# ADR-0017: 新規タグはタグ入力 UI の確定時点で即座に DB 登録する

| 項目 | 内容 |
|------|------|
| ステータス | 採用 |
| 決定日 | 2026-05-25 |

## 背景

`ContentForm.vue`（コンテンツ追加・編集画面）のタグフィールドには、
既存タグから選択 + 未登録タグ名の即時作成機能を持つマルチセレクト UI を使用する。
ユーザーが未登録のタグ名を入力して確定したとき、「DB への INSERT をいつ行うか」について選択が必要だった。

| タイミング | メリット | デメリット |
|----------|---------|-----------|
| **A: 確定時に即時 INSERT** | フォームのバリデーションスキーマをシンプルに保てる（`tag_ids: number[]`）| フォームを途中で破棄しても tags テーブルに行が残る（孤立タグ）|
| B: フォーム送信時にバックエンドで名前 → ID 解決 | 確定したタグのみ INSERT される | バリデーションスキーマが複雑化（`{ id?: number; name?: string }[]`）|

## 決定

**Option A**（確定時点で即時 INSERT）を採用する。

タグ入力 UI でユーザーが新しいタグ名を確定した時点で、`axios.post(route('tags.store'), { name })` を呼んで DB に INSERT し、
返却された ID を選択状態にセットする。フォーム送信時は `tag_ids: number[]` の配列のみを渡す。

```typescript
// ContentForm.vue 内（実装イメージ）
async function handleTagCreate(newTagName: string): Promise<void> {
    const { data } = await axios.post<{ id: number; name: string }>(
        route('tags.store'),
        { name: newTagName },
    );
    selectedTagIds.value.push(data.id);
}
```

## 孤立タグの扱い

- 孤立タグ（`content_tag` に紐づかない `tags` 行）は機能上問題ない（設定画面のタグ一覧に表示される）
- 将来的な問題になる場合は `php artisan tags:cleanup` のような Artisan コマンドでバッチ削除する（現 MVP スコープ外）

## 影響

- `TagController::store` は `axios.post` から呼ばれる JSON レスポンスを返すエンドポイントとして実装する
- `routes/web.php` の `tags.store` は通常の Inertia リダイレクトではなく `response()->json()` を返す
