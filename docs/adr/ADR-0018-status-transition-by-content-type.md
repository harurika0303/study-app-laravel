# ADR-0018: ContentStatus の選択肢を ContentType ごとに制限する

| 項目 | 内容 |
|------|------|
| ステータス | 承認済み |
| 決定日 | 2026-05-25 |

## 背景

`ContentStatus` には `NOT_STARTED / IN_PROGRESS / COMPLETED / PASSED / FAILED` の5値がある。
しかし `PASSED` / `FAILED` は資格（CERTIFICATION）固有の概念であり、書籍・動画には意味をなさない。
また `COMPLETED` は資格学習の結果として不適切である（合否があるため）。

スペックには「ステータス利用ルール」テーブルが存在するが、
**バリデーション・UI・Policy のいずれの層で制限するか**が未定義だった。

## 決定

フロントエンド（UI）とバックエンド（Form Request バリデーション）の両層で制限する。

- **UI**: `<select>` の選択肢を `content_type` に応じてフィルタして表示する
  - BOOK / VIDEO: `NOT_STARTED / IN_PROGRESS / COMPLETED`
  - CERTIFICATION: `NOT_STARTED / IN_PROGRESS / PASSED / FAILED`
- **バックエンド**: `UpdateContentStatusRequest::rules()` で `Rule::in($allowed)` を使い、
  許可されていないステータス値はバリデーションエラーとして拒否する

## 代替案

| 案 | 却下理由 |
|---|---|
| DB 制約で制限 | ContentType ごとに許可値が変わるロジックは CHECK 制約では表現しにくい |
| Policy で制限 | Policy は所有権の確認に使う。ドメインルールの実装場所として不適切 |
| フロントのみで制限 | API に直接リクエストされた場合に不正な値を受け入れてしまう |

## 結果

- `UpdateContentStatusRequest` にコンテンツ種別を参照した動的バリデーションが追加される
- `Contents/Show.vue` の `<select>` コンポーネントは `content.content_type` に基づき選択肢をフィルタする
- `SPEC.md` セクション 6.5 および セクション 7 に実装詳細を記載済み
