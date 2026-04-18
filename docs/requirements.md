# spring-bazaar 要件定義書

| 項目 | 値 |
|------|-----|
| プロジェクト名 | spring-bazaar |
| バージョン | **0.6.1** |
| 作成日 | 2026年4月 |
| 最終更新 | 2026年4月18日 |
| ステータス | Draft |
| スコープ | 商品管理・カート機能(認証は Phase 2) |
| 改版内容(v0.5.2 → v0.6) | 学習目的(Spring Boot / JPA / レイヤード)の再確認に基づき Phase 1 を **α(コア)/ β(応用拡張)** に分割。各機能・API・データフィールドにフェーズタグを付与。Phase 2 認証方針を別紙 [`phase2-plan.md`](phase2-plan.md) に分離。フロント CSS を Tailwind に明記 |

> 改版履歴は末尾付録に記載。v0.4 原本は [`archive/spring-bazaar-requirements-v4.docx`](archive/spring-bazaar-requirements-v4.docx) を参照。

---

## 1. プロジェクト概要

spring-bazaar は Java(Spring Boot)と React(TypeScript)を用いたフルスタック学習プロジェクトです。EC サイトのコアドメイン(商品閲覧・カート・モック注文)を通じて、以下の **学習目的 3 本柱** を実践的に理解することを目的とします。

1. **Spring Boot の REST API 設計** — Controller / DTO / `@Valid` / `@RestControllerAdvice` / ページネーション / CORS
2. **JPA によるデータ永続化** — Entity / Repository / 関連マッピング / トランザクション境界 / Flyway
3. **レイヤードアーキテクチャの実践** — domain / application / presentation / infrastructure の責務分離

Phase 1 は上記 3 本柱を最短で完走するため **α(コア機能)** と **β(応用拡張)** の 2 段階に分割する。α で骨格を通し、β で EC の実務的な拡張(消費税・状態履歴・あとで買う等)を積む。

---

## 2. スコープ

### 2-1. Phase 1α(コア機能)

学習目的 3 本柱を完走するための最小機能。

- 商品一覧・詳細の閲覧(ページネーション対応)
- カートへの商品追加・変更・削除(`ACTIVE` 状態のみ)
- 注文確定(在庫減算あり・決済はモック・単価スナップショットのみ)
- 管理者による商品登録(POST のみ)

### 2-2. Phase 1β(応用拡張)

α の骨格に EC 実務要素を積む。各項目は独立した学びの単位。

- 消費税対応(税抜保持・軽減税率・注文確定時点スナップショット)
- 在庫テーブル分離(`ProductStock`)
- 「あとで買う」(`ACTIVE` / `SAVED_FOR_LATER` 切替)
- 注文状態遷移履歴(`OrderStatusHistory`)+ Aggregate メソッド方式
- カート追加時の重複マージ規則 / 2 段階在庫チェック
- 注文番号採番(`yyyyMMdd-NNNN`)
- エラーレスポンスに `errors[]` を追加

### 2-3. Phase 2(対象外)

以下は Phase 1 では着手しない。設計メモは別紙 [`phase2-plan.md`](phase2-plan.md) に集約する。

- ユーザー認証・JWT 管理
- 商品の更新(PUT)・削除(DELETE)
- 決済処理の実連携
- 注文のキャンセル・返金(データモデルは β で先取り)
- 商品画像のファイルアップロード
- 税率改定マスタ管理(`tax_rates` テーブル化)
- 楽観的ロック(`@Version`)による在庫競合対策
- Docker Compose によるコンテナ化 / CI/CD
- `application/mapper/` を MapStruct に置き換え

---

## 3. 機能要件

| 機能 | フェーズ | 説明 | 優先度 |
|------|---------|------|--------|
| 商品一覧表示 | 1α | カード形式で一覧表示。ページネーション対応(`Page<T>`) | High |
| 商品詳細表示 | 1α | 商品名・価格・説明・在庫数を表示 | High |
| 商品詳細表示(税込併記) | 1β | 税抜/税込価格・税区分を併記 | High |
| カートへの追加 | 1α | 商品詳細からカートに追加。数量指定・在庫チェック | High |
| カートへの追加(マージ / 2 段階チェック) | 1β | 同一商品・同一ステータスの追加は既存行の `quantity` にマージ。在庫チェックはカート追加時・注文確定時の 2 段階 | High |
| カート確認 | 1α | カート内商品の一覧・合計金額を表示 | High |
| カート確認(税抜/税込・グループ化) | 1β | 合計金額を税抜/税込で表示。`ACTIVE` / `SAVED_FOR_LATER` をグループ化して返す | High |
| カートの変更 | 1α | 数量変更・商品削除。数量変更時も在庫チェック | Medium |
| 「あとで買う」への移動 / 復帰 | 1β | カート商品の `SAVED_FOR_LATER` / `ACTIVE` 切替。復帰時に在庫不足なら 409 で拒否(ユーザー側で数量調整) | Medium |
| 注文確定(モック) | 1α | カート内容を `Order` にスナップショット保存・在庫を減算。単価のみスナップショット | Medium |
| 注文確定(税率スナップショット・状態履歴) | 1β | `ACTIVE` のみ対象。税率も確定時点で固定化。注文番号を採番し、`OrderStatusHistory` に `(null → CONFIRMED)` を記録 | Medium |
| 管理者:商品登録 | 1α | 商品名・税抜価格・在庫数・画像 URL を登録(POST のみ) | Low |
| 管理者:商品登録(税区分) | 1β | 税区分(標準 / 軽減)を追加 | Low |

---

## 4. 非機能要件

### パフォーマンス(1α から)

- 商品一覧 API のレスポンスタイム: 500ms 以内(ローカル環境)

### セキュリティ(1α から)

- CORS 設定: フロントエンド(`localhost:5173`)のみ許可
- SQL インジェクション対策: JPA パラメータバインディングを使用
- **CSRF 対策**: Phase 1 は認証なしかつ SPA + REST 構成のため `CsrfConfigurer.disable()` で無効化。Phase 2 で JWT を `Authorization` ヘッダー方式で送る設計のため CSRF は引き続き無効のまま(詳細は [`phase2-plan.md`](phase2-plan.md) §2-5)

### 通貨(1α から)

- Phase 1 は **日本円 (JPY) 固定**
- 金額は `BigDecimal` / `NUMERIC(12, 2)` で保持するが、**表示は整数円**(UI 層で `toFixed(0)` 相当)
- 多通貨対応は Phase 2 以降の検討事項

### バリデーション(1α から)

- `spring-boot-starter-validation` を使用(`@Valid` / `@NotBlank` / `@Min` 等)
- DTO に Bean Validation アノテーションを付与し Controller 層で `@Valid` を適用
- バリデーションメッセージはアノテーションの `message` 属性に日本語で直書き(詳細は [`design/06-exception.md`](design/06-exception.md) §5.1)
- 在庫チェック(`quantity <= stock`)は Service 層でドメインロジックとして実装

### 在庫の同時更新(競合制御)(1α から)

Phase 1 は **楽観ロックなし・「最後勝ち」方針** とする。学習用の単一開発者環境を前提に、在庫減算の競合検知は行わない(同時注文により在庫がマイナスに見える可能性を許容)。

**Phase 2 での強化**: `ProductStock` に `@Version` を付与して楽観ロックを導入。注文確定 Service が `OptimisticLockException` を検知した場合は 409 Conflict を返し、ユーザーに再試行を促す。悲観ロック(`SELECT ... FOR UPDATE`)は採用しない(EC サイトにおいては衝突頻度が低く、楽観ロックの方がスループットが高いため)。

> **補足**: 楽観ロックは「読み取り時のバージョン値を書き込み時に `WHERE version = ?` で検証し、一致しなければ更新 0 件として失敗扱いにする」仕組み。JPA では `@Version Long version` フィールドを追加するだけで Hibernate が自動で SQL を組み立てる。

### 消費税対応(1β から適用)

- **商品価格は税抜保持**(`products.price`)。税込は表示時に計算
- **軽減税率対応**: 商品ごとに税区分を保持(`STANDARD` / `REDUCED`)
- **注文確定時に税率をスナップショット**: `order_items.tax_rate` として固定化し、税率改定を跨ぐキャンセル・返金でも払い戻し額が不変
- 税率は Phase 1 では `application.yml` で定数管理(Phase 2 でマスタテーブル化)
- 税額端数処理: **行単位で切り捨て**(消費税法の一般的運用)

#### 税率の設定仕様(1β)

1β では `application.yml` に以下の階層で保持し、`@ConfigurationProperties` で型安全に受ける。Phase 2 でマスタテーブル化する際に呼び出し側を変えずに差し替えられるよう、参照は `TaxProperties#rateOf(TaxCategory)` のような専用メソッドに集約する。

```yaml
tax:
  rates:
    standard: "0.10"   # 標準税率 10%
    reduced:  "0.08"   # 軽減税率 8%
  rounding: FLOOR       # 行単位切り捨て(消費税法の一般運用)
```

| 項目 | 値 | 理由 |
|------|-----|------|
| 格納型 | `BigDecimal`(YAML では文字列 `"0.10"`) | YAML の素の数値は `double` 経由で丸め誤差を含みうるため、必ず引用符で文字列として記述する |
| バインディング方式 | `@ConfigurationProperties(prefix = "tax")` を付けた `record TaxProperties` | `@Value` より拡張性が高く、Phase 2 で `effectiveFrom` 等のフィールドを足しやすい |
| 参照インターフェース | `TaxProperties#rateOf(TaxCategory) : BigDecimal` | 呼び出し側は Phase 2 のマスタ化でも変更不要 |
| 丸めモード | `RoundingMode.FLOOR` | 要件書本文(行単位切り捨て)と一致 |

**計算例**(1β 以降):

```
リンゴ(軽減 8%)  ¥100 × 2 = ¥200  税: floor(200 × 0.08) = ¥16
ノート(標準 10%) ¥300 × 1 = ¥300  税: floor(300 × 0.10) = ¥30
─────────────────
小計   ¥500   (= subtotal)
消費税 ¥46    (= tax_amount = 16 + 30)
合計   ¥546   (= total_price = subtotal + tax_amount)
```

### 金額・数量の上限(1α から)

- 金額: `@DecimalMax("9999999999.99")`(`NUMERIC(12,2)` の物理上限)
- 数量: `@Max(99)`(カート行単位)

### 保守性(1α から)

- レイヤードアーキテクチャ(domain / application / presentation / infrastructure)に従う
- springdoc-openapi で Swagger UI を自動生成
- `config/` はファイル単位で分割(`SecurityConfig` / `CorsConfig` / `BeanConfig` / `ClockConfig` を 1α、`TaxConfig` を 1β)

---

## 5. API 設計(概要)

ベース URL: `http://localhost:8080`

カート系エンドポイントは全て `X-Cart-Id: {uuid}` リクエストヘッダーで `cartId` を渡す。フロントは localStorage の値をヘッダーにセットする。

| Method | Path | フェーズ | 説明 | レスポンス |
|--------|------|---------|------|-----------|
| GET | `/api/products` | 1α | 商品一覧(ページネーション) | `Page<ProductResponse>` ★ |
| GET | `/api/products/{id}` | 1α | 商品詳細取得 | `ProductResponse` |
| POST | `/api/products` | 1α | 商品登録(管理者用) | `ProductResponse` |
| GET | `/api/cart` | 1α | カート内容取得 | `CartResponse` |
| GET | `/api/cart`(グループ化) | 1β | `ACTIVE` / `SAVED_FOR_LATER` でグループ化して返す | `CartResponse` |
| POST | `/api/cart/items` | 1α | カートに商品追加(常に `status=ACTIVE`) | `CartResponse` |
| POST | `/api/cart/items`(マージ) | 1β | 同一 `(productId, status)` が既存なら `quantity` をマージ | `CartResponse` |
| PATCH | `/api/cart/items/{id}` | 1α | カート商品の数量変更 | `CartResponse` |
| PATCH | `/api/cart/items/{id}`(status) | 1β | ステータス変更(ACTIVE ⇄ SAVED_FOR_LATER)を追加 | `CartResponse` |
| DELETE | `/api/cart/items/{id}` | 1α | カートから商品削除 | 204 No Content |
| POST | `/api/orders` | 1α | 注文確定(モック。単価スナップショットのみ) | `OrderResponse` |
| POST | `/api/orders`(税 / 履歴 / 採番) | 1β | `ACTIVE` 商品のみ対象。税率スナップショット・注文番号採番・状態履歴記録を追加 | `OrderResponse` |

★ 商品一覧は `Page<T>` でページネーション対応。クエリパラメータ: `?page=0&size=20&sort=name,asc`

詳細な request/response スキーマは別紙 [`design/02-api-spec.md`](design/02-api-spec.md) で規定する。

### Phase 2 で追加予定

- `GET /api/orders` — 注文一覧取得(認証連携前提)
- `GET /api/orders/{id}` — 注文詳細取得
- `PATCH /api/orders/{id}/cancel` — 注文キャンセル
- `POST /api/orders/{id}/refund` — 返金処理
- `/api/auth/*` — 認証関連(詳細は [`phase2-plan.md`](phase2-plan.md) §2-4)

---

## 6. バリデーション方針

`spring-boot-starter-validation` を使用。`@Valid` を Controller の `@RequestBody` に付与し、違反時は `@RestControllerAdvice` で 400 Bad Request を返す。

| エンドポイント | フィールド | フェーズ | ルール |
|---------------|-----------|---------|--------|
| POST `/api/products` | `name` | 1α | `@NotBlank` — 商品名は必須 |
| POST `/api/products` | `price` | 1α | `@DecimalMin("0.01")` + `@DecimalMax("9999999999.99")` — 税抜価格 |
| POST `/api/products` | `stock` | 1α | `@Min(0)` — 0 以上の整数 |
| POST `/api/products` | `taxCategory` | 1β | `@NotNull` — `STANDARD` または `REDUCED` |
| POST `/api/cart/items` | `productId` | 1α | `@NotNull` — 商品 ID 必須 |
| POST `/api/cart/items` | `quantity` | 1α | `@Min(1)` + `@Max(99)` — 1 以上 99 以下。在庫数以下かは Service 層でチェック |
| PATCH `/api/cart/items/{id}` | `quantity` | 1α | `@Min(1)` + `@Max(99)` |
| PATCH `/api/cart/items/{id}` | `status` | 1β | `ACTIVE` または `SAVED_FOR_LATER` |

---

## 7. エラーレスポンス形式

全エラーは `@RestControllerAdvice` で統一フォーマットに変換して返す。フロントはこの形式を前提に型定義・エラーハンドリングを実装する。

### 7-1. 共通フォーマット(1α)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "リクエストに誤りがあります",
  "timestamp": "2026-04-15T10:00:00Z"
}
```

| status | error | フェーズ | 使用場面 |
|--------|-------|---------|---------|
| 400 | Bad Request | 1α | バリデーションエラー(`@Valid` 違反)。`message` に概要 |
| 403 | Forbidden | 1α | `CartItemForbiddenException`(他人カートの操作) |
| 404 | Not Found | 1α | `ProductNotFoundException` / `CartItemNotFoundException` 等 |
| 409 | Conflict | 1α | `OutOfStockException`(在庫不足)/ `EmptyCartException`(空カート)/ `CartItemQuantityExceededException`(マージ後 `@Max(99)` 超過) |
| 409 | Conflict | 1β | `IllegalStateTransitionException`(注文状態遷移違反)/ SAVED_FOR_LATER → ACTIVE 復帰時の在庫不足 |
| 500 | Internal Server Error | 1α | 予期しないサーバーエラー |

### 7-2. `errors[]` フィールド(1β で追加)

Bean Validation(`@Valid`)で複数フィールドが同時に違反する可能性があり、フロントはフィールド単位でエラーを表示するため、**1β から 400 のときのみ `errors[]` を付与** する。

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "リクエストに誤りがあります",
  "timestamp": "2026-04-15T10:00:00Z",
  "errors": [
    { "field": "name",  "message": "商品名は必須です" },
    { "field": "price", "message": "0.01 以上で入力してください" }
  ]
}
```

| フィールド | 型 | 備考 |
|-----------|-----|------|
| `errors[]` | 配列 | **400 のときのみ付与**。404 / 409 / 500 では省略(フィールド概念がないため) |
| `errors[].field` | 文字列 | 違反フィールドのパス。ネストは dot / bracket 記法(例: `items[0].quantity`)。Jackson のデフォルトパスと一致 |
| `errors[].message` | 文字列 | フィールド単位のエラーメッセージ(日本語) |
| `message`(外側) | 文字列 | 概要(例: `"リクエストに誤りがあります"`)。フロントはフォーム全体の警告文などに利用 |

フロント型定義(1β 以降):

```typescript
interface ErrorResponse {
  status: number;
  error: string;
  message: string;
  timestamp: string;
  errors?: FieldError[];  // 400 のみ(1β 以降)
}
interface FieldError {
  field: string;
  message: string;
}
```

---

## 8. データモデル

Phase 1 の設計方針: **JPA エンティティ = ドメインモデル**として同一クラスで扱う。規模拡大時に分離を検討する。

物理設計の詳細(DDL・インデックス・制約)は [`design/01-database.md`](design/01-database.md)、Aggregate / ファクトリの設計は [`design/05-domain-order.md`](design/05-domain-order.md) を参照。

### 8-1. Product

| フィールド | 型 | フェーズ | 制約・備考 |
|-----------|-----|---------|-----------|
| `id` | `UUID` | 1α | PK |
| `name` | `String` | 1α | `@NotBlank` |
| `description` | `String` | 1α | 任意 |
| `price` | `BigDecimal` | 1α | `@DecimalMin("0.01")`。**税抜** |
| `stock` | `Integer` | **1α のみ** | `@Min(0)`。1β で `ProductStock` に分離 |
| `taxCategory` | `TaxCategory` (Enum) | 1β | `STANDARD` / `REDUCED`。デフォルト `STANDARD` |
| `imageUrl` | `String` | 1α | 任意。画像ファイル本体アップロードは Phase 2 |
| `createdAt` / `updatedAt` | `OffsetDateTime` | 1α | — |

### 8-2. ProductStock(1β 新設)

在庫情報を Product から分離。変更頻度・責務が異なるため独立エンティティとする。

| フィールド | 型 | 制約・備考 |
|-----------|-----|-----------|
| `productId` | `UUID` | PK 兼 FK → `Product.id`(1 対 1) |
| `quantity` | `Integer` | `@Min(0)` |
| `updatedAt` | `OffsetDateTime` | 在庫変更時刻 |

**背景**: Phase 2 で楽観ロック(`@Version`)を在庫のみに付与することで、商品情報の更新と在庫減算を干渉させない。また入出庫履歴(`stock_movements`)の拡張経路を確保する。

### 8-3. Cart / CartItem

| エンティティ | フィールド | フェーズ | 備考 |
|-------------|-----------|---------|------|
| `Cart` | `cartId: UUID` | 1α | フロント `localStorage` で `crypto.randomUUID()` 生成。サーバーはそのまま受け取る |
| `CartItem` | `id: UUID` | 1α | PK |
| `CartItem` | `cartId: UUID` | 1α | FK → `Cart` |
| `CartItem` | `productId: UUID` | 1α | FK → `Product` |
| `CartItem` | `quantity: Integer` | 1α | カート内でのこの商品の個数。`@Min(1)` + `@Max(99)` |
| `CartItem` | `status: CartItemStatus` | 1β | `ACTIVE` / `SAVED_FOR_LATER`。デフォルト `ACTIVE` |

**カート識別の方針**: 認証が存在しない Phase 1 では、フロント側で `crypto.randomUUID()` を用いて UUID を生成し `localStorage` に保存する。サーバー側セッション管理は不要。

**「あとで買う」(1β)**: `status` を `SAVED_FOR_LATER` にすることで、カート合計・注文確定の対象から外す。注文確定時は `ACTIVE` の行のみを処理する。

**重複マージ規則**: `POST /api/cart/items` で同一 `(cart_id, product_id, status)` が既存の場合、新規行を INSERT せず既存行の `quantity` に加算する。加算後に `@Max(99)` 超なら `CartItemQuantityExceededException`(409)、在庫上限超なら `OutOfStockException`(409)。

> **1α は Service 層でマージ**する([`design/04-domain-cart-product.md`](design/04-domain-cart-product.md) §3.2)。DB の一意制約は張らず、アプリ層で `(cart_id, product_id)` を検索して加算する。1β で `status` 列追加と同時に `UNIQUE (cart_id, product_id, status)` を導入して物理担保する。マージキーも 1α は `(cart_id, product_id)`、1β は `(cart_id, product_id, status=ACTIVE)` に拡張する。

**`SAVED_FOR_LATER → ACTIVE` 復帰時の在庫チェック(1β)**: 復帰時点での在庫が `quantity` を下回る場合は 409 Conflict を返し、遷移を行わない(ユーザー側で数量を減らして再試行する UX)。

### 8-4. Order / OrderItem

| エンティティ | フィールド | フェーズ | 備考 |
|-------------|-----------|---------|------|
| `Order` | `id: UUID` | 1α | PK |
| `Order` | `status: OrderStatus` (Enum) | 1α | 1α は `CONFIRMED` のみ実使用。1β で `PENDING` / `CANCELLED` / `REFUNDED` を Enum に追加(実使用は Phase 2)。現在値キャッシュ(真実源は `OrderStatusHistory`) |
| `Order` | `orderNumber: String` | 1β | 表示用注文番号 `yyyyMMdd-NNNN` |
| `Order` | `totalPrice: BigDecimal` | 1α | 1α は単価 × 数量の合計(税なし)。1β で税込合計 = `subtotal + taxAmount` |
| `Order` | `subtotal: BigDecimal` | 1β | 税抜合計 = Σ(`unit_price` × `quantity`) |
| `Order` | `taxAmount: BigDecimal` | 1β | 税額合計 = Σ(`OrderItem.taxAmount`) |
| `Order` | `createdAt: OffsetDateTime` | 1α | — |
| `OrderItem` | `id: UUID` | 1α | PK |
| `OrderItem` | `productName: String` | 1α | 商品名スナップショット |
| `OrderItem` | `unitPrice: BigDecimal` | 1α | 1α は単価スナップショット。1β では税抜単価であることを明確化 |
| `OrderItem` | `quantity: Integer` | 1α | この明細で注文した個数 |
| `OrderItem` | `taxCategory: TaxCategory` | 1β | 確定時点の税区分 |
| `OrderItem` | `taxRate: BigDecimal` | 1β | 確定時点の税率(例 `0.0800`) |
| `OrderItem` | `taxAmount: BigDecimal` | 1β | この明細の税額(行単位切り捨て) |

**OrderStatus Enum**:
- 1α で実使用: `CONFIRMED` のみ
- 1β で Enum に追加: `PENDING`(Phase 2 決済連携用)/ `CANCELLED` / `REFUNDED`(Phase 2 キャンセル・返金機能用)

### 8-4-1. OrderResponse スキーマ

注文完了画面はフロントが `POST /api/orders` のレスポンスをそのまま描画する方針のため、**レシート生成に必要なフィールドをすべて含める**。行単位の小計・税額もサーバー側で計算して返し、フロントでの再計算を禁止する(丸め規則のズレによる事故防止)。

#### 1α の最小レスポンス

```json
{
  "orderId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
  "status": "CONFIRMED",
  "confirmedAt": "2026-04-15T10:00:00+09:00",
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

#### 1β の拡張レスポンス(税率スナップショット・注文番号あり)

```json
{
  "orderId": "0b3fa4f8-3d1b-4a3d-b8a3-9a7e1c7e2f5a",
  "orderNumber": "20260415-0001",
  "status": "CONFIRMED",
  "confirmedAt": "2026-04-15T10:00:00+09:00",
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
    }
  ],
  "subtotal": "500",
  "taxAmount": "46",
  "totalPrice": "546",
  "currency": "JPY"
}
```

#### フィールド仕様

| フィールド | 型 | フェーズ | 備考 |
|-----------|-----|---------|------|
| `orderId` | UUID 文字列 | 1α | `Order.id`。問合せ・将来の詳細取得 API 用 |
| `orderNumber` | 文字列 | 1β | 表示用注文番号。採番規則: `yyyyMMdd-NNNN`(例 `20260415-0001`)。`NNNN` は日次連番・4 桁ゼロ埋め |
| `status` | `OrderStatus` | 1α | Phase 1 では常に `CONFIRMED` |
| `confirmedAt` | ISO-8601 文字列 | 1α | `Order.createdAt` をそのまま返す。タイムゾーン付き(`+09:00`) |
| `items[]` | 配列 | 1α | 注文明細。空配列は不可 |
| `items[].unitPrice` | **文字列** | 1α | 単価スナップショット。`BigDecimal` を **文字列で送信**(JSON の数値型に落とすと double 精度で崩れるため) |
| `items[].taxRate` | **文字列** | 1β | 確定時点の税率(例 `"0.08"`) |
| `items[].lineSubtotal` | **文字列** | 1β | `unitPrice × quantity`。サーバー側で計算 |
| `items[].taxAmount` | **文字列** | 1β | `floor(lineSubtotal × taxRate)`。サーバー側で計算 |
| `items[].lineTotal` | **文字列** | 1α(税なし)/ 1β(税込) | 1α では `unitPrice × quantity`、1β では `lineSubtotal + taxAmount` |
| `subtotal` | **文字列** | 1β | 税抜合計。サーバー側で計算 |
| `taxAmount`(全体) | **文字列** | 1β | 税額合計。サーバー側で計算 |
| `totalPrice` | **文字列** | 1α(税なし) / 1β(税込) | 注文全体の合計。すべてサーバー側で計算して返す |
| `currency` | 文字列 | 1α | Phase 1 は `"JPY"` 固定。Phase 2 の多通貨対応に備えて常にフィールドとして含める |

#### 採番規則の実装方針(1β)

- 1β は単純に **`orders` テーブルに `order_number` 列(UNIQUE 制約)を追加**し、サービス層で日次連番を採番
- 採番衝突時は `DataIntegrityViolationException` → 1 回リトライ
- 厳密性を求めるなら `order_sequences(date, next_seq)` テーブルで排他的に採番するが、Phase 1 は上記の簡易方式で可

#### 含めないフィールド(Phase 1 全体)

- 配送先住所・支払方法 — モック注文のため未スコープ
- ユーザー情報 — 認証が Phase 2 のため

### 8-5. OrderStatusHistory(1β 新設)

注文の状態遷移履歴。監査ログ兼、キャンセル・返金時の払い戻し計算の根拠。

| フィールド | 型 | 備考 |
|-----------|-----|------|
| `id` | `UUID` | PK |
| `orderId` | `UUID` | FK → `Order` |
| `fromStatus` | `OrderStatus?` | 初回は `null` |
| `toStatus` | `OrderStatus` | — |
| `reason` | `String?` | キャンセル理由など |
| `changedAt` | `OffsetDateTime` | — |
| `changedBy` | `String?` | Phase 2 認証導入時に埋める |

**許容する遷移**(Service 層で検証):

```
(null) → CONFIRMED                   -- Phase 1β で唯一発生
(null) → PENDING → CONFIRMED         -- Phase 2(決済連携)
CONFIRMED → CANCELLED → REFUNDED     -- Phase 2(キャンセル → 返金)
```

**運用ルール**: `Order.status` を変更する Service は、**同一トランザクションで履歴行を 1 件 INSERT** する。これを破ると真実源が壊れるため、**Aggregate メソッド方式で強制**する(詳細は [`design/05-domain-order.md`](design/05-domain-order.md) §2.1)。

### 8-6. 注文確定のライフサイクル

#### 1α のフロー

1. Service 層で Cart の全 `CartItem` を走査し在庫チェック(不足時は `OutOfStockException`)
2. `Product.stock` を減算して保存(楽観ロックは Phase 2 以降)
3. `CartItem` の内容(`productId` / `quantity` / 現時点の `price` / `productName`)を `OrderItem` にコピー(スナップショット)
4. Cart をクリア
5. `Order` を `status=CONFIRMED` で保存
6. `OrderResponse` を返す

#### 1β のフロー(拡張)

1. Service 層で Cart の **`status=ACTIVE`** CartItem のみを走査し在庫チェック(空なら `EmptyCartException`)
2. `ProductStock.quantity` を減算して保存
3. CartItem の内容に加え、**税区分から導出した `taxRate`・行単位税額** を `OrderItem` にスナップショット
4. Cart の `ACTIVE` 行をクリア(`SAVED_FOR_LATER` は残す)
5. **注文番号を採番**(`OrderNumberGenerator` / UNIQUE 衝突時は 1 回リトライ)
6. `Order` を `status=CONFIRMED` で保存(`subtotal` / `taxAmount` / `totalPrice` を計算済みでセット)
7. **`OrderStatusHistory` に `(null → CONFIRMED)` を 1 件 INSERT**
8. `OrderResponse` を返す

---

## 9. アーキテクチャ設計

### 9-1. バックエンド:レイヤー構成

層優先のレイヤードアーキテクチャ。DDD 戦術パターン(Aggregate / Factory / 不変条件)は `domain/model/` で取り入れるが、依存方向の厳密制御(ヘキサゴナル / クリーンアーキテクチャ)や モジュール分離(モジュラーモノリス)は採用しない。学習目的 3 本柱に焦点を絞るための判断。

| パス | 役割 | 備考 |
|------|------|------|
| `domain/model/` | Entity (JPA) | JPA アノテーション付きのドメインモデル。Phase 1 は Entity = ドメインモデル |
| `domain/repository/` | Repository I/F | Spring Data JPA の interface のみ |
| `domain/exception/` | ドメイン例外 | `ProductNotFoundException` / `CartItemNotFoundException` / `CartItemForbiddenException` / `OutOfStockException` / `EmptyCartException` / `CartItemQuantityExceededException`(1α)、`IllegalStateTransitionException`(1β)|
| `application/service/` | UseCase ロジック | トランザクション境界。ドメインモデルを操作し DTO を返す |
| `application/dto/` | Input / Output DTO | 外部公開データ形式。`record` を採用 |
| `application/mapper/` | Entity ↔ DTO 変換 | 手動変換メソッドを集約(Phase 1)。Phase 2 で MapStruct に置き換え |
| `presentation/controller/` | REST Controller | `@RestController`。リクエスト受付と DTO の返却のみ |
| `presentation/advice/` | 例外ハンドラ | `@RestControllerAdvice`。例外 → HTTP レスポンス変換 |
| `infrastructure/config/` | 設定クラス | `SecurityConfig` / `CorsConfig` / `BeanConfig` / `ClockConfig` を 1α、`TaxConfig` を 1β にファイル分割 |
| `infrastructure/persistence/` | Repository 実装 | domain の interface を実装(Phase 1 ではほぼ空) |

### 9-2. フロントエンド:ディレクトリ構成

```
src/api/         axios クライアント・BaseURL・interceptor
src/types/       バックエンド DTO に対応する TypeScript 型
src/components/  汎用 UI パーツ(ドメイン非依存)
src/features/    商品 / カート をドメイン別に分割。各 feature が components / hooks / api を内包
src/hooks/       feature 横断の共通カスタム hooks
src/pages/       React Router v7 のページコンポーネント
```

### 9-3. 状態管理方針

サーバー状態(商品・カート)は TanStack Query で管理。グローバル UI 状態が必要になった場合は Zustand を追加する。

- 商品データ → TanStack Query(キャッシュ・再フェッチ)
- カートデータ → TanStack Query(`cartId` を `localStorage` から取得してリクエスト)
- グローバル UI 状態 → 必要になった時点で Zustand を導入

---

## 10. テスト方針

Phase 1 は TDD(赤 → 緑 → リファクタ)で実装する。

| 種別 | 対象層 | 方針 |
|------|--------|------|
| Unit Test | `application/service/` / `domain/model/`(Aggregate) | Mockito でリポジトリをモック。ビジネスロジック・例外を検証 |
| `@WebMvcTest` | `presentation/controller/` | MockMvc で HTTP リクエスト/レスポンスを検証。Service はモック |
| `@DataJpaTest` | `infrastructure/persistence/` | Testcontainers PostgreSQL でクエリ・CRUD 動作を検証(H2 は使わない) |
| Integration Test | 任意(Phase 2 以降) | `@SpringBootTest` で全層を通した結合テスト |

**Phase 1α の目標**: Service 層の Unit Test と Controller 層の `@WebMvcTest` を最優先で実装する。`@DataJpaTest` はカスタムクエリを書いた場合に追加する。

**Phase 1β で追加する重点テスト**:

- 注文確定時の税額計算(標準 / 軽減税率混在)
- 行単位切り捨ての端数処理
- `SAVED_FOR_LATER` 商品が注文確定対象から除外されること
- `OrderStatusHistory` が `Order.status` 変更と同一トランザクションで INSERT されること
- `OrderNumberGenerator` の採番と衝突時の 1 回リトライ
- CartItem 重複マージと `@Max(99)` / 在庫上限での 409

---

## 11. 技術スタック

### バックエンド

- Java 21 / Spring Boot 3.x
- Spring Data JPA(Hibernate)
- `spring-boot-starter-validation`
- Spring Security(Phase 1 は CORS 設定のみ)
- Gradle(Kotlin DSL)
- PostgreSQL 16
- Flyway(DB マイグレーション)
- springdoc-openapi(Swagger UI)
- Testcontainers(テスト用 PostgreSQL)

### フロントエンド

- React 19 / TypeScript / Vite
- TanStack Query(サーバー状態管理)
- React Router v7
- axios(HTTP クライアント)
- **Tailwind CSS**(学習の焦点を API 連携に置くため、スタイリングはユーティリティファーストで最小化)
- Zustand(グローバル UI 状態 / 必要時に追加)

### 開発環境

- macOS(M4 Pro)/ IntelliJ IDEA / VS Code
- PostgreSQL: Homebrew でローカル起動
- Docker: テスト用に Testcontainers で起動(開発 DB はネイティブ)。Phase 2 で Docker Compose を導入予定

---

## 12. フェーズ計画

### Phase 1α(現フェーズ着手対象)— コア機能

**学習ゴール**: Spring Boot REST API 設計 / JPA 永続化 / レイヤードアーキテクチャの 3 本柱を最短で完走する。

- [ ] Spring Initializr でプロジェクト生成(backend / frontend モノレポ)
- [ ] Flyway による DB マイグレーション(`V1__init.sql` — Product + Cart + CartItem + Order + OrderItem の最小 DDL)
- [ ] Product CRUD API(GET / GET{id} / POST)+ ページネーション
- [ ] バリデーション(`@Valid`)+ 例外ハンドリング(`@RestControllerAdvice` — `ProductNotFound` / `CartItemNotFound` / `CartItemForbidden` / `OutOfStock` / `EmptyCart` / `CartItemQuantityExceeded` / validation)
- [ ] Cart API(取得・追加・変更・削除 / `ACTIVE` のみ)
- [ ] 注文確定 API(在庫減算・単価スナップショット・`CONFIRMED` 固定)
- [ ] React フロントエンド(商品一覧・詳細・カート画面)
- [ ] `cartId` を `localStorage` で管理
- [ ] Service 層 Unit Test + Controller 層 `@WebMvcTest`
- [ ] エラーレスポンス形式(`errors[]` なし・`message` 単一)

**完了条件**: 上記を満たし、ブラウザから「商品一覧 → カート投入 → 注文確定」が通しで動くこと。

### Phase 1β — 応用拡張

α の骨格に EC 実務要素を積む。各項目が独立した TDD の赤 → 緑 → リファクタ 1 サイクルに収まるよう順序化。各ステップごとに Flyway マイグレーションを追加する。

1. [ ] `ProductStock` テーブル分離(`V2__split_product_stock.sql`)— JPA 1:1 関連と Aggregate 境界
2. [ ] 税区分・税率 + `@ConfigurationProperties`(`V3__add_tax.sql`)— `BigDecimal` 精度と設定バインディング
3. [ ] `OrderStatusHistory` + Aggregate メソッド方式(`V4__order_status_history.sql`)— `cascade` と不変条件
4. [ ] `SAVED_FOR_LATER` + 重複マージ + 2 段階在庫チェック(`V5__cart_item_status.sql`)— 複合一意制約とドメインロジック
5. [ ] `orderNumber` 採番(`V6__order_number.sql`)— `Clock` DI と UNIQUE リトライ
6. [ ] エラーレスポンスに `errors[]` フィールド追加 — `@RestControllerAdvice` の応用

### Phase 2(予定)

以下は [`phase2-plan.md`](phase2-plan.md) で詳細設計する。

- JWT 認証の導入 / ユーザー登録・ログイン画面
- 商品の PUT / DELETE エンドポイント
- 楽観的ロック(`@Version`)による在庫競合対策(`ProductStock` に付与)
- 注文キャンセル・返金機能(`CANCELLED` / `REFUNDED` への遷移)
- 税率マスタテーブル(`tax_rates`)化
- 在庫入出庫履歴(`stock_movements`)
- `application/mapper/` を MapStruct に置き換え
- Docker Compose によるコンテナ化 / CI/CD

---

## 付録. 改版履歴

| バージョン | 日付 | 主な変更 |
|-----------|------|---------|
| v0.3 | — | 初版 |
| v0.4 | 2026-04 | `cartId` ヘッダー・`PENDING` 用途・エラーレスポンス形式を明記 |
| v0.5 | 2026-04 | 在庫テーブル分離 / 「あとで買う」追加 / 消費税対応(税抜保持・軽減税率・スナップショット) / 注文ステータス拡張(`CANCELLED` / `REFUNDED`) / 注文状態遷移履歴の導入 |
| v0.5.1 | 2026-04 | レビュー反映: 在庫チェック 2 段階明記 / 「あとで買う」復帰時の 409 拒否 / CartItem 重複マージ規則 / 通貨 JPY 固定 / CSRF 無効化 / 注文完了画面の実現方法 / 金額・数量の上限 / 税額計算例 |
| v0.5.2 | 2026-04-15 | 実装前詰め: 税率設定仕様(`@ConfigurationProperties` / YAML 文字列保持 / `TaxProperties#rateOf`)/ 在庫競合方針(Phase 1 は最後勝ち・Phase 2 で `@Version`)/ Phase 2 認証方針確定 / `OrderResponse` スキーマ確定 / エラーレスポンスに `errors[]` を追加 |
| **v0.6** | **2026-04-18** | **学習目的(Spring Boot / JPA / レイヤード)の再確認に基づき Phase 1 を α(コア)/ β(応用拡張)に分割。各機能・API・データフィールドにフェーズタグを付与。Phase 2 認証方針を別紙 [`phase2-plan.md`](phase2-plan.md) に分離。フロント CSS を Tailwind に明記。テスト用 DB を Testcontainers PostgreSQL に明記** |
| **v0.6.1** | **2026-04-18** | **設計書 04(Cart / Product Aggregate)作成に伴う調整:1α の重複マージは Service 層実装に確定 / `EmptyCartException` / `CartItemQuantityExceededException` を 1α に昇格 / 1β 新規追加の例外は `IllegalStateTransitionException` のみ** |
