# 03. プロジェクト初期セットアップ設計

要件定義書 v0.5.2 に基づく Phase 1 の開発環境・プロジェクト構成の確定事項。
各項目で「**決定**」「**候補と比較**」「**選定理由**」を明示する。

---

## 1. リポジトリ構成

### 決定

**モノレポ** — 単一リポジトリ内に `backend/` と `frontend/` を配置する。

```
spring-bazaar/              (Git ルート)
├── backend/                (Spring Boot)
├── frontend/               (React + Vite)
├── docs/                   (要件書・設計書 — 既存)
├── .gitignore
└── README.md
```

### 候補と比較

| 方式 | 利点 | 欠点 |
|------|------|------|
| **モノレポ**(採用) | 1 PR でフロント・バックエンドを一括レビュー。Docker Compose の `build.context` が直感的。CI で変更範囲を横断検知しやすい | リポジトリサイズが膨らみやすい。各チームの独立リリースが困難 |
| マルチリポ | チーム分割・独立デプロイに強い。言語別ツールチェインが干渉しない | PR が分散し整合確認が煩雑。ローカル開発で 2 リポを clone・同期する手間 |

### 選定理由

- 開発者が **1 名**。チーム分割の恩恵がなく、1 PR で全体を見渡せるモノレポの方が圧倒的に効率的。
- Phase 2 で Docker Compose を導入する際、`docker-compose.yml` をルートに置いて `build: ./backend` / `build: ./frontend` と書ける構成が自然。
- 「リポサイズが膨らむ」欠点は、学習プロジェクト規模では問題にならない。

---

## 2. バックエンド プロジェクト設定

### 2.1 Maven 座標

| 項目 | 値 |
|------|-----|
| **Group** | `dev.tamai` |
| **Artifact** | `spring-bazaar` |
| **ルートパッケージ** | `dev.tamai.springbazaar` |
| **Java** | 21 (LTS) |
| **Spring Boot** | 3.3.x(最新安定) |
| **ビルドツール** | Gradle 8.x / Kotlin DSL |

#### Group に `dev.tamai` を選んだ理由

| 候補 | 評価 |
|------|------|
| `com.example` | Spring Initializr のデフォルト。「チュートリアルの習作」感が強く、ポートフォリオや公開リポでは避けたい |
| `io.github.tamaihideyuki` | GitHub Pages ドメインに準拠した正式な逆ドメイン。正確だが冗長で、パッケージ階層が深くなる |
| **`dev.tamai`**(採用) | 個人開発者向けの `.dev` TLD 風。短く実用的。Maven Central に publish する予定がない学習プロジェクトでは、厳密なドメイン所有権より可読性を優先 |

#### Spring Boot 3.3.x を選んだ理由

| 候補 | 評価 |
|------|------|
| 3.2.x | 安定だが LTS サポート期間が 3.3.x より短い。Virtual Thread 周りの改善が入っていない |
| **3.3.x**(採用) | 2024 年 5 月 GA。Java 21 LTS との組み合わせでテスト実績が豊富。`RestClient` の成熟・`@HttpExchange` の安定・Testcontainers のファーストクラスサポートなど、Phase 2 で活きる機能が揃う |
| 3.4.x (最新) | リリースから日が浅く、サードパーティの互換性(springdoc 等)にまだ不安定な部分がある。学習用でトラブルシューティングに時間を取られたくない |

### 2.2 依存関係

```kotlin
// build.gradle.kts (抜粋)
dependencies {
    // --- Spring Boot Starters ---
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-security")

    // --- Database ---
    runtimeOnly("org.postgresql:postgresql")
    implementation("org.flywaydb:flyway-core")
    implementation("org.flywaydb:flyway-database-postgresql")

    // --- API Documentation ---
    implementation("org.springdoc:springdoc-openapi-starter-webmvc-ui:2.6.0")

    // --- Developer Productivity ---
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    annotationProcessor("org.springframework.boot:spring-boot-configuration-processor")

    // --- Test ---
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
    testRuntimeOnly("org.testcontainers:postgresql")
    testImplementation("org.testcontainers:junit-jupiter")
}
```

#### 各依存関係の選定背景

| 依存 | 役割 | 代替候補 | 不採用理由 |
|------|------|---------|-----------|
| `spring-boot-starter-web` | REST API 基盤 | WebFlux(リアクティブ) | EC サイトのリクエストは I/O バウンドだが、Phase 1 はブロッキングで十分。リアクティブは学習コストが高く、JPA との統合も複雑(`R2DBC` 必須) |
| `spring-boot-starter-data-jpa` | ORM (Hibernate) | MyBatis / jOOQ / Spring Data JDBC | **JPA を選ぶ理由**: 関連(1:1, 1:N)の自動マッピング、ページネーション(`Page<T>`)の組み込みサポート、JPQL によるオブジェクト指向クエリ。**MyBatis** は SQL を直接書けて性能チューニングしやすいが、DTO マッピングの boilerplate が増え、ページネーション実装が手動になる。**jOOQ** は型安全 SQL が魅力だが有料ライセンス(PostgreSQL は無料対象外)。**Spring Data JDBC** は設計がシンプルだがレイジーロードや `Page<T>` 対応が弱い |
| `spring-boot-starter-validation` | Bean Validation | 手動バリデーション | `@Valid` + `@RestControllerAdvice` で宣言的に制約を記述でき、コードの見通しが良い。手動バリデーションは Service 層に条件分岐が増え、漏れやすい |
| `spring-boot-starter-security` | CORS 設定・Phase 2 の認証基盤 | `WebMvcConfigurer#addCorsMappings` のみ | `WebMvcConfigurer` だけでも CORS は設定できるが、Security Filter Chain で一元管理する方が Phase 2 への移行がスムーズ。Security 無しで CORS だけ設定すると、Phase 2 で Security を追加した瞬間にフィルタ順序の競合で CORS が壊れる典型的なハマりポイントを回避 |
| `postgresql` | JDBC ドライバ | H2(開発時) | 本番と開発で DB を揃える方針。H2 は `timestamptz` や `NUMERIC` の挙動が PostgreSQL と微妙に異なり、マイグレーション SQL の互換も保証されない |
| `flyway-core` | DB マイグレーション | Liquibase | **Flyway**: SQL ファイル(`V1__init.sql`)を直接書く。DBA との協業や SQL レベルでの制御がしやすい。セットアップが最小限。**Liquibase**: XML/YAML/JSON で変更セットを記述し、ロールバック定義が書ける。大規模組織向けの機能(diff, 変更セット依存関係)が豊富だが、学習プロジェクトには過剰。本プロジェクトでは SQL を直接管理したいため Flyway を選択 |
| `springdoc-openapi-starter-webmvc-ui` | Swagger UI 自動生成 | SpringFox / 手書き OpenAPI | **SpringFox** は Spring Boot 3.x 未対応で事実上メンテナンス停止。**springdoc** が後継として Spring Boot 3.x + Jakarta EE に完全対応しており、唯一の実質的選択肢 |
| `lombok` | boilerplate 削減 | Java 21 の record / 手書き | 下記 §2.3 で詳述 |
| `spring-boot-configuration-processor` | `@ConfigurationProperties` の IDE 補完 | なし | `TaxProperties` 等の YAML 補完を IntelliJ で有効化するため。ビルド成果物に含まれない(annotation processor のみ) |
| `testcontainers:postgresql` | テスト用の本物 PostgreSQL | H2 インメモリ | **Testcontainers**: Docker で本物の PostgreSQL を起動してテスト。Flyway マイグレーションが本番と同じ SQL で通ることを保証。**H2**: 起動が速いが、PostgreSQL 固有の挙動(`CHECK` 制約、`NUMERIC` の精度、`timestamptz` のタイムゾーン変換)と差異があり、「テストは通るが本番で落ちる」リスクを孕む。本プロジェクトは金額計算の精度が重要なため、最初から本物の DB でテストする |

#### 採用しなかった依存

| ライブラリ | 検討理由 | 不採用理由 |
|-----------|---------|-----------|
| MapStruct | Entity ↔ DTO マッピングの自動化 | Phase 1 のエンティティは 7 つ、マッパーは 3 つ。手動の静的メソッドで十分に管理可能。Phase 2 でマッパー数が増えたら導入(要件書 §12 Phase 2 に明記済み) |
| Zustand | フロントのグローバル状態管理 | Phase 1 はサーバー状態を TanStack Query で管理すれば足りる。グローバル UI 状態の必要性が出た時点で導入 |
| Docker / Docker Compose | コンテナ化 | Phase 2 以降(要件書既決)。Phase 1 はローカル Homebrew PostgreSQL で開発速度を優先 |

### 2.3 Lombok の採用方針

#### 決定

**限定採用** — 以下のアノテーションのみ使用を許可する。

| 許可 | 禁止 |
|------|------|
| `@Getter`, `@RequiredArgsConstructor`, `@Builder`, `@ToString` | `@Data`, `@Setter`, `@AllArgsConstructor`, `@SneakyThrows`, `@val` |

#### 候補と比較

| 方式 | 利点 | 欠点 |
|------|------|------|
| **Lombok 限定採用**(採用) | Service / Controller で `final` フィールド + `@RequiredArgsConstructor` の DI boilerplate を削減。JPA Entity の `@Getter` で getter 手書きを回避 | IDE プラグイン必須。コンパイル時のコード変換が「魔法」で、デバッグ時に生成コードが見えにくい |
| Lombok 不採用(手書き) | ソースコードに嘘がない。IDE 依存なし | getter/constructor が大量に並ぶ。Entity が 7 つ + DTO が 10 以上ある本プロジェクトでは、可読性が著しく落ちる |
| Java 21 の record のみ | 言語ネイティブ。追加依存なし | **JPA Entity に `record` は使えない**(Hibernate は引数なしコンストラクタ + mutable フィールドを要求)。DTO には使えるが Entity は手書きが残り、2 スタイル混在になる |

#### 選定理由

- JPA Entity は `record` が使えないため、`@Getter` + `@RequiredArgsConstructor` で boilerplate を減らすのが現実的。
- `@Data`(`@Setter` + `@EqualsAndHashCode` 含む)を禁止するのは、**JPA Entity で `equals/hashCode` が Hibernate のプロキシと衝突する** 典型バグを防ぐため。`@Setter` は「どこからでも変更可能」を招き、ドメインモデルの不変条件を壊しやすい。
- DTO(Request / Response)は **`record` を採用**。`record` は final・不変・`equals/hashCode` 自動生成で、DTO の用途に完全に合致する。Jackson は record のデシリアライズに対応済み。

**まとめ**: Entity → `@Getter` + `@RequiredArgsConstructor`(Lombok)。DTO → `record`(Java 標準)。

### 2.4 パッケージ階層

```
dev.tamai.springbazaar
├── SpringBazaarApplication.java
├── domain
│   ├── model/       Entity (JPA アノテーション付きドメインモデル)
│   ├── repository/  Spring Data JPA interface
│   └── exception/   ドメイン例外
├── application
│   ├── service/     ユースケースロジック(@Transactional)
│   ├── dto/         入出力 DTO (record)
│   └── mapper/      Entity ↔ DTO 変換(静的メソッド)
├── presentation
│   ├── controller/  @RestController
│   └── advice/      @RestControllerAdvice
└── infrastructure
    ├── config/      SecurityConfig, CorsConfig, TaxConfig, OpenApiConfig, ClockConfig
    └── persistence/ (Phase 1 ではほぼ空)
```

#### なぜこの 4 層か

| 候補 | 説明 | 不採用理由 |
|------|------|-----------|
| **レイヤードアーキテクチャ**(採用) | domain / application / presentation / infrastructure の責務分離 | — |
| ヘキサゴナル(Ports & Adapters) | domain にポート(interface)を切り、infrastructure にアダプタを実装。依存の方向を厳密に制御 | Phase 1 の規模(Entity 7 / Service 3)で interface + 実装に分けると、ファイル数が倍増して学習コストの方が上回る。Spring Data JPA の `JpaRepository` interface は既に「ポート」の役割を果たしており、追加の抽象化層は過剰 |
| 機能別パッケージ(feature-based) | `product/` `cart/` `order/` 内にそれぞれ controller / service / entity を内包 | ドメイン間の依存(CartService → ProductRepository)が パッケージ境界を跨ぐため、アクセス制御が複雑化する。少人数ではレイヤー別の方が横断的な変更(例外ハンドリング追加等)がしやすい |

#### `infrastructure/persistence/` がほぼ空な理由

Spring Data JPA では `domain/repository/` に interface を置くだけで Hibernate が実装を自動生成する。カスタムクエリが必要になった場合に `persistence/` 配下に `CustomXxxRepositoryImpl` を置く余地として確保しているが、Phase 1 では不要。

---

## 3. フロントエンド プロジェクト設定

### 3.1 基本設定

| 項目 | 値 |
|------|-----|
| **Vite テンプレート** | `react-ts`(標準) |
| **Node** | LTS 22.x |
| **パッケージマネージャ** | npm |
| **React** | 19.x |
| **TypeScript** | 5.x (strict mode) |

#### Vite テンプレート: `react-ts` vs `react-swc-ts`

| テンプレート | 利点 | 欠点 |
|-------------|------|------|
| **`react-ts`**(採用) | Babel ベースで、エコシステムの互換性が最も広い。トラブル時の情報が豊富 | SWC 比でビルド速度がやや劣る(数百 ms 単位) |
| `react-swc-ts` | SWC による高速トランスパイル(大規模プロジェクトでは顕著) | Babel プラグインが使えない。エラーメッセージが Babel と異なり、学習段階では情報が少ない |

**選定理由**: Phase 1 のフロントは画面 4〜5 ページ。ビルド速度の差は体感できない規模。Babel ベースの方がエラーメッセージの事例が Web 上に多く、学習時のトラブルシューティングが容易。

#### パッケージマネージャ: npm vs yarn vs pnpm

| ツール | 利点 | 欠点 |
|--------|------|------|
| **npm**(採用) | Node 同梱。追加インストール不要。`package-lock.json` は GitHub Dependabot のデフォルト対応 | `node_modules/` のフラット化で依存の「幽霊参照」が起きうる(pnpm のストリクトモードが解決) |
| yarn (Classic) | 並列インストールが速い。`yarn.lock` | Yarn Classic はメンテナンスモード。Berry (v4) は PnP モードが IDE と相性問題を起こすことがある |
| pnpm | 厳密な依存解決(幽霊参照排除)。ディスク効率 | 追加インストール必要。CI やツールチェインでの `npm` 前提の設定が動かないことがある |

**選定理由**: 1 名開発で依存数も少ない Phase 1 では、追加ツールのセットアップコストを払う理由がない。npm で十分。

### 3.2 フロント依存関係

```json
{
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router": "^7.0.0",
    "@tanstack/react-query": "^5.0.0",
    "axios": "^1.7.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "@types/react": "...",
    "@types/react-dom": "...",
    "vite": "^6.0.0",
    "@vitejs/plugin-react": "..."
  }
}
```

#### 各依存関係の選定背景

| 依存 | 役割 | 代替候補 | 不採用理由 |
|------|------|---------|-----------|
| **React 19** | UI ライブラリ | Vue 3 / Svelte / SolidJS | 要件書既決。補足: React は求人市場でのシェアが最大であり、学習プロジェクトとしてのポートフォリオ価値が高い |
| **React Router v7** | SPA ルーティング | TanStack Router / Next.js | **TanStack Router** は型安全ルーティングが魅力だが、v7 時点でまだ比較的新しく情報が少ない。**Next.js** は SSR/SSG が強みだが、本プロジェクトは完全な SPA + REST 構成のため SSR は不要でむしろ構成が複雑になる。React Router v7 は最も枯れた SPA ルーターで、ドキュメントと事例が豊富 |
| **TanStack Query v5** | サーバー状態管理 | SWR / useEffect + useState / Redux Toolkit Query | **SWR**: API がシンプルで軽量だが、mutation サポートが TanStack Query より弱い(カート操作で `useMutation` + キャッシュ無効化を多用するため重要)。**useEffect + useState**: ローディング/エラー/リフェッチ/キャッシュを全て手書きする必要があり、バグの温床。**RTK Query**: Redux 依存で、本プロジェクトで Redux を採用していないため過剰 |
| **axios** | HTTP クライアント | fetch API (native) / ky / ofetch | **fetch**: ネイティブで依存ゼロだが、`response.ok` チェックを毎回書く必要あり、interceptor がない(`X-Cart-Id` ヘッダーの一元付与に interceptor が必須)、タイムアウト設定が `AbortController` 経由で冗長。**axios**: interceptor で `X-Cart-Id` を全リクエストに自動付与、エラーハンドリングが `try/catch` で直感的、`baseURL` 設定が 1 行。**ky**: fetch ベースでモダンだが、情報量が axios に比べ圧倒的に少ない |

#### 採用しなかったフロント依存

| ライブラリ | 検討理由 | Phase 1 不採用理由 |
|-----------|---------|------------------|
| React Hook Form | フォーム状態管理 + バリデーション | Phase 1 のフォームは「商品登録」と「カート数量変更」の 2 つ。React Hook Form の `setError` が `errors[]` レスポンスと相性が良いのは事実だが、2 フォームに対してライブラリ導入するのは過剰。`useState` + 手動 `setError` で始め、フォーム数が増える Phase 2 で導入判断 |
| Tailwind CSS | ユーティリティファースト CSS | CSS フレームワーク選定は本書のスコープ外(別途決定)。ただし Phase 1 では最低限のスタイリングのみ必要(学習の焦点は API 連携とデータフロー) |
| Zustand | グローバル UI 状態管理 | Phase 1 でグローバル UI 状態の必要性がない。TanStack Query がサーバー状態を管理し、ローカル状態は `useState` で十分。必要が生じた時点で導入(要件書 §9-3 に明記済み) |

### 3.3 ディレクトリ構成

```
frontend/
├── public/
├── src/
│   ├── api/            axios クライアント・interceptor・baseURL
│   ├── types/          バックエンド DTO の TypeScript 型(ErrorResponse 含む)
│   ├── components/     汎用 UI パーツ(ドメイン非依存)
│   ├── features/
│   │   ├── product/    components / hooks / api
│   │   └── cart/       components / hooks / api
│   ├── hooks/          feature 横断の共通カスタム hooks
│   ├── pages/          React Router v7 のページコンポーネント
│   ├── App.tsx         ルーティング定義
│   └── main.tsx        エントリポイント
├── .env.development
├── tsconfig.json
└── vite.config.ts
```

#### `features/` を採用した理由

| 方式 | 説明 | 評価 |
|------|------|------|
| **feature-based**(採用) | ドメイン別(`product/` `cart/`)にコンポーネント・hooks・api を内包 | 関連コードの局所性が高い。「カート機能を触るときに `cart/` だけ見ればいい」 |
| 技術別フラット | `components/` `hooks/` `api/` をトップレベルに全展開 | ファイル数が増えるとドメイン間の境界が曖昧になる。20+ ファイル超えると見通しが悪化 |
| pages 内包 | pages コンポーネント内に hooks・api を直接書く | 再利用性がゼロ。ページ間で同じ API 呼び出しが重複 |

---

## 4. ローカル開発環境

### 4.1 PostgreSQL セットアップ

| 項目 | 値 |
|------|-----|
| **インストール方法** | `brew install postgresql@16` + `brew services start postgresql@16` |
| **DB 名** | `spring_bazaar` |
| **ユーザー名** | `spring_bazaar` |
| **パスワード** | `spring_bazaar` |
| **接続 URL** | `jdbc:postgresql://localhost:5432/spring_bazaar` |

#### Docker vs Homebrew

| 方式 | 利点 | 欠点 |
|------|------|------|
| **Homebrew**(採用) | 起動が速い(ネイティブプロセス)。Apple Silicon に最適化済み。Docker Desktop のリソース消費がない | OS グローバルにインストールされ、他プロジェクトとバージョン競合の可能性 |
| Docker (`docker run`) | 環境の完全隔離。バージョン切替が容易 | Docker Desktop の RAM 消費(2GB+ 常駐)。Testcontainers もテスト中に Docker を使うため、開発中も Docker を起動していると M4 Pro でもメモリ圧迫 |

**選定理由**: Phase 1 はローカル開発のみ。Homebrew の方が起動が速く、Docker Desktop のリソース消費を避けられる。Phase 2 で Docker Compose 化する際に PostgreSQL もコンテナに移行する。

#### 初回セットアップコマンド

```bash
# PostgreSQL インストール・起動
brew install postgresql@16
brew services start postgresql@16

# DB・ユーザー作成
psql postgres -c "CREATE USER spring_bazaar WITH PASSWORD 'spring_bazaar';"
psql postgres -c "CREATE DATABASE spring_bazaar OWNER spring_bazaar;"
```

### 4.2 設定ファイル分離

```
backend/src/main/resources/
├── application.yml           共通設定(税率、Jackson、JPA validate、CORS 値)
├── application-local.yml     ローカル DB 接続・SQL ログ・Flyway dev 拡張
├── application-test.yml      Testcontainers 接続
└── application-local.yml.example   ★Git 管理(コピーして使うテンプレ)
```

| ファイル | Git 管理 | 内容 |
|----------|---------|------|
| `application.yml` | ○ | 税率(`tax.rates.*`)、Jackson 設定、JPA `ddl-auto: validate`、CORS 設定値 |
| `application-local.yml` | **× (`.gitignore`)** | DB URL / user / password、`show-sql: true`、Flyway dev seed location |
| `application-local.yml.example` | ○ | `application-local.yml` のテンプレート(パスワード欄はプレースホルダ) |
| `application-test.yml` | ○ | Testcontainers の動的 URL(`spring.datasource.url` は `@DynamicPropertySource` で上書き) |

#### なぜ `application-local.yml` を `.gitignore` するか

- ローカル環境固有の値(DB パスワード、ポート)がコミットされるのを防ぐ。
- 「`application-local.yml` に何を書けばいいか」は `.example` ファイルで自明にする。
- **Phase 2** で AWS Secrets Manager 等の外部シークレット管理に移行する際に、ローカル設定とのインターフェースが `application-{profile}.yml` で統一されている。

#### デフォルトプロファイル

`application.yml` に `spring.profiles.default: local` を設定。明示的にプロファイルを指定しない場合は自動で `local` が適用される。`@SpringBootTest` は `application-test.yml` を使うよう `@ActiveProfiles("test")` を付与。

### 4.3 ポート

| サービス | ポート | 備考 |
|---------|--------|------|
| Spring Boot (backend) | 8080 | Spring Boot デフォルト |
| Vite (frontend) | 5173 | Vite デフォルト |
| PostgreSQL | 5432 | PostgreSQL デフォルト |

すべてデフォルト値をそのまま使い、設定の複雑性を排除する。ポート競合が発生した場合のみ変更。

### 4.4 フロントエンド環境変数

```env
# frontend/.env.development
VITE_API_URL=http://localhost:8080
```

`axios` の `baseURL` を `import.meta.env.VITE_API_URL` から取得する。本番環境では `.env.production` で API URL を差し替え。

---

## 5. シードデータ

### 5.1 配置と本番投入防止

| 項目 | 値 |
|------|-----|
| **配置** | `backend/src/main/resources/db/migration-dev/V900__seed_dev.sql` |
| **本番投入防止** | Flyway の `locations` を **プロファイル分岐** で制御 |

```yaml
# application.yml (共通 — 本番マイグレーションのみ)
spring:
  flyway:
    locations: classpath:db/migration

# application-local.yml (ローカル — dev seed を追加)
spring:
  flyway:
    locations: classpath:db/migration,classpath:db/migration-dev
```

#### 代替案との比較

| 方式 | 利点 | 欠点 |
|------|------|------|
| **Flyway 追加 location**(採用) | Flyway が管理するため冪等(2 回目以降は skip)。バージョン番号で実行順序を制御可能 | ディレクトリが 2 つに分かれる |
| `spring.sql.init.data-locations` | Spring Boot ネイティブ。シンプル | Flyway と実行タイミングが競合しうる(`spring.jpa.defer-datasource-initialization` が必要)。冪等性保証なし(再起動のたびに INSERT → 重複エラー) |
| `data.sql` + `spring.sql.init.mode=embedded` | 最も簡単 | `embedded` は H2 のみ。PostgreSQL では `always` 指定が必要で、全環境で実行されるリスク |

#### 番号体系

| 範囲 | 用途 |
|------|------|
| `V1` 〜 `V899` | 本番マイグレーション(`db/migration/`) |
| `V900` 〜 `V999` | 開発用シードデータ(`db/migration-dev/`) |

### 5.2 シードデータ内容

商品 8 件 — 税区分・在庫・画像の各パターンを網羅:

| 商品名 | 税区分 | 税抜価格 | 在庫 | 画像 | 確認目的 |
|--------|--------|---------|------|------|---------|
| リンゴ | REDUCED | 100 | 50 | あり | 軽減税率の基本ケース |
| パン | REDUCED | 200 | 30 | あり | 軽減税率 + 中価格帯 |
| 牛乳 | REDUCED | 180 | 0 | あり | **軽減税率 + 在庫切れ** |
| お茶 | REDUCED | 150 | 5 | なし | 軽減税率 + 画像なし + 少在庫 |
| ノート | STANDARD | 300 | 100 | あり | 標準税率の基本ケース |
| ボールペン | STANDARD | 150 | 20 | あり | 標準税率 + 低価格帯 |
| デスクライト | STANDARD | 5000 | 3 | あり | 標準税率 + 高価格帯 + 少在庫 |
| USBケーブル | STANDARD | 800 | 0 | なし | **標準税率 + 在庫切れ + 画像なし** |

---

## 6. CORS 設定

### 決定

```yaml
cors:
  allowed-origins:    [ "http://localhost:5173" ]
  allowed-methods:    [ "GET", "POST", "PATCH", "DELETE", "OPTIONS" ]
  allowed-headers:    [ "Content-Type", "X-Cart-Id", "Authorization" ]
  exposed-headers:    [ "Location" ]
  allow-credentials:  false
  max-age:            3600
```

### 各項目の判断根拠

| 項目 | 値 | 理由 |
|------|-----|------|
| `allowed-origins` | `http://localhost:5173` | Vite のデフォルトポート。本番 URL は Phase 2 で追加 |
| `allowed-methods` | GET, POST, PATCH, DELETE, OPTIONS | OPTIONS はプリフライトリクエスト用。PUT は Phase 1 不使用のため含めない(Phase 2 で追加) |
| `allowed-headers` | Content-Type, X-Cart-Id, Authorization | `Authorization` は Phase 1 では使わないが **先行許可**。Phase 2 で JWT を導入した際に CORS 設定変更が不要になる。「今は要らないけど後で確実に要るもの」は許可リストに先入れしておくのが定石(セキュリティリスクは一切増えない — ヘッダーの**送信**を許可するだけで、認証の検証ロジックは別) |
| `exposed-headers` | Location | `POST /api/products` や `POST /api/orders` のレスポンスに `Location` ヘッダーを含む。CORS では明示的に `exposed-headers` に含めないとフロントの JavaScript から読めない |
| `allow-credentials` | `false` | Phase 1 は Cookie 不使用。`true` にすると `allowed-origins` にワイルドカード(`*`)が使えなくなる制約もある。Phase 2 でリフレッシュトークン用の HttpOnly Cookie を導入する際に、**`/api/auth/refresh` エンドポイントのみ別の `CorsConfigurationSource`** で `true` に上書きする設計 |
| `max-age` | 3600 (1 時間) | プリフライトリクエストのキャッシュ秒数。1 時間はブラウザの上限(Chrome)に近い値。開発中は頻繁にサーバー再起動するが、CORS 設定が変わるのは稀なため長めで問題ない |

### 実装方針

`SecurityFilterChain` 内で `http.cors(cors -> cors.configurationSource(...))` を使用する。`WebMvcConfigurer#addCorsMappings` は **使わない** — Security Filter より後段で適用されるため、Security が先に CORS を拒否する問題が起きうるため。

---

## 7. 日時・タイムゾーン

### 7.1 決定一覧

| 項目 | 決定 | 理由 |
|------|------|------|
| **JVM タイムゾーン** | UTC (`TimeZone.setDefault(TimeZone.getTimeZone("UTC"))`) | DB の `timestamptz` は UTC で保存する前提。JVM も UTC に統一すると、`OffsetDateTime` → DB 保存時の暗黙変換が一切発生せず、バグを未然に防ぐ |
| **DB** | `timestamptz` (UTC 保存) | 設計書 01-database.md で既決 |
| **Jackson シリアライズ** | `spring.jackson.serialization.write-dates-as-timestamps=false` | ISO-8601 文字列出力。数値タイムスタンプはフロントで読みにくく、デバッグ効率が落ちる |
| **API 入力** | `OffsetDateTime` で受付。`Z` も `+09:00` も許容 | クライアント側がどの TZ で送っても受けられる柔軟性 |
| **フロント表示** | JST に変換(`Intl.DateTimeFormat('ja-JP', { timeZone: 'Asia/Tokyo' })`) | ターゲットユーザーが日本国内を前提 |
| **`orderNumber` 採番** | **JST 0時で日付切替** | 注文番号 `yyyyMMdd-NNNN` の「日付」は業務上 JST が自然。UTC 0 時切替だと JST 9:00 でリセットされ、日本のユーザーの感覚とずれる |

### 7.2 `Clock` の DI

**決定**: Service 層が現在時刻を必要とする場合は `Clock` を DI する。`LocalDate.now()` や `Instant.now()` を直接呼ばない。

```java
// ClockConfig.java (infrastructure/config/)
@Configuration
public class ClockConfig {
    @Bean
    public Clock clock() {
        return Clock.system(ZoneId.of("Asia/Tokyo"));
    }
}

// OrderService.java (使用側)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final Clock clock;

    public OrderResponse createOrder(UUID cartId) {
        LocalDate today = LocalDate.now(clock);  // テストで固定可能
        String datePrefix = today.format(DateTimeFormatter.BASIC_ISO_DATE);
        // ...
    }
}
```

#### なぜ `Clock` を DI するか

| 方式 | テスト容易性 | 欠点 |
|------|-------------|------|
| **`Clock` を DI**(採用) | テストで `Clock.fixed(...)` を注入すれば日時を完全固定。注文番号の日付切替テストが確実に書ける | 引数が 1 つ増える(微小なコスト) |
| `LocalDate.now()` 直接呼び | コードが短い | テストで日時を制御できない。深夜 0 時前後のテストが非決定的になる。`PowerMock` 等で static をモックする手もあるが、バイトコード操作が重く脆い |
| `@MockBean` で `Clock` | Spring Context にモックを差し込む | `@WebMvcTest` や単体テストでは Spring Context がないので使えない。`@SpringBootTest` だとテストが遅い |

#### `ZoneId.of("Asia/Tokyo")` を Bean に埋め込む理由

`orderNumber` の採番境界は **ビジネスルール**(JST 0 時切替)。JVM タイムゾーンを UTC にしても、`Clock` Bean の ZoneId が `Asia/Tokyo` なので `LocalDate.now(clock)` は JST の日付を返す。ビジネス上の「今日」と技術上の「UTC 時刻」を分離する設計。

---

## 8. チェックリスト(着手前)

実装着手前に本書の内容が反映されていることを確認する:

- [ ] `backend/` に Spring Initializr で生成したプロジェクトを配置
- [ ] `frontend/` に `npm create vite@latest -- --template react-ts` で生成
- [ ] `.gitignore` に `application-local.yml` / `node_modules/` / `.env.local` 追加
- [ ] `application-local.yml.example` をコミット
- [ ] PostgreSQL ローカルセットアップ実施
- [ ] `V1__init.sql` を `db/migration/` に配置(01-database.md §8 の DDL)
- [ ] `V900__seed_dev.sql` を `db/migration-dev/` に配置
- [ ] `application.yml` に税率設定・Jackson 設定・CORS 値を記述
- [ ] `ClockConfig` Bean を作成
- [ ] backend 起動 → Flyway マイグレーション成功を確認
- [ ] frontend 起動 → `GET /api/products` が CORS エラーなく通ることを確認
