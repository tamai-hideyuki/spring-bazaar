# spring-bazaar Phase 2 計画書

要件定義書 v0.6 §2-3 / §12 Phase 2 で参照される、Phase 2 の設計前提・方針を集約する。

Phase 1 の実装判断を曇らせないよう、Phase 2 関連の詳細設計はすべて本書にまとめる。**Phase 1 の実装中は「方針のメモ」「先取り判断の根拠」として参照するにとどめ、Phase 1 の意思決定には使わない**。

---

## 1. Phase 2 スコープ(確定)

- JWT 認証の導入(Spring Security + JJWT)
- ユーザー登録・ログイン画面
- 商品の PUT / DELETE エンドポイント
- 楽観的ロック(`@Version`)による在庫競合対策(`ProductStock` に付与)
- 注文キャンセル・返金機能(`CANCELLED` / `REFUNDED` への遷移)
- 税率マスタテーブル(`tax_rates`)化(`effective_from` / `effective_to` 管理)
- 在庫入出庫履歴(`stock_movements`)
- `application/mapper/` を MapStruct に置き換え
- Docker Compose によるコンテナ化
- CI/CD(GitHub Actions)
- 放棄カートの TTL クリーンアップ

---

## 2. 認証方針

### 2-1. 全体方針

- 認証方式: **JWT を `Authorization: Bearer <token>` ヘッダーで送信**
- セッション: **ステートレス**(サーバーでセッション保持しない)
- CSRF: **無効のまま**(Cookie に依存しないアクセストークンのため)。ただしリフレッシュトークン経路のみ例外(後述)
- フレームワーク: Spring Security + `spring-security-oauth2-resource-server`(または JJWT)

### 2-2. トークン設計

| 項目 | 値 | 理由 |
|------|-----|------|
| 署名アルゴリズム | **HS256**(共有秘密鍵) | 単一バックエンド・学習用として十分。必要時に RS256 へ移行 |
| アクセストークン有効期限 | **15 分** | 短命に倒し、漏洩リスクを最小化 |
| リフレッシュトークン | **あり / 有効期限 7 日** | UX を落とさない。DB 保管し revoke 可能にする |
| アクセストークン保存先(フロント) | **メモリ(JS 変数)** | XSS 耐性のため `localStorage` を避ける |
| リフレッシュトークン保存先(フロント) | **HttpOnly + Secure + SameSite=Strict Cookie** | JS から読めないため XSS 耐性が高い |
| パスワードハッシュ | **BCrypt**(`BCryptPasswordEncoder`) | Spring Security 標準 |

### 2-3. JWT クレーム

```json
{
  "sub": "<user UUID>",
  "roles": ["USER"],
  "iat": 1744672800,
  "exp": 1744673700
}
```

- `sub`: ユーザーの UUID
- `roles`: `USER` または `ADMIN`。`POST /api/products` は `ADMIN` のみ許可
- 追加クレームは最小限(`email` 等は入れない。必要なら `/api/me` で都度取得)

### 2-4. 認証エンドポイント

| Method | Path | 説明 |
|--------|------|------|
| POST | `/api/auth/register` | ユーザー登録 |
| POST | `/api/auth/login` | ログイン。access を JSON で返し、refresh を Set-Cookie |
| POST | `/api/auth/refresh` | refresh Cookie を検証し新しい access を発行。**このエンドポイントのみ CSRF トークン要求**(Cookie 認証のため) |
| POST | `/api/auth/logout` | refresh トークンを revoke |

### 2-5. CSRF の扱い

Phase 1 から通して `CsrfConfigurer.disable()` を基本とし、**`/api/auth/refresh` のみ例外的に CSRF トークンを要求**する。理由:

- アクセストークン経路は `Authorization` ヘッダー + メモリ保存のため CSRF 攻撃不可
- リフレッシュ経路は HttpOnly Cookie を使うため、CSRF 対策が必要

### 2-6. 認可の粒度

- `GET` / `POST /api/cart/*` / `POST /api/orders`: 認証済みユーザー(`USER` 以上)
- `POST /api/products` / `PUT` / `DELETE /api/products/{id}`: `ADMIN` ロールのみ
- `GET /api/products`, `GET /api/products/{id}`: 匿名許可(公開商品カタログ)

### 2-7. Phase 1 の前提整合

- `cartId` ヘッダー方式は Phase 2 で「認証済みユーザーの `userId` を優先、未ログイン時は `cartId` フォールバック」に移行する想定。Phase 1 のエンドポイント仕様は Phase 2 で破壊的変更なしに拡張可能。
- 既存のエラーレスポンス形式(要件書 §7)は Phase 2 で 401 Unauthorized / 403 Forbidden を追加するのみ。
- エラー統一形式のため、`AuthenticationEntryPoint` / `AccessDeniedHandler` を実装して同形式(`ErrorResponse`)で返す。

---

## 3. その他 Phase 2 の先取り設計

### 3-1. 楽観的ロック

`ProductStock` に `@Version Long version` を付与。`OrderService` が `OptimisticLockException` を検知したら 409 Conflict(`InventoryConflictException` 等にラップ)を返し、ユーザーに再試行を促す。悲観ロック(`SELECT ... FOR UPDATE`)は採用しない。

### 3-2. 注文キャンセル・返金

`Order` Aggregate に以下のメソッドを追加(Phase 1 の [`design/05-domain-order.md`](design/05-domain-order.md) §2.4 で設計先取り済み):

```java
public void cancel(String reason, OffsetDateTime now) { ... }  // CONFIRMED → CANCELLED
public void refund(OffsetDateTime now) { ... }                 // CANCELLED → REFUNDED
```

状態遷移違反は `IllegalStateTransitionException`(Phase 1 から先行作成済み)で 409 を返す。

### 3-3. 税率マスタ化

`tax_rates(tax_category, rate, effective_from, effective_to)` を追加。`TaxProperties#rateOf(TaxCategory)` の呼び出し側は変更なし(インターフェース固定で差し替え可能な設計を Phase 1β 時点で確立済み)。

### 3-4. 在庫履歴

`stock_movements(id, product_id, delta, reason, occurred_at)` を追加。`ProductStock#decrease` / `increase` 内で履歴行を 1 件 INSERT。

### 3-5. キャンセル時の在庫戻し戦略

- 全量戻す想定([`design/01-database.md`](design/01-database.md) §10 未決 #6 に準拠)
- `Order.cancel` 内で `stock_movements` に `delta=+quantity` を記録しつつ `ProductStock.quantity` を加算

### 3-6. `order_sequences` テーブル化(高並行対策)

Phase 1β の簡易採番(`SELECT COUNT + UNIQUE + 1 回リトライ`)で衝突が実測された場合のみ、`order_sequences(date, next_seq)` を導入して `SELECT ... FOR UPDATE` で排他採番。学習用の単一開発者環境では不要の見込み。

---

## 4. Phase 2 着手時の作業

- [ ] 本書を「Phase 2 要件定義書 v1.0」に昇格
- [ ] [`design/`](design/) 配下に `07-auth.md` / `08-cancel-refund.md` / `09-docker-compose.md` / `10-ci-cd.md` 等を追加
- [ ] 要件書 §2-3 / §12 から Phase 2 記述を削除(本書に一本化)
