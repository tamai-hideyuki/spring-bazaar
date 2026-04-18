# 02. API スキーマ詳細設計

要件定義書 v0.6 §5「API 設計(概要)」を物理的に詰めた仕様書。Phase 1 スコープ。
リクエスト / レスポンスの JSON スキーマ・バリデーション・エラー時の挙動を本書で確定する。

エンドポイント・フィールドには **フェーズタグ** を付与し、1α(コア)と 1β(応用拡張)で何が異なるかを明示する。

---

## 1. 共通仕様

### 1.1 ベース情報

| 項目 | 値 |
|------|-----|
| ベース URL(開発) | `http://localhost:8080` |
| API プレフィックス | `/api` |
| Content-Type | `application/json; charset=UTF-8`(リクエスト / レスポンス共通) |
| 文字コード | UTF-8 |
| 認証 | **なし**(Phase 1)。Phase 2 で `Authorization: Bearer <JWT>` を追加 |

### 1.2 共通リクエストヘッダー

| ヘッダー | 必須 | 対象 | 説明 |
|----------|------|------|------|
| `Content-Type: application/json` | ボディあり時 | 全 POST / PATCH | — |
| `X-Cart-Id: <UUID>` | カート系・注文確定 | `/api/cart/*`, `POST /api/orders` | フロント `localStorage` 生成の UUIDv4。サーバーは upsert で `carts` 行を保証 |

`X-Cart-Id` の値が UUID 形式に合致しない場合は **400 Bad Request**(`errors[0].field = "X-Cart-Id"`。1α では `errors[]` がないため `message` に集約)。

### 1.3 値の表現規約

| 型 | JSON 表現 | 理由・例 |
|----|----------|---------|
| UUID | 文字列(RFC 4122) | 例: `"0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a"` |
| 金額 (`BigDecimal`) | **文字列** | JSON 数値型は double 経由で精度が崩れる。例: `"100"` `"546"` |
| 税率 (`BigDecimal`) | **文字列**(1β) | 例: `"0.08"` `"0.10"` |
| 日時 (`OffsetDateTime`) | ISO-8601 文字列(オフセット付き) | 例: `"2026-04-16T10:00:00+09:00"` |
| 列挙 | 文字列(大文字 SNAKE) | `"ACTIVE"` `"REDUCED"` 等 |
| 数量 (`Integer`) | 数値 | `1` 〜 `99` |
| ページ番号 | 数値(0 始まり) | Spring Data Pageable 準拠 |

> **金額・税率を必ず文字列で送る理由**: `BigDecimal` は Jackson のデフォルト設定では JSON 数値型(double)に変換される過程で精度が崩れうる。サーバー側は `@JsonFormat(shape = JsonFormat.Shape.STRING)` または `WRITE_BIGDECIMAL_AS_PLAIN` で文字列出力に固定する。

### 1.4 列挙値

| 名前 | 値 | フェーズ | 用途 |
|------|-----|---------|------|
| `TaxCategory` | `STANDARD` / `REDUCED` | 1β | 税区分 |
| `CartItemStatus` | `ACTIVE` / `SAVED_FOR_LATER` | 1β | カート行の状態 |
| `OrderStatus` | `PENDING` / `CONFIRMED` / `CANCELLED` / `REFUNDED` | 1α(値は `CONFIRMED` のみ)/ 1β(Enum 拡張、実使用は `CONFIRMED` のまま) | 注文状態 |

### 1.5 共通レスポンス: `ErrorResponse`

要件書 §7 に準拠。すべてのエラーは `@RestControllerAdvice` で本形式に統一する。

#### 1α の共通フォーマット(`errors[]` なし)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "リクエストに誤りがあります",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

#### 1β で追加される `errors[]` フィールド

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

| フィールド | 型 | フェーズ | 備考 |
|-----------|-----|---------|------|
| `status` | number | 1α | HTTP ステータスコード |
| `error` | string | 1α | HTTP リーズンフレーズ(`Bad Request` 等) |
| `message` | string | 1α | 概要メッセージ(日本語) |
| `timestamp` | string | 1α | サーバー時刻(UTC, ISO-8601 + `Z`) |
| `errors[]` | array | **1β** | **400 のみ付与**。404 / 409 / 500 では省略 |
| `errors[].field` | string | **1β** | 違反フィールドのパス。ネストは `items[0].quantity` の dot/bracket 記法 |
| `errors[].message` | string | **1β** | フィールド単位のメッセージ |

### 1.6 ページネーション: `Page<T>`(1α から)

商品一覧で使用。Spring Data の `Page<T>` をそのまま JSON 化せず、本仕様で安定化したスキーマで返す。

```json
{
  "content": [ /* T の配列 */ ],
  "page": {
    "number": 0,
    "size": 20,
    "totalElements": 123,
    "totalPages": 7,
    "first": true,
    "last": false,
    "numberOfElements": 20
  },
  "sort": "name,asc"
}
```

クエリパラメータ:

| 名前 | 型 | 既定値 | 制約 | 備考 |
|------|-----|--------|------|------|
| `page` | int | `0` | `>= 0` | 0 始まり |
| `size` | int | `20` | `1` ≤ size ≤ **`100`** | 上限を明示(過大取得の防止) |
| `sort` | string | `name,asc` | `<field>,<asc\|desc>` | Phase 1 は `name` `price` `createdAt` を許可 |

範囲外指定はサーバー側で 400 を返すのではなく **既定値にクランプ**(`size > 100` は `100` に丸める)。実装は `PageableHandlerMethodArgumentResolverCustomizer` で `maxPageSize` を設定。

---

## 2. Products(商品)

### 2.1 `GET /api/products` — 商品一覧取得(1α)

#### Request

| 項目 | 値 |
|------|-----|
| Method | GET |
| Path | `/api/products` |
| Headers | — |
| Query | `page` / `size` / `sort`(§1.6 参照) |
| Body | — |

#### Response (200 OK): `Page<ProductResponse>`

##### 1α のレスポンス例

```json
{
  "content": [
    {
      "id": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
      "name": "リンゴ",
      "description": "青森産。1個 100g 程度。",
      "price": "100",
      "stock": 50,
      "imageUrl": "https://example.com/apple.png",
      "createdAt": "2026-04-15T10:00:00+09:00",
      "updatedAt": "2026-04-15T10:00:00+09:00"
    }
  ],
  "page": { "number": 0, "size": 20, "totalElements": 1, "totalPages": 1, "first": true, "last": true, "numberOfElements": 1 },
  "sort": "name,asc"
}
```

##### 1β のレスポンス例(`taxCategory` 追加)

```json
{
  "content": [
    {
      "id": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
      "name": "リンゴ",
      "description": "青森産。1個 100g 程度。",
      "price": "100",
      "taxCategory": "REDUCED",
      "stock": 50,
      "imageUrl": "https://example.com/apple.png",
      "createdAt": "2026-04-15T10:00:00+09:00",
      "updatedAt": "2026-04-15T10:00:00+09:00"
    }
  ],
  "page": { "number": 0, "size": 20, "totalElements": 1, "totalPages": 1, "first": true, "last": true, "numberOfElements": 1 },
  "sort": "name,asc"
}
```

#### `ProductResponse` フィールド仕様

| フィールド | 型 | フェーズ | 必須 | 備考 |
|-----------|-----|---------|------|------|
| `id` | UUID | 1α | ○ | — |
| `name` | string | 1α | ○ | 最大 255 文字 |
| `description` | string \| null | 1α | — | 任意。最大長は仕様未定(`TEXT`) |
| `price` | 文字列 BigDecimal | 1α | ○ | **税抜**。`>= 0.01` |
| `taxCategory` | `TaxCategory` | 1β | ○ | `STANDARD` または `REDUCED` |
| `stock` | int | 1α | ○ | 1α は `products.stock`、1β は `product_stocks.quantity` を JOIN して返す |
| `imageUrl` | string \| null | 1α | — | 任意 |
| `createdAt` | OffsetDateTime | 1α | ○ | — |
| `updatedAt` | OffsetDateTime | 1α | ○ | — |

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | `sort` のフィールド名が許可外(`name` / `price` / `createdAt` 以外)。Spring Data の `PropertyReferenceException` を advice で 400 にマッピング |

---

### 2.2 `GET /api/products/{id}` — 商品詳細取得(1α)

#### Request

| 項目 | 値 |
|------|-----|
| Method | GET |
| Path | `/api/products/{id}` |
| Path 変数 | `id`(UUID) |
| Body | — |

#### Response (200 OK): `ProductResponse`

§2.1 と同形式(1α / 1β で同じ差異)。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | `id` が UUID 形式でない |
| 404 | 1α | `ProductNotFoundException` — 該当商品なし |

---

### 2.3 `POST /api/products` — 商品登録(管理者用)(1α)

> Phase 1 は認証なしで誰でも叩ける(ローカル開発前提)。Phase 2 で `ADMIN` ロール必須化。

#### Request

| 項目 | 値 |
|------|-----|
| Method | POST |
| Path | `/api/products` |
| Headers | `Content-Type: application/json` |
| Body | `ProductCreateRequest` |

##### 1α の Request Body

```json
{
  "name": "リンゴ",
  "description": "青森産。1個 100g 程度。",
  "price": "100",
  "stock": 50,
  "imageUrl": "https://example.com/apple.png"
}
```

##### 1β の Request Body(`taxCategory` 追加)

```json
{
  "name": "リンゴ",
  "description": "青森産。1個 100g 程度。",
  "price": "100",
  "taxCategory": "REDUCED",
  "stock": 50,
  "imageUrl": "https://example.com/apple.png"
}
```

#### `ProductCreateRequest` バリデーション

| フィールド | 型 | フェーズ | 必須 | バリデーション |
|-----------|-----|---------|------|---------------|
| `name` | string | 1α | ○ | `@NotBlank`、最大 255 文字 |
| `description` | string | 1α | — | 任意 |
| `price` | 文字列 BigDecimal | 1α | ○ | `@DecimalMin("0.01")` + `@DecimalMax("9999999999.99")` |
| `taxCategory` | `TaxCategory` | 1β | ○ | `@NotNull`。`STANDARD` または `REDUCED` |
| `stock` | int | 1α | ○ | `@Min(0)` |
| `imageUrl` | string | 1α | — | 任意。最大 2048 文字 |

#### 副作用

- **1α**: `products` 行を作成(`stock` 列を同一行に持つ)
- **1β**: 同一トランザクションで `products` と `product_stocks`(`quantity = stock`)の 2 行を作成

#### Response (201 Created): `ProductResponse`

§2.1 と同形式。`Location: /api/products/{id}` ヘッダーを付与。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | バリデーション違反(1α は `message` 単一、1β は `errors[]` に詳細) |

---

## 3. Cart(カート)

### 3.1 `GET /api/cart` — カート内容取得

`X-Cart-Id` のカートが未存在の場合も **空のカートを返す**(404 にしない。フロントが起動直後に叩いても自然に動作させるため)。

#### Request(1α / 1β 共通)

| 項目 | 値 |
|------|-----|
| Method | GET |
| Path | `/api/cart` |
| Headers | `X-Cart-Id: <UUID>` |
| Body | — |

#### Response (200 OK): `CartResponse`

##### 1α のレスポンス例(フラット構造・税なし)

```json
{
  "cartId": "8c3a3b6e-2c4f-4f1f-bbe8-2f1b9d3b7cce",
  "items": [
    {
      "id": "11111111-1111-1111-1111-111111111111",
      "productId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
      "productName": "リンゴ",
      "unitPrice": "100",
      "quantity": 2,
      "lineTotal": "200",
      "stockAvailable": 48
    }
  ],
  "totalPrice": "200",
  "currency": "JPY"
}
```

##### 1β のレスポンス例(`status` でグループ化・税あり)

```json
{
  "cartId": "8c3a3b6e-2c4f-4f1f-bbe8-2f1b9d3b7cce",
  "active": {
    "items": [
      {
        "id": "11111111-1111-1111-1111-111111111111",
        "productId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
        "productName": "リンゴ",
        "unitPrice": "100",
        "quantity": 2,
        "taxCategory": "REDUCED",
        "taxRate": "0.08",
        "lineSubtotal": "200",
        "taxAmount": "16",
        "lineTotal": "216",
        "stockAvailable": 48
      }
    ],
    "subtotal": "200",
    "taxAmount": "16",
    "totalPrice": "216"
  },
  "savedForLater": {
    "items": [
      {
        "id": "22222222-2222-2222-2222-222222222222",
        "productId": "f1c8...",
        "productName": "ノート",
        "unitPrice": "300",
        "quantity": 1,
        "taxCategory": "STANDARD",
        "taxRate": "0.10",
        "lineSubtotal": "300",
        "taxAmount": "30",
        "lineTotal": "330",
        "stockAvailable": 0
      }
    ]
  },
  "currency": "JPY"
}
```

#### フィールド仕様

| パス | 型 | フェーズ | 備考 |
|------|-----|---------|------|
| `cartId` | UUID | 1α | リクエストの `X-Cart-Id` をそのまま返す |
| **1α のみ** | | | |
| `items[]` | array | 1α のみ | 1β で `active.items[]` / `savedForLater.items[]` に分離 |
| `totalPrice` | 文字列 BigDecimal | 1α のみ | `Σ lineTotal`。1β では `active.totalPrice` に移動 |
| **1β で追加 / 変更** | | | |
| `active.items[]` | array | 1β | 空配列可 |
| `active.items[].taxCategory` | `TaxCategory` | 1β | 現在の税区分 |
| `active.items[].taxRate` | 文字列 BigDecimal | 1β | 現在適用される税率(`TaxProperties#rateOf`) |
| `active.items[].lineSubtotal` | 文字列 BigDecimal | 1β | `unitPrice × quantity`。サーバー側計算 |
| `active.items[].taxAmount` | 文字列 BigDecimal | 1β | `floor(lineSubtotal × taxRate)`。サーバー側計算 |
| `active.subtotal` / `taxAmount` / `totalPrice` | 文字列 BigDecimal | 1β | `active` の合計 |
| `savedForLater.items[]` | array | 1β | 構造は active と同一。**合計フィールドは出さない**(注文対象外のため) |
| **両フェーズ共通(フラット構造 or active 内)** | | | |
| `id` | UUID | 1α | `cart_items.id`。PATCH/DELETE のパス変数 |
| `productId` | UUID | 1α | — |
| `productName` | string | 1α | **現在の** 商品名(注文確定までスナップショットしない) |
| `unitPrice` | 文字列 BigDecimal | 1α | **現在の** 単価(1α は税抜 / 税込の区別なし、1β は税抜) |
| `quantity` | int | 1α | `1` 〜 `99` |
| `lineTotal` | 文字列 BigDecimal | 1α | 1α は `unitPrice × quantity`、1β は `lineSubtotal + taxAmount` |
| `stockAvailable` | int | 1α | 現在の在庫(1α は `products.stock`、1β は `product_stocks.quantity`)。在庫切れバッジ表示用 |
| `currency` | string | 1α | Phase 1 は `"JPY"` 固定 |

> **注意**: `lineSubtotal` 等のカート上の金額は **あくまで現時点の参考値**。注文確定時に再計算され `order_items.tax_rate` 等にスナップショットされる。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | `X-Cart-Id` が未指定または UUID 形式でない |

---

### 3.2 `POST /api/cart/items` — カートに商品追加

#### Request(1α / 1β 共通)

| 項目 | 値 |
|------|-----|
| Method | POST |
| Path | `/api/cart/items` |
| Headers | `X-Cart-Id` / `Content-Type: application/json` |
| Body | `CartItemAddRequest` |

```json
{
  "productId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
  "quantity": 2
}
```

#### バリデーション

| フィールド | 型 | フェーズ | 必須 | バリデーション |
|-----------|-----|---------|------|---------------|
| `productId` | UUID | 1α | ○ | `@NotNull` |
| `quantity` | int | 1α | ○ | `@Min(1)` + `@Max(99)` |

`status` は受け付けない(常に `ACTIVE` で追加)。1β の `SAVED_FOR_LATER` への切替は PATCH を使う。

#### サーバー処理

##### 1α

1. `X-Cart-Id` のカートを upsert(なければ INSERT)
2. **Service 層で `(cart_id, product_id)` をキーに既存行を検索**([`04-domain-cart-product.md`](04-domain-cart-product.md) §3.2 で確定):
   - 既存行があれば `quantity` に加算(マージ)
   - なければ新規 INSERT
3. 加算後 / 新規 の `quantity` が `@Max(99)` を超えるなら `CartItemQuantityExceededException`(409)
4. 加算後 / 新規 の `quantity` が **在庫数を超える** なら `OutOfStockException`(409)
5. 在庫減算は **行わない**(注文確定時のみ減らす)

> 1α では DB に `(cart_id, product_id)` の UNIQUE 制約を張らず、アプリ層でマージする。1β で `status` 列追加と同時に `UNIQUE (cart_id, product_id, status)` を導入する([`01-database.md`](01-database.md) §8.5)。

##### 1β

1. `X-Cart-Id` のカートを upsert(なければ INSERT)
2. **重複マージ**: `(cart_id, product_id, status='ACTIVE')` の既存行があれば、新規 INSERT せず既存行の `quantity` に加算
3. 加算後の `quantity` が `@Max(99)` を超える、または **在庫数を超える** 場合は **409 Conflict**(ロールバック)
4. 在庫減算は行わない(注文確定時のみ)

#### Response (200 OK): `CartResponse`

§3.1 と同形式(1α はフラット、1β はグループ化)。**追加後のカート全体** を返す(フロント側のキャッシュ更新を簡素化するため)。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | バリデーション違反 / `X-Cart-Id` 不正 |
| 404 | 1α | `productId` の商品が存在しない(`ProductNotFoundException`) |
| 409 | 1α | `OutOfStockException`(`quantity` が在庫を超過) / `CartItemQuantityExceededException`(マージ後 `@Max(99)` 超過) |

---

### 3.3 `PATCH /api/cart/items/{id}` — カート商品の数量 / ステータス変更

#### Request

| 項目 | 値 |
|------|-----|
| Method | PATCH |
| Path | `/api/cart/items/{id}` |
| Path 変数 | `id`(UUID。`cart_items.id`) |
| Headers | `X-Cart-Id` / `Content-Type: application/json` |
| Body | `CartItemUpdateRequest` |

##### 1α の Request Body(数量のみ)

```json
{ "quantity": 3 }
```

##### 1β の Request Body(`status` 追加)

```json
{
  "quantity": 3,
  "status": "SAVED_FOR_LATER"
}
```

#### バリデーション

| フィールド | 型 | フェーズ | 必須 | バリデーション |
|-----------|-----|---------|------|---------------|
| `quantity` | int | 1α | — | `@Min(1)` + `@Max(99)`。省略時は変更なし |
| `status` | `CartItemStatus` | 1β | — | `ACTIVE` または `SAVED_FOR_LATER`。省略時は変更なし |

**両方省略は不可**(400 を返す)。1α は `quantity` が唯一のフィールド、1β では少なくとも 1 フィールドは必要。

#### サーバー処理

##### 1α

1. パス変数 `id` の `cart_items` を取得。`X-Cart-Id` と `cart_id` が不一致なら **403 Forbidden**(他人カート操作の防止)
2. `quantity` 変更時: 新値が **現在の在庫を超える** なら 409

##### 1β(在庫チェックの 2 段階)

1. パス変数 `id` の `cart_items` を取得。`X-Cart-Id` と `cart_id` が不一致なら **403 Forbidden**
2. `quantity` 変更時: 新値が **現在の在庫を超える** なら 409
3. `status` を `SAVED_FOR_LATER → ACTIVE` に戻す場合: **その時点の在庫が `quantity` を下回るなら 409**(ユーザー側で数量を減らして再試行する UX)
4. `status` を `ACTIVE → SAVED_FOR_LATER` に変える場合: 在庫チェック不要
5. 重複マージ: ステータス変更により `(cart_id, product_id, status)` が既存行と衝突する場合、既存行に `quantity` をマージし本行は DELETE。マージ後合計が `@Max(99)` または在庫を超える場合は 409

#### Response (200 OK): `CartResponse`

§3.1 と同形式。**変更後のカート全体** を返す。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | バリデーション違反 / 全フィールド省略 / `X-Cart-Id` 不正 |
| 403 | 1α | パス変数 `id` のカート行が `X-Cart-Id` のカートに属さない |
| 404 | 1α | `id` の `cart_items` が存在しない |
| 409 | 1α | 在庫不足(数量変更時) |
| 409 | 1β | SAVED_FOR_LATER → ACTIVE 復帰時の在庫不足 / マージ時の `@Max(99)` or 在庫超過 |

---

### 3.4 `DELETE /api/cart/items/{id}` — カートから商品削除(1α)

#### Request

| 項目 | 値 |
|------|-----|
| Method | DELETE |
| Path | `/api/cart/items/{id}` |
| Path 変数 | `id`(UUID。`cart_items.id`) |
| Headers | `X-Cart-Id` |
| Body | — |

#### Response (204 No Content)

ボディなし。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | `X-Cart-Id` 不正 |
| 403 | 1α | パス変数 `id` のカート行が `X-Cart-Id` のカートに属さない |
| 404 | 1α | `id` の `cart_items` が存在しない |

> 冪等性: 既に削除済みの `id` を再度 DELETE する場合は 404 を返す(`DELETE` の冪等性は HTTP 仕様上「副作用が同じ」を意味するだけで、レスポンスは異なってよい)。

---

## 4. Orders(注文)

### 4.1 `POST /api/orders` — 注文確定(モック)

1α は Cart 内全商品を対象、1β は **`ACTIVE`** 商品のみを対象に注文を確定する。1β では `SAVED_FOR_LATER` は残す。

#### Request(1α / 1β 共通)

| 項目 | 値 |
|------|-----|
| Method | POST |
| Path | `/api/orders` |
| Headers | `X-Cart-Id` / `Content-Type: application/json` |
| Body | **空オブジェクト `{}` または省略** |

> Phase 1 は決済モック・配送先なしのため、リクエストボディは空。Phase 2 で `paymentMethod` 等を追加する余地として `application/json` は残す。

#### サーバー処理

##### 1α のフロー

1. `X-Cart-Id` のカートの全 `cart_items` を取得
2. **空なら 409**(`EmptyCartException` — [`04-domain-cart-product.md`](04-domain-cart-product.md) §7 で 1α から先行導入することが確定)
3. 各行で在庫チェック(不足あれば 409、`OutOfStockException`)
4. `products.quantity` を行ごとに減算
5. `cart_items` の値を `order_items` にコピー(商品名・単価・数量)
6. `cart_items` を全削除
7. `orders` を `status=CONFIRMED`、`total_price` をセットして INSERT
8. `OrderResponse`(1α 最小版)を返す(201 Created)

##### 1β のフロー(要件書 §8-6)

1. `X-Cart-Id` のカートで `status='ACTIVE'` の `cart_items` を全取得
2. **空なら 409**(`EmptyCartException`)
3. 各行で在庫チェック(不足あれば 409、`OutOfStockException`)
4. `product_stocks.quantity` を行ごとに減算(楽観ロックなし)
5. `cart_items` の値を `order_items` にコピー(商品名・税抜単価・税区分・税率・行単位税額)
6. `cart_items` の `ACTIVE` 行を DELETE(`SAVED_FOR_LATER` は残す)
7. **注文番号を採番**(`OrderNumberGenerator` / UNIQUE 衝突時は 1 回リトライ)
8. `orders` を `status=CONFIRMED`、`subtotal` / `tax_amount` / `total_price` / `order_number` をセットして INSERT
9. **`order_status_history` に `(from=null, to=CONFIRMED)` を 1 件 INSERT**
10. `OrderResponse`(1β 拡張版)を返す(201 Created)

すべて **同一トランザクション**。途中で例外が出れば全ロールバック。

#### Response (201 Created): `OrderResponse`

`Location: /api/orders/{orderId}` ヘッダーを付与(取得 API は Phase 2 だが将来互換のため付ける)。

##### 1α のレスポンス例

```json
{
  "orderId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
  "status": "CONFIRMED",
  "confirmedAt": "2026-04-16T10:00:00+09:00",
  "items": [
    {
      "orderItemId": "a1c8...",
      "productId": "p1f2...",
      "productName": "リンゴ",
      "unitPrice": "100",
      "quantity": 2,
      "lineTotal": "200"
    }
  ],
  "totalPrice": "200",
  "currency": "JPY"
}
```

##### 1β のレスポンス例(`orderNumber` / 税関連 / `subtotal` 追加)

```json
{
  "orderId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
  "orderNumber": "20260416-0001",
  "status": "CONFIRMED",
  "confirmedAt": "2026-04-16T10:00:00+09:00",
  "items": [
    {
      "orderItemId": "a1c8...",
      "productId": "p1f2...",
      "productName": "リンゴ",
      "unitPrice": "100",
      "quantity": 2,
      "taxCategory": "REDUCED",
      "taxRate": "0.08",
      "lineSubtotal": "200",
      "taxAmount": "16",
      "lineTotal": "216"
    },
    {
      "orderItemId": "b2d9...",
      "productId": "p3a4...",
      "productName": "ノート",
      "unitPrice": "300",
      "quantity": 1,
      "taxCategory": "STANDARD",
      "taxRate": "0.10",
      "lineSubtotal": "300",
      "taxAmount": "30",
      "lineTotal": "330"
    }
  ],
  "subtotal": "500",
  "taxAmount": "46",
  "totalPrice": "546",
  "currency": "JPY"
}
```

#### `OrderResponse` フィールド仕様

| フィールド | 型 | フェーズ | 備考 |
|-----------|-----|---------|------|
| `orderId` | UUID | 1α | `Order.id` |
| `orderNumber` | string | 1β | `yyyyMMdd-NNNN` |
| `status` | `OrderStatus` | 1α | Phase 1 では常に `CONFIRMED` |
| `confirmedAt` | ISO-8601 | 1α | `Order.createdAt` を `OffsetDateTime` で返す(タイムゾーン付き) |
| `items[].orderItemId` / `productId` / `productName` / `unitPrice` / `quantity` | — | 1α | スナップショット |
| `items[].taxCategory` / `taxRate` / `lineSubtotal` / `taxAmount` | — | 1β | — |
| `items[].lineTotal` | 文字列 | 1α(単純積)/ 1β(税込) | サーバー側計算 |
| `subtotal` | 文字列 | 1β | 税抜合計 |
| `taxAmount`(全体) | 文字列 | 1β | 税額合計 |
| `totalPrice` | 文字列 | 1α | 1α は単純合計、1β は税込 |
| `currency` | string | 1α | `"JPY"` 固定 |

すべての金額・税率は **文字列**(JSON 数値型に落とさない)。

#### エラー

| ステータス | フェーズ | 発生条件 |
|-----------|---------|---------|
| 400 | 1α | `X-Cart-Id` 不正 |
| 404 | 1α | カートまたは商品が存在しない |
| 409 | 1α | `OutOfStockException`(在庫不足)/ `EmptyCartException`(カート空) |
| 409 | 1β | 注文番号採番の 2 回目も衝突(`DataIntegrityViolationException` がそのまま 500 に落ちる可能性あり) |

---

## 5. Phase 2 で追加予定エンドポイント(本書では仕様未確定)

詳細は [`../phase2-plan.md`](../phase2-plan.md) を参照。

| Method | Path | 説明 |
|--------|------|------|
| GET | `/api/orders` | 注文一覧取得(認証連携前提) |
| GET | `/api/orders/{id}` | 注文詳細取得 |
| PATCH | `/api/orders/{id}/cancel` | 注文キャンセル(`CONFIRMED → CANCELLED`) |
| POST | `/api/orders/{id}/refund` | 返金処理(`CANCELLED → REFUNDED`) |
| POST | `/api/auth/register` | ユーザー登録 |
| POST | `/api/auth/login` | ログイン(access JWT 返却 + refresh Cookie) |
| POST | `/api/auth/refresh` | アクセストークン再発行(CSRF 要求) |
| POST | `/api/auth/logout` | リフレッシュトークン revoke |
| PUT | `/api/products/{id}` | 商品更新 |
| DELETE | `/api/products/{id}` | 商品削除 |

---

## 6. レスポンス例集(エラー)

### 6.1 バリデーション違反(400)

##### 1α(`errors[]` なし)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "商品名は必須です",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

##### 1β(`errors[]` あり)

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

### 6.2 商品未存在(404)

```json
{
  "status": 404,
  "error": "Not Found",
  "message": "指定された商品は存在しません",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

### 6.3 在庫不足(409)

```json
{
  "status": 409,
  "error": "Conflict",
  "message": "在庫が不足しています(リンゴ: 在庫 1 / 要求 3)",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

### 6.4 注文状態遷移違反(409, 1β で導入・Phase 2 で実使用)

```json
{
  "status": 409,
  "error": "Conflict",
  "message": "CONFIRMED から PENDING への遷移は許可されていません",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

### 6.5 他人カート操作(403)

```json
{
  "status": 403,
  "error": "Forbidden",
  "message": "このカート行を操作する権限がありません",
  "timestamp": "2026-04-16T10:00:00Z"
}
```

---

## 7. OpenAPI / Swagger UI

`springdoc-openapi` で `/v3/api-docs` および `/swagger-ui.html` を公開する。

- 本書の内容を **DTO の `@Schema` アノテーション** で機械可読な形に落とす(コードと仕様の二重管理を避ける)
- `Page<T>` のラップ JSON は `springdoc-openapi-starter-webmvc-ui` の独自実装で揃える(デフォルトの Spring Page スキーマは安定しないため、§1.6 の整形クラスを別途用意)

---

## 8. 未決事項

| # | 項目 | 暫定決定 | 備考 |
|---|------|---------|------|
| 1 | `description` の最大長 | 未制限(`TEXT`) | UI 表示で truncate 想定。必要なら `@Size(max = N)` 追加 |
| 2 | `imageUrl` の URL バリデーション | フォーマット検証なし | Phase 2 でファイルアップロード化する際に再検討 |
| 3 | カートが他人 ID と衝突した場合の扱い | UUIDv4 衝突は天文学的に稀 → 考慮しない | Phase 2 認証導入で恒久解決 |
| 4 | `POST /api/cart/items` のレスポンスに `Location` を返すか | **返さない**(`CartResponse` 全体を返すため) | RESTful 厳密派は分かれるが UX 優先 |
| 5 | `OrderResponse` への `cartId` 含有 | 含めない | カートはクリア済みのため意味薄。必要なら追加 |
| 6 | 1α カート重複時の挙動 | [`04-domain-cart-product.md`](04-domain-cart-product.md) §3.2 で確定:Service 層で `(cart_id, product_id)` をキーに `quantity` を加算マージ。DB UNIQUE は 1β で追加 | — |
| 7 | フロント型生成戦略(手書き vs OpenAPI 自動生成) | Phase 1α は手書き、膨らんだら `openapi-typescript` を検討 | 1β 完了時に再評価 |
