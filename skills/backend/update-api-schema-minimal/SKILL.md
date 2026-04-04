---
name: backend-update-api-schema-minimal
description: APIスキーマ変更時に、ビルドを通すための最小修正だけを行うスキル。新規エンドポイントは handler で `return nil, fmt.Errorf("not implemented")` を返して StrictServerInterface の不足を埋める。ドメイン/リポジトリ実装は原則追加しない。既存エンドポイントは必要最小限のみ追従する。都度コミットが必要な場面で使う。
---

OpenAPI スキーマ変更に追従しつつ、ビルドを通すことだけを目的に最小変更で進める。

## 手順

### 1. スキーマ差分を確認する

`main` との差分を確認する。

```bash
git diff main...HEAD -- openapi/openapi.yaml
```

### 2. 生成コードを更新する

API コードを再生成する。

```bash
task generate
```

`task generate` が Go patch version 不一致で失敗する場合は `mise` 経由で実行する。

```bash
mise exec -- task generate
```

### 3. 生成差分から不足実装を特定する

追加された `StrictServerInterface` メソッドを確認する。

```bash
git diff -- generated/
```

### 4. handler に最小実装を追加する

- 関数ごとにファイルを分けて追加する。
- 未実装の新規エンドポイントは次だけを返す。

```go
return nil, fmt.Errorf("not implemented")
```

- 未使用引数は `_ context.Context` のようにブランク識別子で受ける。
- `not implemented` のみを返す関数には TODO コメントを付けない。
- ドメイン層・サービス層・リポジトリ層は、ビルドを通すために必須な場合のみ最小変更する。

### 5. ビルドとテストを実行する

```bash
mise exec -- task build
mise exec -- task test
```

### 6. 修正ごとにコミットする

- 生成コード更新
- handler の未実装追加
- 命名/体裁修正

のように、論理単位ごとに小さくコミットする。まとめてコミットしない。
