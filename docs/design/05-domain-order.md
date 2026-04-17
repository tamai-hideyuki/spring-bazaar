# 05. Order ドメイン設計

要件定義書 v0.5.2 §8-4〜8-6(注文確定ライフサイクル・`OrderResponse` スキーマ・`OrderStatusHistory`)と [`01-database.md`](01-database.md) を実装レベルで詰めた設計書。Phase 1 スコープ。

**ゴール**: 注文確定ユースケースを TDD で実装するための、Aggregate の契約とテスト観点を確定する。

---

## 1. 方針

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

## 2. Order Aggregate 設計

### 2.1 責務

- 注文ヘッダー情報(`id` / `orderNumber` / `status` / 金額合計 / `createdAt`)の保持
- 明細(`items`)と状態履歴(`statusHistory`)の所有
- 状態遷移メソッドの提供(Phase 1 は `confirm` のみ)
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

    /** Phase 1 で唯一の状態遷移。カート → 注文への確定 */
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

YAGNI に従い Phase 1 では **書かない**。ただし `IllegalStateTransitionException` はハンドラ・テストを含め先行作成する([`06-exception.md`](06-exception.md) §2.1 末尾の注記)。

---

## 3. OrderItem の生成

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

### 3.3 端数処理の具体例(要件書 v0.5.2 §4 と一致)

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

## 4. 注文番号採番

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

## 5. 注文確定ユースケース(`OrderService#createOrderInternal`)

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

`OrderResponse` の生成は `application/mapper/OrderMapper.java`(静的メソッド)。要件書 §8-4-1 / [`02-api-spec.md`](02-api-spec.md) §4.1 に従う。

### 6.1 注意点

- **金額・税率は文字列**。`BigDecimal#toPlainString()` を使う(`toString()` では `"1E+2"` 等の指数表記になりうる)
- `confirmedAt` は `Order.createdAt`(`OffsetDateTime`)をそのまま返す
- `currency` は `"JPY"` 固定(要件書 §4)
- `lineSubtotal` / `lineTotal` は `OrderItem` の getter(`getLineSubtotal()`)または計算で導出(DB 保持しない)

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

---

## 7. テスト設計(TDD)

### 7.1 レイヤー別テスト観点

| 層 | テスト種別 | 赤 → 緑で書く観点 |
|----|-----------|-------------------|
| `Order` Aggregate | 純 Java Unit Test | `confirm` が status=CONFIRMED にする / `statusHistory` に 1 件追加 / totals が items と整合 / 空 items で `EmptyCartException` |
| `OrderItem#snapshot` | 純 Java Unit Test | 行単位税額の切り捨て / 税率・商品名・単価のスナップショット |
| `ProductStock#decrease` | 純 Java Unit Test | 十分在庫で減算成功 / 不足で `OutOfStockException` / 残量 0 境界 |
| `OrderNumberGenerator` | Mockito で `OrderRepository` モック + `Clock.fixed` | 初日 `20260416-0001` / 2 件目 `20260416-0002` / 翌日 `20260417-0001` |
| `OrderService#createOrder` | Mockito Unit Test | 正常フロー / 空カートで 409 / 在庫不足で 409 / 採番衝突時の 1 回リトライ / ACTIVE のみ削除・SFL は残る |
| `OrderController` | `@WebMvcTest` | 正常時 201 + `OrderResponse` / 各例外の HTTP マッピング([`06-exception.md`](06-exception.md) §7) |
| 結合(任意) | `@DataJpaTest` | cascade で `OrderItem` / `statusHistory` が同時 INSERT される / `UNIQUE(order_number)` が効く |

### 7.2 Aggregate テストの雛形

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

### 7.3 採番テストの雛形

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

### 7.4 `OrderService` のリトライテスト

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

TDD の順序で並べる。**上から順に赤 → 緑 → リファクタで実装**する。

- [ ] `domain/exception/EmptyCartException` + `IllegalStateTransitionException` 作成([`06-exception.md`](06-exception.md) §2.1 先行)
- [ ] `ProductStockTest` → `ProductStock#decrease` 実装(§5.2)
- [ ] `OrderItemTest` → `OrderItem#snapshot` 実装(§3)
- [ ] `OrderTest` → `Order.confirm` 実装(§2)
- [ ] `OrderNumberGeneratorTest` → `OrderNumberGenerator` 実装(§4.2)
- [ ] `OrderServiceTest` → `OrderService#createOrder` 実装(§5・採番リトライ含む)
- [ ] `OrderMapperTest` → `OrderMapper#toResponse` 実装(§6)
- [ ] `OrderControllerTest`(`@WebMvcTest`) → `OrderController#create` 実装
- [ ] 結合確認(任意): `@DataJpaTest` で cascade が効くこと
