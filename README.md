# study-app-laravel

書籍・動画コース・資格学習の進捗をセッション単位で記録し、ダッシュボードで可視化する学習記録管理アプリです。

## 技術スタック

| レイヤ | 採用技術 | バージョン |
|--------|----------|-----------|
| バックエンド | Laravel | 11.x |
| 実行環境 | PHP | 8.3 |
| フロントエンド | Inertia.js + Vue 3 + TypeScript | Inertia 2.x / Vue 3.x |
| ビルドツール | Vite | 5.x |
| UI スタイリング | Tailwind CSS | 3.x |
| グラフ | Chart.js + vue-chartjs | 4.x |
| データベース | MySQL | 8.0 |
| キャッシュ / セッション | Redis | 7.x |
| Web サーバー | Nginx | 1.25 |
| コンテナ | Docker / Docker Compose | — |

---

## 前提条件

ローカル開発を始める前に、以下がインストール済みであることを確認してください。

- **Docker** 24.0 以上 — [インストール手順](https://docs.docker.com/get-docker/)
- **Docker Compose** v2.20 以上（Docker Desktop に同梱）

PHP・Node.js・Composer はホストマシンへのインストール不要です。すべてコンテナ内で完結します。

---

## ローカル開発環境のセットアップ

### 1. リポジトリのクローン

```bash
git clone <repository-url> study-app-laravel
cd study-app-laravel
```

### 2. 環境変数ファイルの作成

```bash
cp .env.example .env
```

`.env` は開発用のデフォルト値がすでに設定されています。通常の開発では変更不要ですが、
必要に応じて以下の値を確認してください。

```dotenv
APP_NAME="学習記録アプリ"
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=study_app
DB_USERNAME=app_user
DB_PASSWORD=secret

CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

REDIS_HOST=redis
REDIS_PORT=6379

FILESYSTEM_DISK=local
```

### 3. Docker コンテナのビルド・起動

```bash
docker compose up -d --build
```

初回は Docker イメージのビルドに数分かかります。起動後は以下のサービスが利用できます。

| サービス名 | 役割 | アクセス先 |
|-----------|------|-----------|
| `app` | PHP-FPM 8.3（Laravel 本体） | — |
| `nginx` | Web サーバー（リバースプロキシ） | http://localhost |
| `db` | MySQL 8.0 | localhost:3306 |
| `redis` | Redis 7（セッション・キャッシュ） | localhost:6379 |
| `vite` | Vite 開発サーバー（Hot Module Replacement） | http://localhost:5173 |

コンテナの起動状態を確認するには:

```bash
docker compose ps
```

### 4. アプリケーションキーの生成

```bash
docker compose exec app php artisan key:generate
```

### 5. データベースのマイグレーション＆初期シーディング

```bash
docker compose exec app php artisan migrate --seed
```

マイグレーションのみ実行する場合:

```bash
docker compose exec app php artisan migrate
```

### 6. ストレージのシンボリックリンク作成

サムネイル画像などのアップロードファイルを公開するために実行します。

```bash
docker compose exec app php artisan storage:link
```

### 7. 動作確認

ブラウザで http://localhost を開き、ログイン画面が表示されれば完了です。

---

## 開発ワークフロー

### よく使うコマンド

```bash
# コンテナ内で Artisan コマンドを実行
docker compose exec app php artisan <コマンド>

# コンテナ内でコマンドを実行（シェル）
docker compose exec app bash

# Composer パッケージの追加
docker compose exec app composer require <パッケージ名>

# npm パッケージの追加（Vite コンテナで実行）
docker compose exec vite npm install <パッケージ名>

# フロントエンドのビルド（開発用 HMR は vite サービスが自動起動）
docker compose exec vite npm run build

# Laravel のルーティング確認
docker compose exec app php artisan route:list
```

### ログの確認

```bash
# Laravel アプリケーションログ（リアルタイム）
docker compose exec app tail -f storage/logs/laravel.log

# Nginx アクセスログ
docker compose logs nginx -f

# 全サービスのログ
docker compose logs -f
```

### データベース操作

```bash
# MySQL に接続
docker compose exec db mysql -u app_user -psecret study_app

# マイグレーションのロールバック
docker compose exec app php artisan migrate:rollback

# データベースのリフレッシュ（全マイグレーションをやり直し）
docker compose exec app php artisan migrate:fresh --seed
```

---

## テスト

```bash
# PHPUnit（Feature + Unit テスト）
docker compose exec app php artisan test

# 特定のテストクラスのみ実行
docker compose exec app php artisan test --filter=ContentControllerTest

# カバレッジレポートの生成（HTML）
docker compose exec app php artisan test --coverage-html storage/coverage

# Vitest（フロントエンドユニットテスト）
docker compose exec vite npm run test

# Vitest（ウォッチモード）
docker compose exec vite npm run test:watch
```

---

## コンテナの停止・削除

```bash
# コンテナの停止（データは保持）
docker compose stop

# コンテナの削除（データは保持）
docker compose down

# コンテナ・ボリューム（DB データ含む）を完全削除
docker compose down -v
```

---

## 本番環境へのデプロイ

本番用の設定は `docker-compose.prod.yml` に分離されています。

```bash
# 本番環境の起動
docker compose -f docker-compose.prod.yml up -d --build

# マイグレーションの実行
docker compose -f docker-compose.prod.yml exec app php artisan migrate --force

# 各種キャッシュの生成（本番では必須）
docker compose -f docker-compose.prod.yml exec app php artisan config:cache
docker compose -f docker-compose.prod.yml exec app php artisan route:cache
docker compose -f docker-compose.prod.yml exec app php artisan view:cache
docker compose -f docker-compose.prod.yml exec app php artisan event:cache
```

本番環境では `.env` の以下の値を必ず変更してください。

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_KEY=<php artisan key:generate で生成>
APP_URL=https://your-domain.com

DB_PASSWORD=<強力なパスワード>
```

---

## ディレクトリ構成（主要部分）

```
study-app-laravel/
├── app/
│   ├── Enums/                       # PHP 8.1+ バックド列挙型
│   │   ├── ContentType.php          #   BOOK / VIDEO / CERTIFICATION
│   │   └── ContentStatus.php        #   NOT_STARTED / IN_PROGRESS / ...
│   ├── Http/
│   │   ├── Controllers/             # リクエスト処理・Inertia レスポンス返却
│   │   ├── Requests/                # Form Request（バリデーション）
│   │   └── Resources/               # API Resource（JSON 整形）
│   ├── Models/                      # Eloquent モデル
│   ├── Policies/                    # 認可ロジック（Gate / Policy）
│   └── Services/                    # ビジネスロジック（Controller を薄く保つ）
├── database/
│   ├── migrations/                  # スキーマ定義
│   └── factories/                   # テスト用ファクトリ
├── resources/
│   ├── js/
│   │   ├── Pages/                   # Inertia ページコンポーネント（Vue）
│   │   ├── Components/              # 再利用可能な Vue コンポーネント
│   │   ├── Layouts/                 # ページレイアウト（AppLayout.vue 等）
│   │   ├── composables/             # Vue 3 Composable（useContentForm 等）
│   │   └── types.ts                 # TypeScript 型定義
│   └── views/
│       └── app.blade.php            # Inertia エントリーポイント
├── routes/
│   ├── web.php                      # Web ルート
│   └── auth.php                     # 認証ルート（Breeze 生成）
├── docker/
│   ├── nginx/
│   │   └── default.conf             # Nginx 設定
│   └── php/
│       ├── Dockerfile               # PHP-FPM イメージ定義
│       └── php.ini                  # PHP 設定
├── tests/
│   ├── Feature/                     # 機能テスト（HTTP リクエスト〜DB まで）
│   └── Unit/                        # ユニットテスト（単一クラス）
├── docker-compose.yml               # 開発環境
├── docker-compose.prod.yml          # 本番環境
├── vite.config.ts
└── .env.example
```

---

## トラブルシューティング

### コンテナが起動しない

```bash
# ログを確認
docker compose logs app
docker compose logs db
```

### 権限エラー（storage/ や bootstrap/cache/）

```bash
docker compose exec app chmod -R 775 storage bootstrap/cache
docker compose exec app chown -R www-data:www-data storage bootstrap/cache
```

### ポート競合

デフォルトでは 80 番（Nginx）・3306 番（MySQL）・6379 番（Redis）・5173 番（Vite）を使用します。
競合する場合は `docker-compose.yml` の `ports` セクションを変更してください。

```yaml
# 例: Nginx を 8080 番に変更
ports:
  - "8080:80"
```

### MySQL への初回接続が失敗する

コンテナ起動直後は MySQL の初期化に数秒かかります。少し待ってから再度実行してください。

```bash
docker compose exec db mysqladmin -u app_user -psecret ping --wait=30
```
