# コーディングルール

> **このドキュメントは AI が実装時に必ず参照するルール集です。**
> アーキテクチャ的な決定理由は各 ADR を参照してください。

---

## 1. PHP 全般

- **`declare(strict_types=1)`** をすべての PHP ファイルの先頭に付与する
- **PSR-12** コーディングスタンダードに準拠する（`laravel/pint` で自動整形）
- **`mixed` 型・型なし引数・型なし戻り値の禁止** — すべての引数と戻り値に型宣言を付与する
- **`null` の返却には nullable 型 `?Type` または `Type|null` を使用する** — `false` を「値なし」として返すことを禁止する
- **早期 return（ガード節）を使う** — ネストを深くしない
- **定数は PHP 8.1+ の Backed Enum で表現する**（`ContentType::class`, `ContentStatus::class`）

```php
<?php

declare(strict_types=1);

namespace App\Services;

use App\Models\User;
use App\Models\LearningSession;

final class DashboardService
{
    public function getWeeklySummary(User $user, string $timezone): int
    {
        // ✅ 早期 return + 型安全
        $start = now($timezone)->startOfWeek()->utc();

        return (int) LearningSession::where('user_id', $user->id)
            ->where('started_at', '>=', $start)
            ->sum(\DB::raw('TIMESTAMPDIFF(MINUTE, started_at, ended_at)'));
    }
}
```

---

## 2. ファイル・クラス命名

| 対象 | 規則 | 例 |
|------|------|----|
| クラスファイル | PascalCase | `ContentController.php`, `LearningContent.php` |
| インターフェース | PascalCase + `Interface` サフィックス | `ContentQueryInterface.php` |
| Enum | PascalCase | `ContentType.php`, `ContentStatus.php` |
| マイグレーション | snake_case（Artisan 自動生成名） | `2024_01_01_000000_create_learning_contents_table.php` |
| テストクラス | 対象クラス名 + `Test` | `ContentControllerTest.php` |
| Vue コンポーネント | PascalCase | `ContentCard.vue`, `SessionForm.vue` |
| Vue ページ | PascalCase（Inertia の規約） | `Contents/Index.vue`, `Dashboard.vue` |
| TypeScript ファイル | camelCase | `useContentForm.ts`, `useSessionTimer.ts` |
| Composable | `use` プレフィックス + PascalCase | `useContentForm.ts` |

---

## 3. Controller 設計

### 責務

Controller は薄く保つ。以下のみを行う:

1. リクエストのバリデーション（Form Request に委譲）
2. 認可チェック（`$this->authorize()` または `$this->authorizeForUser()`）
3. Service / Model の呼び出し
4. Inertia レスポンスまたはリダイレクトの返却

**ビジネスロジック（集計クエリ・複雑な条件分岐等）は Service クラスに実装する。**

```php
// ✅ 正しいパターン
public function store(StoreContentRequest $request): RedirectResponse
{
    $this->authorize('create', LearningContent::class);

    $content = $this->contentService->createContent($request->user(), $request->validated());

    return redirect()->route('contents.show', $content)
        ->with('success', 'コンテンツを追加しました');
}

// ❌ 悪いパターン（Controller にビジネスロジックを書かない）
public function store(Request $request): RedirectResponse
{
    $validated = $request->validate([...]);
    $content = new LearningContent();
    $content->user_id = auth()->id();
    // ...複雑なロジック...
    $content->save();
    return redirect()->route('contents.index');
}
```

---

## 4. Eloquent モデル

- **`$guarded = []` を使わない** — `$fillable` を明示的に定義する
- **アクセサ / ミューテータは PHP 8.0+ の新形式（`Attribute::make`）を使う**
- **リレーション名は snake_case で定義する**（Eloquent の規約）
- **N+1 問題を防ぐ** — Controller/Service では必要なリレーションを `with()` で eager load する

```php
// app/Models/LearningContent.php
class LearningContent extends Model
{
    protected $fillable = [
        'user_id', 'title', 'content_type', 'author', 'url',
        'description', 'status', 'notes', 'thumbnail_path',
    ];

    protected $casts = [
        'content_type' => ContentType::class,
        'status'       => ContentStatus::class,
    ];

    // ✅ PHP 8.0+ アクセサ形式
    protected function thumbnailUrl(): Attribute
    {
        return Attribute::make(
            get: fn () => $this->thumbnail_path
                ? \Storage::url($this->thumbnail_path)
                : null,
        );
    }
}
```

---

## 5. Form Request（バリデーション）

- バリデーションは必ず **Form Request クラス**に切り出す。`$request->validate()` の直書きを禁止する
- `authorize()` は原則 `true` を返す（認可は Controller の `$this->authorize()` で行う）
- カスタムメッセージは `messages()` メソッドに日本語で記述する

```php
// app/Http/Requests/StoreSessionRequest.php
class StoreSessionRequest extends FormRequest
{
    public function authorize(): bool
    {
        return true;
    }

    public function rules(): array
    {
        return [
            'started_at' => ['required', 'date'],
            'ended_at'   => ['required', 'date', 'after:started_at'],
            'memo'       => ['nullable', 'string', 'max:1000'],
        ];
    }

    public function messages(): array
    {
        return [
            'ended_at.after' => '終了日時は開始日時より後にしてください。',
        ];
    }
}
```

---

## 6. API Resource（レスポンス整形）

- Controller から JSON データを返す場合は必ず **API Resource** を使う。`$model->toArray()` の直接返却を禁止する
- コレクションには `ResourceClass::collection($collection)` を使う
- `when()` / `whenLoaded()` で条件付きフィールドを制御する

```php
// app/Http/Resources/ContentResource.php
public function toArray(Request $request): array
{
    return [
        'id'             => $this->id,
        'title'          => $this->title,
        'content_type'   => $this->content_type->value,
        'status'         => $this->status->value,
        'thumbnail_url'  => $this->thumbnail_url, // アクセサ経由
        'tags'           => TagResource::collection($this->whenLoaded('tags')),
        // ✅ eager load されていない場合は undefined になるため N+1 を防げる
        'total_study_minutes' => (int) ($this->total_study_minutes ?? 0),
        'updated_at'     => $this->updated_at->toISOString(),
    ];
}
```

---

## 7. 認可（Policy）

- **すべてのリソース操作に Policy を適用する**。Controller で `$this->authorize()` を必ず呼ぶ
- Policy クラスは `app/Policies/` に配置し、`AppServiceProvider` で登録する
- ログインユーザーの ID とリソースの `user_id` が一致するかを必ず検証する

```php
// ✅ Controller 内の認可チェック
public function update(UpdateContentRequest $request, LearningContent $content): RedirectResponse
{
    $this->authorize('update', $content); // ← 必ず呼ぶ
    // ...
}
```

---

## 8. データフェッチ・ミューテーション（ADR-0007 参照）

### データ取得（Controller → Inertia::render）

```php
// ✅ 正しいパターン: Controller でデータを取得し Inertia に渡す
return Inertia::render('Contents/Index', [
    'contents' => ContentResource::collection($contents),
    'filters'  => $request->only(['q', 'type', 'status']),
]);
```

### データ更新（Inertia router → Controller）

```typescript
// ✅ Inertia の useForm または router を使う
const form = useForm({ title: '', content_type: 'BOOK' });
form.post(route('contents.store'));

// ✅ 個別フィールドの即時更新
router.patch(route('contents.status', props.content.id), { status: newStatus });

// ❌ axios / fetch で直接 API を叩かない（タグ即時作成を除く）
```

### タグ即時作成（例外）

タグの即時作成のみ、`axios.post` を使用する（ADR-0017 参照）。

```typescript
// ✅ タグ新規作成のみ axios を許可
const { data } = await axios.post<{ id: number; name: string }>(
    route('tags.store'),
    { name: newTagName },
);
```

---

## 9. リダイレクト後のフラッシュメッセージ

- 成功時: `->with('success', 'コンテンツを追加しました')`
- エラー時: `->with('error', '操作に失敗しました')`
- フラッシュは `HandleInertiaRequests` ミドルウェアの `share()` で全ページに伝播させる
- フロントエンドでは `AppLayout.vue` が `usePage().props.flash` を監視してトーストを表示する

```php
// app/Http/Middleware/HandleInertiaRequests.php
public function share(Request $request): array
{
    return array_merge(parent::share($request), [
        'auth' => [
            'user' => $request->user(),
        ],
        'flash' => [
            'success' => fn () => $request->session()->get('success'),
            'error'   => fn () => $request->session()->get('error'),
        ],
    ]);
}
```

---

## 10. Vue 3 / TypeScript（フロントエンド）

### Composition API（`<script setup>`）を必ず使う

```vue
<!-- ✅ Composition API を使う -->
<script setup lang="ts">
import { ref, computed } from 'vue';
import { useForm, router } from '@inertiajs/vue3';

const props = defineProps<{ content: Content }>();
const form = useForm({ title: props.content.title });
</script>

<!-- ❌ Options API は使わない -->
<script lang="ts">
export default {
  data() { return { title: '' }; }
}
</script>
```

### Props の型定義

- Props は TypeScript インターフェースで型付けする。`any` の使用を禁止する
- Inertia ページコンポーネントの Props は `defineProps<PageProps>()` で受け取る

### Tailwind CSS

- クラスの結合は `clsx` または三項演算子を使う
- インラインスタイル（`style=""`）は禁止。Tailwind クラスで表現できない場合のみ CSS 変数を使う

```vue
<!-- ✅ clsx + tailwind クラス -->
<span :class="clsx('px-2 py-1 rounded text-sm', {
    'bg-green-100 text-green-800': status === 'COMPLETED',
    'bg-blue-100 text-blue-800': status === 'IN_PROGRESS',
})">
  {{ statusLabel }}
</span>
```

---

## 11. テスト

### PHPUnit（`tests/Feature/` および `tests/Unit/`）

- **Feature テスト**: HTTP リクエスト → Controller → DB → レスポンスまでをカバーする
- **Unit テスト**: 単一の Service クラス / Eloquent メソッドをテストする
- テストには `RefreshDatabase` トレイトを使い、毎回クリーンな状態でテストする
- Factory を必ず使う。`new Model([...])` の直書きを禁止する

```php
// tests/Feature/ContentTest.php
class ContentTest extends TestCase
{
    use RefreshDatabase;

    public function test_authenticated_user_can_create_content(): void
    {
        $user = User::factory()->create();

        $this->actingAs($user)
            ->post(route('contents.store'), [
                'title'        => 'Laravel入門',
                'content_type' => 'BOOK',
                'status'       => 'NOT_STARTED',
            ])
            ->assertRedirect()
            ->assertSessionHas('success');

        $this->assertDatabaseHas('learning_contents', [
            'user_id' => $user->id,
            'title'   => 'Laravel入門',
        ]);
    }
}
```

### Vitest（フロントエンドユニットテスト）

- 純粋なロジックを持つ Composable（`useSessionTimer.ts` 等）を対象とする
- Vue コンポーネントの E2E テストはスコープ外

---

## 12. コードフォーマット

### PHP（Laravel Pint）

```json
// pint.json
{
    "preset": "laravel"
}
```

コミット前に `composer pint` を実行する。

### JavaScript / TypeScript（ESLint + Prettier）

`.prettierrc`:
```json
{
    "semi": true,
    "singleQuote": true,
    "printWidth": 100,
    "tabWidth": 4,
    "trailingComma": "all",
    "plugins": ["prettier-plugin-tailwindcss"]
}
```

---

## 13. セキュリティ

- **CSRF**: Inertia は `X-XSRF-TOKEN` を自動付与するため、Laravel のデフォルト CSRF 保護が有効
- **SQL インジェクション**: Eloquent の Query Builder を使う。生クエリは `DB::raw` に PDO バインディングを必ず使う
- **XSS**: Vue の `{{ }}` は自動エスケープ。`v-html` の使用を禁止する
- **ファイルアップロード**: MIME タイプ・ファイルサイズを Form Request でバリデーションし、`store()` メソッドで `storage/app/public` にのみ保存する。元のファイル名はそのまま使用せず、`Storage::put` が生成するランダムなパスを使う
- **認可**: すべての Controller メソッドで `$this->authorize()` を呼び、他ユーザーのリソースへのアクセスを防ぐ
