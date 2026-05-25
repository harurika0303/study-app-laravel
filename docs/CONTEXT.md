# CONTEXT

## 用語集

### LearningContent（学習コンテンツ）
ユーザーが学習対象として登録した書籍・動画コース・資格のエンティティ。

**同義語**: コンテンツ
**非同義語**: LearningSession（学習の「記録」であり、コンテンツ自体ではない）

---

### LearningSession（学習セッション）
ある LearningContent に対して行った1回分の学習を記録したエンティティ。開始日時・終了日時・メモを持つ。

**同義語**: セッション
**非同義語**: LearningContent（学習対象の情報であり、記録ではない）

---

### ContentType（コンテンツ種別）
LearningContent の種類を表す列挙型。`BOOK`（書籍）/ `VIDEO`（動画コース）/ `CERTIFICATION`（資格）の3値。

**非同義語**: ContentStatus（進捗状態を表す別の列挙型）

---

### ContentStatus（コンテンツステータス）
LearningContent の進捗状態を表す列挙型。ContentType によって使用できる値が異なる。

| ContentType | 使用可能なステータス |
|---|---|
| BOOK / VIDEO | NOT_STARTED / IN_PROGRESS / COMPLETED |
| CERTIFICATION | NOT_STARTED / IN_PROGRESS / PASSED / FAILED |

**非同義語**: ContentType（種別を表す別の列挙型）

---

### Tag（タグ）
ユーザーが LearningContent に付与するラベル。ユーザースコープで一意（`user_id + name` がユニーク制約）。

**非同義語**: ContentType（タグは任意のラベルであり、固定の分類ではない）

---

### 孤立タグ
`content_tag` 中間テーブルに紐づいていない Tag 行。ADR-0017 の即時 INSERT 方式により発生しうる。機能上は問題なし。

---

### タイムゾーン Cookie（`tz`）
ブラウザで取得したユーザーのタイムゾーン文字列（例: `Asia/Tokyo`）を格納する Cookie。DB は UTC 保存のため、表示時・集計時にこの値を使用する。

---

### エクスポート
ユーザーの全 LearningContent と全 LearningSession を JSON または CSV 形式でダウンロードする機能。ファイル名は `study-app-export-YYYYMMDD.json/.csv`。

**非同義語**: インポート（ADR-0015 によりスコープ外）

---

### セッションページネーション
コンテンツ詳細画面のセッション一覧は 1 ページあたり 20 件で表示する。全件取得は行わない。

---

### メール認証
ユーザー登録後にメール確認リンクを送信し、確認済みでないと認証済みルートにアクセスできない仕組み。`routes/web.php` の `verified` ミドルウェアで制御する。開発時は `MAIL_MAILER=log` でログ出力。

---

### MonthlyStat
`getMonthlyStats` が返す月次集計の1エントリ。`month`（YYYY-MM）・`added_count`（新規追加数）・`completed_count`（完了数）・`study_minutes`（学習時間）の4フィールドを持つ。完了数は `updated_at` を完了日時のプロキシとして算出する。

**非同義語**: heatmapData（日次集計）

---

### ContentTypeStats
`getContentTypeStats` が返す `{BOOK: N, VIDEO: N, CERTIFICATION: N}` 形式のコンテンツ件数マップ。`ContentTypeChart`（円グラフ）の入力データ。

---

### Error.vue
`Pages/Error.vue` に実装する統一エラーページ。HTTP ステータスコード（403/404/500）を props として受け取り、対応する日本語メッセージを表示する。

---

### CalendarHeatmap カラーパレット
0分=グレー、1〜30分・31〜60分・61分以上の3段階を濃緑で表現するGitHub形式のヒートマップ配色。

---

### タグ ComboBox
ContentForm のタグ入力に使用する自作コンポーネント。テキスト入力で既存タグをフィルタし、Enter キーまたは「追加」ボタンで確定する。未登録タグ名を確定した場合は即時 DB INSERT（ADR-0017）。1コンテンツ最大10タグ。
