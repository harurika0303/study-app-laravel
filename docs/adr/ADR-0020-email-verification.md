# ADR-0020: メール認証を有効化する

| 項目 | 内容 |
|------|------|
| ステータス | 承認済み |
| 決定日 | 2026-05-25 |

## 背景

`routes/web.php` のすべての認証済みルートに `verified` ミドルウェアが適用されている。
`verified` は Laravel の `EnsureEmailIsVerified` ミドルウェアであり、
`User` モデルが `MustVerifyEmail` を実装している場合にのみ機能する。

**メール認証を有効にするかどうか**を明示的に決定する必要があった。

| 選択肢 | メリット | デメリット |
|--------|---------|-----------|
| **A: 有効化** | セキュリティ向上。未確認メールアドレスのなりすましを防止 | 登録フローに一手間増える。開発時はメール確認が必要 |
| B: 無効化（`verified` を除去） | 登録後すぐに使い始められる | 到達不可能なアドレスでも登録できる |

## 決定

**Option A**（メール認証を有効化）を採用する。

```php
// app/Models/User.php
class User extends Authenticatable implements MustVerifyEmail
{
    // ...
}
```

開発時は `.env` の `MAIL_MAILER=log` を使用し、`storage/logs/laravel.log` にメール内容が出力される。

## 結果

- `User` モデルに `implements MustVerifyEmail` を追加する
- Laravel Breeze が生成する `/email/verify` および `/email/verification-notification` ルートをそのまま使用する
- `routes/web.php` の `verified` ミドルウェアはそのまま維持する
- 本番環境では `MAIL_MAILER` を SMTP 等に変更する（`.env.example` に `MAIL_MAILER=log` として明示）
