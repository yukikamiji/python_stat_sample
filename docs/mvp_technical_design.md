# MVP技術設計書: Web記事収集・AI要約アプリ

## 1. 前提とMVPスコープ

本設計書は、Python / Streamlit / SQLite / SQLAlchemy / OpenAI API / BeautifulSoup を用いた、個人開発向けのシンプルなMVPを前提とする。対象アプリは「URLを登録し、Webページ本文を抽出し、AIで要約して保存・閲覧する」ことを最小価値とする。

### MVPで実現すること

- ユーザーがURLを手動登録する
- BeautifulSoupでページ本文を取得・整形する
- OpenAI APIで要約を生成する
- URL、タイトル、本文、要約、処理状態をSQLiteに保存する
- Streamlit画面で記事一覧、詳細、要約結果を閲覧する
- 失敗時の原因を画面とDBに残し、再実行できるようにする

### MVPでやらないこと

- ユーザー認証
- 複数ユーザー対応
- 自動定期クローリング
- ベクトル検索
- 非同期ジョブキュー
- 高度な本文抽出アルゴリズム
- 管理画面や権限管理

これらは将来拡張を考慮した構成に留める。

## 2. システムアーキテクチャ

### 全体構成

```text
[Browser]
   |
   v
[Streamlit UI]
   |
   +--> [Application Services]
   |        |
   |        +--> ArticleService
   |        +--> CrawlService
   |        +--> SummaryService
   |
   +--> [SQLAlchemy Repository]
   |        |
   |        v
   |     [SQLite]
   |
   +--> [External APIs]
            |
            +--> Web Page HTTP Fetch
            +--> OpenAI API
```

### レイヤー構成

| レイヤー | 役割 | 主な技術 |
| --- | --- | --- |
| Presentation | 画面表示、入力受付、処理結果表示 | Streamlit |
| Application | ユースケース制御、サービス連携 | Python |
| Domain | 記事、要約、処理状態などの業務概念 | Python dataclass / Enum |
| Infrastructure | DBアクセス、HTTP取得、OpenAI API呼び出し | SQLAlchemy / requests / BeautifulSoup / OpenAI SDK |
| Storage | 永続化 | SQLite |

### MVPでの処理方針

- Streamlitから同期的に「取得 → 抽出 → 要約 → 保存」を実行する
- DBアクセスはSQLAlchemy ORM経由に統一する
- SQLite固有のSQLに依存せず、PostgreSQL移行を容易にする
- 外部API呼び出しはサービスクラスに閉じ込め、画面から直接呼ばない

## 3. ディレクトリ構成

```text
project-root/
├── app.py
├── requirements.txt
├── .env.example
├── README.md
├── data/
│   └── app.db
├── docs/
│   └── mvp_technical_design.md
├── src/
│   ├── __init__.py
│   ├── config.py
│   ├── database.py
│   ├── models/
│   │   ├── __init__.py
│   │   ├── article.py
│   │   └── summary.py
│   ├── repositories/
│   │   ├── __init__.py
│   │   └── article_repository.py
│   ├── services/
│   │   ├── __init__.py
│   │   ├── article_service.py
│   │   ├── crawl_service.py
│   │   └── summary_service.py
│   ├── clients/
│   │   ├── __init__.py
│   │   ├── openai_client.py
│   │   └── web_client.py
│   └── utils/
│       ├── __init__.py
│       ├── text.py
│       └── logging.py
└── tests/
    ├── test_crawl_service.py
    ├── test_summary_service.py
    └── test_article_repository.py
```

### 主要ファイルの責務

| ファイル | 責務 |
| --- | --- |
| `app.py` | Streamlitの画面定義とサービス呼び出し |
| `src/config.py` | 環境変数、DB URL、OpenAI設定の読み込み |
| `src/database.py` | SQLAlchemy engine / session / Base定義 |
| `src/models/` | SQLAlchemy ORMモデル |
| `src/repositories/` | DB CRUD処理 |
| `src/services/` | ユースケース単位の処理 |
| `src/clients/` | 外部API・HTTP通信の詳細 |
| `src/utils/` | テキスト整形、ログなどの共通処理 |

## 4. DB設計

### 設計方針

- MVPではSQLiteを使用する
- SQLAlchemy ORMを利用し、DB差し替え時の影響を抑える
- PostgreSQL移行を見据え、型や制約は一般的なRDBで扱いやすいものにする
- 要約履歴を保持できるよう、記事と要約を分離する
- 将来のクローリング、ベクトル検索に備えた拡張カラムを最小限用意する

### ER図

```text
articles 1 --- N summaries
articles 1 --- N crawl_logs
```

### テーブル一覧

| テーブル | 目的 |
| --- | --- |
| `articles` | URL、タイトル、本文、取得状態を管理 |
| `summaries` | 要約結果、利用モデル、プロンプト種別を管理 |
| `crawl_logs` | 取得・抽出・要約の実行ログを管理 |

## 5. テーブル定義

### articles

| カラム | 型 | 制約 | 説明 |
| --- | --- | --- | --- |
| `id` | Integer | PK | 記事ID |
| `url` | String(2048) | NOT NULL, UNIQUE | 記事URL |
| `title` | String(512) | NULL | ページタイトル |
| `content` | Text | NULL | 抽出済み本文 |
| `content_hash` | String(64) | NULL, INDEX | 本文のSHA-256ハッシュ |
| `source_domain` | String(255) | NULL, INDEX | URLのドメイン |
| `status` | String(32) | NOT NULL, INDEX | `pending` / `fetched` / `summarized` / `failed` |
| `error_message` | Text | NULL | 最新エラー内容 |
| `fetched_at` | DateTime | NULL | 最終取得日時 |
| `created_at` | DateTime | NOT NULL | 作成日時 |
| `updated_at` | DateTime | NOT NULL | 更新日時 |

### summaries

| カラム | 型 | 制約 | 説明 |
| --- | --- | --- | --- |
| `id` | Integer | PK | 要約ID |
| `article_id` | Integer | FK, NOT NULL, INDEX | `articles.id` |
| `summary_text` | Text | NOT NULL | 要約本文 |
| `model_name` | String(128) | NOT NULL | 使用したOpenAIモデル名 |
| `prompt_version` | String(64) | NOT NULL | プロンプトバージョン |
| `input_tokens` | Integer | NULL | 入力トークン数 |
| `output_tokens` | Integer | NULL | 出力トークン数 |
| `created_at` | DateTime | NOT NULL | 作成日時 |

### crawl_logs

| カラム | 型 | 制約 | 説明 |
| --- | --- | --- | --- |
| `id` | Integer | PK | ログID |
| `article_id` | Integer | FK, NULL, INDEX | 対象記事ID。URL登録前の失敗ではNULL可 |
| `url` | String(2048) | NOT NULL | 処理対象URL |
| `step` | String(64) | NOT NULL, INDEX | `fetch` / `parse` / `summarize` / `save` |
| `status` | String(32) | NOT NULL, INDEX | `success` / `failed` |
| `message` | Text | NULL | 詳細メッセージ |
| `created_at` | DateTime | NOT NULL | 作成日時 |

### SQLAlchemyモデル例

```python
class Article(Base):
    __tablename__ = "articles"

    id = Column(Integer, primary_key=True)
    url = Column(String(2048), nullable=False, unique=True)
    title = Column(String(512), nullable=True)
    content = Column(Text, nullable=True)
    content_hash = Column(String(64), nullable=True, index=True)
    source_domain = Column(String(255), nullable=True, index=True)
    status = Column(String(32), nullable=False, index=True, default="pending")
    error_message = Column(Text, nullable=True)
    fetched_at = Column(DateTime, nullable=True)
    created_at = Column(DateTime, nullable=False, default=datetime.utcnow)
    updated_at = Column(DateTime, nullable=False, default=datetime.utcnow, onupdate=datetime.utcnow)
```

## 6. API設計

MVPでは外部公開APIは作らず、Streamlit UIからApplication Serviceを直接呼び出す。将来的にFastAPI等を追加しやすいよう、サービスメソッドの入出力をAPI境界として設計する。

### Application Service API

#### ArticleService.register_url

| 項目 | 内容 |
| --- | --- |
| 目的 | URLを登録し、既存記事があれば返す |
| 入力 | `url: str` |
| 出力 | `Article` |
| 主なエラー | URL形式不正、DB保存失敗 |

#### ArticleService.process_article

| 項目 | 内容 |
| --- | --- |
| 目的 | URL取得、本文抽出、要約、保存を一括実行 |
| 入力 | `url: str` |
| 出力 | `ProcessResult(article_id, status, summary_id, error_message)` |
| 主なエラー | HTTPエラー、本文抽出失敗、OpenAI APIエラー |

#### ArticleService.list_articles

| 項目 | 内容 |
| --- | --- |
| 目的 | 記事一覧を取得する |
| 入力 | `status: str | None`, `keyword: str | None`, `limit: int`, `offset: int` |
| 出力 | `list[Article]` |
| 主なエラー | DB取得失敗 |

#### ArticleService.get_article_detail

| 項目 | 内容 |
| --- | --- |
| 目的 | 記事本文と最新要約を取得する |
| 入力 | `article_id: int` |
| 出力 | `ArticleDetail(article, latest_summary)` |
| 主なエラー | 記事不存在 |

### 将来のHTTP API案

| Method | Path | 目的 |
| --- | --- | --- |
| `POST` | `/articles` | URL登録 |
| `POST` | `/articles/{id}/process` | 取得・要約実行 |
| `GET` | `/articles` | 記事一覧 |
| `GET` | `/articles/{id}` | 記事詳細 |
| `GET` | `/summaries/{id}` | 要約詳細 |

## 7. 画面設計

### 画面一覧

| 画面 | 目的 | MVP優先度 |
| --- | --- | --- |
| URL登録・要約実行画面 | URL入力、要約実行、結果表示 | High |
| 記事一覧画面 | 保存済み記事の確認、検索、ステータス確認 | High |
| 記事詳細画面 | 本文、要約、エラー情報の確認 | High |
| 設定確認画面 | モデル名、DB接続先、プロンプトバージョン確認 | Low |

### URL登録・要約実行画面

```text
[URL入力欄]
[取得して要約ボタン]

処理中ステータス:
- URL検証
- ページ取得
- 本文抽出
- AI要約
- DB保存

結果:
- タイトル
- URL
- 要約
- 本文抜粋
- エラーがあればエラー詳細
```

### 記事一覧画面

```text
[キーワード検索]
[ステータスフィルタ]

| ID | タイトル | ドメイン | ステータス | 取得日時 | 詳細ボタン |
```

### 記事詳細画面

```text
タイトル
URL
ステータス

[最新要約]
[本文表示]
[再取得・再要約ボタン]
[エラーメッセージ]
[処理ログ]
```

### Streamlit実装方針

- `st.tabs()`で「URL登録」「記事一覧」「記事詳細」を分ける
- MVPではページ分割より単一`app.py`を優先する
- 処理中は`st.spinner()`を使用する
- 成功時は`st.success()`、失敗時は`st.error()`を使用する
- 詳細表示対象の記事IDは`st.session_state`で保持する

## 8. AI要約処理フロー

### 処理フロー

```text
1. URL入力
2. URL形式チェック
3. articlesにpendingで登録、または既存記事を取得
4. HTTP GETでHTML取得
5. BeautifulSoupでtitleと本文候補を抽出
6. 本文を正規化
7. 本文が長い場合はMVPでは先頭から上限文字数に切り詰め
8. OpenAI APIへ要約リクエスト
9. summariesに要約を保存
10. articles.statusをsummarizedに更新
11. 画面に要約を表示
```

### 本文抽出方針

MVPでは以下の順で本文を抽出する。

1. `article`タグ
2. `main`タグ
3. `body`タグ
4. 取得できない場合は失敗扱い

除去対象は以下とする。

- `script`
- `style`
- `nav`
- `footer`
- `header`
- `aside`
- `noscript`

### 要約プロンプト方針

```text
あなたはWeb記事を簡潔に要約するアシスタントです。
以下の本文を日本語で要約してください。

出力形式:
- 3行要約
- 重要ポイント5つ
- 想定読者

本文:
{content}
```

### トークン・文字数制御

- MVPでは厳密なトークン分割は行わず、文字数上限で制御する
- `MAX_CONTENT_CHARS=12000`程度を初期値とする
- 長文分割要約は将来対応とする
- モデル名、最大出力トークン、temperatureは環境変数で変更可能にする

## 9. クラス設計

### 主要クラス一覧

| クラス | 配置 | 責務 |
| --- | --- | --- |
| `Article` | `src/models/article.py` | 記事ORMモデル |
| `Summary` | `src/models/summary.py` | 要約ORMモデル |
| `CrawlLog` | `src/models/crawl_log.py` | 処理ログORMモデル |
| `ArticleRepository` | `src/repositories/article_repository.py` | 記事・要約・ログの永続化 |
| `WebClient` | `src/clients/web_client.py` | HTTP GET、タイムアウト制御 |
| `OpenAIClient` | `src/clients/openai_client.py` | OpenAI API呼び出し |
| `CrawlService` | `src/services/crawl_service.py` | HTML取得と本文抽出 |
| `SummaryService` | `src/services/summary_service.py` | 要約プロンプト生成と要約実行 |
| `ArticleService` | `src/services/article_service.py` | URL登録から要約保存までのユースケース制御 |

### ArticleService

```python
class ArticleService:
    def __init__(
        self,
        article_repository: ArticleRepository,
        crawl_service: CrawlService,
        summary_service: SummaryService,
    ):
        self.article_repository = article_repository
        self.crawl_service = crawl_service
        self.summary_service = summary_service

    def process_article(self, url: str) -> ProcessResult:
        """URL登録、本文取得、要約、保存を実行する。"""
```

### CrawlService

```python
class CrawlService:
    def __init__(self, web_client: WebClient):
        self.web_client = web_client

    def fetch_and_extract(self, url: str) -> ExtractedContent:
        """HTMLを取得し、titleと本文を抽出する。"""
```

### SummaryService

```python
class SummaryService:
    def __init__(self, openai_client: OpenAIClient, model_name: str):
        self.openai_client = openai_client
        self.model_name = model_name

    def summarize(self, content: str) -> SummaryResult:
        """本文から要約を生成する。"""
```

### DTO

```python
@dataclass
class ExtractedContent:
    title: str | None
    content: str

@dataclass
class SummaryResult:
    summary_text: str
    model_name: str
    input_tokens: int | None = None
    output_tokens: int | None = None

@dataclass
class ProcessResult:
    article_id: int | None
    status: str
    summary_id: int | None = None
    error_message: str | None = None
```

## 10. エラーハンドリング方針

### 基本方針

- 画面にはユーザーが理解できる短いメッセージを表示する
- DBには開発者が原因調査できる詳細メッセージを保存する
- 外部通信エラーは必ずタイムアウトを設定する
- 例外はApplication Service境界で捕捉し、`ProcessResult`として返す
- importに対するtry/exceptは使用しない

### エラー分類

| 分類 | 例 | 画面表示 | DB記録 |
| --- | --- | --- | --- |
| 入力エラー | URL形式不正 | URLを確認してください | validation failed |
| HTTPエラー | 404、500、timeout | ページ取得に失敗しました | HTTP status / timeout詳細 |
| 抽出エラー | 本文が空 | 本文を抽出できませんでした | parse failed |
| OpenAI APIエラー | rate limit、認証失敗 | 要約生成に失敗しました | API error detail |
| DBエラー | unique制約、接続失敗 | 保存に失敗しました | SQLAlchemy error detail |

### リトライ方針

- MVPでは自動リトライしない
- 画面上の「再取得・再要約」ボタンで手動再実行する
- 将来はHTTP 429 / 5xx / timeoutに限定して指数バックオフを導入する

### ログ方針

- `crawl_logs`に処理ステップ単位の結果を残す
- Python標準`logging`でもコンソールログを出す
- APIキーや本文全量などの機密・大量データはログに出さない

## 11. 将来拡張を考慮した構成

### PostgreSQL移行

- `DATABASE_URL`を環境変数化する
- SQLAlchemy ORMとAlembicを使う前提にする
- SQLite依存のSQL、`rowid`、独自関数を使わない
- `Text`、`DateTime`、`String`、`Integer`など一般的な型を優先する

### 自動クローリング対応

将来追加する想定コンポーネントは以下とする。

```text
scheduler/
├── crawler_job.py
└── source_feed_job.py
```

追加テーブル案:

| テーブル | 目的 |
| --- | --- |
| `crawl_sources` | 定期取得対象のサイト、RSS、URL一覧を管理 |
| `crawl_jobs` | クロールジョブの実行状態を管理 |

MVP時点では`crawl_logs`と`articles.source_domain`を用意しておくことで、移行しやすくする。

### ベクトル検索対応

将来追加する想定カラム・テーブルは以下とする。

| 対象 | 内容 |
| --- | --- |
| `article_embeddings` | 記事または要約のembeddingを保存 |
| `embedding_model` | 使用したembeddingモデル名 |
| `chunk_index` | 長文分割時のチャンク番号 |
| `vector_store_id` | 外部ベクトルDB利用時の参照ID |

MVPではembeddingを保存しないが、要約と本文を別テーブルで管理し、`article_id`を中心に拡張できる構成にする。

### 非同期処理対応

将来的に処理時間が長くなった場合は、以下の順で拡張する。

1. Streamlit同期処理のまま、処理ステップを細分化する
2. APScheduler等で定期実行を追加する
3. RQ / Celery等のジョブキューを追加する
4. Web UIとバックグラウンドワーカーを分離する

### APIサーバー対応

- Application ServiceをStreamlitから独立させておく
- FastAPIを追加する場合も同じサービスを再利用する
- 認証・認可はAPIサーバー追加時に設計する

## 12. 開発優先順位

### Phase 1: MVP必須

1. プロジェクト雛形作成
2. 設定読み込み、DB初期化
3. `articles`、`summaries`、`crawl_logs`のSQLAlchemyモデル作成
4. URL登録機能
5. HTTP取得とBeautifulSoup本文抽出
6. OpenAI API要約処理
7. 要約結果保存
8. StreamlitでURL入力と要約結果表示
9. 記事一覧・詳細表示
10. 基本的なエラー表示とログ保存

### Phase 2: MVP改善

1. キーワード検索
2. ステータスフィルタ
3. 再取得・再要約
4. プロンプトバージョン管理
5. Alembic導入
6. 単体テスト追加
7. 本文抽出精度改善

### Phase 3: 拡張準備

1. 自動クロール元管理
2. 定期実行ジョブ
3. 長文分割要約
4. embedding生成
5. ベクトル検索
6. PostgreSQL移行
7. FastAPI追加
8. ユーザー認証

## 13. MVP実装時の推奨環境変数

```text
DATABASE_URL=sqlite:///data/app.db
OPENAI_API_KEY=your_api_key
OPENAI_MODEL=gpt-4.1-mini
SUMMARY_PROMPT_VERSION=v1
HTTP_TIMEOUT_SECONDS=15
MAX_CONTENT_CHARS=12000
```

## 14. MVP完了条件

- Streamlit画面からURLを入力できる
- WebページのHTMLを取得できる
- 本文とタイトルを抽出できる
- OpenAI APIで日本語要約を生成できる
- 記事、要約、処理ログをSQLiteへ保存できる
- 保存済み記事の一覧と詳細を確認できる
- 失敗時に画面とDBで原因を確認できる
