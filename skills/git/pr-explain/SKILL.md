---
name: pr-explain
description: PR やローカル変更の内容を包括的に収集・分析し、対応概要・背景・実装の順で丁寧に解説する。引数なしの場合はローカル変更 → PR → 直近コミットの優先度で解説対象を決定する。Use when PR の解説、変更内容の説明、コミットの概要把握を求められた際に使用する。
---

# PR 解説ワークフロー

PR の変更内容を包括的に収集・分析し、対応概要・背景説明・実装説明の順で丁寧に解説する。引数として PR の URL または番号を受け取れる。

## 重要な原則

1. **情報を包括的に収集する** - diff だけでなく、PR description、レビューコメント、作成者のコメント、コミット履歴すべてから情報を集める
2. **解説は常に日本語で行う** - 技術用語や固有名詞は原文のまま維持
3. **日本語の記述は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用
4. **理解しやすい順序で解説する** - コードの変更順ではなく、処理の流れやアーキテクチャの観点から説明
5. **レビュー議論は背景・実装に自然に統合する** - 独立セクションにせず、関連する文脈に織り込む

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "解説対象の特定: 引数、ローカル変更、PR、直近コミットの優先度で対象を特定" })
todo({ action: "add", text: "PR 情報の収集: メタデータ、diff、レビューコメント、PR コメントを収集" })
todo({ action: "add", text: "コードベースの確認: 変更対象のコードを読み、文脈を把握" })
todo({ action: "add", text: "解説の作成・出力: 収集した情報をもとに解説を作成" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. 解説対象の特定

引数が渡された場合、PR として特定する。URL 形式と番号形式の両方に対応する:

```bash
# URL 形式: https://github.com/owner/repo/pull/123
# owner, repo, number を URL から抽出し、以降の全ての gh pr view / gh pr diff
# に -R <owner>/<repo> を付与する (-R なしではカレントリポジトリの同一番号の
# 無関係な PR を解説してしまう可能性がある)
gh pr view <number> -R <owner>/<repo> --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'

# 番号形式: 123 または #123
# 現在のリポジトリの PR として扱う
gh pr view <number> --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'

# owner/repo を取得 (番号形式の場合、GraphQL クエリで使用)
gh repo view --json owner,name --jq '"\(.owner.login)/\(.name)"'
```

URL から owner/repo を抽出できない場合は、現在のリポジトリの owner/repo を使用する。

引数が渡されていない場合、以下の優先度で解説対象を決定する:

1. **staged/unstaged の変更がある場合** → ローカル変更を解説対象とする

   ```bash
   git diff --stat
   git diff --cached --stat
   ```

   いずれかに変更があれば、`git diff` と `git diff --cached` の内容を解説する。この場合、Step 2 の PR 情報収集はスキップし、diff の内容とコードベースの確認のみで解説を作成する。

2. **現在のブランチに PR が紐づいている場合** → その PR を解説対象とする

   ```bash
   gh pr view --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'
   ```

3. **上記いずれでもない場合** → 直近のコミットを解説対象とする
   ```bash
   git show HEAD
   ```
   この場合も Step 2 の PR 情報収集はスキップし、コミットの diff 内容とコードベースの確認で解説を作成する。

### 2. PR 情報の収集

以下の情報を **並列で** 収集する:

**PR メタデータと description:**

```bash
# URL 指定の PR の場合は以降の全コマンドに -R <owner>/<repo> を付与する
gh pr view <number> -R <owner>/<repo> --json title,body,author,baseRefName,headRefName,labels,additions,deletions,changedFiles,createdAt
```

**コミット履歴:**

```bash
gh pr view <number> -R <owner>/<repo> --json commits --jq '.commits[] | "\(.oid[0:7]) \(.messageHeadline)\n\(.messageBody)"'
```

**コード差分:**

```bash
# 変更ファイル一覧
gh pr diff <number> -R <owner>/<repo> --name-only

# 全差分
gh pr diff <number> -R <owner>/<repo>
```

**レビューコメント (インライン):**

```bash
# レビュースレッドのコメントを取得 (GraphQL で完全な議論を取得)
# <owner>, <repo>, <number> は実際の値に置き換える
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <number>) {
      reviews(first: 50) {
        nodes {
          body
          state
          author { login }
        }
      }
      reviewThreads(first: 100) {
        nodes {
          isResolved
          comments(first: 20) {
            nodes {
              body
              path
              line
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}'
```

**PR コメント (一般):**

```bash
gh pr view <number> --json comments --jq '.comments[] | "\(.author.login): \(.body)"'
```

### 3. コードベースの確認

diff だけでは文脈が不十分な場合、変更対象ファイルの全体像を確認する:

- 変更されたファイルを read tool で読み、変更箇所の前後の文脈を把握する
- 関連する型定義や依存先のコードも必要に応じて確認する
- 新規ファイルの場合は、類似ファイルがあれば参照して設計意図を把握する

**確認の判断基準:**

| 状況             | 対応                              |
| ---------------- | --------------------------------- |
| 新規ファイル追加 | 周辺の類似ファイルを確認          |
| 既存ファイル変更 | 変更箇所の前後を read tool で確認 |
| 型定義の変更     | 使用箇所を grep tool で確認       |
| 設定変更         | 関連する設定ファイルを確認        |

### 4. 解説の作成・出力

収集した情報を以下の構造で解説する:

```markdown
## 対応概要

[PR の変更内容を 2-3 文で簡潔に説明する。何が変わったのかを端的に伝える。]

## 背景説明

[なぜこの変更が必要だったのかを説明する。以下の情報源から背景を構成する:]

- PR の description に記載された動機や課題
- 関連する Issue の内容
- レビューでの議論から明らかになった設計判断の理由
- コミットメッセージに含まれる意図の説明

## 実装説明

[変更内容を処理の流れが理解しやすい順序で丁寧に解説する:]

- コードの変更順 (diff 順) ではなく、論理的な理解の順序で説明する
- 新しい概念やデータ構造を先に説明してから、それを使う処理を説明する
- ファイルパスと行番号を含めて具体的に参照する
- レビューで議論になった設計判断は、該当箇所の説明に自然に織り込む
```

**解説の品質基準:**

- **対応概要**: 読むだけで変更の全体像が掴めること
- **背景説明**: 「なぜこの変更をしたのか」が明確にわかること
- **実装説明**: コードを読まなくても処理の流れが理解できること

**大規模な差分 (目安: 変更ファイル 30 以上) の場合:**

コンテキストウィンドウの制約上、全ファイルの diff を一度に処理できないため、以下の戦略を取る:

- 全ファイルの diff を一度に読み込まず、変更ファイル一覧から主要な変更を特定する
- ディレクトリ単位で変更の傾向をまとめ、重要なファイルに絞って詳細解説する
- 自動生成やリネームのみの変更はまとめて言及し、個別解説は省略する

**実装説明の順序決定方法:**

1. アーキテクチャレベルの変更 (新しいモジュール、ディレクトリ構成の変更) → 先に説明
2. データモデル・型定義の変更 → 次に説明
3. コアロジックの変更 → 処理の流れに沿って説明
4. UI・表示の変更 → 最後に説明
5. 設定・インフラの変更 → 関連する箇所で説明

## エラーハンドリング

### gh CLI が使用できない場合

`gh api graphql` でメインワークフローと同じ GraphQL クエリを実行する:

```bash
# ステップ 2 の GraphQL クエリを gh api graphql で実行
# <owner>, <repo>, <number> は実際の値に置き換える
gh api graphql -F query='...'

# PR メタデータは REST API で取得
gh api repos/<owner>/<repo>/pulls/<number>
```

### 外部リポジトリの PR の場合

URL から owner/repo を抽出し、`gh api` で情報を取得する:

```bash
# リポジトリをクローンせずに PR 情報を取得
gh api repos/<owner>/<repo>/pulls/<number>
gh api repos/<owner>/<repo>/pulls/<number>/comments
gh api repos/<owner>/<repo>/pulls/<number>/reviews
```

### PR が見つからない場合

```
指定された PR が見つかりません。URL または PR 番号を確認してください。
例: /skill:pr-explain https://github.com/owner/repo/pull/123
```
