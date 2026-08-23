---
name: pr-create
description: 現在のブランチから Draft Pull Request を作成する。テンプレート準拠、ラベル自動選択を行う。Use when PR の作成、プルリクエストの作成を求められた際に使用する。
---

# PR 作成ワークフロー

現在のブランチから Draft Pull Request を作成する。引数で `--base <branch>` が指定された場合はそれをベースブランチとして使用する。

## 重要な原則

1. **PR タイトル・description の言語は対象リポジトリに従う** - 既存の PR やコミット履歴を確認し、リポジトリで使用されている言語 (日本語/英語等) に合わせる
2. **日本語で PR タイトル・description を書く場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用する
3. **PR は常に Draft として作成する**
4. **PR タイトルは Conventional Commits に準拠する** - コミットが 1 つの場合はそのメッセージをそのまま使用し、2 つ以上の場合は commit-proposer subagent で生成する
5. **PR テンプレートがある場合は必ず準拠する**
6. **ラベルはリポジトリに存在するもののみ使用する**

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "事前確認: ブランチ状態、リモート差分を確認" })
todo({ action: "add", text: "未コミット変更のコミット: unstaged/staged の変更がある場合のみ commit スキルを実行" })
todo({ action: "add", text: "PR テンプレートの確認: PULL_REQUEST_TEMPLATE.md を探索・読み込み" })
todo({ action: "add", text: "PR タイトルの生成: コミットが 1 つならそのメッセージを使用、2 つ以上なら commit-proposer subagent で生成" })
todo({ action: "add", text: "ラベルの選択: リポジトリのラベル一覧から適切なものを選択" })
todo({ action: "add", text: "Draft PR 作成: gh pr create --draft で PR を作成" })
todo({ action: "add", text: "完了報告: PR URL を報告し、ブラウザで開く" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. 事前確認

以下を確認する:

```bash
# 現在のブランチと状態を確認
git status
git branch --show-current

# ベースブランチを確認 (引数で指定されていない場合は main または master)
git remote show origin | grep 'HEAD branch'

# リモートとの差分を確認 (<base> は上記で確認したベースブランチに置き換える)
git log origin/<base>..HEAD --oneline
```

**確認事項:**

- 未コミットの変更があるか (次のステップで対応)
- リモートにプッシュ済みであること
- ベースブランチとの差分があること

未プッシュの場合は `git push -u origin <branch>` を実行する。

### 2. 未コミット変更のコミット

`git status` の結果から unstaged または staged の変更がある場合のみ実行する。変更がない場合はこのステップをスキップする。

**変更がある場合:** commit スキル (`../commit/SKILL.md`) を read し、その手順に従って実行する。

commit スキルがステージング、コミットメッセージ生成、ユーザー承認、コミット実行を行う。

コミット完了後、未プッシュであれば `git push -u origin <branch>` を実行する。

### 3. PR テンプレートの確認

```bash
# テンプレートファイルを探す
ls -la .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || \
ls -la .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null || \
ls -la docs/PULL_REQUEST_TEMPLATE.md 2>/dev/null
```

テンプレートが存在する場合は read tool で内容を確認し、そのフォーマットに準拠した description を作成する。

### 4. PR タイトルの生成

`git log origin/<base>..HEAD --oneline` を実行し、コミット数を確認する (ステップ 2 で新規コミットが追加された可能性があるため、必ずここで再取得する)。

**コミットが 1 つの場合:** `git log origin/<base>..HEAD -1 --format='%s'` で subject のみ取得し、そのまま PR タイトルとして使用する。commit-proposer subagent の呼び出しはスキップする。

**コミットが 2 つ以上の場合:** commit-proposer subagent を subagent tool で呼び出す。

```
subagent({
  subagent_type: "commit-proposer",
  description: "PR タイトル候補の生成",
  prompt: "PR のコミット履歴から PR タイトル候補を提案してください。ベースブランチ: <base>。PR タイトルとして Conventional Commits 形式で提案してください。"
})
```

subagent がコミット履歴の分析、commitlint 設定の確認、PR タイトル候補の生成を実行する。

### 5. ラベルの選択

```bash
# リポジトリのラベル一覧を取得
gh label list --json name,description
```

変更内容に基づいて適切なラベルを選択する:

| 変更タイプ       | 推奨ラベル               |
| ---------------- | ------------------------ |
| 新機能追加       | `enhancement`, `feature` |
| バグ修正         | `bug`, `fix`             |
| ドキュメント     | `documentation`, `docs`  |
| リファクタリング | `refactor`, `tech-debt`  |
| テスト追加       | `test`, `testing`        |
| 依存関係更新     | `dependencies`           |
| 破壊的変更       | `breaking-change`        |

存在しないラベルは使用しない。

### 6. Draft PR 作成

Draft PR を作成する:

```bash
gh pr create \
  --draft \
  --title "<タイトル>" \
  --body "<説明>" \
  --base <ベースブランチ> \
  --label "<ラベル1>,<ラベル2>" \
  --assignee @me
```

### 7. 完了報告

作成された PR の URL を報告し、ブラウザで開く:

```bash
# PR の URL を取得
gh pr view --json url --jq '.url'

# ブラウザで PR を開く
gh pr view --web
```

**報告フォーマット:**

```
## Draft PR 作成完了

- **PR:** #<number>
- **タイトル:** <タイトル>
- **URL:** <url>
- **状態:** Draft

ブラウザで PR を開きました。
```

## エラーハンドリング

### gh CLI が使用できない場合

GitHub MCP server が構成されていれば `mcp` tool 経由でフォールバックする (PR 作成は draft: true を指定、ラベル一覧の取得も同様)。構成されていない場合はユーザーに gh CLI のセットアップを案内する。

### 認証エラー

```bash
gh auth status
gh auth login
```

### ブランチが存在しない

```bash
git push -u origin $(git branch --show-current)
```
