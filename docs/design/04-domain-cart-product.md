# 04. Cart / Product ドメイン設計

要件定義書 v0.6 §8-1〜8-3 および [`01-database.md`](01-database.md) / [`02-api-spec.md`](02-api-spec.md) §2-§3 を実装レベルで詰めた設計書。Phase 1 スコープ。

**ゴール**: Cart / Product ドメインの実装契約を TDD で進められる粒度に確定し、1α 実装レディ状態にする。要件書・設計書 01 / 02 / 06 内に残る「設計書 04 で確定」マーカーをすべて埋める。

本書は **1α(簡易版)と 1β(応用拡張)の両方** を扱う。構成は [`05-domain-order.md`](05-domain-order.md) と揃える。

---

## 1. 方針

| 項目 | 決定 |
|------|------|
| Cart の扱い | **Aggregate ルート**。`Cart` が `List<CartItem>` を `@OneToMany` で所有 |
| Product の扱い | 1α は **単一エンティティ**(`stock` 内包)。1β で `Product` + `ProductStock` の 1:1 に分離 |
| 重複マージ規則 | 1α は **Service 層で `(cart_id, product_id)` を検索して `quantity` を加算**(DB UNIQUE 制約なし)。1β で DB に UNIQUE を張り、Cart Aggregate メソッドに責務移動 |
| 空カート例外 | **1α から `EmptyCartException` を先行導入**(1β で追加しない) |
| 在庫チェックの位置 | `Product#decreaseStock`(1α)/ `ProductStock#decrease`(1β)に集約 |
| カート行の所有権 | `X-Cart-Id` と `CartItem.cart_id` の一致検証は **CartService 層で実施** |
| カート upsert 並行制御 | Phase 1 は「最後勝ち」方針。`DataIntegrityViolationException` は実装では握らない(1 名開発で発生しない) |

### 1.1 05(Order)との責務分担

| ドメイン | 設計書 | 主な責務 |
|---------|-------|---------|
| Cart / CartItem | **04(本書)** | カート行の追加・変更・削除・SAVED_FOR_LATER 切替(1β) |
| Product / ProductStock | **04(本書)** | 商品マスタ、在庫保持・減算 |
| Order / OrderItem / OrderStatusHistory | 05 | 注文確定、税額スナップショット、状態履歴 |

`OrderService` は Cart / Product から値を読み、Order を生成する(05 §5)。本書は Cart 側からの呼び出しルート(`addItem` / `updateQuantity` / `remove`)を扱う。

---

## 2. Product ドメイン

### 2.1 Phase 1α の `Product`

`stock` をフィールドに内包する単一エンティティ。

```java
@Entity
@Table(name = "products")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Product {

    @Id
    private UUID id;

    @Column(nullable = false) private String name;
    @Column private String description;
    @Column(nullable = false) private BigDecimal price;

    @Column(nullable = false) private int stock;          // 1α のみ

    @Column private String imageUrl;
    @Column(nullable = false) private OffsetDateTime createdAt;
    @Column(nullable = false) private OffsetDateTime updatedAt;

    /** 1α の商品登録ファクトリ */
    public static Product register(
        String name, String description, BigDecimal price,
        int stock, String imageUrl, OffsetDateTime now
    ) {
        Product p = new Product();
        p.id = UUID.randomUUID();
        p.name = name;
        p.description = description;
        p.price = price;
        p.stock = stock;
        p.imageUrl = imageUrl;
        p.createdAt = now;
        p.updatedAt = now;
        return p;
    }

    /** 在庫減算。不足なら OutOfStockException */
    public void decreaseStock(int amount) {
        if (stock < amount) {
            throw new OutOfStockException(name, stock, amount);
        }
        stock -= amount;
    }

    /** 在庫チェックのみ(減算しない) */
    public boolean hasStock(int amount) {
        return stock >= amount;
    }
}
```

**責務**:
- 商品情報の保持
- 在庫の減算・確認(1β で `ProductStock` に移管)

### 2.2 Phase 1β の `Product` + `ProductStock`

`stock` を別エンティティに分離。Product 登録時は **同一トランザクションで両方を作成**するファクトリを設ける。

```java
// Product(1β)
@Entity
@Table(name = "products")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Product {

    @Id private UUID id;
    @Column(nullable = false) private String name;
    @Column private String description;
    @Column(nullable = false) private BigDecimal price;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false) private TaxCategory taxCategory;  // 1β 追加

    @Column private String imageUrl;
    @Column(nullable = false) private OffsetDateTime createdAt;
    @Column(nullable = false) private OffsetDateTime updatedAt;

    @OneToOne(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true,
              fetch = FetchType.LAZY)
    private ProductStock stock;

    /** 1β の商品登録ファクトリ。Product + ProductStock を同時生成 */
    public static Product register(
        String name, String description, BigDecimal price,
        TaxCategory taxCategory, int initialStock, String imageUrl,
        OffsetDateTime now
    ) {
        Product p = new Product();
        p.id = UUID.randomUUID();
        p.name = name;
        p.description = description;
        p.price = price;
        p.taxCategory = taxCategory;
        p.imageUrl = imageUrl;
        p.createdAt = now;
        p.updatedAt = now;
        p.stock = new ProductStock(p, initialStock, now);  // 1:1 を同時生成
        return p;
    }
}

// ProductStock(1β 新設)
@Entity
@Table(name = "product_stocks")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class ProductStock {

    @Id
    @Column(name = "product_id")
    private UUID productId;

    @MapsId
    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "product_id")
    private Product product;

    @Column(nullable = false) private int quantity;
    @Column(nullable = false) private OffsetDateTime updatedAt;

    ProductStock(Product product, int quantity, OffsetDateTime now) {  // package-private
        this.product = product;
        this.quantity = quantity;
        this.updatedAt = now;
    }

    public void decrease(int amount) {
        if (quantity < amount) {
            throw new OutOfStockException(product.getName(), quantity, amount);
        }
        quantity -= amount;
    }

    public boolean has(int amount) {
        return quantity >= amount;
    }
}
```

### 2.3 N+1 対策

`GET /api/products`(一覧)は `ProductResponse.stock` を返すため、1β では `ProductStock` を JOIN で取得する必要がある。デフォルトの LAZY では N+1 になる。

**方針**: `ProductRepository` に `@EntityGraph(attributePaths = "stock")` を付けた専用メソッドを用意する。

```java
public interface ProductRepository extends JpaRepository<Product, UUID> {

    @EntityGraph(attributePaths = "stock")
    Page<Product> findAllWithStock(Pageable pageable);

    @EntityGraph(attributePaths = "stock")
    Optional<Product> findByIdWithStock(UUID id);
}
```

1α では `stock` がフィールドに含まれるため不要。1β への昇格時に追加する。

---

## 3. Cart ドメイン

### 3.1 Cart Aggregate の所有範囲

`Cart` は **Aggregate ルート**として `List<CartItem>` を所有する。`CartItem` の追加・削除・変更は `Cart` 経由で行う(直接 `CartItemRepository#save` を呼ばない)。

**理由**:
- JPA の `@OneToMany` + `cascade` を学ぶ教材価値
- 1β でのマージ規則・SAVED_FOR_LATER 遷移を Aggregate メソッド内に閉じ込められる
- `GET /api/cart` で `cart.getItems()` を使える

**注意**: Order Aggregate([`05-domain-order.md`](05-domain-order.md))とは異なり、状態履歴のような不変条件はないため、Aggregate メソッド方式は 1β の重複マージ・SFL 遷移でのみ使う。1α は薄い Aggregate で構わない。

### 3.2 Phase 1α の `Cart` / `CartItem`

```java
// Cart(1α)
@Entity
@Table(name = "carts")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Cart {

    @Id
    private UUID id;

    @Column(nullable = false) private OffsetDateTime createdAt;
    @Column(nullable = false) private OffsetDateTime updatedAt;

    @OneToMany(mappedBy = "cart", cascade = CascadeType.ALL, orphanRemoval = true,
               fetch = FetchType.LAZY)
    private final List<CartItem> items = new ArrayList<>();

    /** フロント生成 UUID を受け取って新規 Cart を作る */
    public static Cart createWithId(UUID cartId, OffsetDateTime now) {
        Cart c = new Cart();
        c.id = cartId;
        c.createdAt = now;
        c.updatedAt = now;
        return c;
    }

    /**
     * 1α の商品追加。同一 productId が既存なら quantity を加算。
     * 加算後に在庫を超える / @Max(99) を超える場合は例外。
     */
    public CartItem addItem(Product product, int quantity, OffsetDateTime now) {
        Optional<CartItem> existing = items.stream()
            .filter(i -> i.getProductId().equals(product.getId()))
            .findFirst();

        if (existing.isPresent()) {
            existing.get().addQuantity(quantity, product, now);  // マージ
            return existing.get();
        }

        if (!product.hasStock(quantity)) {
            throw new OutOfStockException(product.getName(), product.getStock(), quantity);
        }
        CartItem item = CartItem.create(this, product.getId(), quantity, now);
        items.add(item);
        this.updatedAt = now;
        return item;
    }

    /** 1α の数量変更 */
    public void updateQuantity(UUID cartItemId, int newQuantity, Product product, OffsetDateTime now) {
        CartItem item = findItem(cartItemId);
        if (!product.hasStock(newQuantity)) {
            throw new OutOfStockException(product.getName(), product.getStock(), newQuantity);
        }
        item.changeQuantity(newQuantity, now);
        this.updatedAt = now;
    }

    /** 1α のカート行削除 */
    public void removeItem(UUID cartItemId, OffsetDateTime now) {
        CartItem item = findItem(cartItemId);
        items.remove(item);
        this.updatedAt = now;
    }

    private CartItem findItem(UUID cartItemId) {
        return items.stream()
            .filter(i -> i.getId().equals(cartItemId))
            .findFirst()
            .orElseThrow(() -> new CartItemNotFoundException(cartItemId));
    }

    public boolean isEmpty() {
        return items.isEmpty();
    }
}

// CartItem(1α)
@Entity
@Table(name = "cart_items")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class CartItem {

    @Id
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "cart_id", nullable = false)
    private Cart cart;

    @Column(name = "product_id", nullable = false)
    private UUID productId;

    @Column(nullable = false) private int quantity;
    @Column(nullable = false) private OffsetDateTime createdAt;
    @Column(nullable = false) private OffsetDateTime updatedAt;

    static CartItem create(Cart cart, UUID productId, int quantity, OffsetDateTime now) {
        CartItem item = new CartItem();
        item.id = UUID.randomUUID();
        item.cart = cart;
        item.productId = productId;
        item.quantity = quantity;
        item.createdAt = now;
        item.updatedAt = now;
        return item;
    }

    /** 1α の加算マージ。上限・在庫を超えないよう呼び出し側が Product を渡す */
    void addQuantity(int delta, Product product, OffsetDateTime now) {
        int merged = this.quantity + delta;
        if (merged > 99) {
            throw new CartItemQuantityExceededException(product.getName(), merged, 99);
        }
        if (!product.hasStock(merged)) {
            throw new OutOfStockException(product.getName(), product.getStock(), merged);
        }
        this.quantity = merged;
        this.updatedAt = now;
    }

    void changeQuantity(int newQuantity, OffsetDateTime now) {
        this.quantity = newQuantity;
        this.updatedAt = now;
    }
}
```

> **注**: `CartItemQuantityExceededException` は [`06-exception.md`](06-exception.md) §2.2 で 1β 追加としているが、1α のマージ動作でも同じ例外を投げるため、**1α の時点で先行導入する**([`06-exception.md`](06-exception.md) §2.1 に移動する扱い)。本書の採用判断が 06 に反映される。

### 3.3 Phase 1β の Cart 拡張

1β では以下を追加する:

1. `CartItem` に `status: CartItemStatus` を追加(`ACTIVE` / `SAVED_FOR_LATER`)
2. DB に `UNIQUE (cart_id, product_id, status)` を張る([`01-database.md`](01-database.md) §8.5)
3. `Cart#addItem` のマージキーを `(productId, status=ACTIVE)` に拡張
4. `Cart#moveToSavedForLater(cartItemId)` / `Cart#activateFromSavedForLater(cartItemId, Product)` を追加
5. `activateFromSavedForLater` では在庫チェックを行い、不足時は `OutOfStockException`

```java
// 1β で追加・差し替えるメソッド(抜粋)
public class Cart {

    public CartItem addItem(Product product, int quantity, OffsetDateTime now) {
        // 既存行のマッチキーを (productId, status=ACTIVE) に拡張
        Optional<CartItem> existing = items.stream()
            .filter(i -> i.getProductId().equals(product.getId())
                      && i.getStatus() == CartItemStatus.ACTIVE)
            .findFirst();
        // ...以降 1α と同じ
    }

    /** 1β: ACTIVE → SAVED_FOR_LATER */
    public void moveToSavedForLater(UUID cartItemId, OffsetDateTime now) {
        CartItem item = findItem(cartItemId);
        // 既存の SAVED_FOR_LATER 行とマージ
        Optional<CartItem> existingSFL = items.stream()
            .filter(i -> i.getProductId().equals(item.getProductId())
                      && i.getStatus() == CartItemStatus.SAVED_FOR_LATER
                      && !i.getId().equals(item.getId()))
            .findFirst();

        if (existingSFL.isPresent()) {
            existingSFL.get().mergeQuantityOnly(item.getQuantity(), now);
            items.remove(item);  // orphanRemoval で DELETE
        } else {
            item.changeStatus(CartItemStatus.SAVED_FOR_LATER, now);
        }
        this.updatedAt = now;
    }

    /** 1β: SAVED_FOR_LATER → ACTIVE(在庫チェック込み) */
    public void activateFromSavedForLater(UUID cartItemId, Product product, OffsetDateTime now) {
        CartItem item = findItem(cartItemId);
        if (!product.getStock().has(item.getQuantity())) {
            throw new OutOfStockException(product.getName(),
                product.getStock().getQuantity(), item.getQuantity());
        }
        // 既存の ACTIVE 行とマージ
        Optional<CartItem> existingActive = items.stream()
            .filter(i -> i.getProductId().equals(item.getProductId())
                      && i.getStatus() == CartItemStatus.ACTIVE
                      && !i.getId().equals(item.getId()))
            .findFirst();

        if (existingActive.isPresent()) {
            existingActive.get().addQuantity(item.getQuantity(), product, now);
            items.remove(item);
        } else {
            item.changeStatus(CartItemStatus.ACTIVE, now);
        }
        this.updatedAt = now;
    }

    /** 1β: 注文確定用。ACTIVE の行のみを返す */
    public List<CartItem> activeItems() {
        return items.stream()
            .filter(i -> i.getStatus() == CartItemStatus.ACTIVE)
            .toList();
    }
}
```

---

## 4. CartService(ユースケース層)

### 4.1 Cart upsert

`X-Cart-Id` を受け取って `Cart` を取得、なければ新規作成する。1α / 1β 共通。

```java
@Service
@RequiredArgsConstructor
public class CartService {

    private final CartRepository cartRepository;
    private final ProductRepository productRepository;
    private final Clock clock;

    @Transactional
    public Cart findOrCreate(UUID cartId) {
        return cartRepository.findById(cartId)
            .orElseGet(() -> cartRepository.save(
                Cart.createWithId(cartId, OffsetDateTime.now(clock))));
    }

    // addItem / updateQuantity / removeItem / moveToSavedForLater 等のメソッド
}
```

### 4.2 カート行の所有権チェック

`PATCH /api/cart/items/{id}` / `DELETE /api/cart/items/{id}` は、パス変数の `cartItemId` が `X-Cart-Id` のカートに属するか検証する。Aggregate 内で `findItem` した際に見つからなければ `CartItemNotFoundException`(404)、別カートに属していれば `CartItemForbiddenException`(403)。

```java
@Transactional
public CartResponse updateQuantity(UUID cartId, UUID cartItemId, int newQuantity) {
    Cart cart = cartRepository.findById(cartId)
        .orElseThrow(() -> new CartItemNotFoundException(cartItemId));

    // cart.findItem の前に所有権を検証したい場合は、
    // CartItem を別途 findById して cart_id 比較する
    CartItem target = cartItemRepository.findById(cartItemId)
        .orElseThrow(() -> new CartItemNotFoundException(cartItemId));
    if (!target.getCart().getId().equals(cartId)) {
        throw new CartItemForbiddenException(cartId, target.getCart().getId());
    }

    Product product = productRepository.findById(target.getProductId())
        .orElseThrow(() -> new ProductNotFoundException(target.getProductId()));

    cart.updateQuantity(cartItemId, newQuantity, product, OffsetDateTime.now(clock));
    return cartMapper.toResponse(cart, productRepository);  // 1α 版 or 1β 版
}
```

> **補足**: Aggregate の原則論で言えば「Cart 経由でしか CartItem に触らない」が、PATCH / DELETE のパス変数から辿るため、所有権検証のためだけに `CartItemRepository` を使うのは許容する(学習プロジェクトの実用性優先)。

### 4.3 `CartResponse` の組み立て

- **1α**: フラット構造 `items[]` + `totalPrice`(税なし)
- **1β**: `active` / `savedForLater` のグループ化 + 税情報

`CartMapper#toResponse` を 1α 版と 1β 版で分けて定義する(05 §6 の `OrderMapper` と同じ方式)。

---

## 5. Controller / DTO

### 5.1 `X-Cart-Id` の UUID 検証

`@RequestHeader("X-Cart-Id") UUID cartId` で受けると、Spring が自動で UUID パースする。不正値は `MethodArgumentTypeMismatchException` → advice で 400([`06-exception.md`](06-exception.md) §4.1)。`@UUID` 等の追加アノテーションは不要。

```java
@RestController
@RequestMapping("/api/cart")
@RequiredArgsConstructor
public class CartController {

    private final CartService cartService;

    @GetMapping
    public CartResponse getCart(@RequestHeader("X-Cart-Id") UUID cartId) {
        return cartService.getCart(cartId);
    }

    @PostMapping("/items")
    public CartResponse addItem(
        @RequestHeader("X-Cart-Id") UUID cartId,
        @RequestBody @Valid CartItemAddRequest req
    ) {
        return cartService.addItem(cartId, req.productId(), req.quantity());
    }

    @PatchMapping("/items/{id}")
    public CartResponse updateItem(
        @RequestHeader("X-Cart-Id") UUID cartId,
        @PathVariable UUID id,
        @RequestBody @Valid CartItemUpdateRequest req
    ) {
        return cartService.updateItem(cartId, id, req);
    }

    @DeleteMapping("/items/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void removeItem(
        @RequestHeader("X-Cart-Id") UUID cartId,
        @PathVariable UUID id
    ) {
        cartService.removeItem(cartId, id);
    }
}
```

### 5.2 Request DTO(1α)

```java
public record CartItemAddRequest(
    @NotNull(message = "商品 ID は必須です")
    UUID productId,

    @Min(value = 1,  message = "数量は 1 以上で入力してください")
    @Max(value = 99, message = "数量は 99 以下で入力してください")
    int quantity
) {}

// 1α
public record CartItemUpdateRequest(
    @Min(value = 1,  message = "数量は 1 以上で入力してください")
    @Max(value = 99, message = "数量は 99 以下で入力してください")
    Integer quantity   // null = 変更なし(1α では quantity 必須化しても良い)
) {}
```

### 5.3 Request DTO(1β で `status` 追加)

```java
public record CartItemUpdateRequest(
    @Min(value = 1,  message = "数量は 1 以上で入力してください")
    @Max(value = 99, message = "数量は 99 以下で入力してください")
    Integer quantity,

    CartItemStatus status  // null = 変更なし
) {
    /** 両方省略は不可 */
    @AssertTrue(message = "数量またはステータスのいずれかを指定してください")
    public boolean isAtLeastOneFieldPresent() {
        return quantity != null || status != null;
    }
}
```

### 5.4 Product 登録 Request

```java
// 1α
public record ProductCreateRequest(
    @NotBlank(message = "商品名は必須です") String name,
    String description,
    @NotNull(message = "価格は必須です")
    @DecimalMin(value = "0.01", message = "0.01 以上で入力してください")
    @DecimalMax(value = "9999999999.99", message = "価格が大きすぎます")
    BigDecimal price,
    @Min(value = 0, message = "在庫数は 0 以上で入力してください") int stock,
    String imageUrl
) {}

// 1β で taxCategory を追加
public record ProductCreateRequest(
    @NotBlank(message = "商品名は必須です") String name,
    String description,
    @NotNull @DecimalMin("0.01") @DecimalMax("9999999999.99") BigDecimal price,
    @NotNull(message = "税区分は必須です") TaxCategory taxCategory,
    @Min(0) int stock,
    String imageUrl
) {}
```

---

## 6. 例外との対応

[`06-exception.md`](06-exception.md) との対応関係を本書時点で確定する。**`CartItemQuantityExceededException` は 06 で 1β に分類されていたが、本書の方針(1α からマージあり)により 1α に昇格する**。06 を 1α 5 例外に更新する必要がある。

| 例外 | フェーズ(確定) | 発生箇所(本書) |
|------|----------------|-----------------|
| `ProductNotFoundException` | 1α | §4 Service 層 |
| `CartItemNotFoundException` | 1α | §3.2 `Cart#findItem` |
| `CartItemForbiddenException` | 1α | §4.2 所有権検証 |
| `OutOfStockException` | 1α | §2 `Product#decreaseStock` / §3 `Cart#addItem` |
| **`CartItemQuantityExceededException`** | **1α に昇格** | §3.2 `CartItem#addQuantity` |
| **`EmptyCartException`** | **1α に昇格** | §7 注文確定時(`OrderService` から) |
| `IllegalStateTransitionException` | 1β | (05 §2.4 で実使用は Phase 2) |

### 6.1 [`06-exception.md`](06-exception.md) への反映点(本書決定に基づく修正指示)

- §2.1「Phase 1α のドメイン例外」を **5 例外 + EmptyCartException(合計 6)** に拡張
- §2.2「Phase 1β で追加するドメイン例外」を **`IllegalStateTransitionException` のみの 1 例外** に縮小
- §8.1 / §8.2 チェックリストを同様に調整

---

## 7. 空カート例外の扱い(1α で確定)

要件書・05 / 06 で保留していた「1α の空カート例外」について本書で確定する。

### 選択肢

| 案 | 評価 |
|----|------|
| A. `EmptyCartException` を 1α から先行導入 | ○ 意味が明確、1β 移行で変更不要 |
| B. `OutOfStockException` で代用 | ✗ 意味のミスマッチ(在庫の問題ではない) |
| C. 空カートで 400 を返す | ✗ HTTP セマンティクスに反する(サーバー状態の問題なので 409 が適切) |

### 決定

**案 A を採用**。1α の注文確定時、`Cart#isEmpty()` が `true` なら `OrderService` が `EmptyCartException` を投げる。06 の例外カタログは本決定に従い修正する。

---

## 8. カート upsert の並行制御

Phase 1 は「最後勝ち」方針([`01-database.md`](01-database.md) §1)。`CartService#findOrCreate` で `findById` → `save(new Cart(...))` の間に並行リクエストが入ると PK 競合する可能性があるが、Phase 1 の単一開発者環境では実質発生しない。

**1α / 1β 共通方針**:
- `DataIntegrityViolationException` は握らない
- もし発生したら 500 を返す(advice でキャッチ済み)
- Phase 2 で並行性が問題になったら `ON CONFLICT DO NOTHING` 相当(JPA 的には `findById` を再実行)を導入

---

## 9. テスト設計(TDD)

### 9.1 1α のレイヤー別テスト観点

| 層 | テスト種別 | 赤 → 緑で書く観点 |
|----|-----------|-------------------|
| `Product#register` | 純 Java Unit Test | 必須フィールドが入る / `id` が生成される |
| `Product#decreaseStock` | 純 Java Unit Test | 十分在庫で減算成功 / 不足で `OutOfStockException` / 残量 0 境界 |
| `Cart#createWithId` | 純 Java Unit Test | フロント生成 UUID が保持される |
| `Cart#addItem`(新規) | 純 Java Unit Test | 在庫内で成功 / 在庫不足で `OutOfStockException` |
| `Cart#addItem`(マージ) | 純 Java Unit Test | 同一 `productId` で既存行に加算される / 加算後 99 超で `CartItemQuantityExceededException` / 加算後在庫超で `OutOfStockException` |
| `Cart#updateQuantity` | 純 Java Unit Test | 在庫内で成功 / 在庫不足で 409 / 未存在 `cartItemId` で `CartItemNotFoundException` |
| `Cart#removeItem` | 純 Java Unit Test | 行が削除される / 未存在で `CartItemNotFoundException` |
| `Cart#isEmpty` | 純 Java Unit Test | 空 / 非空 |
| `CartService#addItem` | Mockito Unit Test | Cart upsert / 商品取得 / 重複マージの流れ / 例外ケース |
| `CartService#updateItem` | Mockito Unit Test | 所有権検証(403)/ 正常更新 |
| `ProductService#register` | Mockito Unit Test | Product が保存される(1β では ProductStock も同時) |
| `CartController` | `@WebMvcTest` | 各エンドポイントの HTTP 応答 / 例外の HTTP マッピング |
| `ProductController` | `@WebMvcTest` | 一覧 / 詳細 / 登録 |

### 9.2 1β で追加されるテスト観点

| 層 | テスト種別 | 観点 |
|----|-----------|------|
| `Product#register`(1β) | 純 Java Unit Test | `ProductStock` が同時生成される / `taxCategory` が保持される |
| `ProductStock#decrease` | 純 Java Unit Test | 05 §7.1 と同じ |
| `Cart#addItem`(マージ with status) | 純 Java Unit Test | `(productId, ACTIVE)` キーでのマージ / `SAVED_FOR_LATER` 行は別扱い |
| `Cart#moveToSavedForLater` | 純 Java Unit Test | 新規 SFL 行が作られる / 既存 SFL とマージされる |
| `Cart#activateFromSavedForLater` | 純 Java Unit Test | 在庫内で成功 / 在庫不足で `OutOfStockException` / 既存 ACTIVE とマージ |
| `Cart#activeItems` | 純 Java Unit Test | `ACTIVE` のみ返る |
| 結合(任意) | `@DataJpaTest` | `UNIQUE (cart_id, product_id, status)` が効く |

### 9.3 Cart Aggregate テストの雛形(1α)

```java
class CartTest {

    private static final OffsetDateTime NOW =
        OffsetDateTime.parse("2026-04-16T10:00:00+09:00");

    @Test
    void addItem_mergesQuantityWhenSameProduct() {
        Cart cart = Cart.createWithId(UUID.randomUUID(), NOW);
        Product apple = sampleProduct("リンゴ", "100", 50);

        cart.addItem(apple, 2, NOW);
        cart.addItem(apple, 3, NOW);

        assertThat(cart.getItems()).hasSize(1);
        assertThat(cart.getItems().get(0).getQuantity()).isEqualTo(5);
    }

    @Test
    void addItem_throwsWhenMergedQuantityExceeds99() {
        Cart cart = Cart.createWithId(UUID.randomUUID(), NOW);
        Product apple = sampleProduct("リンゴ", "100", 200);
        cart.addItem(apple, 50, NOW);

        assertThatThrownBy(() -> cart.addItem(apple, 60, NOW))
            .isInstanceOf(CartItemQuantityExceededException.class);
    }

    @Test
    void addItem_throwsWhenStockInsufficient() {
        Cart cart = Cart.createWithId(UUID.randomUUID(), NOW);
        Product apple = sampleProduct("リンゴ", "100", 3);

        assertThatThrownBy(() -> cart.addItem(apple, 5, NOW))
            .isInstanceOf(OutOfStockException.class);
    }
}
```

---

## 10. チェックリスト(着手前)

### 10.1 Phase 1α の実装順序(TDD)

上から順に **赤 → 緑 → リファクタ** で実装する。

- [ ] `domain/exception/` に 1α 6 例外を作成([`06-exception.md`](06-exception.md) §2.1 / 本書 §6 で確定した 6 例外)
- [ ] `ProductTest` → `Product#register` / `decreaseStock` / `hasStock` 実装
- [ ] `ProductServiceTest` → `ProductService#register` / `findAll` / `findById` 実装
- [ ] `ProductControllerTest`(`@WebMvcTest`)→ `ProductController` 実装
- [ ] `CartTest` → `Cart#createWithId` / `addItem`(新規・マージ)/ `updateQuantity` / `removeItem` / `isEmpty` 実装
- [ ] `CartItemTest`(任意)→ `CartItem#addQuantity` / `changeQuantity` の境界テスト
- [ ] `CartServiceTest` → `CartService` の各ユースケース(upsert・追加・変更・削除・所有権検証)
- [ ] `CartControllerTest`(`@WebMvcTest`)→ `CartController` 実装
- [ ] `OrderTest` → `Order.create` 実装([`05-domain-order.md`](05-domain-order.md) §8.1)
- [ ] `OrderServiceTest` → `OrderService#createOrder`(1α 版)実装
- [ ] `OrderControllerTest` → `OrderController` 実装
- [ ] 結合確認(任意): `@DataJpaTest` で Cart `@OneToMany` cascade が効くこと

### 10.2 Phase 1β で追加する作業

- [ ] `IllegalStateTransitionException` を追加(1β で初登場)
- [ ] `ProductStock` エンティティを作成(`V2__split_product_stock.sql`)
- [ ] `Product` を 1:1 所有に改修(`ProductStock` を `@OneToOne`)
- [ ] `ProductRepository` に `@EntityGraph` 版メソッドを追加
- [ ] `Product#register` を 1β 版(`taxCategory` / `ProductStock` 同時生成)に差し替え
- [ ] `CartItem` に `status` 追加(`V5__cart_item_status.sql`)
- [ ] `Cart#addItem` のマージキーを `(productId, status=ACTIVE)` に拡張
- [ ] `Cart#moveToSavedForLater` / `activateFromSavedForLater` を実装
- [ ] `Cart#activeItems` を追加(`OrderService` から利用)
- [ ] `CartItemUpdateRequest` に `status` + `@AssertTrue` を追加
- [ ] `CartMapper` を 1β 版(グループ化・税)に差し替え
- [ ] 05 側の `Order.confirm` / 税・採番・履歴への移行(05 §8.2)と同期して実施
