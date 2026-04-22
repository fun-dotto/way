---
name: pr
description: ベースブランチを明示または既定値(main)で決定し、ベースとの差分全体を根拠に日本語でPRタイトル/本文を作成する。PRテンプレートがあれば形式を継承し、自己アサイン付きのDraft PRを作成する。
license: MIT
metadata:
  author: Kanta Oikawa
  version: "1.0"
---

PRを一貫した手順で作成するためのスキル。

## 目的

- ベースブランチを明確に定める（未指定時は `main`）。
- ベースブランチとの差分全体を根拠にPRタイトル/本文を作る。
- PRタイトル・本文は日本語で記述する。
- PRテンプレートがある場合は、その形式を維持して本文を作る。
- PRを `Draft` で作成し、自分自身をアサインする。

## 手順

### 1. ベースブランチを決める

- 指定がなければ `main` をベースブランチにする。
- 指定がある場合は、そのブランチをベースブランチにする。

例:

```bash
# 未指定の場合
BASE_BRANCH=main

# 指定がある場合
BASE_BRANCH=<指定されたブランチ名>
```

### 2. ベースとの差分全体を確認する

PR本文はステージ状態ではなく、ブランチ差分全体から作る。

```bash
git fetch origin

# 変更ファイルと差分量を把握
git diff --stat origin/${BASE_BRANCH}...HEAD

# 差分の実体を確認
git diff origin/${BASE_BRANCH}...HEAD

# コミット一覧を確認
git log --oneline origin/${BASE_BRANCH}..HEAD
```

- この差分全体を要約して、PRタイトルと本文を作る。
- 一部のコミットメッセージだけで本文を作らない。

### 3. PRテンプレートの有無を確認する

```bash
ls .github/PULL_REQUEST_TEMPLATE.md .github/pull_request_template.md .github/PULL_REQUEST_TEMPLATE/*.md 2>/dev/null
```

- テンプレートが存在する場合は、見出し・チェック項目・構造を維持したまま内容を埋める。
- テンプレートが複数ある場合は、リポジトリ運用で指定されたものを優先する。

### 4. PRタイトル/本文を日本語で作成する

- タイトルは変更の主目的を日本語で簡潔に表現する。
- 本文はベース差分全体を要約し、背景・変更点・確認内容を日本語で記述する。
- テンプレートがある場合は、テンプレートの節ごとに日本語で埋める。

### 5. Draft + 自己アサインでPRを作成する

```bash
gh pr create \
  --base "${BASE_BRANCH}" \
  --draft \
  --assignee "@me" \
  --title "<日本語タイトル>" \
  --body-file "<本文ファイル>"
```

- 必ず `--draft` を付ける。
- 必ず `--assignee @me` を付ける。
- タイトル・本文は日本語に統一する。

### 6. 作成後に内容を確認する

```bash
gh pr view --json title,body,isDraft,assignees,baseRefName,url
```

- `isDraft=true` であることを確認する。
- assignees に自分が含まれていることを確認する。
- タイトル/本文が日本語で、かつ差分全体を反映していることを確認する。

## 禁止事項

- ベースブランチ未指定時に `main` 以外を既定値にすること。
- ベース差分を確認せずにPRタイトル/本文を書くこと。
- 英語でPRタイトル/本文を書くこと。
- テンプレートがあるのに独自形式へ崩すこと。
- Draft 以外で作成すること。
- 自分をアサインせずに作成すること。
