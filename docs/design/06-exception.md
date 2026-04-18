# 06. 例外ハンドリング設計

要件定義書 v0.6 §7 および [`02-api-spec.md`](02-api-spec.md) §1.5・§6 を物理実装レベルで詰めた設計書。Phase 1 スコープ。

TDD で赤 → 緑 → リファクタを回すため、**「例外クラス : ハンドラ : テスト = 1 : 1 : 1」** の対応を原則とする。

例外クラス・ハンドラは **1α / 1β** に分類し、1α で最小限・1β で追加分を明示する。

---

## 1. 方針

| 項目 | 決定 |
|------|------|
| ハンドラ構造 | **例外クラスごとに `@ExceptionHandler` をベタに書く**(抽象化しない) |
| メッセージ言語 | 日本語 |
| ドメイン例外の文言 | **例外クラスのコンストラクタで生成**(advice 側で組み立てない) |
| Bean Validation メッセージ | **アノテーションの `message` 属性で日本語直書き** |
| `timestamp` 生成 | DI した `Clock` から取得(テストで固定可能) |
| advice の配置 | `presentation/advice/GlobalExceptionHandler.java`(1 ファイル集約) |
| `errors[]` フィールド | **1β で追加**。1α は `message` 単一 |

### 1.1 なぜ抽象化しないか

Phase 1 で扱う例外クラスは最終的に 7 種(1α 4 種 + 1β 追加 3 種)。`DomainException(ErrorCode)` のような基底クラス + Enum 分岐方式は、TDD のサイクルを「例外追加 → Enum 追加 → 分岐追加 → テスト追加」の 4 手にしてしまう。ベタ書きなら「例外追加 → ハンドラ追加 → テスト追加」の 3 手で済み、かつ IDE のジャンプで全挙動が見渡せる。Phase 2 で例外数が倍になったら再評価する。

### 1.2 メッセージをどこで生成するか

```java
// Good: 例外クラスが文言を知る
throw new ProductNotFoundException(productId);
// → 内部で "指定された商品は存在しません" を生成

// Bad: advice 側で組み立てる
// → 例外の意味が advice と例外クラスの 2 箇所に分散し、追跡が面倒
```

ドメイン例外は **「投げた側がユーザー向け文言を確定させる」** 方針。テストも例外クラス側の Unit Test 1 本で済む。

---

## 2. 例外カタログ

配置: `domain/exception/`

### 2.1 Phase 1α のドメイン例外

| クラス | HTTP | 発生箇所 | コンストラクタ引数 | メッセージ例 |
|--------|------|---------|-------------------|--------------|
| `ProductNotFoundException` | 404 | 商品取得時に未存在 | `UUID productId` | `"指定された商品は存在しません"` |
| `CartItemNotFoundException` | 404 | `cart_items.id` 未存在 | `UUID cartItemId` | `"指定されたカート商品は存在しません"` |
| `CartItemForbiddenException` | 403 | `X-Cart-Id` と `cart_items.cart_id` 不一致 | `UUID headerCartId, UUID itemCartId` | `"このカート行を操作する権限がありません"` |
| `OutOfStockException` | 409 | 在庫不足(追加・変更・注文確定) | `String productName, int stock, int requested` | `"在庫が不足しています(リンゴ: 在庫 1 / 要求 3)"` |
| `EmptyCartException` | 409 | 空カートで注文確定(1α から) | なし | `"カートが空のため注文できません"` |
| `CartItemQuantityExceededException` | 409 | マージ後 `quantity > 99`(1α からマージ規則あり) | `String productName, int merged, int limit` | `"カート内の数量は 99 を超えられません(リンゴ: 要求 120)"` |

> **1α の空カート扱い**: [`04-domain-cart-product.md`](04-domain-cart-product.md) §7 で確定した通り、**`EmptyCartException` を 1α から先行導入**する。1β で追加しない。
>
> **1α の重複マージ**: 同様に [`04-domain-cart-product.md`](04-domain-cart-product.md) §3.2 で 1α から Service 層マージを採用することが確定したため、`CartItemQuantityExceededException` も 1α で先行導入する。

### 2.2 Phase 1β で追加するドメイン例外

| クラス | HTTP | 発生箇所 | コンストラクタ引数 | メッセージ例 |
|--------|------|---------|-------------------|--------------|
| `IllegalStateTransitionException` | 409 | 注文状態遷移違反(Phase 2 で実使用、1β では基盤のみ) | `OrderStatus from, OrderStatus to` | `"CONFIRMED から PENDING への遷移は許可されていません"` |

> **`IllegalStateTransitionException` の扱い**: Phase 1 では `null → CONFIRMED` の 1 遷移しか発生せず本例外が投げられる経路は無いが、[`05-domain-order.md`](05-domain-order.md) の Aggregate 設計で将来の `cancel()` / `refund()` と同じ書き方にするため、1β でクラスを先行作成する(ハンドラとテストも含む)。

### 2.3 Spring が投げる標準例外(マッピングのみ / 1α から)

| 例外 | HTTP | 発生条件 |
|------|------|---------|
| `MethodArgumentNotValidException` | 400 | `@Valid` 違反(1α は `message` に概要 / 1β で `errors[]` に詳細を展開) |
| `HandlerMethodValidationException` | 400 | `@RequestParam` / `@PathVariable` に対する制約違反 |
| `HttpMessageNotReadableException` | 400 | リクエストボディが不正 JSON |
| `MissingRequestHeaderException` | 400 | `X-Cart-Id` など必須ヘッダー欠落 |
| `MethodArgumentTypeMismatchException` | 400 | パス変数・ヘッダー値の型変換失敗(UUID 不正含む) |
| `PropertyReferenceException` | 400 | Spring Data のソートフィールド名が許可外(§[02-api-spec.md](02-api-spec.md) §2.1) |
| `NoResourceFoundException` | 404 | 未定義パス(Spring 6+) |
| `HttpRequestMethodNotSupportedException` | 405 | メソッド不一致 |
| `Exception`(フォールバック) | 500 | 予期しない例外 |

---

## 3. ErrorResponse の生成規則

要件書 §7 および [`02-api-spec.md`](02-api-spec.md) §1.5 を実装レベルで確定。

### 3.1 フィールド生成

| フィールド | 値の生成方法 | フェーズ |
|-----------|--------------|---------|
| `status` | `HttpStatus#value()` | 1α |
| `error` | `HttpStatus#getReasonPhrase()`(`"Bad Request"` 等の英語) | 1α |
| `message` | 例外クラスの `getMessage()`(ドメイン例外)または固定文言(Spring 標準例外ごと) | 1α |
| `timestamp` | `Clock` から `Instant.now(clock)` → `DateTimeFormatter.ISO_INSTANT` で文字列化(UTC + `Z`) | 1α |
| `errors[]` | **1β で追加**。400 のときのみ付与(404 / 409 / 500 では **キー自体を省略**) | 1β |

### 3.2 `errors[]` の付与ルール(1β)

1β で `errors[]` が現れるのは以下のケースのみ:

| ケース | `errors[]` の中身 |
|-------|------------------|
| `MethodArgumentNotValidException` | `BindingResult.getFieldErrors()` を 1:1 にマッピング |
| `HandlerMethodValidationException` | 制約違反 1 件ごとにフィールド名を復元して追加 |

それ以外(`ProductNotFoundException` 等、概念的に「フィールド」を持たない例外)では `errors` キーを JSON から外す。Jackson 側は DTO に `@JsonInclude(JsonInclude.Include.NON_NULL)` を付け、`null` なら出力しない。

### 3.3 JSON 出力例

#### 1α(`errors[]` なし)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "商品名は必須です",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

> 1α では `MethodArgumentNotValidException` が投げられた際、`BindingResult` の **最初のフィールドエラー** の `defaultMessage` を `message` に入れる(簡易版)。

#### 1β(400 のみ `errors[]` あり)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "リクエストに誤りがあります",
  "timestamp": "2026-04-16T10:00:00Z",
  "errors": [
    { "field": "name",  "message": "商品名は必須です" },
    { "field": "price", "message": "0.01 以上で入力してください" }
  ]
}
```

#### 404(`errors[]` なし / 1α・1β 共通)

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "指定された商品は存在しません",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

---

## 4. `GlobalExceptionHandler` のスケルトン

1α では以下の **基本ハンドラ** のみ実装し、1β で `errors[]` 対応と追加例外のハンドラを足す。

### 4.1 1α 版スケルトン

```java
@RestControllerAdvice
@RequiredArgsConstructor
@Slf4j
public class GlobalExceptionHandler {

    private final Clock clock;

    // --- 400 ---
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        // 1α: 最初のフィールドエラーを message に入れる
        String firstMessage = ex.getBindingResult().getFieldErrors().stream()
            .findFirst()
            .map(fe -> fe.getDefaultMessage())
            .orElse("リクエストに誤りがあります");
        return build(HttpStatus.BAD_REQUEST, firstMessage);
    }

    @ExceptionHandler(MissingRequestHeaderException.class)
    public ResponseEntity<ErrorResponse> handleMissingHeader(MissingRequestHeaderException ex) {
        return build(HttpStatus.BAD_REQUEST,
            "必須ヘッダーが不足しています: " + ex.getHeaderName());
    }

    @ExceptionHandler(MethodArgumentTypeMismatchException.class)
    public ResponseEntity<ErrorResponse> handleTypeMismatch(MethodArgumentTypeMismatchException ex) {
        return build(HttpStatus.BAD_REQUEST,
            "パラメータの型が不正です: " + ex.getName());
    }

    @ExceptionHandler(HttpMessageNotReadableException.class)
    public ResponseEntity<ErrorResponse> handleUnreadable(HttpMessageNotReadableException ex) {
        return build(HttpStatus.BAD_REQUEST, "リクエストボディの形式が不正です");
    }

    @ExceptionHandler(PropertyReferenceException.class)
    public ResponseEntity<ErrorResponse> handleSortField(PropertyReferenceException ex) {
        return build(HttpStatus.BAD_REQUEST, "指定されたソートフィールドは許可されていません: " + ex.getPropertyName());
    }

    // --- 403 ---
    @ExceptionHandler(CartItemForbiddenException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(CartItemForbiddenException ex) {
        return build(HttpStatus.FORBIDDEN, ex.getMessage());
    }

    // --- 404 ---
    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleProductNotFound(ProductNotFoundException ex) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(CartItemNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleCartItemNotFound(CartItemNotFoundException ex) {
        return build(HttpStatus.NOT_FOUND, ex.getMessage());
    }

    @ExceptionHandler(NoResourceFoundException.class)
    public ResponseEntity<ErrorResponse> handleNoResource(NoResourceFoundException ex) {
        return build(HttpStatus.NOT_FOUND, "指定されたリソースは存在しません");
    }

    // --- 405 ---
    @ExceptionHandler(HttpRequestMethodNotSupportedException.class)
    public ResponseEntity<ErrorResponse> handleMethodNotAllowed(HttpRequestMethodNotSupportedException ex) {
        return build(HttpStatus.METHOD_NOT_ALLOWED,
            "このメソッドは許可されていません: " + ex.getMethod());
    }

    // --- 409 ---
    @ExceptionHandler(OutOfStockException.class)
    public ResponseEntity<ErrorResponse> handleOutOfStock(OutOfStockException ex) {
        return build(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(EmptyCartException.class)
    public ResponseEntity<ErrorResponse> handleEmptyCart(EmptyCartException ex) {
        return build(HttpStatus.CONFLICT, ex.getMessage());
    }

    @ExceptionHandler(CartItemQuantityExceededException.class)
    public ResponseEntity<ErrorResponse> handleQuantityExceeded(CartItemQuantityExceededException ex) {
        return build(HttpStatus.CONFLICT, ex.getMessage());
    }

    // --- 500 ---
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleUnknown(Exception ex) {
        log.error("Unhandled exception", ex);
        return build(HttpStatus.INTERNAL_SERVER_ERROR, "サーバー内部エラーが発生しました");
    }

    private ResponseEntity<ErrorResponse> build(HttpStatus status, String message) {
        ErrorResponse body = new ErrorResponse(
            status.value(),
            status.getReasonPhrase(),
            message,
            Instant.now(clock),
            null  // 1α は errors[] を持たない
        );
        return ResponseEntity.status(status).body(body);
    }
}
```

### 4.2 1β で追加・差し替えるハンドラ

1β 移行時は `handleValidation` を `errors[]` 対応版に差し替え、`IllegalStateTransitionException` のハンドラを足す。

```java
// --- 400(1β で差し替え) ---
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
    List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
        .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
        .toList();
    return build(HttpStatus.BAD_REQUEST, "リクエストに誤りがあります", errors);
}

// --- 409(1β で追加) ---
@ExceptionHandler(IllegalStateTransitionException.class)
public ResponseEntity<ErrorResponse> handleIllegalTransition(IllegalStateTransitionException ex) {
    return build(HttpStatus.CONFLICT, ex.getMessage());
}

// `build` メソッドを多重化(errors 付き版を追加)
private ResponseEntity<ErrorResponse> build(HttpStatus status, String message, List<FieldError> errors) {
    ErrorResponse body = new ErrorResponse(
        status.value(), status.getReasonPhrase(), message, Instant.now(clock), errors
    );
    return ResponseEntity.status(status).body(body);
}
```

### 4.3 `ErrorResponse` / `FieldError` DTO

配置: `application/dto/`

```java
@JsonInclude(JsonInclude.Include.NON_NULL)
public record ErrorResponse(
    int status,
    String error,
    String message,
    Instant timestamp,
    List<FieldError> errors   // 1α では常に null、1β で 400 のみ入る
) {}

public record FieldError(String field, String message) {}
```

> **補足**: 1α の時点で `errors` フィールドを持たせ、常に `null` を入れておく(`@JsonInclude(NON_NULL)` で JSON から除外される)。1β で中身を入れるだけにして、DTO の record シグネチャ変更を避ける。

---

## 5. Bean Validation メッセージの日本語化(1α から)

### 5.1 方針: **アノテーションの `message` 属性で日本語直書き**

```java
public record ProductCreateRequest(
    @NotBlank(message = "商品名は必須です")
    String name,

    String description,

    @NotNull(message = "価格は必須です")
    @DecimalMin(value = "0.01", message = "0.01 以上で入力してください")
    @DecimalMax(value = "9999999999.99", message = "価格が大きすぎます")
    BigDecimal price,

    @NotNull(message = "税区分は必須です")          // 1β で追加
    TaxCategory taxCategory,

    @Min(value = 0, message = "在庫数は 0 以上で入力してください")
    int stock,

    String imageUrl
) {}
```

| 候補 | 評価 |
|------|------|
| **`message` 直書き**(採用) | フィールドの真横に文言がありジャンプ不要。TDD で赤になったときに同じファイルで修正できる |
| `ValidationMessages.properties` + `{key}` | i18n 本番運用には有効だが、Phase 1 は日本語のみ。プロパティファイル参照で IDE の往復が増える |
| `MessageSource` に `{0}`・`{1}` で動的生成 | 柔軟だがメッセージが Java とプロパティ両方に分散する |

**移行性**: 多言語化が必要になったら `message = "..."` を `message = "{product.name.notBlank}"` に機械的置換し、プロパティファイルに寄せる。先送りで問題ない。

### 5.2 `errors[].field` のパス表記(1β)

| 違反箇所 | `field` の値(Spring デフォルト) |
|---------|--------------------------------|
| トップレベル | `name` |
| ネストオブジェクト | `address.zipcode` |
| 配列要素 | `items[0].quantity` |

Jackson のプロパティパスと一致しており、フロント側の型でもそのまま使える。advice で再加工しない。

---

## 6. ロギング方針(1α から)

| ステータス | ログレベル | 備考 |
|-----------|-----------|------|
| 400 / 403 / 404 / 405 / 409 | **ログ出力しない**(正常な業務エラー) | 大量に出るとノイズになる。必要なら `DEBUG` |
| 500 | `ERROR`(スタックトレース付き) | 想定外 → 即座に気づきたい |

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ErrorResponse> handleUnknown(Exception ex) {
    log.error("Unhandled exception", ex);
    return build(HttpStatus.INTERNAL_SERVER_ERROR, "サーバー内部エラーが発生しました");
}
```

業務例外にスタックトレースが欲しくなったら、個別の `@ExceptionHandler` で `log.debug("...", ex)` を足す(本設計書では入れない)。

---

## 7. テスト設計(TDD)

### 7.1 原則

**例外クラス : ハンドラメソッド : テストメソッド = 1 : 1 : 1**

新しい例外を追加する流れ:

1. **赤**: advice テストに「この例外が投げられたら xxx でこのメッセージ」を書く
2. **緑**: 例外クラス作成 → `@ExceptionHandler` メソッド追加
3. リファクタ: 原則として行わない(重複は 10 行未満なので許容)

### 7.2 `@WebMvcTest` の構成

```java
@WebMvcTest(controllers = DummyController.class)
@Import({ GlobalExceptionHandler.class, TestClockConfig.class })
class GlobalExceptionHandlerTest {

    @Autowired MockMvc mockMvc;

    @Test
    void productNotFound_returns404() throws Exception {
        mockMvc.perform(get("/dummy/product-not-found"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.status").value(404))
            .andExpect(jsonPath("$.error").value("Not Found"))
            .andExpect(jsonPath("$.message").value("指定された商品は存在しません"))
            .andExpect(jsonPath("$.timestamp").value("2026-04-16T10:00:00Z"))
            .andExpect(jsonPath("$.errors").doesNotExist());
    }

    // 1β で追加
    @Test
    void validation_returns400WithErrorsArray() throws Exception {
        mockMvc.perform(post("/dummy/validate")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{}"))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors").isArray())
            .andExpect(jsonPath("$.errors[?(@.field=='name')]").exists());
    }
    // ... 例外ごとに 1 メソッド
}
```

### 7.3 `DummyController` 方式(1α から)

各例外を確実に発火させるため、テスト専用 `DummyController` を `src/test/java/` 配下に置く(本番コードには含めない)。

```java
@RestController
@RequestMapping("/dummy")
class DummyController {
    @GetMapping("/product-not-found")
    void productNotFound() { throw new ProductNotFoundException(UUID.randomUUID()); }

    @GetMapping("/out-of-stock")
    void outOfStock() { throw new OutOfStockException("リンゴ", 1, 3); }

    @PostMapping("/validate")
    void validate(@RequestBody @Valid ProductCreateRequest req) { /* 何もしない */ }

    // ... 例外ごとに 1 エンドポイント
}
```

| 候補 | 評価 |
|------|------|
| **DummyController で発火**(採用) | 本物の Controller 依存なくハンドラだけをテスト可能。他の Controller 実装が壊れていてもテストが通る |
| 本物の `ProductController` を `@WebMvcTest` | 例外発火条件を整えるのにサービス層のモック設定が必要で準備が重い |
| Unit Test で `GlobalExceptionHandler` を直接 `new` | MockMvc を使わないため `@JsonInclude` 等のシリアライズ挙動が検証できない |

### 7.4 `Clock` の固定(1α から)

```java
@TestConfiguration
class TestClockConfig {
    @Bean
    Clock clock() {
        return Clock.fixed(Instant.parse("2026-04-16T10:00:00Z"), ZoneOffset.UTC);
    }
}
```

`timestamp` を決定論的に検証できる。

### 7.5 例外クラス自体の Unit Test(1α から)

メッセージ組み立てを検証する軽いテストを書く(特に `OutOfStockException` のようにフォーマット文字列を使うもの)。

```java
class OutOfStockExceptionTest {
    @Test
    void messageIncludesProductNameAndNumbers() {
        OutOfStockException ex = new OutOfStockException("リンゴ", 1, 3);
        assertThat(ex.getMessage())
            .isEqualTo("在庫が不足しています(リンゴ: 在庫 1 / 要求 3)");
    }
}
```

---

## 8. チェックリスト(着手前)

### 8.1 Phase 1α の実装順序

- [ ] `application/dto/ErrorResponse.java` / `FieldError.java` を作成(§4.3)
- [ ] `domain/exception/` に 1α 6 例外クラスを作成(§2.1)
- [ ] `presentation/advice/GlobalExceptionHandler.java`(1α 版)を作成(§4.1)
- [ ] `ClockConfig` が DI されていることを確認([`03-project-setup.md`](03-project-setup.md) §7.2 既決)
- [ ] 全 Request DTO で Bean Validation の `message` を日本語化(§5.1)
- [ ] `GlobalExceptionHandlerTest` で 1α 例外ごとに 1 テストメソッドを赤で書けている(§7)
- [ ] 500 だけ `log.error` していることを確認(§6)

### 8.2 Phase 1β で追加する作業

- [ ] `domain/exception/IllegalStateTransitionException` を追加(§2.2)
- [ ] `GlobalExceptionHandler#handleValidation` を `errors[]` 対応版に差し替え(§4.2)
- [ ] `build(HttpStatus, String, List<FieldError>)` のオーバーロードを追加
- [ ] `IllegalStateTransitionException` のハンドラを実装(§4.2)
- [ ] `GlobalExceptionHandlerTest` に `errors[]` 検証 / `IllegalStateTransition` のテストを追加(§7.2)
- [ ] `DummyController` に `IllegalStateTransition` 用のエンドポイントを追加(§7.3)
