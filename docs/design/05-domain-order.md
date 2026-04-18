# 05. Order ドメイン設計

要件定義書 v0.6 §8-4〜8-6 および [`01-database.md`](01-database.md) を実装レベルで詰めた設計書。Phase 1 スコープ。

**ゴール**: 注文確定ユースケースを TDD で実装するための、Aggregate の契約とテスト観点を確定する。

本書は **1α(簡易版)と 1β(本格版)の両方** を扱う。まず §0 で 1α の簡易実装を示し、§1 以降で 1β の本格版(Aggregate + Factory + 状態履歴 + 採番)を詳述する。1α → 1β はリファクタで進化させる前提。

---

## 0. Phase 1α の簡易 Order

### 0.1 方針(1α)

1α は学習目的 3 本柱(Spring Boot / JPA / レイヤード)を最短で通すため、Order ドメインを **最小限の JPA エンティティ** として実装する。Aggregate メソッド・ファクトリ・状態履歴・採番は 1β で導入するため、1α では採用しない。

| 項目 | 1α の決定 |
|------|----------|
| 状態変更 | なし(常に `CONFIRMED` で作成) |
| 状態履歴 | なし(`OrderStatusHistory` テーブル自体が 1β で新設) |
| 税額計算 | なし(`totalPrice = Σ(unitPrice × quantity)`) |
| 注文番号 | なし(`id: UUID` のみ) |
| Clock の DI | 任意(1α では `OffsetDateTime.now()` でも可)。1β で必須化 |
| トランザクション境界 | `OrderService#createOrder` メソッド全体(`@Transactional`) |

### 0.2 クラス構造(1α)

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Order {

    @Id
    private UUID id;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false)
    private BigDecimal totalPrice;

    @Column(nullable = false)
    private OffsetDateTime createdAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private final List<OrderItem> items = new ArrayList<>();

    /** 1α の注文生成。空なら EmptyCartException(1α から先行導入 — [04] §7 で確定) */
    public static Order create(List<OrderItem> items, OffsetDateTime now) {
        if (items.isEmpty()) {
            throw new EmptyCartException();
        }
        Order order = new Order();
        order.id = UUID.randomUUID();
        order.status = OrderStatus.CONFIRMED;
        order.createdAt = now;
        items.forEach(order::attachItem);
        order.totalPrice = items.stream()
            .map(OrderItem::getLineTotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        return order;
    }

    private void attachItem(OrderItem item) {
        items.add(item);
        item.assignTo(this);
    }
}
```

### 0.3 `OrderItem`(1α)

```java
@Entity
@Table(name = "order_items")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderItem {

    @Id private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Column(nullable = false) private UUID productId;
    @Column(nullable = false) private String productName;
    @Column(nullable = false) private BigDecimal unitPrice;
    @Column(nullable = false) private int quantity;

    public static OrderItem snapshot(CartItem cartItem, Product product) {
        OrderItem item = new OrderItem();
        item.id = UUID.randomUUID();
        item.productId = product.getId();
        item.productName = product.getName();
        item.unitPrice = product.getPrice();
        item.quantity = cartItem.getQuantity();
        return item;
    }

    public BigDecimal getLineTotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }

    void assignTo(Order order) { this.order = order; }
}
```

### 0.4 注文確定ユースケース(1α)

```
1. Cart の全 CartItem を取得
   └─ 空なら 409(1α 用の例外または OutOfStockException で代用)
2. 各 CartItem について:
   ├─ Product を取得(なければ ProductNotFoundException)
   ├─ stock チェック(不足なら OutOfStockException)
   ├─ Product.stock を減算
   └─ OrderItem.snapshot(cartItem, product) を生成
3. Order.create(items, now) で Entity 生成
4. orderRepository.save(order)
   └─ cascade で OrderItem も INSERT
5. cartItemRepository.deleteByCartId(cartId)
6. OrderMapper.toResponse(order) を返却
```

全体を `@Transactional` で囲む。

### 0.5 1α → 1β の進化ポイント

1β 移行時は以下をリファクタで追加する(§1 以降で詳述):
1. `OrderStatusHistory` を `Order` に `@OneToMany` で所有させ、状態遷移時に 1 件追加する制約を Aggregate メソッド `confirm` に移す
2. `OrderItem` に `taxCategory` / `taxRate` / `taxAmount` を追加。`snapshot` が `taxRate` を受け取り `taxAmount` を `FLOOR` で算出
3. `Order` に `subtotal` / `taxAmount` / `totalPrice` を保持(`recalculateTotals`)
4. `Order` に `orderNumber` を追加し、`OrderNumberGenerator` で採番
5. `Clock` を `OrderService` / `OrderNumberGenerator` に DI

---

## 1. 方針(1β)

| 項目 | 決定 |
|------|------|
| 状態変更の強制 | **Aggregate メソッド**(`Order.confirm(...)` 等)内で自身の `status` と `statusHistory` を同時に更新 |
| Order と履歴の関係 | `Order` が `List<OrderStatusHistory>` を所有し、JPA cascade で同時保存 |
| トランザクション境界 | `OrderService#createOrder` メソッド全体(`@Transactional`) |
| 税額計算の位置 | **`OrderItem.snapshot(...)` ファクトリ内**(確定時点で一度だけ計算しスナップショット) |
| 在庫減算の位置 | `ProductStock#decrease(int, String)` メソッド |
| 注文番号採番 | `OrderNumberGenerator` Bean。衝突時 **1 回リトライ** |
| Clock | DI 済み([`03-project-setup.md`](03-project-setup.md) §7.2)。`LocalDate.now(clock)` で JST 日付取得 |

### 1.1 なぜ Aggregate メソッド方式か

要件書 §8-5 運用ルール:「`Order.status` を変更する Service は、同一トランザクションで履歴行を 1 件 INSERT する。これを破ると真実源が壊れる」。

Service 層で「status 変更の次の行に `history.save()` を書く」運用では忘れを防げない。

**採用**: Aggregate が状態変更メソッドを持ち、メソッド内で自身の `status` と `history` の両方を更新する。Service はこのメソッドを呼ぶだけで「忘れる余地」がなくなる。

**TDD との相性**: Spring Context を起動せず `new Order(...).confirm(...)` の Unit Test で「`history` が 1 件追加された」「`status` が `CONFIRMED` になった」を赤 → 緑で書ける。

### 1.2 なぜ税額計算を `OrderItem` ファクトリに置くか

- 要件書 §8 スナップショット方針: 確定時点の単価・税率・税額を `order_items` に保持
- 1 箇所で「入力 → スナップショット」を閉じ込めると、Service が何度呼んでも結果が同じで副作用がない → TDD で書きやすい

---

## 2. Order Aggregate 設計(1β)

### 2.1 責務

- 注文ヘッダー情報(`id` / `orderNumber` / `status` / 金額合計 / `createdAt`)の保持
- 明細(`items`)と状態履歴(`statusHistory`)の所有
- 状態遷移メソッドの提供(Phase 1β は `confirm` のみ)
- 金額合計の再計算(`items` と整合)

### 2.2 クラス構造

```java
@Entity
@Table(name = "orders")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)  // JPA 用
public class Order {

    @Id
    private UUID id;

    @Column(nullable = false, unique = true)
    private String orderNumber;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private OrderStatus status;

    @Column(nullable = false) private BigDecimal subtotal;
    @Column(nullable = false) private BigDecimal taxAmount;
    @Column(nullable = false) private BigDecimal totalPrice;

    @Column(nullable = false)
    private OffsetDateTime createdAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private final List<OrderItem> items = new ArrayList<>();

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private final List<OrderStatusHistory> statusHistory = new ArrayList<>();

    /** Phase 1β で唯一の状態遷移。カート → 注文への確定 */
    public static Order confirm(
        String orderNumber,
        List<OrderItem> items,
        OffsetDateTime now
    ) {
        if (items.isEmpty()) {
            throw new EmptyCartException();
        }
        Order order = new Order();
        order.id = UUID.randomUUID();
        order.orderNumber = orderNumber;
        order.status = OrderStatus.CONFIRMED;
        order.createdAt = now;
        items.forEach(order::attachItem);
        order.recalculateTotals();
        order.recordTransition(null, OrderStatus.CONFIRMED, null, now);
        return order;
    }

    private void attachItem(OrderItem item) {
        items.add(item);
        item.assignTo(this);  // package-private
    }

    private void recalculateTotals() {
        this.subtotal = items.stream()
            .map(OrderItem::getLineSubtotal)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        this.taxAmount = items.stream()
            .map(OrderItem::getTaxAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
        this.totalPrice = this.subtotal.add(this.taxAmount);
    }

    private void recordTransition(
        OrderStatus from, OrderStatus to, String reason, OffsetDateTime at
    ) {
        statusHistory.add(new OrderStatusHistory(this, from, to, reason, at));
    }
}
```

### 2.3 不変条件

| 条件 | 強制方法 |
|------|---------|
| `items` は空でない | `confirm` ファクトリ内で `EmptyCartException` |
| `status` 変更時は `statusHistory` に 1 件追加される | `confirm` / Phase 2 の `cancel` / `refund` 内で `recordTransition` を呼ぶ |
| `subtotal` / `taxAmount` / `totalPrice` は `items` と整合 | `recalculateTotals` を `confirm` 内で呼ぶ(外部から書き換え不可) |
| `orderNumber` は UNIQUE | DB 制約 + Service 層の採番リトライ(§4.3) |

### 2.4 Phase 2 で追加予定のメソッド(予告・Phase 1 では書かない)

```java
public void cancel(String reason, OffsetDateTime now) {
    if (status != OrderStatus.CONFIRMED) {
        throw new IllegalStateTransitionException(status, OrderStatus.CANCELLED);
    }
    recordTransition(status, OrderStatus.CANCELLED, reason, now);
    status = OrderStatus.CANCELLED;
}

public void refund(OffsetDateTime now) {
    if (status != OrderStatus.CANCELLED) {
        throw new IllegalStateTransitionException(status, OrderStatus.REFUNDED);
    }
    recordTransition(status, OrderStatus.REFUNDED, null, now);
    status = OrderStatus.REFUNDED;
}
```

YAGNI に従い Phase 1 では **書かない**。ただし `IllegalStateTransitionException` はハンドラ・テストを含め 1β で先行作成する([`06-exception.md`](06-exception.md) §2.1 末尾の注記)。

---

## 3. OrderItem の生成(1β)

### 3.1 ファクトリメソッド

税額計算は確定時点で 1 度だけ行い、`OrderItem` に封じ込める。

```java
@Entity
@Table(name = "order_items")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class OrderItem {

    @Id private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id", nullable = false)
    private Order order;

    @Column(nullable = false) private UUID productId;
    @Column(nullable = false) private String productName;
    @Column(nullable = false) private BigDecimal unitPrice;
    @Column(nullable = false) private int quantity;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false) private TaxCategory taxCategory;

    @Column(nullable = false) private BigDecimal taxRate;
    @Column(nullable = false) private BigDecimal taxAmount;

    /** カート行から注文明細を確定する(税率スナップショット・行単位税額計算) */
    public static OrderItem snapshot(
        CartItem cartItem,
        Product product,
        BigDecimal taxRate  // TaxProperties#rateOf(product.taxCategory)
    ) {
        OrderItem item = new OrderItem();
        item.id = UUID.randomUUID();
        item.productId = product.getId();
        item.productName = product.getName();
        item.unitPrice = product.getPrice();
        item.quantity = cartItem.getQuantity();
        item.taxCategory = product.getTaxCategory();
        item.taxRate = taxRate;
        item.taxAmount = calculateLineTax(item.unitPrice, item.quantity, taxRate);
        return item;
    }

    /** 行単位の税額 = floor(unitPrice × quantity × taxRate) */
    private static BigDecimal calculateLineTax(
        BigDecimal unitPrice, int quantity, BigDecimal taxRate
    ) {
        return unitPrice
            .multiply(BigDecimal.valueOf(quantity))
            .multiply(taxRate)
            .setScale(0, RoundingMode.FLOOR);
    }

    public BigDecimal getLineSubtotal() {
        return unitPrice.multiply(BigDecimal.valueOf(quantity));
    }

    void assignTo(Order order) {  // package-private
        this.order = order;
    }
}
```

### 3.2 なぜ `Product` 全体を渡すか

`productName` / `unitPrice` / `taxCategory` をスナップショットするため `Product` 参照が必要。引数で 3 つ受けるより `Product` 1 つの方が呼び出し側が簡潔で、将来スナップショット項目が増えても引数シグネチャが変わらない。

### 3.3 端数処理の具体例(要件書 v0.6 §4 と一致)

| 商品 | 税区分 | 単価 | 数量 | 行小計 | 税率 | 税額計算 | 税額 |
|------|--------|------|------|--------|------|---------|------|
| リンゴ | REDUCED | 100 | 2 | 200 | 0.08 | `floor(200 × 0.08) = floor(16) = 16` | **16** |
| ノート | STANDARD | 300 | 1 | 300 | 0.10 | `floor(300 × 0.10) = floor(30) = 30` | **30** |
| 合計 |   |   |   | **500** |   | 16 + 30 | **46** |

総額: `500 + 46 = 546`

### 3.4 JPA マッピング上の注意

- `Order` 側の `@OneToMany(mappedBy = "order")` の所有側は `OrderItem`。必ず双方向で参照を張る(`OrderItem.order` が null のままだと FK が入らない)
- `Order#attachItem` 内で `item.assignTo(this)` を呼び、親側リストと子側参照を同期させる
- `cascade = CascadeType.ALL + orphanRemoval = true` により `Order` の `save` / `delete` だけで `OrderItem` も連鎖

---

## 4. 注文番号採番(1β)

### 4.1 仕様

- フォーマット: `yyyyMMdd-NNNN`(`NNNN` = 日次 4 桁ゼロ埋め連番)
- 日付境界: **JST 0 時**([`03-project-setup.md`](03-project-setup.md) §7.1 既決)
- 例: `20260416-0001`

### 4.2 実装

```java
@Component
@RequiredArgsConstructor
public class OrderNumberGenerator {

    private final OrderRepository orderRepository;
    private final Clock clock;

    private static final DateTimeFormatter DATE_FMT = DateTimeFormatter.ofPattern("yyyyMMdd");

    public String generate() {
        LocalDate today = LocalDate.now(clock);
        String datePrefix = today.format(DATE_FMT);
        long todayCount = orderRepository.countByOrderNumberStartingWith(datePrefix + "-");
        int nextSeq = (int) todayCount + 1;
        return "%s-%04d".formatted(datePrefix, nextSeq);
    }
}
```

### 4.3 衝突時のリトライ

`orders.order_number` に `UNIQUE` 制約があるため、並行採番で同じ値が生成されると `DataIntegrityViolationException` が飛ぶ。要件書 §8-4-1 に従い **1 回だけリトライ** する。

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    public OrderResponse createOrder(UUID cartId) {
        try {
            return createOrderInternal(cartId);
        } catch (DataIntegrityViolationException e) {
            // 採番衝突の可能性 → 1 回だけ再試行
            return createOrderInternal(cartId);
        }
    }

    @Transactional
    protected OrderResponse createOrderInternal(UUID cartId) { /* §5 */ }
}
```

> **Phase 1 は 1 名開発のため衝突は起きない**が、Phase 2 の負荷試験に備えて仕込んでおく。それ以上の厳密性(`order_sequences` テーブル等)は Phase 2 で評価([`01-database.md`](01-database.md) §10 未決 #8 に準拠)。

### 4.4 採番の排他性について(なぜ count + 1 で許されるか)

- Phase 1 の「最後勝ち」方針([`01-database.md`](01-database.md) §1 / 要件書 §4 在庫競合)と同じ思想: 同時採番の衝突は `UNIQUE` 制約で検知 → リトライで吸収
- `SELECT COUNT` + `INSERT` は論理的にアトミックではないが、UNIQUE 制約が最終防衛線として働く
- Phase 2 で高並行になったら `order_sequences(date, next_seq)` テーブル + `SELECT ... FOR UPDATE` に差し替える

---

## 5. 注文確定ユースケース(`OrderService#createOrderInternal` / 1β)

### 5.1 フロー

```
1. Cart の ACTIVE な CartItem を取得
   └─ 空なら → EmptyCartException

2. 各 CartItem について:
   ├─ Product を取得(なければ ProductNotFoundException)
   ├─ ProductStock を取得
   ├─ stock.decrease(quantity, productName)   ← 在庫不足なら OutOfStockException
   ├─ TaxProperties#rateOf(product.taxCategory) で税率解決
   └─ OrderItem.snapshot(cartItem, product, taxRate) を生成

3. OrderNumberGenerator#generate() で orderNumber を採番

4. Order.confirm(orderNumber, items, now) で Aggregate 生成
   └─ 内部で recalculateTotals + recordTransition が実行される

5. orderRepository.save(order)
   └─ cascade で OrderItem / OrderStatusHistory も INSERT

6. cartItemRepository.deleteByCartIdAndStatus(cartId, ACTIVE)
   └─ SAVED_FOR_LATER は残す

7. OrderMapper#toResponse(order) で OrderResponse を組み立てて返却
```

全体を `@Transactional` で囲む。例外発生時は全ロールバック(在庫も戻る)。

### 5.2 `ProductStock#decrease` の実装

```java
public void decrease(int amount, String productName) {
    if (quantity < amount) {
        throw new OutOfStockException(productName, quantity, amount);
    }
    quantity -= amount;
    // updated_at は @PreUpdate で自動更新(Clock 経由での上書きは Phase 2 検討)
}
```

### 5.3 `now` の取得

`OrderService` は `Clock` を DI し、`OffsetDateTime.now(clock)` でユースケース開始時刻を固定する。これを `Order.confirm(...)` と `OrderNumberGenerator#generate(...)`(内部で `LocalDate.now(clock)`)に渡すことで、同一トランザクション内で同じ「現在」を共有する。

```java
OffsetDateTime now = OffsetDateTime.now(clock);
String orderNumber = orderNumberGenerator.generate();  // 内部で LocalDate.now(clock)
Order order = Order.confirm(orderNumber, items, now);
```

---

## 6. DTO マッピング(`OrderMapper`)

`OrderResponse` の生成は `application/mapper/OrderMapper.java`(静的メソッド)。要件書 §8-4-1 / [`02-api-spec.md`](02-api-spec.md) §4 に従う。

1α と 1β で出力項目が異なるため、メソッドを分けて定義する(テストも分かれる)。

### 6.1 注意点

- **金額・税率は文字列**。`BigDecimal#toPlainString()` を使う(`toString()` では `"1E+2"` 等の指数表記になりうる)
- `confirmedAt` は `Order.createdAt`(`OffsetDateTime`)をそのまま返す
- `currency` は `"JPY"` 固定(要件書 §4)
- 1α の `lineTotal` は `unitPrice × quantity`
- 1β の `lineSubtotal` / `lineTotal` は `OrderItem` の getter または計算で導出(DB 保持しない)

### 6.2 1α 版

```java
public static OrderResponse1Alpha toResponse1Alpha(Order order) {
    List<OrderItemResponse1Alpha> items = order.getItems().stream()
        .map(OrderMapper::toItemResponse1Alpha)
        .toList();
    return new OrderResponse1Alpha(
        order.getId(),
        order.getStatus(),
        order.getCreatedAt(),
        items,
        order.getTotalPrice().toPlainString(),
        "JPY"
    );
}
```

### 6.3 1β 版

```java
public static OrderResponse toResponse(Order order) {
    List<OrderItemResponse> items = order.getItems().stream()
        .map(OrderMapper::toItemResponse)
        .toList();
    return new OrderResponse(
        order.getId(),
        order.getOrderNumber(),
        order.getStatus(),
        order.getCreatedAt(),
        items,
        order.getSubtotal().toPlainString(),
        order.getTaxAmount().toPlainString(),
        order.getTotalPrice().toPlainString(),
        "JPY"
    );
}
```

> **補足**: 1α → 1β 移行時は `OrderResponse1Alpha` を削除し、`OrderResponse` のみに集約する。DTO の命名は実装時に短いもの(`OrderResponse` と `OrderResponseV1` 等)に寄せても可。

---

## 7. テスト設計(TDD)

### 7.1 1α のレイヤー別テスト観点

| 層 | テスト種別 | 赤 → 緑で書く観点 |
|----|-----------|-------------------|
| `Order.create` | 純 Java Unit Test | `status=CONFIRMED` / `totalPrice` が items と整合 / 空 items で例外 |
| `OrderItem.snapshot`(1α) | 純 Java Unit Test | 商品名・単価のスナップショット |
| `Product#decreaseStock`(1α) | 純 Java Unit Test | 十分在庫で減算 / 不足で `OutOfStockException` / 残量 0 境界 |
| `OrderService#createOrder` | Mockito Unit Test | 正常フロー / 空カートで 409 / 在庫不足で 409 / Cart がクリアされる |
| `OrderController` | `@WebMvcTest` | 正常時 201 + `OrderResponse1Alpha` / 各例外の HTTP マッピング |
| 結合(任意) | `@DataJpaTest` | cascade で `OrderItem` が同時 INSERT される |

### 7.2 1β で追加されるテスト観点

| 層 | テスト種別 | 赤 → 緑で書く観点 |
|----|-----------|-------------------|
| `Order.confirm`(1β) | 純 Java Unit Test | `statusHistory` に 1 件追加される / `subtotal`/`taxAmount`/`totalPrice` が整合 / `EmptyCartException` |
| `OrderItem.snapshot`(1β) | 純 Java Unit Test | 行単位税額の切り捨て / 税率・税区分のスナップショット |
| `ProductStock#decrease` | 純 Java Unit Test | 十分在庫で減算 / 不足で `OutOfStockException` / 残量 0 境界 |
| `OrderNumberGenerator` | Mockito + `Clock.fixed` | 初日 `20260416-0001` / 2 件目 `20260416-0002` / 翌日 `20260417-0001` |
| `OrderService#createOrder`(1β) | Mockito Unit Test | 採番衝突時の 1 回リトライ / `ACTIVE` のみ削除・`SFL` は残る |

### 7.3 1β の Aggregate テストの雛形

```java
class OrderTest {

    private static final OffsetDateTime NOW =
        OffsetDateTime.parse("2026-04-16T10:00:00+09:00");

    @Test
    void confirm_withValidItems_setsStatusToConfirmed() {
        Order order = Order.confirm("20260416-0001", List.of(sampleItem()), NOW);
        assertThat(order.getStatus()).isEqualTo(OrderStatus.CONFIRMED);
    }

    @Test
    void confirm_appendsOneHistoryEntry() {
        Order order = Order.confirm("20260416-0001", List.of(sampleItem()), NOW);
        assertThat(order.getStatusHistory()).hasSize(1);
        assertThat(order.getStatusHistory().get(0).getFromStatus()).isNull();
        assertThat(order.getStatusHistory().get(0).getToStatus())
            .isEqualTo(OrderStatus.CONFIRMED);
    }

    @Test
    void confirm_recalculatesTotalsFromItems() {
        OrderItem apple = orderItem("リンゴ", "100", 2, "0.08"); // 小計200, 税16
        OrderItem note  = orderItem("ノート", "300", 1, "0.10"); // 小計300, 税30
        Order order = Order.confirm("X", List.of(apple, note), NOW);
        assertThat(order.getSubtotal()).isEqualByComparingTo("500");
        assertThat(order.getTaxAmount()).isEqualByComparingTo("46");
        assertThat(order.getTotalPrice()).isEqualByComparingTo("546");
    }

    @Test
    void confirm_withEmptyItems_throws() {
        assertThatThrownBy(() -> Order.confirm("X", List.of(), NOW))
            .isInstanceOf(EmptyCartException.class);
    }
}
```

### 7.4 採番テストの雛形(1β)

```java
class OrderNumberGeneratorTest {

    @Mock OrderRepository orderRepository;
    Clock clock = Clock.fixed(
        Instant.parse("2026-04-16T01:00:00Z"),  // JST 10:00
        ZoneId.of("Asia/Tokyo"));

    @Test
    void generatesFirstNumberOfDay() {
        when(orderRepository.countByOrderNumberStartingWith("20260416-")).thenReturn(0L);
        OrderNumberGenerator gen = new OrderNumberGenerator(orderRepository, clock);
        assertThat(gen.generate()).isEqualTo("20260416-0001");
    }

    @Test
    void generatesNextSequence() {
        when(orderRepository.countByOrderNumberStartingWith("20260416-")).thenReturn(5L);
        OrderNumberGenerator gen = new OrderNumberGenerator(orderRepository, clock);
        assertThat(gen.generate()).isEqualTo("20260416-0006");
    }
}
```

### 7.5 `OrderService` のリトライテスト(1β)

```java
@Test
void createOrder_retriesOnceOnOrderNumberCollision() {
    // 1 回目の save で衝突、2 回目で成功するよう設定
    when(orderRepository.save(any()))
        .thenThrow(new DataIntegrityViolationException("unique violation"))
        .thenAnswer(inv -> inv.getArgument(0));

    OrderResponse res = orderService.createOrder(cartId);

    verify(orderRepository, times(2)).save(any());
    assertThat(res.status()).isEqualTo(OrderStatus.CONFIRMED);
}

@Test
void createOrder_throwsWhenSecondAttemptAlsoFails() {
    when(orderRepository.save(any()))
        .thenThrow(new DataIntegrityViolationException("1"))
        .thenThrow(new DataIntegrityViolationException("2"));

    assertThatThrownBy(() -> orderService.createOrder(cartId))
        .isInstanceOf(DataIntegrityViolationException.class);
}
```

---

## 8. チェックリスト(着手前)

### 8.1 Phase 1α の実装順序(TDD)

上から順に **赤 → 緑 → リファクタ** で実装する。

- [ ] `domain/exception/` に `ProductNotFoundException` / `OutOfStockException` / `CartItemNotFoundException` / `CartItemForbiddenException` を作成([`06-exception.md`](06-exception.md) §2.1 の 1α 分)
- [ ] `ProductTest`(`decreaseStock` 相当)→ `Product#decreaseStock` 実装
- [ ] `OrderItemTest`(1α)→ `OrderItem.snapshot(cartItem, product)` 実装
- [ ] `OrderTest`(1α)→ `Order.create(items, now)` 実装
- [ ] `OrderServiceTest`(1α)→ `OrderService#createOrder` 実装(採番リトライなし)
- [ ] `OrderMapperTest`(1α)→ `OrderMapper#toResponse1Alpha` 実装
- [ ] `OrderControllerTest`(`@WebMvcTest`)→ `OrderController#create` 実装

### 8.2 Phase 1β への拡張(リファクタ + 機能追加)

1α 完了後、以下の順で追加する。各ステップごとに Flyway マイグレーションを足す([`01-database.md`](01-database.md) §7)。

- [ ] `domain/exception/` に `EmptyCartException` / `IllegalStateTransitionException` / `CartItemQuantityExceededException` を追加
- [ ] `ProductStock` エンティティを作成(`V2__split_product_stock.sql`)し、`Product#decreaseStock` を `ProductStock#decrease` に移す
- [ ] `OrderItem` に税フィールドを追加(`V3__add_tax.sql`)。`OrderItemTest` を 1β 観点で拡張
- [ ] `OrderStatusHistory` を新設(`V4__order_status_history.sql`)。`Order` を Aggregate メソッド方式にリファクタ(`create` → `confirm`)
- [ ] `OrderNumberGenerator` を実装(`V6__order_number.sql`)
- [ ] `OrderService` に採番リトライを追加
- [ ] `OrderMapper#toResponse`(1β 版)を追加し、Controller の戻り型を差し替え
- [ ] 結合確認(任意): `@DataJpaTest` で cascade が効くこと
