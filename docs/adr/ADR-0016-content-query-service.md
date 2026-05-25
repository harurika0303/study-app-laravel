# ADR-0016: コンテンツ一覧取得に ContentQueryService を使用する

| 項目 | 内容 |
|------|------|
| ステータス | 採用 |
| 決定日 | 2026-05-25 |

## 背景

コンテンツ一覧画面では以下の要件がある。

1. フィルター（タイトル検索・コンテンツ種別・ステータス・タグ）
2. 4 種類のソート（更新日降順・作成日降順・タイトル昇順・**学習時間降順**）
3. 各カードに「累計学習時間」を表示する
4. ページネーション（LIMIT/OFFSET）と総件数を同時に取得する

累計学習時間は `learning_sessions` の `TIMESTAMPDIFF(MINUTE, started_at, ended_at)` の合計値であり、
`learning_contents` と LEFT JOIN + GROUP BY + SUM が必要。
Eloquent の `select()` だけでは動的 ORDER BY（`study_time_desc` など）を安全に組み立てられず、
集計 + ページネーション + 総件数の同時取得も複雑になる。

## 決定

`ContentQueryService` クラスを作成し、フィルタ・ソート・ページネーションをすべてこのクラスに集約する。

```php
// app/Services/ContentQueryService.php
$query = LearningContent::where('user_id', $user->id)
    ->with('tags')
    ->withSum('sessions as total_study_minutes',
        DB::raw('TIMESTAMPDIFF(MINUTE, started_at, ended_at)'));

// フィルター・ソートを適用
$paginator = $query->paginate(20, page: $page);
```

`ContentController::index` からは `ContentQueryService::getPaginatedContents()` を呼ぶだけにする。

## 却下した代替案

| 案 | 却下理由 |
|----|----------|
| Controller に直接クエリを書く | テストが困難になり、Controller が肥大化する |
| Eloquent Scope で全条件を実装 | ソート・集計・ページネーションが複雑に絡み合い可読性が低下する |
| 生 SQL（`DB::select`）| ORM の恩恵（eager load・Resource 変換）が使えない |

## 影響
- `ContentQueryService` は `tests/Unit/ContentQueryServiceTest.php` でユニットテストを書く
- Controller はサービスへの委譲のみを行い、クエリロジックを持たない
