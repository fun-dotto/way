---
name: deploy
description: リポジトリと環境をカンマ区切りで指定し、GitHub Actions の workflow_dispatch でデプロイをトリガーする。
---

複数リポジトリ・複数環境へのデプロイを一括トリガーするスキル。

## 引数の形式

```
/deploy <repos> <envs>
```

- `<repos>`: カンマ区切りのリポジトリ名。`fun-dotto/` プレフィックスは省略可。
- `<envs>`: カンマ区切りの環境名。

例:

```
/deploy announcement-api,user-api dev,stg,qa
/deploy fun-dotto/announcement-api dev
```

引数が不足している場合はユーザーに確認する。

## リポジトリとワークフロー名の対応

| リポジトリ | ワークフロー名 |
|---|---|
| fun-dotto/dotto-admin-web | CD |
| 上記以外 | Deploy |

## 有効な環境名

- `dev`
- `stg`
- `qa`
- `prod`

指定された環境名がこの一覧に含まれない場合はエラーとしてユーザーに伝える。

## 手順

### 1. 引数を解析する

- リポジトリ名に `fun-dotto/` プレフィックスがなければ付与する。
- カンマ区切りを分割し、リポジトリ一覧と環境一覧を得る。
- 環境名が有効な環境名に含まれるか検証する。

### 2. デプロイをトリガーする

リポジトリと環境の全組み合わせに対して `gh workflow run` を実行する。

```bash
gh workflow run <ワークフロー名> --repo <リポジトリ> -f environment_name=<環境名>
```

- ワークフロー名はリポジトリごとに上記の対応表で決定する。
- コマンドが失敗した場合はその組み合わせを記録し、続行する。

### 3. 結果を報告する

全組み合わせの実行結果を表形式でユーザーに報告する。

例:

| リポジトリ | dev | stg | qa |
|---|---|---|---|
| fun-dotto/announcement-api | ok | ok | ok |
| fun-dotto/user-api | ok | ok | failed |

## 禁止事項

- `prod` 環境が指定された場合、実行前に必ずユーザーへ確認を取ること。確認なしで prod デプロイを実行しない。
- 存在しないリポジトリやワークフローに対して繰り返しリトライしないこと。
