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
│   │   │   └── ExportController.php
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

- ログインユーザーのメールアドレス（`usePage().props.auth.user.email`）を表示
- 「ログアウト」ボタン → `router.post(route('logout'))` でログアウト

**フラッシュメッセージ**

- Controller から `with('success', 'メッセージ')` または `with('error', 'メッセージ')` でセットする
- `AppLayout.vue` で `usePage().props.flash` を監視し、Tailwind の Toast 表示コンポーネントで通知する

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
```

#### 表示コンポーネント

| コンポーネント | データ | 説明 |
|---|---|---|
| `StudySummary` | 今週・今月の合計学習時間（分）| `DashboardService::getWeeklySummary` / `getMonthlySummary` |
| `CalendarHeatmap` | 日ごとの学習時間 | 過去 365 日。深さ 4 段階（0 / 1〜30 / 31〜60 / 61〜 分）|
| `ContentTypeChart` | BOOK/VIDEO/CERTIFICATION の件数 | `getContentTypeStats` |
| `StudyTimeLineChart` | 月ごとの累計完了数・累計学習時間 | 過去 12 ヶ月の折れ線グラフ（Chart.js） |
| `InProgressList` | status = IN_PROGRESS のコンテンツ | 最大 5 件。タイトル・種別・最終セッション日時 |

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
- **カード表示項目**: サムネイル画像（なければコンテンツ種別アイコン）、タイトル、コンテンツ種別バッジ、ステータスバッジ、タグ一覧、累計学習時間、最終更新日
- **Empty State（0件時）**: 「まだコンテンツがありません」+ 「最初のコンテンツを追加する」ボタン
- **フィルターリセット**: フィルターが 1 つ以上適用中の場合のみ「フィルターをリセット」ボタンを表示。クリックで `router.get(route('contents.index'))` を呼ぶ

---

### 6.5 学習コンテンツ詳細 `/contents/{id}`（認証必須）

#### Controller

```php
public function show(Request $request, LearningContent $content): Response
{
    $this->authorize('view', $content);

    return Inertia::render('Contents/Show', [
        'content'  => new ContentResource($content->load('tags')),
        'sessions' => SessionResource::collection(
            $content->sessions()->latest('started_at')->get()
        ),
    ]);
}
```

#### 上部: コンテンツ情報

- タイトル・種別・著者/URL・説明（閲覧のみ。編集は `/contents/{id}/edit`）
- **ステータス（インライン変更可）**: `<select>` の change イベントで `router.patch(route('contents.status', id), { status })` を即時送信
- **タグ（追加・削除可）**: タグ選択 UI。変更のたびに `router.patch(route('contents.tags', id), { tag_ids })` を即時送信
- **メモ**: テキストエリア + 保存ボタン。`router.patch(route('contents.notes', id), { notes })` を送信

#### 下部: セッション履歴

- セッション一覧テーブル: 開始日時 / 終了日時 / 学習時間（分）/ メモ / 削除ボタン（確認ダイアログ付き）
- **「セッションを記録する」ボタン**: クリックでモーダルを開く
- **ステータス自動遷移（ADR-0014 参照）**: `SessionController::store` 内で、コンテンツの現在のステータスが `NOT_STARTED` の場合のみ自動的に `IN_PROGRESS` に更新する

#### タイマー機能

`useSessionTimer.ts` composable で実装する。

- **「今すぐ開始」ボタン**: クリック時の UTC 日時を `localStorage` の `timer_start_{contentId}` に保存
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
| タグ | multi-select（既存から選択 or 新規作成） | 任意 | 常時 |
| サムネイル | file（JPEG/PNG/WebP・2MB以内）| 任意 | 常時 |
| メモ | textarea | 任意・2000文字以内 | 常時 |

#### サムネイルアップロードフロー

1. `<input type="file">` でファイルを選択
2. `useForm` の `transform` で `FormData` に変換し、`router.post()` でマルチパート送信
3. `ContentController::store` 内で `$request->file('thumbnail')->store('thumbnails', 'public')` でアップロードし、戻り値のパスを `thumbnail_path` に保存する
4. 公開 URL は `Storage::url($content->thumbnail_path)` で生成する（`ContentResource` で計算して返す）

#### 新規タグの即時作成（ADR-0017 参照）

- タグ入力 UI でユーザーが新しいタグ名を入力し確定した時点で、`axios.post(route('tags.store'), { name })` を呼んで DB に INSERT する
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
        onSuccess: () => router.visit(route('contents.show', newContentId)),
    });
}
```

- 送信成功後: `/contents/{新しいid}` へリダイレクト

---

### 6.7 学習コンテンツ編集 `/contents/{id}/edit`

- `ContentForm` と同じフォームに既存データを preload して表示
- `router.patch(route('contents.update', id), form)` で送信
- 送信成功後: `/contents/{id}` へリダイレクト
- 「削除」ボタン → 確認ダイアログ後 `router.delete(route('contents.destroy', id))` → 一覧へリダイレクト

---

### 6.8 設定 `/settings`（認証必須）

#### タブ構成

**タグ管理タブ**
- タグ一覧（名前・使用コンテンツ数）
- タグ追加フォーム（名前のみ・20文字以内・重複不可）
- タグ名変更（インライン編集 → `router.patch(route('tags.update', id))`)
- タグ削除（`router.delete(route('tags.destroy', id))`）

**データ管理タブ**
- エクスポートボタン（JSON / CSV を選択してダウンロード → `GET /export?format=json`）

---

### 6.9 パスワードリセット

Laravel Breeze が生成する標準フローを使用する。

- `/forgot-password`: メールアドレス入力 → パスワードリセットメール送信
- `/reset-password/{token}`: 新しいパスワード入力 → パスワード変更

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
            'tag_ids'       => ['nullable', 'array'],
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

Controller では `$this->authorize('view', $content)` を必ず呼ぶ。

---

## 9. API Resource（JSON 整形）

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

## 13. 非機能要件

| 項目 | 要件 |
|---|---|
| セキュリティ | CSRF トークン（Laravel デフォルト）。`authorize()` を全 Controller で必ず呼ぶ。ファイルアップロードは MIME 検証後に `storage/app/public` へ保存し、`/storage` 以外のパスへの直接アクセスは不可とする |
| 型安全性 | PHP: `declare(strict_types=1)` を全 PHP ファイルに付与。バックド列挙型（ContentType・ContentStatus）で型安全を確保する。TypeScript: `strict: true` |
| エラーハンドリング | Controller のエラーは Laravel の例外ハンドラが処理。Inertia 向けには `Handler::render` でエラーページを返す。ユーザー向けメッセージはすべて日本語で記述する |
| レスポンシブ | PC ブラウザを主ターゲット。最小幅 1024px を想定。モバイル対応は任意 |
| ロケール | `config/app.php` の `locale` を `ja` に設定。バリデーションメッセージは `lang/ja/` で日本語化する |
| ローディング UI | Inertia の `router.visit` 中は `usePage().props` の変化を監視し、グローバルのプログレスバーを表示する（`@inertiajs/progress` または CSS トランジション） |
| タイムゾーン | DB は UTC 保存。ユーザーのタイムゾーンはブラウザで取得し Cookie `tz` にセットする（`AppLayout.vue` の `onMounted` で実行）。Controller では `$request->cookie('tz', 'Asia/Tokyo')` で読み取る |
