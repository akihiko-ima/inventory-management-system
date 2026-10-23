# アプリケーションの使い方

本ドキュメントでは、在庫管理システム（FastAPI Full Stack Template ベース）をローカル環境で起動し、利用するまでの手順と、`.env` ファイルに記載する設定項目について解説します。

---

## 1. 構成概要

| コンポーネント | 技術                                         | 役割                                                  |
| -------------- | -------------------------------------------- | ----------------------------------------------------- |
| backend        | FastAPI / SQLModel / Alembic                 | REST API（`/api/v1`）、ビルド済みフロントエンドの配信 |
| frontend       | React / Vite / TanStack Router（bun で管理） | 管理画面 UI                                           |
| db             | PostgreSQL 18                                | データ永続化                                          |
| mailpit        | Mailpit                                      | 開発用のメール受信サーバー（実際には送信しない）      |
| adminer        | Adminer                                      | DB を Web ブラウザから操作するツール                  |
| pgadmin        | pgAdmin 4                                    | DB を Web ブラウザから操作・管理するツール（開発時のみ） |
| proxy          | Traefik                                      | リバースプロキシ（Docker Compose 利用時）             |

---

## 2. 事前準備

以下のツールをインストールしておきます。

- [Docker](https://www.docker.com/)（Docker Compose v2 を含む）
- [uv](https://docs.astral.sh/uv/)（Python パッケージ管理。Python バージョンは `.python-version` を参照）
- [bun](https://bun.sh/)（フロントエンドの依存管理・開発サーバー）

### `.env` ファイルの作成

`.env` は `.gitignore` の対象のため、リポジトリには含まれていません。サンプルからコピーして作成します。

```bash
cp .env.sample .env
```

作成後、必要に応じて値を編集してください（各項目の意味は「[5. `.env` ファイルの設定項目](#5-env-ファイルの設定項目)」を参照）。

> **注意**: `SECRET_KEY` / `FIRST_SUPERUSER_PASSWORD` / `POSTGRES_PASSWORD` が `changethis` のままだと、`FASTAPI_ENV=development` 以外の環境ではバックエンドが起動エラーになります（開発環境では警告のみ）。

---

## 3. 起動方法

起動方法は 2 通りあります。開発時は「方法 A」を推奨します。

### 方法 A: ローカル開発（DB・メール・pgAdmin のみ Docker）

1. DB、Mailpit、pgAdmin を起動します。

   ```bash
   docker compose up -d db mailpit pgadmin
   ```

2. `backend` ディレクトリで依存関係をインストールし、DB を初期化します。

   ```bash
   cd backend
   uv sync
   uv run bash scripts/prestart.sh
   ```

   `prestart.sh` は以下を実行します。
   - `alembic upgrade head` … マイグレーションを適用してテーブルを作成
   - `python app/initial_data.py` … `.env` の `FIRST_SUPERUSER` で初期管理者ユーザーを作成

3. バックエンドの開発サーバーを起動します。

   ```bash
   uv run fastapi dev
   ```

4. 別ターミナルで、**プロジェクトルート**からフロントエンドを起動します。

   ```bash
   bun install
   bun run dev
   ```

5. ブラウザで以下にアクセスします。

   | URL                          | 内容                                |
   | ---------------------------- | ----------------------------------- |
   | <http://localhost:5173>      | フロントエンド（Vite 開発サーバー） |
   | <http://localhost:8000>      | バックエンド API                    |
   | <http://localhost:8000/docs> | Swagger UI（API ドキュメント）      |
   | <http://localhost:8025>      | Mailpit（送信メールの確認）         |
   | <http://localhost:5050>      | pgAdmin（DB 管理）                  |

> フロントエンドの API 接続先は `frontend/.env` の `VITE_API_URL`（既定値: `http://localhost:8000`）で設定されています。

#### フロントエンドを FastAPI から配信する場合

```bash
cd frontend
bun run build
```

ビルド結果は `backend/app/frontend` に出力され、<http://localhost:8000> でアクセスできます。フロントエンドを変更した場合は再ビルドが必要です。

### 方法 B: すべて Docker Compose で起動

```bash
docker compose run --rm backend bash scripts/prestart.sh
docker compose watch
```

| URL                          | 内容                                     |
| ---------------------------- | ---------------------------------------- |
| <http://localhost:8000>      | アプリケーション（フロントエンド + API） |
| <http://localhost:8000/docs> | Swagger UI                               |
| <http://localhost:8080>      | Adminer（DB 管理）                       |
| <http://localhost:8090>      | Traefik ダッシュボード                   |
| <http://localhost:8025>      | Mailpit                                  |
| <http://localhost:5050>      | pgAdmin（DB 管理）                       |

- 方法 A で起動中の FastAPI サーバーがある場合は、ポート `8000` が競合するため停止してから起動してください。
- 初回起動時は全サービスの準備に 1 分程度かかることがあります。`docker compose logs backend` で状況を確認できます。
- `.env` を変更した場合は、スタックを再起動してください。

#### pgAdmin の使い方

pgAdmin は開発用のため `compose.override.yml` に定義しており、本番デプロイ（`compose.deploy.yml`）には含まれません。方法 A・方法 B のどちらでも一緒に起動します。

1. <http://localhost:5050> を開き、`.env` の `FIRST_SUPERUSER`（メールアドレス）と `FIRST_SUPERUSER_PASSWORD` でログインします。
2. 「Add New Server」でサーバーを登録します。

   | 項目                 | 値                                  |
   | -------------------- | ----------------------------------- |
   | Host name/address    | `db`                                |
   | Port                 | `5432`                              |
   | Maintenance database | `app`                               |
   | Username             | `postgres`                          |
   | Password             | `.env` の `POSTGRES_PASSWORD` の値 |

> **注意**: pgAdmin のログイン情報は初回起動時に `pgadmin-data` ボリュームへ保存されます。後から `.env` の `FIRST_SUPERUSER` / `FIRST_SUPERUSER_PASSWORD` を変更しても反映されないため、変更を反映したい場合は `docker compose down` の後にボリュームを削除してから再起動してください（ボリューム名は `docker volume ls` で確認できます。例: `docker volume rm inventory-management-system_pgadmin-data`）。

---

## 4. アプリケーションの操作

### ログイン

1. フロントエンドを開くとログイン画面が表示されます。
2. `.env` の `FIRST_SUPERUSER` / `FIRST_SUPERUSER_PASSWORD` でログインします（管理者権限）。

### 主な画面

| 画面             | パス                | 内容                                                              |
| ---------------- | ------------------- | ----------------------------------------------------------------- |
| ダッシュボード   | `/`                 | ログインユーザーの情報表示                                        |
| Items            | `/items`            | アイテム（在庫）の一覧・追加・編集・削除                          |
| Admin            | `/admin`            | ユーザー管理（管理者のみ）                                        |
| User Settings    | `/settings`         | プロフィール・パスワードの変更、アカウント削除                    |
| Sign Up          | `/signup`           | 新規ユーザー登録                                                  |
| パスワード再設定 | `/recover-password` | 登録メールアドレスへ再設定メールを送信（開発時は Mailpit で確認） |

### API を直接利用する場合

<http://localhost:8000/docs> の Swagger UI から試せます。

1. 右上の **Authorize** をクリックし、ユーザー名（メールアドレス）とパスワードを入力します。
2. 以降、認証が必要なエンドポイントを実行できます。

主なエンドポイント（プレフィックス `/api/v1`）:

| メソッド             | パス                   | 内容                                   |
| -------------------- | ---------------------- | -------------------------------------- |
| POST                 | `/login/access-token`  | ログイン（アクセストークン取得）       |
| GET / POST           | `/items/`              | アイテム一覧取得 / 作成                |
| GET / PUT / DELETE   | `/items/{id}`          | アイテム取得 / 更新 / 削除             |
| GET / PATCH / DELETE | `/users/me`            | 自分のユーザー情報の取得 / 更新 / 削除 |
| POST                 | `/users/signup`        | ユーザー登録                           |
| GET / POST           | `/users/`              | ユーザー一覧 / 作成（管理者のみ）      |
| GET                  | `/utils/health-check/` | ヘルスチェック                         |

> `FASTAPI_ENV=development` の場合のみ、開発用の `/private/users/` エンドポイントも有効になります。

---

## 5. `.env` ファイルの設定項目

`.env` はプロジェクトルートに配置し、以下の 2 か所から読み込まれます。

- **バックエンド**: `backend/app/core/config.py` の `Settings` クラスが `../.env` を読み込みます。
- **Docker Compose**: `compose.yml` 内の `${変数名}` の展開に使われ、必要な値がコンテナへ渡されます。

> `.env` 内のホスト名（`localhost`）はローカル実行用です。Docker Compose で起動した場合、DB ホストは `db`、SMTP ホストは `mailpit` に自動で上書きされます。

### 5.1 `.env.sample` に記載されている項目

| 変数名                     | 必須 | サンプル値                                                      | 説明                                                                                                                                                                                                                                                               |
| -------------------------- | ---- | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `FASTAPI_ENV`              | 任意 | `development`                                                   | `development` を指定すると開発モードになります。`changethis` などの既定パスワードが警告のみで許容され、開発用 `/private` API が有効になり、Sentry が無効になります。本番では削除（未設定）してください。                                                           |
| `PROJECT_NAME`             | 必須 | `"Full Stack FastAPI Project"`                                  | アプリケーション名。API ドキュメントのタイトルや、メール送信者名（`EMAILS_FROM_NAME` 未設定時）に使われます。                                                                                                                                                      |
| `SECRET_KEY`               | 必須 | `changethis`                                                    | JWT アクセストークンの署名に使う秘密鍵。本番では必ずランダムな値に変更してください。生成例: `openssl rand -hex 32`                                                                                                                                                |
| `FIRST_SUPERUSER`          | 必須 | `admin@example.com`                                             | 初期データ作成時（`prestart.sh`）に作成される管理者ユーザーのメールアドレス。ログイン ID になります。pgAdmin のログインメールアドレスにも使われます。                                                                                                                                                        |
| `FIRST_SUPERUSER_PASSWORD` | 必須 | `changethis`                                                    | 上記管理者ユーザーのパスワード。pgAdmin のログインパスワードにも使われます。本番では必ず変更してください。                                                                                                                                                                                                     |
| `SMTP_HOST`                | 任意 | `localhost`                                                     | メール送信に使う SMTP サーバーのホスト名。開発時は Mailpit を指します。`SMTP_HOST` と `EMAILS_FROM_EMAIL` の両方が設定されている場合のみメール送信が有効になります。                                                                                               |
| `EMAILS_FROM_EMAIL`        | 任意 | `info@example.com`                                              | 送信メールの差出人アドレス。                                                                                                                                                                                                                                       |
| `SMTP_TLS`                 | 任意 | `False`                                                         | SMTP 接続で STARTTLS を使うか。既定値は `True`。Mailpit を使う開発時は `False` にします。                                                                                                                                                                          |
| `SMTP_PORT`                | 任意 | `1025`                                                          | SMTP サーバーのポート番号。既定値は `587`。Mailpit は `1025`。                                                                                                                                                                                                     |
| `POSTGRES_PASSWORD`        | 必須 | `changethis`                                                    | PostgreSQL の `postgres` ユーザーのパスワード。Docker Compose の `db` コンテナ作成時、および `DATABASE_URL` の組み立てに使われます。本番では必ず変更してください。                                                                                                 |
| `DATABASE_URL`             | 必須 | `postgresql://postgres:${POSTGRES_PASSWORD}@localhost:5432/app` | バックエンドが接続する DB の URL。形式は `postgresql://<ユーザー>:<パスワード>@<ホスト>:<ポート>/<DB名>`。`postgresql://` は内部で自動的に `postgresql+psycopg://` に変換されます。Docker Compose 利用時はホストが `db` に置き換えられた値がコンテナに渡されます。 |

### 5.2 必要に応じて追加できる項目

`.env.sample` には記載されていませんが、`.env` に追記することで設定を変更できます。

| 変数名                           | 既定値                  | 説明                                                                                      |
| -------------------------------- | ----------------------- | ----------------------------------------------------------------------------------------- |
| `SMTP_USER`                      | なし                    | SMTP 認証のユーザー名。                                                                   |
| `SMTP_PASSWORD`                  | なし                    | SMTP 認証のパスワード。                                                                   |
| `SMTP_SSL`                       | `False`                 | SMTP 接続で SSL（SMTPS）を使うか。                                                        |
| `EMAILS_FROM_NAME`               | `PROJECT_NAME` の値     | 送信メールの差出人名。                                                                    |
| `EMAIL_RESET_TOKEN_EXPIRE_HOURS` | `48`                    | パスワード再設定メールのリンク有効期限（時間）。                                          |
| `EMAIL_TEST_USER`                | `test@example.com`      | テスト用のメールアドレス。                                                                |
| `ACCESS_TOKEN_EXPIRE_MINUTES`    | `11520`（8 日）         | ログイン後のアクセストークン有効期限（分）。                                              |
| `FRONTEND_HOST`                  | `http://localhost:5173` | フロントエンドの URL。CORS の許可オリジン、およびメール内リンクの生成に使われます。       |
| `SENTRY_DSN`                     | なし                    | Sentry のエラー監視 DSN。`FASTAPI_ENV=development` 以外のときのみ有効です。               |
| `DOMAIN`                         | `localhost`             | Docker Compose（Traefik）で使うドメイン名。デプロイ用 `compose.deploy.yml` では必須です。 |

### 5.3 `.env` の記載例（ローカル開発用）

```dotenv
# 開発モード
FASTAPI_ENV=development

PROJECT_NAME="Inventory Management System"

# セキュリティ（本番では必ず変更）
SECRET_KEY=changethis
FIRST_SUPERUSER=admin@example.com
FIRST_SUPERUSER_PASSWORD=changethis

# メール（Mailpit）
SMTP_HOST=localhost
EMAILS_FROM_EMAIL=info@example.com
SMTP_TLS=False
SMTP_PORT=1025

# PostgreSQL
POSTGRES_PASSWORD=changethis
DATABASE_URL=postgresql://postgres:${POSTGRES_PASSWORD}@localhost:5432/app
```

### 5.4 フロントエンド用 `frontend/.env`

フロントエンドは別途 `frontend/.env` を読み込みます。

| 変数名         | 既定値                  | 説明                                                      |
| -------------- | ----------------------- | --------------------------------------------------------- |
| `VITE_API_URL` | `http://localhost:8000` | フロントエンドから呼び出すバックエンド API の URL。       |
| `MAILPIT_HOST` | `http://localhost:8025` | E2E テスト（Playwright）で Mailpit を参照するための URL。 |

---

## 6. テスト

```bash
# バックエンド（DB 起動済みであること）
cd backend
uv run bash scripts/test.sh

# フロントエンド E2E（Playwright）
bun run test
```

## 7. 参考

- [development.md](../development.md) … 開発手順（英語）
- [backend/README.md](../backend/README.md) … バックエンドの詳細
- [frontend/README.md](../frontend/README.md) … フロントエンドの詳細
