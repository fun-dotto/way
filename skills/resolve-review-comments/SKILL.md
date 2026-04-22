---
name: resolve-review-comments
description: "GitHub PR のレビューコメント対応を end-to-end で進めたいときに使います。現在の PR の未解決スレッドを特定し、内容を整理し、順に修正し、修正ごとにコミットし、対応済みスレッドを `gh` で Resolve します。"
license: MIT
metadata:
  author: Kanta Oikawa
  version: "1.0"
---

# GH PR コメント対応

## 概要

チェックアウト済みのリポジトリで、GitHub Pull Request のレビュー指摘を順番に処理するときに使うスキルです。
対象は「未解決レビューコメントの洗い出し -> 修正 -> 修正ごとのコミット -> 対応済みスレッドの Resolve」という一連の流れです。

## 使うタイミング

- ユーザーが未解決の PR コメントや review thread を列挙したいとき
- ユーザーが PR 指摘を 1 件ずつ対応したいとき
- ユーザーが対応ごとにコミットを分けたいとき
- ユーザーが対応済みスレッドを GitHub 上で `Resolved` にしたいとき

PR 文脈のない一般的なコードレビューには使いません。GitHub 上のスレッド更新が不要で、ローカル要約だけ欲しいケースにも向きません。

## 前提条件

- リポジトリに GitHub remote がある
- `gh` がインストール済みで認証されている
- `gh pr` / `gh api` 用のネットワークアクセスがある
- 現在ブランチに紐づく PR がある、または PR 番号が分かっている

## ワークフロー

### 1. 対象 PR を特定する

最初に、現在ブランチに対応する PR を特定します。

- ブランチ確認: `git branch --show-current`
- 必要に応じて GitHub 認証確認: `gh auth status`
- PR 特定: `gh pr status`

`gh pr status` だけで足りない場合は `gh pr view <number>` を使います。

### 2. 未解決スレッドを列挙する

スレッドの Resolve 状態は `gh pr view --json` では取りづらいので、基本は GraphQL を使います。

例えば以下のクエリを使います。

```bash
gh api graphql -f query='
query {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          isOutdated
          path
          line
          comments(first: 10) {
            nodes {
              body
              url
            }
          }
        }
      }
    }
  }
}'
```

取得後は `isResolved == false` のものだけに絞ります。

ユーザー向け一覧には以下を含めます。

- コメント本文から読める severity (`MUST`, `SHOULD`, `NITS`)
- ファイルと行番号
- 短い要約
- GitHub discussion URL

### 3. 対応要否をトリアージする

列挙した未解決スレッドについて、修正に着手する前に 1 件ずつ対応要否を判断します。各スレッドを次のいずれかに分類します。

- **対応する**: 指摘に妥当性があり、コード変更で解消すべきもの
- **対応不要 (Resolve)**: すでにブランチ上で解消済み、または現行コードで満たされているもの
- **対応見送り (Resolve しない)**: スコープ外、別 PR / 別 issue に切り出すべき、指摘が現状のコード方針と不整合、など
- **判断保留**: 以下の「判断に迷う場合」に該当するもの

判断に迷う場合は独断で進めず、必ずユーザーに判断を仰ぎます。典型例:

- 指摘の意図や前提が読み取れない
- 対応すると設計方針や他の指摘と衝突する
- スコープ外に見えるが、PR の目的との線引きが曖昧
- 対応コストが大きく、本 PR で対応すべきか別 PR に分けるべきか判断がつかない
- 指摘自体に同意できない、または反論の余地がある

ユーザーに確認するときは、対象スレッドの URL・要約・選択肢 (対応する / 対応不要で Resolve / 見送って Resolve しない / 別 PR に切り出す など) を提示します。ユーザーの回答を得てから §4 以降に進みます。

トリアージ結果はユーザーに一覧で共有し、合意を取ってから修正に着手します。

### 4. 順番に処理する

ユーザーから別指定がない限り、「対応する」に分類したスレッドを決まった順序で処理します。

推奨順序:

1. `MUST`
2. `SHOULD`
3. `NITS`
4. 各 severity 内では GitHub 上の順序を保つ

コード変更前にやること:

- 関連ファイルと周辺ロジックを読む
- その指摘がすでにブランチ上で解消済みか再確認する
- すでに解消済みなら、無意味な差分を作らずスレッドだけ Resolve する

### 5. 各指摘を修正する

各スレッドについて:

- 指摘を解消する最小限で筋の通ったコード変更を入れる
- 環境が許す範囲でローカル検証する
- 関係ない修正を同じコミットに混ぜない

複数の未解決コメントが 1 つの意図的なリファクタで同時に片付く場合はまとめてもよいですが、コミットの責務は狭く保ち、実際に解消されたスレッドだけを Resolve します。

### 6. 修正ごとにコミットする

修正 1 件ごと、または密接に関連する修正セットごとに 1 コミット作ります。

ルール:

- コミットメッセージは命令形にする
- 合うなら conventional commits を優先する
- ユーザーの指示がない限り amend しない
- ワークスペース内の無関係な変更を含めない

典型的な流れ:

```bash
git status --short
git add <paths>
git commit -m "fix(scope): address PR feedback"
```

dirty worktree で `git add .` を無造作に使わないでください。

### 7. 対応済みスレッドを Resolve する

コード変更をコミットしたら、対応する review thread を GraphQL で Resolve します。

```bash
gh api graphql \
  -f query='mutation($threadId: ID!) { resolveReviewThread(input: {threadId: $threadId}) { thread { id isResolved } } }' \
  -F threadId=THREAD_ID
```

以下のどちらかを満たすときだけ Resolve します。

- コード変更がコミット済みで、そのコメントを明確に解消している
- 現在コードですでに満たされていて、自分で確認済みである

部分対応のまま、未検証のまま、またはブロックされたままのスレッドは Resolve しません。

### 8. 未解決件数を再確認する

各バッチの後、または最後に、未解決スレッドを再取得して残件数を確認します。

最終確認には以下のようなクエリを使います。

```bash
gh api graphql -f query='
query {
  repository(owner: "OWNER", name: "REPO") {
    pullRequest(number: PR_NUMBER) {
      reviewThreads(first: 100) {
        nodes { isResolved }
      }
    }
  }
}' --jq '[.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false)] | length'
```

## ガードレール

- 壊れやすいスクレイピングより `gh api graphql` を優先する
- `gh` がネットワークやサンドボックス制限で失敗したら、必要な承認を取って再試行する
- git 操作は直列で進める。`git add` と `git commit` を並列実行しない
- `index.lock` 競合が出たら、lock の状態を確認してから再試行する
- ユーザーがクリーンビルドやテストを求めている場合は、スレッドを Resolve する前に先に検証する
- ビルド失敗が依存や環境由来なら、その点を明示し、完全検証済みとは言わない
- 設計議論が未決着のスレッドは Resolve しない

## 返答の期待値

ユーザーへの報告では以下を含めます。

- 求められたら未解決コメント一覧を分かりやすく出す
- 作成した各コミットハッシュを示す
- 検証できなかった点があれば明示する
- 最後に未解決スレッド件数を添える
