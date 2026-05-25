# 学習記録管理アプリ 仕様書（Laravel 版）

> **このドキュメントは AI が自立実装するための完全仕様書です。**
> 不明点があれば ADR または README.md を参照してください。

---

## 1. アプリ概要

| 項目 | 内容 |
|------|------|
| アプリ名 | study-app-laravel |
| 目的 | 書籍・動画コース・資格学習の進捗をセッション単位で記録し、ダッシュボードで可視化する |
| 主な利用者 | 個人（マルチユーザー対応設計） |
| プラットフォーム | Web アプリ（PC ブラウザ中心） |
| AI 機能 | なし |

---

## 2. 技術スタック

| レイヤ | 採用技術 | バージョン目安 |
|--------|----------|---------------|
| バックエンド | Laravel | 11.x |
| 実行環境 | PHP | 8.3 |
| フロントエンド | Inertia.js + Vue 3 + TypeScript | Inertia 2.x / Vue 3.x |
| ビルドツール | Vite | 5.x |
| UI スタイリング | Tailwind CSS | 3.x |
| グラフ | Chart.js + vue-chartjs | 4.x |
| カレンダーヒートマップ | vue-cal-heatmap または自作コンポーネント | — |
| 認証 | Laravel Breeze（Inertia + Vue スタック） | — |
| データベース | MySQL | 8.0 |
| キャッシュ / セッション | Redis | 7.x |
| Web サーバー | Nginx | 1.25 |
| コンテナ | Docker / Docker Compose | — |
| Linter / Formatter | PHP-CS-Fixer (PHP) + ESLint + Prettier (JS/TS) | — |

### 主要 Composer パッケージ

```
require:
  laravel/framework: ^11.0
  laravel/breeze: ^2.0
  inertiajs/inertia-laravel: ^2.0

require-dev:
  phpunit/phpunit: ^11.0
  fakerphp/faker: ^1.23
  laravel/pint: ^1.0
```

### 主要 npm パッケージ

```
dependencies:
  vue
  @inertiajs/vue3
  @vitejs/plugin-vue
  tailwindcss
  chart.js
  vue-chartjs
  @vueuse/core
  axios

devDependencies:
  typescript
  vite
  laravel-vite-plugin
  eslint
  prettier
  prettier-plugin-tailwindcss
  vitest
  @vue/test-utils
```

---

## 3. ディレクトリ構成

```
study-app-laravel/
├── app/
│   ├── Enums/
│   │   ├── ContentType.php          # BOOK / VIDEO / CERTIFICATION
│   │   └── ContentStatus.php        # NOT_STARTED / IN_PROGRESS / COMPLETED / PASSED / FAILED
│   ├── Http/
│   │   ├── Controllers/
│   │   │   ├── DashboardController.php
│   │   │   ├── ContentController.php
│   │   │   ├── SessionController.php
│   │   │   ├── TagController.php
│   │   │   ├── ExportController.php
│   │   │   └── SettingsController.php
│   │   ├── Requests/
│   │   │   ├── StoreContentRequest.php
│   │   │   ├── UpdateContentRequest.php
│   │   │   ├── UpdateContentStatusRequest.php
│   │   │   ├── UpdateContentTagsRequest.php
│   │   │   ├── UpdateContentNotesRequest.php
│   │   │   ├── StoreSessionRequest.php
│   │   │   └── StoreTagRequest.php
│   │   └── Resources/
│   │       ├── ContentResource.php
│   │       ├── ContentCollection.php
│   │       ├── SessionResource.php
│   │       └── TagResource.php
│   ├── Models/
│   │   ├── User.php
│   │   ├── LearningContent.php
│   │   ├── LearningSession.php
│   │   └── Tag.php
│   ├── Policies/
│   │   ├── ContentPolicy.php
│   │   └── SessionPolicy.php
│   └── Services/
│       ├── DashboardService.php     # ダッシュボード集計ロジック
│       └── ContentQueryService.php  # コンテンツ一覧フィルタ・ソート・ページネーション
├── database/
│   ├── migrations/
│   │   ├── xxxx_create_learning_contents_table.php
│   │   ├── xxxx_create_learning_sessions_table.php
│   │   ├── xxxx_create_tags_table.php
│   │   └── xxxx_create_content_tag_table.php
│   ├── seeders/
│   │   └── DatabaseSeeder.php   # サンプルユーザ1名 + コンテンツ10件 + セッション30件
│   └── factories/
│       ├── LearningContentFactory.php
│       ├── LearningSessionFactory.php
│       └── TagFactory.php
├── resources/
│   ├── js/
│   │   ├── app.ts                   # Inertia エントリーポイント
│   │   ├── types.ts                 # TypeScript 型定義（Inertia shared data 等）
│   │   ├── Pages/
│   │   │   ├── Auth/
│   │   │   │   ├── Login.vue        # ログイン
│   │   │   │   ├── Register.vue     # ユーザー登録
│   │   │   │   ├── ForgotPassword.vue
│   │   │   │   └── ResetPassword.vue
│   │   │   ├── Dashboard.vue        # ダッシュボード
│   │   │   ├── Error.vue            # エラーページ（403/404/500 統一）
│   │   │   ├── Contents/
│   │   │   │   ├── Index.vue        # 一覧
│   │   │   │   ├── Show.vue         # 詳細 + セッション履歴
│   │   │   │   ├── Create.vue       # 新規追加
│   │   │   │   └── Edit.vue         # 編集
│   │   │   └── Settings/
│   │   │       └── Index.vue        # 設定（タグ管理・エクスポート）
│   │   ├── Components/
│   │   │   ├── Dashboard/
│   │   │   │   ├── StatCard.vue
│   │   │   │   ├── StudySummary.vue
│   │   │   │   ├── CalendarHeatmap.vue
│   │   │   │   ├── ContentTypeChart.vue
│   │   │   │   ├── StudyTimeLineChart.vue
│   │   │   │   └── InProgressList.vue
│   │   │   ├── Contents/
│   │   │   │   ├── ContentCard.vue
│   │   │   │   ├── ContentForm.vue
│   │   │   │   ├── StatusBadge.vue
│   │   │   │   └── TagBadge.vue
│   │   │   ├── Sessions/
│   │   │   │   ├── SessionForm.vue
│   │   │   │   └── SessionList.vue
│   │   │   └── Layout/
│   │   │       ├── Sidebar.vue
│   │   │       ├── Header.vue
│   │   │       └── AppLayout.vue
│   │   ├── Layouts/
│   │   │   └── AppLayout.vue        # 認証済みページ共通レイアウト
│   │   └── composables/
│   │       ├── useContentForm.ts    # コンテンツフォームロジック
│   │       ├── useSessionTimer.ts   # タイマー（localStorage）ロジック
│   │       └── useFlash.ts          # フラッシュメッセージ
│   └── views/
│       └── app.blade.php            # Inertia エントリーポイント HTML
├── routes/
│   ├── web.php
│   └── auth.php
├── storage/
│   └── app/
│       └── public/
│           └── thumbnails/          # アップロード画像
├── tests/
│   ├── Feature/
│   │   ├── Auth/
│   │   ├── ContentTest.php
│   │   ├── SessionTest.php
│   │   └── TagTest.php
│   └── Unit/
│       ├── ContentQueryServiceTest.php
│       └── DashboardServiceTest.php
├── docker/
│   ├── nginx/
│   │   └── default.conf
│   └── php/
│       ├── Dockerfile
│       └── php.ini
├── docker-compose.yml
├── docker-compose.prod.yml
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
└── .env.example
```

---

## 4. データモデル（DB スキーマ）

Laravel マイグレーションで定義する。

### learning_contents

```php
// database/migrations/xxxx_create_learning_contents_table.php
Schema::create('learning_contents', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('title', 100);
    $table->string('content_type', 20); // ContentType Enum: BOOK / VIDEO / CERTIFICATION
    $table->string('author', 100)->nullable();
    $table->string('url', 2048)->nullable();
    $table->text('description')->nullable();
    $table->string('status', 20)->default('NOT_STARTED'); // ContentStatus Enum
    $table->text('notes')->nullable();
    $table->string('thumbnail_path')->nullable(); // Storage パス（公開 URL ではなくパスを保存）
    $table->timestamps();

    $table->index('user_id');
    $table->index('status');
});
```

### learning_sessions

```php
// database/migrations/xxxx_create_learning_sessions_table.php
Schema::create('learning_sessions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->foreignId('content_id')
          ->references('id')->on('learning_contents')
          ->cascadeOnDelete();
    $table->dateTime('started_at');
    $table->dateTime('ended_at');
    // duration_minutes は DB の generated column ではなく Eloquent accessor で算出する
    $table->text('memo')->nullable();
    $table->timestamp('created_at')->useCurrent();

    $table->index('user_id');
    $table->index('content_id');
    $table->index('started_at');
});
```

### tags

```php
// database/migrations/xxxx_create_tags_table.php
Schema::create('tags', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->cascadeOnDelete();
    $table->string('name', 20);
    $table->timestamp('created_at')->useCurrent();

    $table->unique(['user_id', 'name']);
    $table->index('user_id');
});
```

### content_tag（中間テーブル）

```php
// database/migrations/xxxx_create_content_tag_table.php
Schema::create('content_tag', function (Blueprint $table) {
    $table->foreignId('content_id')
          ->references('id')->on('learning_contents')
          ->cascadeOnDelete();
    $table->foreignId('tag_id')
          ->references('id')->on('tags')
          ->cascadeOnDelete();
    $table->primary(['content_id', 'tag_id']);
});
```

### Eloquent モデル例

```php
// app/Models/LearningContent.php
class LearningContent extends Model
{
    use HasFactory;

    protected $casts = [
        'content_type' => ContentType::class,
        'status'       => ContentStatus::class,
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function sessions(): HasMany
    {
        return $this->hasMany(LearningSession::class, 'content_id');
    }

    public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class, 'content_tag');
    }

    // duration_minutes を Eloquent accessor で算出
    public function getTotalStudyMinutesAttribute(): int
    {
        return (int) $this->sessions()->sum(
            DB::raw('TIMESTAMPDIFF(MINUTE, started_at, ended_at)')
        );
    }
}
```

```php
// app/Models/LearningSession.php
class LearningSession extends Model
{
    public $timestamps = false;

    protected $casts = [
        'started_at' => 'datetime',
        'ended_at'   => 'datetime',
    ];

    public function getDurationMinutesAttribute(): int
    {
        return (int) $this->started_at->diffInMinutes($this->ended_at);
    }
}
```

### ステータス利用ルール

| content_type | 使用できるステータス |
|---|---|
| BOOK | NOT_STARTED / IN_PROGRESS / COMPLETED |
| VIDEO | NOT_STARTED / IN_PROGRESS / COMPLETED |
| CERTIFICATION | NOT_STARTED / IN_PROGRESS / PASSED / FAILED |

---

## 5. ルーティング（`routes/web.php`）

```php
// 認証不要ルートは auth.php で定義（Laravel Breeze が自動生成）

Route::middleware(['auth', 'verified'])->group(function () {
    // ダッシュボード
    Route::get('/', [DashboardController::class, 'index'])->name('dashboard');

    // 学習コンテンツ
    Route::resource('contents', ContentController::class);
    Route::patch('contents/{content}/status', [ContentController::class, 'updateStatus'])
         ->name('contents.status');
    Route::patch('contents/{content}/tags',   [ContentController::class, 'updateTags'])
         ->name('contents.tags');
    Route::patch('contents/{content}/notes',  [ContentController::class, 'updateNotes'])
         ->name('contents.notes');

    // 学習セッション（コンテンツに紐づく）
    Route::post  ('contents/{content}/sessions',        [SessionController::class, 'store'])
         ->name('sessions.store');
    Route::delete('contents/{content}/sessions/{session}', [SessionController::class, 'destroy'])
         ->name('sessions.destroy');

    // タグ
    Route::apiResource('tags', TagController::class)->only(['index', 'store', 'update', 'destroy']);

    // エクスポート
    Route::get('export', [ExportController::class, 'download'])->name('export');

    // 設定
    Route::get('settings', [SettingsController::class, 'index'])->name('settings');

    // エクスポート（再掲：ExportController に分離）
    // Route::get('export', ...) は ExportController に実装
});
```

---

## 6. 画面一覧と仕様

### 6.0 共通レイアウト（認証済みページ）

`Layouts/AppLayout.vue` に実装するサイドバー＋ヘッダーのレイアウト。
Inertia の `<slot />` でページコンテンツを展開する。

**Sidebar.vue（左サイドバー）**

| ナビ項目 | ルート名 | アイコン |
|----------|---------|---------|
| ダッシュボード | `dashboard` | ホームアイコン |
| 学習コンテンツ | `contents.index` | 本アイコン |
| 設定 | `settings` | 歯車アイコン |

- アクティブなリンクはハイライト表示（`usePage().url` で現在パスを判定）

**Header.vue（上部ヘッダー）**

- ログインユーザーのメールアドレス（`usePage().props.auth.user.email`）を `/profile` へのリンクとして表示
- 「ログアウト」ボタン → `router.post(route('logout'))` でログアウト

**フラッシュメッセージ**

- Controller から `with('success', 'メッセージ')` または `with('error', 'メッセージ')` でセットする
- `AppLayout.vue` で `usePage().props.flash` を監視し、Tailwind の Toast 表示コンポーネントで通知する
- Toast は **3 秒後に自動消去**する

**共通バッジ Tailwind クラス**

| 対象 | 値 | Tailwind クラス |
|---|---|---|
| コンテンツ種別バッジ | BOOK | `bg-blue-100 text-blue-800` |
| コンテンツ種別バッジ | VIDEO | `bg-red-100 text-red-800` |
| コンテンツ種別バッジ | CERTIFICATION | `bg-green-100 text-green-800` |
| ステータスバッジ | NOT_STARTED | `bg-gray-100 text-gray-800` |
| ステータスバッジ | IN_PROGRESS | `bg-yellow-100 text-yellow-800` |
| ステータスバッジ | COMPLETED | `bg-blue-100 text-blue-800` |
| ステータスバッジ | PASSED | `bg-green-100 text-green-800` |
| ステータスバッジ | FAILED | `bg-red-100 text-red-800` |

**共通ページネーション UI**

- 「前」「次」ボタン + ページ番号リスト（Tailwind スタイルのボタン）
- `Pagination.vue` コンポーネントに切り出し、`links` プロパティ（Laravel の `LengthAwarePaginator` が返す `links` 配列）を受け取って表示する

---

### 6.1 ログイン画面 `/login`

- **表示要素**: メールアドレス入力、パスワード入力、ログインボタン、ユーザー登録へのリンク、「パスワードを忘れた方はこちら」リンク
- **バリデーション**: メール形式・必須（サーバーサイドの Form Request）
- **成功時**: `/`（ダッシュボード）へリダイレクト
- **エラー時**: Inertia の `$page.props.errors` でフォーム下部にエラーを表示

### 6.2 ユーザー登録画面 `/register`

- **表示要素**: 名前・メールアドレス・パスワード・確認用パスワード入力、登録ボタン
- **バリデーション**: `RegisteredUserController` の `StoreRequest`（Breeze 標準）
- **成功時**: `/`（ダッシュボード）へリダイレクト

### 6.3 ダッシュボード `/`（認証必須）

#### Controller

```php
// app/Http/Controllers/DashboardController.php
public function index(Request $request): Response
{
    $user = $request->user();
    $tz   = $request->cookie('tz', 'Asia/Tokyo');

    return Inertia::render('Dashboard', [
        'weeklySummary'    => $this->dashboardService->getWeeklySummary($user, $tz),
        'monthlySummary'   => $this->dashboardService->getMonthlySummary($user, $tz),
        'heatmapData'      => $this->dashboardService->getHeatmapData($user, $tz),
        'contentTypeStats' => $this->dashboardService->getContentTypeStats($user),
        'monthlyStats'     => $this->dashboardService->getMonthlyStats($user, $tz),
        'inProgressContents' => $this->dashboardService->getInProgressContents($user),
    ]);
}
```

#### DashboardService の集計クエリ例

```php
// app/Services/DashboardService.php

public function getWeeklySummary(User $user, string $tz): int
{
    $start = now($tz)->startOfWeek();
    return LearningSession::where('user_id', $user->id)
        ->where('started_at', '>=', $start->utc())
        ->sum(DB::raw('TIMESTAMPDIFF(MINUTE, started_at, ended_at)'));
}

public function getHeatmapData(User $user, string $tz, int $days = 365): array
{
    $start = now($tz)->subDays($days - 1)->startOfDay()->utc();
    return LearningSession::where('user_id', $user->id)
        ->where('started_at', '>=', $start)
        ->selectRaw('DATE(CONVERT_TZ(started_at, "+00:00", ?)) as date, SUM(TIMESTAMPDIFF(MINUTE, started_at, ended_at)) as total_minutes', [$tz])
        ->groupBy('date')
        ->orderBy('date')
        ->get()
        ->toArray();
}

public function getInProgressContents(User $user): array
{
    return LearningContent::where('user_id', $user->id)
        ->where('status', ContentStatus::InProgress)
        ->withMax('sessions', 'started_at')
        ->orderByDesc('sessions_max_started_at')
        ->limit(5)
        ->get(['id', 'title', 'content_type'])
        ->toArray();
}

public function getContentTypeStats(User $user): array
{
    // ContentTypeChart 用: コンテンツ種別の件数マップ {BOOK: N, VIDEO: N, CERTIFICATION: N}
    $counts = LearningContent::where('user_id', $user->id)
        ->selectRaw('content_type, COUNT(*) as count')
        ->groupBy('content_type')
        ->get()
        ->mapWithKeys(fn ($row) => [$row->content_type => (int) $row->count]);

    return [
        'BOOK'          => $counts['BOOK'] ?? 0,
        'VIDEO'         => $counts['VIDEO'] ?? 0,
        'CERTIFICATION' => $counts['CERTIFICATION'] ?? 0,
    ];
}

public function getMonthlyStats(User $user, string $tz): array
{
    // StudyTimeLineChart 用: 過去 12 ヶ月の月ごとの「新規追加数」「完了数」「学習時間（分）」
    // 完了数は updated_at をプロキシとして使用
    $since = now($tz)->subMonths(11)->startOfMonth()->utc();

    $added = LearningContent::where('user_id', $user->id)
        ->selectRaw('DATE_FORMAT(CONVERT_TZ(created_at, "+00:00", ?), "%Y-%m") as month, COUNT(*) as count', [$tz])
        ->where('created_at', '>=', $since)
        ->groupBy('month')
        ->get()->pluck('count', 'month');

    $completed = LearningContent::where('user_id', $user->id)
        ->whereIn('status', [ContentStatus::Completed->value, ContentStatus::Passed->value])
        ->selectRaw('DATE_FORMAT(CONVERT_TZ(updated_at, "+00:00", ?), "%Y-%m") as month, COUNT(*) as count', [$tz])
        ->where('updated_at', '>=', $since)
        ->groupBy('month')
        ->get()->pluck('count', 'month');

    $studyTime = LearningSession::where('user_id', $user->id)
        ->selectRaw('DATE_FORMAT(CONVERT_TZ(started_at, "+00:00", ?), "%Y-%m") as month, SUM(TIMESTAMPDIFF(MINUTE, started_at, ended_at)) as total_minutes', [$tz])
        ->where('started_at', '>=', $since)
        ->groupBy('month')
        ->get()->pluck('total_minutes', 'month');

    $months = collect(range(11, 0))->map(fn ($i) => now($tz)->subMonths($i)->format('Y-m'));

    return $months->map(fn ($m) => [
        'month'           => $m,
        'added_count'     => (int) ($added[$m] ?? 0),
        'completed_count' => (int) ($completed[$m] ?? 0),
        'study_minutes'   => (int) ($studyTime[$m] ?? 0),
    ])->values()->toArray();
}
```

#### 表示コンポーネント

| コンポーネント | データ | 説明 |
|---|---|---|
| `StudySummary` | 今週・今月の合計学習時間（分）| `DashboardService::getWeeklySummary` / `getMonthlySummary`。表示形式は 60 分未満は「XX 分」、以上は「X 時間 YY 分」 |
| `CalendarHeatmap` | 日ごとの学習時間 | 過去 365 日。深さ 4 段階（0 / 1〜30 / 31〜60 / 61〜 分）。**学習データ 0 の場合は全セルをグレー表示** |
| `ContentTypeChart` | BOOK/VIDEO/CERTIFICATION の件数 | `getContentTypeStats` 。**ドーナツチャート**（Chart.js Doughnut）。色: BOOK=`#3B82F6`（青）/ VIDEO=`#EF4444`（赤）/ CERTIFICATION=`#22C55E`（緑）|
| `StudyTimeLineChart` | 月ごとの累計完了数・累計学習時間 | 過去 12 ヶ月の折れ線グラフ（Chart.js Line）。**2 軸構成**: 左 Y 軸=件数、右 Y 軸=学習時間（分）。線の色: 新規追加=`#3B82F6`（青）/ 完了=`#22C55E`（緑）/ 学習時間=`#F97316`（橙）|
| `InProgressList` | status = IN_PROGRESS のコンテンツ | 最大 5 件。タイトル・種別・最終セッション日時。タイトルクリックで `/contents/{id}` へ遷移。件数 0 の場合は「学習中コンテンツなし」テキストを表示 |

---

### 6.4 学習コンテンツ一覧 `/contents`（認証必須）

#### Controller

```php
// app/Http/Controllers/ContentController.php
public function index(Request $request): Response
{
    $contents = $this->contentQueryService->getPaginatedContents(
        user:   $request->user(),
        q:      $request->query('q'),
        type:   $request->query('type'),
        status: $request->query('status'),
        tagId:  $request->query('tag'),
        sort:   $request->query('sort', 'updated_at_desc'),
        page:   (int) $request->query('page', 1),
    );

    return Inertia::render('Contents/Index', [
        'contents' => ContentResource::collection($contents),
        'tags'     => TagResource::collection(Tag::where('user_id', $request->user()->id)->get()),
        'filters'  => $request->only(['q', 'type', 'status', 'tag', 'sort']),
    ]);
}
```

#### ContentQueryService

```php
// app/Services/ContentQueryService.php
public function getPaginatedContents(
    User    $user,
    ?string $q,
    ?string $type,
    ?string $status,
    ?string $tagId,
    string  $sort = 'updated_at_desc',
    int     $page = 1,
    int     $perPage = 20,
): LengthAwarePaginator {
    $query = LearningContent::where('user_id', $user->id)
        ->with('tags')
        ->withSum('sessions as total_study_minutes', DB::raw('TIMESTAMPDIFF(MINUTE, started_at, ended_at)'));

    if ($q) {
        $query->where('title', 'like', "%{$q}%");
    }
    if ($type) {
        $query->where('content_type', $type);
    }
    if ($status) {
        $query->where('status', $status);
    }
    if ($tagId) {
        $query->whereHas('tags', fn ($q) => $q->where('tags.id', $tagId));
    }

    $query->orderBy(...match ($sort) {
        'created_at_desc' => ['created_at', 'desc'],
        'title_asc'       => ['title', 'asc'],
        'study_time_desc' => ['total_study_minutes', 'desc'],
        default           => ['updated_at', 'desc'],
    });

    return $query->paginate($perPage, page: $page);
}
```

- **ページネーション**: 1 ページあたり **20 件**。Laravel の `LengthAwarePaginator` を使用し、Inertia に `ContentResource::collection($paginator)` として渡す。ページネーションリンクは `contents.links` から生成する。
- **カード表示項目**: サムネイル画像（なければコンテンツ種別アイコンを灰背景中央表示）、タイトル、コンテンツ種別バッジ、ステータスバッジ、タグ一覧、累計学習時間（60 分未満は「XX 分」・以上は「X 時間 YY 分」）、最終更新日
- **カードクリック**: カード全体が `/contents/{id}` へのリンク（`<a>` タグで包む）
- **サムネイル表示**: `object-cover` で正方形にクロップ表示
- **Empty State（0件時）**: 「まだコンテンツがありません」+ 「最初のコンテンツを追加する」ボタン
- **フィルターの URL 反映**: フィルター内容（q / type / status / tag / sort）をクエリストリングに反映する。ブラウザの戻る・ URL ペーストでフィルター条件を再現可能。Vue 側は `router.get(route('contents.index'), filters, { preserveState: true })` で送信する
- **フィルターリセット**: フィルターが 1 つ以上適用中の場合のみ「フィルターをリセット」ボタンを表示。クリックで `router.get(route('contents.index'))` を呼ぶ

---

### 6.5 学習コンテンツ詳細 `/contents/{id}`（認証必須）

#### Controller

```php
public function show(Request $request, LearningContent $content): Response
{
    $this->authorize('view', $content);

    $sessions = $content->sessions()
        ->latest('started_at')
        ->paginate(20);

    return Inertia::render('Contents/Show', [
        'content'  => new ContentResource($content->load('tags')),
        'sessions' => SessionResource::collection($sessions),
    ]);
}
```

#### 上部: コンテンツ情報

コンテンツ詳細画面は **2 カラムレイアウト**。

- **左カラム**: コンテンツ情報（タイトル・種別・著者/URL・説明・ステータス・タグ・メモ）
- **右カラム**: タイマー UI（今すぐ開始 / 終了ボタン）と「セッションを記録する」ボタン

- タイトル・種別・著者/URL・説明（閲覧のみ。編集は `/contents/{id}/edit`）
- **ステータス（インライン変更可）**: `<select>` の change イベントで `router.patch(route('contents.status', id), { status })` を即時送信。選択肢は `content_type` に応じてフィルタする（ADR-0018 参照）
  - BOOK / VIDEO: `NOT_STARTED / IN_PROGRESS / COMPLETED`
  - CERTIFICATION: `NOT_STARTED / IN_PROGRESS / PASSED / FAILED`
- **タグ（追加・削除可）**: タグ選択 UI。変更のたびに `router.patch(route('contents.tags', id), { tag_ids })` を即時送信
- **メモ**: テキストエリア + 保存ボタン。`router.patch(route('contents.notes', id), { notes })` を送信

#### 下部: セッション履歴

- セッション一覧テーブル: 開始日時 / 終了日時 / 学習時間（分）/ メモ / 削除ボタン（`window.confirm('このセッションを削除しますか？')` 後に `router.delete`）
- **ページネーション**: 1 ページあたり **20 件**。`sessions.links` でページ送りを表示する
- **Empty State（0 件時）**: 「まだセッションが記録されていません」と表示する
- **「セッションを記録する」ボタン**: クリックでモーダルを開く

#### セッション記録モーダル（SessionForm.vue）

| フィールド | 型 | バリデーション |
|---|---|---|
| 開始日時（`started_at`） | datetime-local | 必須 |
| 終了日時（`ended_at`） | datetime-local | 必須・`started_at` より後 |
| メモ（`memo`） | textarea | 任意・1000文字以内 |

- 入力値はユーザーのタイムゾーン（Cookie `tz`）で表示・送信し、Controller で UTC に変換
- バリデーションエラーは `$page.props.errors` でフィールド直下に表示
- **ステータス自動遷移（ADR-0014 参照）**: `SessionController::store` 内で、コンテンツの現在のステータスが `NOT_STARTED` の場合のみ自動的に `IN_PROGRESS` に更新する

#### タイマー機能

`useSessionTimer.ts` composable で実装する。

- **「今すぐ開始」ボタン**: クリック時の UTC 日時を `localStorage` の `timer_start_{contentId}` に保存。ブラウザを閉じても開始時刻は残る
- **計測中表示**: `localStorage` に開始時刻が存在する間は「計測中 — XX 分進行中」をボタン展示。経過分数はページロード時に計算（リアルタイムカウントアップは不要）
- **「終了」ボタン**: `localStorage` から開始時刻を読み取り、`started_at` と `ended_at`（現在時刻）を事前入力した状態でモーダルを開く。送信後またはモーダルを閉じた後に `localStorage` キーを削除する

---

### 6.6 学習コンテンツ追加 `/contents/create`（認証必須）

#### ContentForm フィールド

| フィールド | 型 | バリデーション | 表示条件 |
|---|---|---|---|
| タイトル | text | 必須・100文字以内 | 常時 |
| コンテンツ種別 | select | 必須 | 常時 |
| 著者名 | text | 任意・100文字以内 | BOOK のみ |
| URL | url | 任意・URL形式・2048文字以内 | VIDEO・CERTIFICATION |
| 説明 | textarea | 任意・500文字以内 | 常時 |
| ステータス | select | 必須・デフォルト NOT_STARTED | 常時 |
| タグ | multi-select（既存から選択 or 新規作成） | 任意・最大10タグ | 常時 |
| サムネイル | file（JPEG/PNG/WebP・2MB以内）| 任意 | 常時 |
| メモ | textarea | 任意・2000文字以内 | 常時 |

#### サムネイルアップロードフロー

1. `<input type="file">` でファイルを選択
2. `useForm` の `transform` で `FormData` に変換し、`router.post()` でマルチパート送信
3. `ContentController::store` 内で `$request->file('thumbnail')->store('thumbnails', 'public')` でアップロードし、戻り値のパスを `thumbnail_path` に保存する
4. 公開 URL は `Storage::url($content->thumbnail_path)` で生成する（`ContentResource` で計算して返す）

#### 新規タグの即時作成（ADR-0017 参照）

- タグ入力 UI（自作 ComboBox）でユーザーが新しいタグ名を入力し**Enter キーまたは「追加」ボタン**で確定した時点で、`axios.post(route('tags.store'), { name })` を呼んで DB に INSERT する
- 候補ドロップダウンで既存タグのフィルタリングも行う（入力内容で前方一致検索）
- 返却された ID を選択状態に追加し、フォーム送信時は `tag_ids`（ID 配列）のみを渡す

#### フォーム送信

```typescript
// Inertia useForm を使用
const form = useForm({
    title: '',
    content_type: 'BOOK',
    // ...
    tag_ids: [] as number[],
    thumbnail: null as File | null,
});

function submit() {
    form.post(route('contents.store'), {
        forceFormData: true, // ファイルアップロードのため
    });
}
```

- 送信成功後: `/contents/{id}` へリダイレクト
  - `ContentController::store` は `return redirect()->route('contents.show', $content)` を返す。Inertia は誤誘なしに追従する

---

### 6.7 学習コンテンツ編集 `/contents/{id}/edit`

- `ContentForm` と同じフォームに既存データを preload して表示
- **サムネイルプレビュー**: `thumbnail_path` があれば現在のサムネイル画像をフォーム内に表示する。新しいファイルを選択するまで旧画像が表示される
- `router.patch(route('contents.update', id), form)` で送信
- 送信成功後: `/contents/{id}` へリダイレクト
- 「削除」ボタン → `window.confirm('このコンテンツを削除しますか？関連するセッション履歴もすべて削除されます。')` 後に `router.delete(route('contents.destroy', id))` → 一覧へリダイレクト

#### サムネイル削除ルール（ADR-0019 参照）

```php
// ContentController::update（差し替え時）
if ($request->hasFile('thumbnail') && $content->thumbnail_path) {
    Storage::disk('public')->delete($content->thumbnail_path);
}

// ContentController::destroy（コンテンツ削除時）
if ($content->thumbnail_path) {
    Storage::disk('public')->delete($content->thumbnail_path);
}
$content->delete();
```

---

### 6.8 設定 `/settings`（認証必須）

#### Controller

```php
// app/Http/Controllers/SettingsController.php
public function index(Request $request): Response
{
    return Inertia::render('Settings/Index', [
        'tags' => TagResource::collection(
            Tag::where('user_id', $request->user()->id)
               ->withCount('contents')
               ->orderBy('name')
               ->get()
        ),
    ]);
}
```

#### タブ構成

**タグ管理タブ**
- タグ一覧（名前・使用コンテンツ数）
- タグ追加フォーム（名前のみ・20文字以内・重複不可）
- タグ名変更（インライン編集 → `router.patch(route('tags.update', id))`)
- タグ削除（`router.delete(route('tags.destroy', id))`）—ダイアログなしで即度削除。DB の CASCADE で `content_tag` からも自動削除される

**データ管理タブ**
- エクスポートボタン（JSON / CSV を選択してダウンロード → `GET /export?format=json|csv`）

#### ExportController の仕様

```php
// app/Http/Controllers/ExportController.php
public function download(Request $request): StreamedResponse|JsonResponse
{
    $user = $request->user();
    $format = $request->query('format', 'json'); // 'json' | 'csv'
    $filename = 'study-app-export-' . now()->format('Ymd') . '.' . $format;

    $contents = LearningContent::where('user_id', $user->id)
        ->with(['tags', 'sessions'])
        ->get();

    if ($format === 'csv') {
        return $this->streamCsv($contents, $filename);
    }

    return response()->json($contents->toArray())
        ->header('Content-Disposition', 'attachment; filename="' . $filename . '"');
}
```

- **JSON 出力内容**: `learning_contents` 全件（`tags`・`sessions` をネスト）
- **CSV 出力内容**: コンテンツを 1 行 1 件で出力（セッションは `total_study_minutes` のみ集計して列に含める）

**CSV 列定義**

```
id, title, content_type, status, author, url, description, total_study_minutes, tags, created_at
```

- `tags` 列: タグ名をカンマ区切りで連結（例: `PHP,Laravel`）。タグなしの場合は空文字列
- **セッション詳細を CSV に含めるかどうか**: コンテンツ行のみ（セッション別行は出力しない）

---

### 6.9 パスワードリセット

Laravel Breeze が生成する標準フローを使用する。

- `/forgot-password`: メールアドレス入力 → パスワードリセットメール送信
- `/reset-password/{token}`: 新しいパスワード入力 → パスワード変更

---

### 6.10 プロフィール `/profile`

Laravel Breeze が自動生成する `/profile` ページをそのまま維持する。

- 名前・メール変更 / パスワード変更 / アカウント削除
- サイドバーのナビゲーションには追加しない（直接 URL アクセス、またはヘッダーのユーザー名クリックで遷移）
- `ProfileController`・`ProfileUpdateRequest`・`Pages/Profile/Edit.vue` は Breeze 自動生成のまま使用

---

### 6.11 エラーページ

Inertia の error ハンドリングを使用し、`Pages/Error.vue` を 1 ファイルで 403 / 404 / 500 を統一表示する。

#### セットアップ

```typescript
// resources/js/app.ts
createInertiaApp({
    resolve: name => resolvePageComponent(
        `./Pages/${name}.vue`,
        import.meta.glob('./Pages/**/*.vue'),
    ),
    // ...
});
```

```php
// app/Exceptions/Handler.php
// Inertia の エラーページを返すカスタマイズ
// → ADR-0001 に従い Inertia 標準の renderForConsoleOutput 使用
```

```vue
<!-- resources/js/Pages/Error.vue -->
<script setup lang="ts">
const props = defineProps<{ status: number }>();
const title = computed(() => ({
    403: 'アクセスが許可されていません',
    404: 'ページが見つかりません',
    500: 'サーバーエラーが発生しました',
}[props.status] ?? 'エラーが発生しました'));
</script>
```

- `Handler::render` で Inertiaリクエストの場合は `Inertia::render('Error', ['status' => $e->getStatusCode()])` を返す

---

## 7. バリデーション（Form Request）

```php
// app/Http/Requests/StoreContentRequest.php
class StoreContentRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true; // Controller で Policy を使用
    }

    public function rules(): array
    {
        return [
            'title'         => ['required', 'string', 'max:100'],
            'content_type'  => ['required', Rule::enum(ContentType::class)],
            'author'        => ['nullable', 'string', 'max:100'],
            'url'           => ['nullable', 'url', 'max:2048'],
            'description'   => ['nullable', 'string', 'max:500'],
            'status'        => ['required', Rule::enum(ContentStatus::class)],
            'notes'         => ['nullable', 'string', 'max:2000'],
            'tag_ids'       => ['nullable', 'array', 'max:10'],
            'tag_ids.*'     => ['integer', 'exists:tags,id'],
            'thumbnail'     => ['nullable', 'image', 'mimes:jpeg,png,webp', 'max:2048'],
        ];
    }
}
```

```php
// app/Http/Requests/StoreSessionRequest.php
class StoreSessionRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'started_at' => ['required', 'date'],
            'ended_at'   => ['required', 'date', 'after:started_at'],
            'memo'       => ['nullable', 'string', 'max:1000'],
        ];
    }
}
```

```php
// app/Http/Requests/UpdateContentStatusRequest.php
class UpdateContentStatusRequest extends FormRequest
{
    public function rules(): array
    {
        /** @var LearningContent $content */
        $content = $this->route('content');
        $allowed = match ($content->content_type) {
            ContentType::Book, ContentType::Video => [
                ContentStatus::NotStarted->value,
                ContentStatus::InProgress->value,
                ContentStatus::Completed->value,
            ],
            ContentType::Certification => [
                ContentStatus::NotStarted->value,
                ContentStatus::InProgress->value,
                ContentStatus::Passed->value,
                ContentStatus::Failed->value,
            ],
        };

        return [
            'status' => ['required', Rule::in($allowed)],
        ];
    }
}
```

```php
// app/Http/Requests/UpdateContentRequest.php
// StoreContentRequest と同一ルール。コンテンツ編集（PUT/PATCH）時に使用する。
class UpdateContentRequest extends FormRequest
{
    public function authorize(): bool { return true; }

    public function rules(): array
    {
        return [
            'title'         => ['required', 'string', 'max:100'],
            'content_type'  => ['required', Rule::enum(ContentType::class)],
            'author'        => ['nullable', 'string', 'max:100'],
            'url'           => ['nullable', 'url', 'max:2048'],
            'description'   => ['nullable', 'string', 'max:500'],
            'status'        => ['required', Rule::enum(ContentStatus::class)],
            'notes'         => ['nullable', 'string', 'max:2000'],
            'tag_ids'       => ['nullable', 'array', 'max:10'],
            'tag_ids.*'     => ['integer', 'exists:tags,id'],
            'thumbnail'     => ['nullable', 'image', 'mimes:jpeg,png,webp', 'max:2048'],
        ];
    }
}
```

```php
// app/Http/Requests/UpdateContentTagsRequest.php
// PATCH /contents/{id}/tags 専用
class UpdateContentTagsRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'tag_ids'   => ['nullable', 'array', 'max:10'],
            'tag_ids.*' => ['integer', 'exists:tags,id'],
        ];
    }
}
```

```php
// app/Http/Requests/UpdateContentNotesRequest.php
// PATCH /contents/{id}/notes 専用
class UpdateContentNotesRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'notes' => ['nullable', 'string', 'max:2000'],
        ];
    }
}
```

```php
// app/Http/Requests/StoreTagRequest.php
class StoreTagRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name' => [
                'required',
                'string',
                'max:20',
                Rule::unique('tags')->where(fn ($q) => $q->where('user_id', $this->user()->id)),
            ],
        ];
    }
}
```

---

## 8. 認可（Policy）

```php
// app/Policies/ContentPolicy.php
class ContentPolicy
{
    public function view(User $user, LearningContent $content): bool
    {
        return $user->id === $content->user_id;
    }

    public function update(User $user, LearningContent $content): bool
    {
        return $user->id === $content->user_id;
    }

    public function delete(User $user, LearningContent $content): bool
    {
        return $user->id === $content->user_id;
    }
}
```

```php
// app/Policies/SessionPolicy.php
class SessionPolicy
{
    public function delete(User $user, LearningSession $session): bool
    {
        return $user->id === $session->user_id;
    }
}
```

Controller では `$this->authorize('view', $content)` を必ず呼ぶ。
`SessionController::destroy` では `$this->authorize('delete', $session)` を呼ぶ。

---

## 9. API Resource（JSON 整形）

> `ContentCollection.php` は `ContentResource::collection()` と同等の実質ラッパーのみ。`ResourceCollection` を継承しているが追加ロジックはない。

```php
// app/Http/Resources/ContentResource.php
class ContentResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'                  => $this->id,
            'title'               => $this->title,
            'content_type'        => $this->content_type->value,
            'status'              => $this->status->value,
            'author'              => $this->author,
            'url'                 => $this->url,
            'description'         => $this->description,
            'notes'               => $this->notes,
            'thumbnail_url'       => $this->thumbnail_path
                                        ? Storage::url($this->thumbnail_path)
                                        : null,
            'total_study_minutes' => (int) ($this->total_study_minutes ?? 0),
            'tags'                => TagResource::collection($this->whenLoaded('tags')),
            'created_at'          => $this->created_at->toISOString(),
            'updated_at'          => $this->updated_at->toISOString(),
        ];
    }
}
```

---

## 9-2. その他 API Resource

```php
// app/Http/Resources/SessionResource.php
class SessionResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'               => $this->id,
            'content_id'       => $this->content_id,
            'started_at'       => $this->started_at->toISOString(),
            'ended_at'         => $this->ended_at->toISOString(),
            'duration_minutes' => $this->duration_minutes, // Eloquent accessor
            'memo'             => $this->memo,
            'created_at'       => $this->created_at->toISOString(),
        ];
    }
}
```

```php
// app/Http/Resources/TagResource.php
class TagResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return [
            'id'             => $this->id,
            'name'           => $this->name,
            'contents_count' => $this->whenCounted('contents'), // withCount 時のみ含む
            'created_at'     => $this->created_at->toISOString(),
        ];
    }
}
```

---

## 10. TypeScript 型定義（`resources/js/types.ts`）

```typescript
export type ContentType = 'BOOK' | 'VIDEO' | 'CERTIFICATION';

export type ContentStatus =
    | 'NOT_STARTED'
    | 'IN_PROGRESS'
    | 'COMPLETED'
    | 'PASSED'
    | 'FAILED';

export interface Content {
    id: number;
    title: string;
    content_type: ContentType;
    status: ContentStatus;
    author: string | null;
    url: string | null;
    description: string | null;
    notes: string | null;
    thumbnail_url: string | null;
    total_study_minutes: number;
    tags: Tag[];
    created_at: string;
    updated_at: string;
}

export interface Session {
    id: number;
    content_id: number;
    started_at: string;
    ended_at: string;
    duration_minutes: number;
    memo: string | null;
    created_at: string;
}

export interface Tag {
    id: number;
    name: string;
    created_at: string;
}

export interface MonthlyStat {
    month: string;           // 'YYYY-MM' 形式
    added_count: number;
    completed_count: number;
    study_minutes: number;
}

export type ContentTypeStats = Record<ContentType, number>;

// Inertia shared data
export interface PageProps {
    auth: {
        user: {
            id: number;
            name: string;
            email: string;
        };
    };
    flash: {
        success?: string;
        error?: string;
    };
}
```

---

## 10-2. Composable 仕様

### `useFlash.ts`

`usePage().props.flash` を監視し、Toast の表示/非表示を管理する。

```typescript
// resources/js/composables/useFlash.ts
export function useFlash() {
    const flash = computed(() => usePage().props.flash as { success?: string; error?: string });
    const message = ref<string | null>(null);
    const type = ref<'success' | 'error'>('success');

    watch(flash, (val) => {
        if (val.success) { message.value = val.success; type.value = 'success'; }
        else if (val.error)  { message.value = val.error;   type.value = 'error'; }
        if (message.value) {
            setTimeout(() => { message.value = null; }, 3000);
        }
    });

    return { message, type };
}
```

`AppLayout.vue` 内でこの composable を呼び出し、`message` が非 null の間だけ Toast を表示する。

### `useContentForm.ts`

Inertia `useForm` のラッパー + コンテンツ種別による `author` / `url` フィールドの切り替えロジックを提供する。

```typescript
// resources/js/composables/useContentForm.ts
export function useContentForm(initial?: Partial<ContentFormData>) {
    const form = useForm<ContentFormData>({ /* 初期値 */ ...initial });

    // コンテンツ種別が変わったら author / url をリセット
    watch(() => form.content_type, (type) => {
        if (type === 'BOOK') { form.url = ''; }
        else { form.author = ''; }
    });

    // author フィールドを表示するか
    const showAuthor = computed(() => form.content_type === 'BOOK');
    // url フィールドを表示するか
    const showUrl = computed(() => form.content_type !== 'BOOK');

    return { form, showAuthor, showUrl };
}
```

---

## 11. 環境変数（`.env.example`）

```dotenv
APP_NAME="学習記録アプリ"
APP_ENV=local
APP_KEY=
APP_DEBUG=true
APP_URL=http://localhost

LOG_CHANNEL=stack
LOG_LEVEL=debug

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
REDIS_PASSWORD=null
REDIS_PORT=6379

FILESYSTEM_DISK=local

MAIL_MAILER=log
MAIL_FROM_ADDRESS="noreply@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

---

## 12. Docker 構成

### `docker-compose.yml`（開発環境）

```yaml
services:
  app:
    build:
      context: ./docker/php
      dockerfile: Dockerfile
    volumes:
      - .:/var/www/html
      - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
    environment:
      - APP_ENV=local

  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
    volumes:
      - .:/var/www/html
      - ./docker/nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - app

  db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootsecret
      MYSQL_DATABASE: study_app
      MYSQL_USER: app_user
      MYSQL_PASSWORD: secret
    volumes:
      - db_data:/var/lib/mysql
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "app_user", "-psecret"]
      interval: 5s
      timeout: 5s
      retries: 10

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  vite:
    image: node:20-alpine
    working_dir: /var/www/html
    volumes:
      - .:/var/www/html
    ports:
      - "5173:5173"
    command: sh -c "npm install && npm run dev"
    environment:
      - VITE_HOST=0.0.0.0

volumes:
  db_data:
  redis_data:
```

### `docker/php/Dockerfile`

```dockerfile
FROM php:8.3-fpm-alpine

RUN apk add --no-cache \
    git \
    zip \
    unzip \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    oniguruma-dev \
    libxml2-dev \
    redis

RUN docker-php-ext-configure gd --with-freetype --with-jpeg \
    && docker-php-ext-install \
        pdo_mysql \
        mbstring \
        exif \
        pcntl \
        bcmath \
        gd \
        opcache

RUN pecl install redis && docker-php-ext-enable redis

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

WORKDIR /var/www/html

RUN addgroup -g 1000 appgroup && adduser -u 1000 -G appgroup -s /bin/sh -D appuser
USER appuser
```

### `docker/nginx/default.conf`

```nginx
server {
    listen 80;
    server_name localhost;
    root /var/www/html/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

---

## 14. データベースシーダー（`database/seeders/DatabaseSeeder.php`）

開発・デモ環境用のシードデータ。`php artisan db:seed` で投入する。

```php
// database/seeders/DatabaseSeeder.php
public function run(): void
{
    // サンプルユーザー 1 名（固定認証汗）
    $user = User::factory()->create([
        'name'              => 'Test User',
        'email'             => 'test@example.com',
        'password'          => Hash::make('password'),
        'email_verified_at' => now(),
    ]);

    // コンテンツ 10 件
    $contents = LearningContent::factory(10)->create(['user_id' => $user->id]);

    // セッション 30 件（コンテンツに均等配分）
    $contents->each(function ($content) use ($user) {
        LearningSession::factory(3)->create([
            'user_id'    => $user->id,
            'content_id' => $content->id,
        ]);
    });

    // タグ 5 件 + コンテンツにランダム付与
    $tags = Tag::factory(5)->create(['user_id' => $user->id]);
    $contents->each(fn ($c) => $c->tags()->attach(
        $tags->random(rand(1, 3))->pluck('id')
    ));
}
```

---

## 13. 非機能要件

| 項目 | 要件 |
|---|---|
| セキュリティ | CSRF トークン（Laravel デフォルト）。`authorize()` を全 Controller で必ず呼ぶ。ファイルアップロードは MIME 検証後に `storage/app/public` へ保存し、`/storage` 以外のパスへの直接アクセスは不可とする |
| 型安全性 | PHP: `declare(strict_types=1)` を全 PHP ファイルに付与。バックド列挙型（ContentType・ContentStatus）で型安全を確保する。TypeScript: `strict: true` |
| エラーハンドリング | Controller のエラーは Laravel の例外ハンドラが処理。Inertia 向けには `Handler::render` でエラーページを返す。ユーザー向けメッセージはすべて日本語で記述する |
| レスポンシブ | **デスクトップのみ**対応。最小幅 1024px を想定。モバイル対応はスコープ外 |
| ロケール | `config/app.php` の `locale` を `ja` に設定。バリデーションメッセージは `lang/ja/` で日本語化する |
| ローディング UI | `@inertiajs/progress` パッケージを使用してページ遺移中にトップバーのプログレスバーを表示する |
| タイムゾーン | DB は UTC 保存。`AppLayout.vue` の `onMounted` で `Intl.DateTimeFormat().resolvedOptions().timeZone` を取得し、`document.cookie = 'tz=' + tz + '; path=/'` で Cookie に保存する。Controller では `$request->cookie('tz', 'Asia/Tokyo')` で読み取る |
| アクセシビリティ | 現時点ではスコープ外。ARIA 属性の付与などは行わない |
| メール認証 | `MustVerifyEmail` インターフェースを `User` モデルに実装し、メール認証を有効化する。開発時は `MAIL_MAILER=log` でログにメールを出力する。`routes/web.php` の `verified` ミドルウェアはそのまま使用する |
