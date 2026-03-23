---
name: backend-update-api-schema
description: APIスキーマの変更に伴う、バックエンドサービスの変更
---

OpenAPIスキーマ (`openapi/openapi.yaml`) の変更に伴い、コード生成と実装の更新を行うスキル。

## 共通ルール

- 各手順の作業が完了したら、都度コミットすること。まとめてコミットしない。

## 手順

### 1. スキーマの変更内容を把握する

mainブランチとの差分を確認し、OpenAPIスキーマにどのような変更があったかを把握する。

```bash
git diff main...HEAD -- openapi/openapi.yaml
```

### 2. コードを生成する

oapi-codegenでAPIコードを再生成する。

```bash
task generate
```

### 3. 生成コードの差分を確認する

生成されたコードの変更点を確認する。新しいレスポンス型、リクエスト型、インターフェースの変更を把握する。

```bash
git diff -- generated/
```

### 4. 実装を変更する

生成コードの変更に合わせて、以下のファイルを必要に応じて修正する。

#### 対象ファイル

- `internal/handler/` — ハンドラー実装。`StrictServerInterface` を満たすメソッド群。
- `internal/handler/converter.go` — ドメインモデルとAPIモデルの変換関数。
- `internal/service/` — ビジネスロジック。
- `internal/repository/` — データアクセス層。
- `internal/domain/` — ドメインモデル・エラー定義。

#### 実装パターン

**新しいレスポンスステータスコードが追加された場合:**

生成コードに新しいレスポンス型（例: `AnnouncementsV1Detail404Response`）が追加される。
ハンドラーで適切な条件に応じてその型を返すように修正する。

例: 404レスポンスが追加された場合

```go
// Before
if errors.Is(err, domain.ErrNotFound) {
    return nil, err
}

// After
if errors.Is(err, domain.ErrNotFound) {
    return api.AnnouncementsV1Detail404Response{}, nil
}
```

**新しいエンドポイントが追加された場合:**

1. `internal/handler/` に新しいハンドラーファイルを作成
2. 必要に応じて `internal/service/`, `internal/repository/` にメソッドを追加
3. `internal/handler/converter.go` に変換関数を追加

**リクエスト/レスポンスのフィールドが変更された場合:**

1. `internal/domain/` のモデルを更新
2. `internal/handler/converter.go` の変換関数を更新
3. `internal/database/` のクエリを更新

### 5. テストを更新する

変更に応じてテストを追加・修正する。

- `internal/handler/` のテストファイル (`*_test.go`)
- 既存テストが壊れていないか確認

### 6. ビルド・テストを実行する

```bash
task build && task test
```

全てパスすることを確認する。
