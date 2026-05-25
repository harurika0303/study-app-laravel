# ADR-0019: サムネイル画像は削除・差し替え時に物理削除する

| 項目 | 内容 |
|------|------|
| ステータス | 承認済み |
| 決定日 | 2026-05-25 |

## 背景

`learning_contents.thumbnail_path` にアップロードファイルのストレージパスを保存する。
コンテンツが削除される、またはサムネイルが別ファイルに差し替えられたとき、
**旧ファイルをどう扱うか**を決定する必要があった。

| 選択肢 | メリット | デメリット |
|--------|---------|-----------|
| **A: 即時物理削除** | ストレージを無駄に消費しない。孤立ファイルが発生しない | Controller でファイル削除ロジックが必要 |
| B: 削除せず放置（定期クリーンアップ） | Controller がシンプル | 孤立ファイルが増え続ける。クリーンアップ Artisan コマンドの追加実装が必要 |

## 決定

**Option A**（即時物理削除）を採用する。

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

## 結果

- `ContentController::update` と `ContentController::destroy` にファイル削除ロジックが追加される
- ストレージに孤立ファイルが残らない
- アップロード失敗時に DB が更新されないよう、`DB::transaction` 内でファイル保存と DB 更新を行う
