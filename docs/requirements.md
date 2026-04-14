# spring-bazaar 要件定義書

| 項目 | 値 |
|------|-----|
| プロジェクト名 | spring-bazaar |
| バージョン | **0.5.1** |
| 作成日 | 2026年4月 |
| ステータス | Draft |
| スコープ | 商品管理・カート機能(認証は Phase 2) |
| 改版内容(v0.4 → v0.5) | 在庫テーブル分離 / 「あとで買う」追加 / 消費税対応(税抜保持・軽減税率・スナップショット) / 注文ステータス拡張(CANCELLED・REFUNDED) / 注文状態遷移履歴の導入 |
| 改版内容(v0.5 → v0.5.1) | 在庫チェック2段階明記 / 「あとで買う」復帰時の在庫不足を 409 拒否 / CartItem 重複マージ規則 / 通貨 JPY 固定 / CSRF 無効化方針 / 注文完了画面の実現方法 / 金額・数量の上限 / 税額計算例 |

> v0.4 原本は [`archive/spring-bazaar-requirements-v4.docx`](archive/spring-bazaar-requirements-v4.docx) を参照。

---

## 1. プロジェクト概要

spring-bazaar は Java(Spring Boot)と React(TypeScript)を用いたフルスタック学習プロジェクトです。EC サイトのコアドメイン(商品閲覧・カート・モック注文)を通じて、Spring Boot の REST API 設計・JPA によるデータ永続化・レイヤードアーキテクチャの実践的な理解を目的とします。

---

## 2. スコープ

### 2-1. 対象(In Scope)

- 商品一覧・詳細の閲覧(ページネーション対応)
- カートへの商品追加・変更・削除
- **「あとで買う」(カート内商品のステータス切替: `ACTIVE` / `SAVED_FOR_LATER`)** ★v0.5
- 注文確定(在庫減算あり・決済はモック)
- 管理者による商品登録(POST のみ)

### 2-2. 対象外(Out of Scope)

- ユーザー認証・JWT 管理 → Phase 2
- 商品の更新(PUT)・削除(DELETE) → Phase 2
- 決済処理の実連携 → Phase 2
- **注文のキャンセル・返金 → Phase 2(データモデルは v0.5 で先取り)** ★v0.5
- 商品画像のファイルアップロード → Phase 2
- **税率改定マスタ管理(`tax_rates` テーブル化) → Phase 2(Phase 1 は定数運用)** ★v0.5

---

## 3. 機能要件

| 機能 | 説明 | 優先度 |
|------|------|--------|
| 商品一覧表示 | 商品をカード形式で一覧表示。ページネーション対応(`Page<T>`) | High |
| 商品詳細表示 | 商品名・**税抜/税込価格**・説明・在庫数を表示 ★v0.5 | High |
| カートへの追加 | 商品詳細からカートに追加。数量指定。**在庫チェックは2段階(カート追加時・注文確定時)** ★v0.5.1。同一商品・同一ステータスの追加は既存行の `quantity` にマージ ★v0.5.1 | High |
| カート確認 | カート内商品の一覧・合計金額(税抜/税込)を表示 ★v0.5 | High |
| カートの変更 | 数量変更・商品削除が可能。**数量変更時も在庫チェック** ★v0.5.1 | Medium |
| **「あとで買う」への移動 / 復帰** | カート商品を `SAVED_FOR_LATER` に移動、または `ACTIVE` に戻す ★v0.5。**復帰時に在庫不足なら 409 Conflict で拒否(ユーザー側で数量調整)** ★v0.5.1 | Medium |
| 注文確定(モック) | カート内 `ACTIVE` 商品のみを `Order` にスナップショット保存・在庫を減算。**税率も確定時点で固定化** ★v0.5。**完了画面は `POST /api/orders` レスポンスの `OrderResponse` をフロントがそのまま表示**(注文一覧 API は Phase 2) ★v0.5.1 | Medium |
| 管理者:商品登録 | 商品名・税抜価格・在庫数・画像 URL・**税区分(標準/軽減)** を登録(POST のみ) ★v0.5 | Low |

---

## 4. 非機能要件

### パフォーマンス

- 商品一覧 API のレスポンスタイム: 500ms 以内(ローカル環境)

### セキュリティ

- CORS 設定: フロントエンド(`localhost:5173`)のみ許可
- SQL インジェクション対策: JPA パラメータバインディングを使用
- **CSRF 対策**: Phase 1 は認証なしかつ SPA + REST 構成のため `CsrfConfigurer.disable()` で**無効化**。Phase 2 で JWT を `Authorization` ヘッダー方式で送る設計とし、CSRF は引き続き無効のままとする ★v0.5.1

### 通貨 ★v0.5.1

- Phase 1 は **日本円 (JPY) 固定**
- 金額は `BigDecimal` / `NUMERIC(12, 2)` で保持するが、**表示は整数円**(UI 層で `toFixed(0)` 相当)
- 多通貨対応は Phase 2 以降の検討事項

### バリデーション

- `spring-boot-starter-validation` を使用(`@Valid` / `@NotBlank` / `@Min` 等)
- DTO に Bean Validation アノテーションを付与し Controller 層で `@Valid` を適用
- 在庫チェック(`quantity <= stock`)は Service 層でドメインロジックとして実装

### 消費税対応 ★v0.5

- **商品価格は税抜保持**(`products.price`)。税込は表示時に計算
- **軽減税率対応**: 商品ごとに税区分を保持(`STANDARD` / `REDUCED`)
- **注文確定時に税率をスナップショット**: `order_items.tax_rate` として固定化し、税率改定を跨ぐキャンセル・返金でも払い戻し額が不変
- 税率は Phase 1 では `application.yml` で定数管理(Phase 2 でマスタテーブル化)
- 税額端数処理: **行単位で切り捨て**(消費税法の一般的運用)

**計算例** ★v0.5.1:

```
リンゴ(軽減 8%)  ¥100 × 2 = ¥200  税: floor(200 × 0.08) = ¥16
ノート(標準 10%) ¥300 × 1 = ¥300  税: floor(300 × 0.10) = ¥30
─────────────────
小計   ¥500   (= subtotal)
消費税 ¥46    (= tax_amount = 16 + 30)
合計   ¥546   (= total_price = subtotal + tax_amount)
```

### 保守性

- レイヤードアーキテクチャ(domain / application / presentation / infrastructure)に従う
- springdoc-openapi で Swagger UI を自動生成
- `config/` はファイル単位で分割(SecurityConfig / CorsConfig / BeanConfig)

---

## 5. API 設計(概要)

ベース URL: `http://localhost:8080`

カート系エンドポイントは全て `X-Cart-Id: {uuid}` リクエストヘッダーで `cartId` を渡す。フロントは localStorage の値をヘッダーにセットする。

| Method | Path | 説明 | レスポンス |
|--------|------|------|-----------|
| GET | `/api/products` | 商品一覧(ページネーション) | `Page<ProductResponse>` ★ |
| GET | `/api/products/{id}` | 商品詳細取得 | `ProductResponse` |
| POST | `/api/products` | 商品登録(管理者用) | `ProductResponse` |
| GET | `/api/cart` | カート内容取得(`ACTIVE` / `SAVED_FOR_LATER` をグループ化して返す) ★v0.5 | `CartResponse` |
| POST | `/api/cart/items` | カートに商品追加(既定 `status=ACTIVE`)。**同一 `(productId, status)` が既存なら `quantity` をマージ** ★v0.5.1 | `CartResponse` |
| PATCH | `/api/cart/items/{id}` | カート商品の数量変更・**ステータス変更**(ACTIVE ⇄ SAVED_FOR_LATER) ★v0.5 | `CartResponse` |
| DELETE | `/api/cart/items/{id}` | カートから商品削除 | 204 No Content |
| POST | `/api/orders` | 注文確定(モック)。`ACTIVE` 商品のみが対象 ★v0.5 | `OrderResponse` |

★ 商品一覧は `Page<T>` でページネーション対応。クエリパラメータ: `?page=0&size=20&sort=name,asc`

詳細な request/response スキーマは別紙 `design/02-api-spec.md` で規定する。

### Phase 2 で追加予定

- `GET /api/orders` — 注文一覧取得(認証連携前提) ★v0.5.1
- `GET /api/orders/{id}` — 注文詳細取得 ★v0.5.1
- `PATCH /api/orders/{id}/cancel` — 注文キャンセル
- `POST /api/orders/{id}/refund` — 返金処理

---

## 6. バリデーション方針

`spring-boot-starter-validation` を使用。`@Valid` を Controller の `@RequestBody` に付与し、違反時は `@RestControllerAdvice` で 400 Bad Request を返す。

| エンドポイント | フィールド | ルール |
|---------------|-----------|--------|
| POST `/api/products` | `name` | `@NotBlank` — 商品名は必須 |
| POST `/api/products` | `price` | `@DecimalMin("0.01")` + **`@DecimalMax("9999999999.99")`** ★v0.5.1 — 税抜価格 |
| POST `/api/products` | `stock` | `@Min(0)` — 0以上の整数 |
| POST `/api/products` | `taxCategory` ★v0.5 | `@NotNull` — `STANDARD` または `REDUCED` |
| POST `/api/cart/items` | `productId` | `@NotNull` — 商品ID必須 |
| POST `/api/cart/items` | `quantity` | `@Min(1)` + **`@Max(99)`** ★v0.5.1 — 1以上99以下。在庫数以下かは Service 層でチェック |
| PATCH `/api/cart/items/{id}` | `quantity` | `@Min(1)` + **`@Max(99)`** ★v0.5.1 |
| PATCH `/api/cart/items/{id}` | `status` ★v0.5 | `ACTIVE` または `SAVED_FOR_LATER` |

---

## 7. エラーレスポンス形式

全エラーは `@RestControllerAdvice` で統一フォーマットに変換して返す。フロントはこの形式を前提に型定義・エラーハンドリングを実装する。

### レスポンスボディ(共通フォーマット)

```json
{
  "status": 400,
  "error": "Bad Request",
  "message": "name は必須です",
  "timestamp": "2026-04-13T10:00:00Z"
}
```

| status | error | 使用場面 |
|--------|-------|---------|
| 400 | Bad Request | バリデーションエラー(`@Valid` 違反)。`message` にフィールドエラー詳細 |
| 404 | Not Found | `ProductNotFoundException` など。対象リソースが存在しない |
| 409 | Conflict | `OutOfStockException`(在庫不足 — カート追加・数量変更・**SAVED_FOR_LATER → ACTIVE 復帰** ★v0.5.1・注文確定のいずれでも発生)。**`IllegalStateTransitionException`(注文状態遷移違反)★v0.5** |
| 500 | Internal Server Error | 予期しないサーバーエラー |

---

## 8. データモデル ★v0.5 大幅改訂

Phase 1 の設計方針: **JPA エンティティ = ドメインモデル**として同一クラスで扱う。規模拡大時に分離を検討する。

物理設計の詳細(DDL・インデックス・制約)は [`design/01-database.md`](design/01-database.md) を参照。

### 8-1. Product

| フィールド | 型 | 制約・備考 |
|-----------|-----|-----------|
| `id` | `UUID` | PK |
| `name` | `String` | `@NotBlank` |
| `description` | `String` | 任意 |
| `price` | `BigDecimal` | `@DecimalMin("0.01")`。**税抜** ★v0.5 |
| **`taxCategory`** ★v0.5 | `TaxCategory` (Enum) | `STANDARD` / `REDUCED`。デフォルト `STANDARD` |
| `imageUrl` | `String` | 任意。画像ファイル本体アップロードは Phase 2 |
| `createdAt` / `updatedAt` | `OffsetDateTime` | — |

**変更点**: `stock` フィールドは `ProductStock` に分離 ★v0.5

### 8-2. ProductStock ★v0.5 新設

在庫情報を Product から分離。変更頻度・責務が異なるため独立エンティティとする。

| フィールド | 型 | 制約・備考 |
|-----------|-----|-----------|
| `productId` | `UUID` | PK 兼 FK → `Product.id`(1対1) |
| `quantity` | `Integer` | `@Min(0)` |
| `updatedAt` | `OffsetDateTime` | 在庫変更時刻 |

**背景**: Phase 2 で楽観ロック(`@Version`)を在庫のみに付与することで、商品情報の更新と在庫減算を干渉させない。また入出庫履歴(`stock_movements`)の拡張経路を確保する。

### 8-3. Cart / CartItem

| エンティティ | フィールド | 備考 |
|-------------|-----------|------|
| `Cart` | `cartId: UUID` | フロント `localStorage` で `crypto.randomUUID()` 生成。サーバーはそのまま受け取る |
| `CartItem` | `id: UUID` | PK |
| `CartItem` | `cartId: UUID` | FK → `Cart` |
| `CartItem` | `productId: UUID` | FK → `Product` |
| `CartItem` | `quantity: Integer` | **カート内でのこの商品の個数**。`@Min(1)` |
| `CartItem` | **`status: CartItemStatus`** ★v0.5 | `ACTIVE` / `SAVED_FOR_LATER`。デフォルト `ACTIVE` |

**カート識別の方針**: 認証が存在しない Phase 1 では、フロント側で `crypto.randomUUID()` を用いて UUID を生成し `localStorage` に保存する。サーバー側セッション管理は不要。

**「あとで買う」** ★v0.5: `status` を `SAVED_FOR_LATER` にすることで、カート合計・注文確定の対象から外す。注文確定時は `ACTIVE` の行のみを処理する。

**重複マージ規則** ★v0.5.1: `POST /api/cart/items` で同一 `(cart_id, product_id, status)` が既存の場合、新規行を INSERT せず既存行の `quantity` に加算する。加算後に `@Max(99)` または在庫上限を超える場合は 409 Conflict を返す。

**`SAVED_FOR_LATER → ACTIVE` 復帰時の在庫チェック** ★v0.5.1: 復帰時点での在庫が `quantity` を下回る場合は 409 Conflict を返し、遷移を行わない(ユーザー側で数量を減らして再試行する UX)。

### 8-4. Order / OrderItem

| エンティティ | フィールド | 備考 |
|-------------|-----------|------|
| `Order` | `id: UUID` | PK |
| `Order` | `status: OrderStatus` (Enum) | **`PENDING` / `CONFIRMED` / `CANCELLED` / `REFUNDED`** ★v0.5。現在値キャッシュ(真実源は `OrderStatusHistory`) |
| `Order` | **`subtotal: BigDecimal`** ★v0.5 | 税抜合計 = Σ(`unit_price` × `quantity`) |
| `Order` | **`taxAmount: BigDecimal`** ★v0.5 | 税額合計 = Σ(`OrderItem.taxAmount`) |
| `Order` | `totalPrice: BigDecimal` | 税込合計 = `subtotal` + `taxAmount` |
| `Order` | `createdAt: OffsetDateTime` | — |
| `OrderItem` | `id: UUID` | PK |
| `OrderItem` | `productName: String` ★v0.5 | 商品名スナップショット |
| `OrderItem` | `unitPrice: BigDecimal` | **税抜**単価スナップショット |
| `OrderItem` | `quantity: Integer` | この明細で注文した個数 |
| `OrderItem` | **`taxCategory: TaxCategory`** ★v0.5 | 確定時点の税区分 |
| `OrderItem` | **`taxRate: BigDecimal`** ★v0.5 | 確定時点の税率(例 `0.0800`) |
| `OrderItem` | **`taxAmount: BigDecimal`** ★v0.5 | この明細の税額(行単位切り捨て) |

**OrderStatus Enum**:
- Phase 1 で実使用: `CONFIRMED` のみ
- `PENDING`: Phase 2 決済連携用
- `CANCELLED` / `REFUNDED`: Phase 2 キャンセル・返金機能用(スキーマは先取り)

### 8-5. OrderStatusHistory ★v0.5 新設

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
(null) → CONFIRMED                   -- Phase 1 で唯一発生
(null) → PENDING → CONFIRMED         -- Phase 2(決済連携)
CONFIRMED → CANCELLED → REFUNDED     -- Phase 2(キャンセル→返金)
```

**運用ルール**: `Order.status` を変更する Service は、**同一トランザクションで履歴行を1件 INSERT** する。これを破ると真実源が壊れるため、基盤で強制する。

### 8-6. 注文確定のライフサイクル

1. Service 層で Cart の **`status=ACTIVE`** CartItem のみを走査し在庫チェック(不足時は `OutOfStockException`) ★v0.5
2. `ProductStock.quantity` を減算して保存(楽観ロックは Phase 2 以降) ★v0.5
3. CartItem の内容(`productId` / `quantity` / 現時点の `price` / 税区分から導出した `taxRate` / 行単位税額)を `OrderItem` にコピー(スナップショット) ★v0.5
4. Cart の `ACTIVE` 行をクリア(`SAVED_FOR_LATER` は残す) ★v0.5
5. `Order` を `status=CONFIRMED` で保存
6. **`OrderStatusHistory` に `(null → CONFIRMED)` を1件 INSERT** ★v0.5
7. `OrderResponse` を返す

---

## 9. アーキテクチャ設計

### 9-1. バックエンド:レイヤー構成

| パス | 役割 | 備考 |
|------|------|------|
| `domain/model/` | Entity (JPA) | JPA アノテーション付きのドメインモデル。Phase 1 は Entity = ドメインモデル |
| `domain/repository/` | Repository I/F | Spring Data JPA の interface のみ |
| `domain/exception/` | ドメイン例外 | `ProductNotFoundException` / `OutOfStockException` / **`IllegalStateTransitionException`** ★v0.5 など |
| `application/service/` | UseCase ロジック | トランザクション境界。ドメインモデルを操作し DTO を返す |
| `application/dto/` | Input / Output DTO | 外部公開データ形式。`@Valid` アノテーションを付与 |
| `application/mapper/` | Entity ↔ DTO 変換 | 手動変換メソッドを集約(Phase 1) |
| `presentation/controller/` | REST Controller | `@RestController`。リクエスト受付と DTO の返却のみ |
| `presentation/advice/` | 例外ハンドラ | `@RestControllerAdvice`。例外 → HTTP レスポンス変換 |
| `infrastructure/config/` | 設定クラス | `SecurityConfig` / `CorsConfig` / `BeanConfig` / **`TaxConfig`** ★v0.5 をファイル分割 |
| `infrastructure/persistence/` | Repository 実装 | domain の interface を実装 |

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

| 種別 | 対象層 | 方針 |
|------|--------|------|
| Unit Test | `application/service/` | Mockito でリポジトリをモック。ビジネスロジック・例外を検証 |
| `@WebMvcTest` | `presentation/controller/` | MockMvc で HTTP リクエスト/レスポンスを検証。Service はモック |
| `@DataJpaTest` | `infrastructure/persistence/` | H2 インメモリ DB でクエリ・CRUD 動作を検証 |
| Integration Test | 任意(Phase 2 以降) | `@SpringBootTest` で全層を通した結合テスト |

**Phase 1 の目標**: Service 層の Unit Test と Controller 層の `@WebMvcTest` を最優先で実装する。`@DataJpaTest` はカスタムクエリを書いた場合に追加する。

**v0.5 で追加する重点テスト** ★v0.5:

- 注文確定時の税額計算(標準/軽減税率混在)
- 行単位切り捨ての端数処理
- `SAVED_FOR_LATER` 商品が注文確定対象から除外されること
- `OrderStatusHistory` が `Order.status` 変更と同一トランザクションで INSERT されること

---

## 11. 技術スタック

### バックエンド

- Java 21 / Spring Boot 3.x
- Spring Data JPA(Hibernate)
- `spring-boot-starter-validation`
- Spring Security(Phase 1 は CORS 設定のみ)
- Gradle(Kotlin DSL)
- PostgreSQL 16
- **Flyway(DB マイグレーション)** ★v0.5
- springdoc-openapi(Swagger UI)

### フロントエンド

- React 19 / TypeScript / Vite
- TanStack Query(サーバー状態管理)
- React Router v7
- axios(HTTP クライアント)
- Zustand(グローバル UI 状態 / 必要時に追加)

### 開発環境

- macOS(M4 Pro)/ IntelliJ IDEA / VS Code
- PostgreSQL: Homebrew でローカル起動
- Docker: Phase 2 以降で導入予定

---

## 12. フェーズ計画

### Phase 1(現フェーズ)

- Spring Initializr でプロジェクト生成
- **Flyway による DB マイグレーション(`V1__init.sql`)** ★v0.5
- Product CRUD API(GET / GET{id} / POST)+ ページネーション
- **ProductStock テーブルへの在庫分離** ★v0.5
- バリデーション(`@Valid`)+ 例外ハンドリング(`@RestControllerAdvice`)
- Cart API(取得・追加・変更・削除)+ **「あとで買う」切替** ★v0.5
- 注文確定 API(在庫減算・スナップショット保存・**税額確定時点スナップショット**)★v0.5
- **`OrderStatusHistory` の記録(`(null → CONFIRMED)` のみ)** ★v0.5
- React フロントエンド(商品一覧・詳細・カート画面・**「あとで買う」一覧**) ★v0.5
- `cartId` を `localStorage` で管理
- Service 層 Unit Test + Controller 層 `@WebMvcTest`

### Phase 2(予定)

- JWT 認証の導入(Spring Security + JJWT)
- ユーザー登録・ログイン画面
- 商品の PUT / DELETE エンドポイント
- 楽観的ロック(`@Version`)による在庫競合対策(`ProductStock` に付与) ★v0.5
- **注文キャンセル・返金機能(`CANCELLED` / `REFUNDED` への遷移)** ★v0.5
- **税率マスタテーブル(`tax_rates`)化(`effective_from` / `effective_to` 管理)** ★v0.5
- **在庫入出庫履歴(`stock_movements`)** ★v0.5
- `application/mapper/` を MapStruct に置き換え
- Docker Compose によるコンテナ化
- CI/CD(GitHub Actions)

---

## 付録. 改版履歴

| バージョン | 日付 | 主な変更 |
|-----------|------|---------|
| v0.3 | — | 初版 |
| v0.4 | 2026-04 | `cartId` ヘッダー・`PENDING` 用途・エラーレスポンス形式を明記 |
| v0.5 | 2026-04 | 在庫テーブル分離 / 「あとで買う」追加 / 消費税対応(税抜保持・軽減税率・スナップショット) / 注文ステータス拡張(`CANCELLED` / `REFUNDED`) / 注文状態遷移履歴の導入 |
| **v0.5.1** | 2026-04 | レビュー反映: 在庫チェック2段階明記 / 「あとで買う」復帰時の409拒否 / CartItem 重複マージ規則 / 通貨 JPY 固定 / CSRF 無効化 / 注文完了画面の実現方法 / 金額・数量の上限 / 税額計算例 |
