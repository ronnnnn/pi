---
name: pr-auto
description: 現在のブランチから ready for review の Pull Request を作成し、open 後はそのまま pr-watch スキルで監視・自動修正に移行する。コミット・PR タイトル・description はユーザー確認なしで自動生成・実行する。Use when PR を一気通貫で作成して監視したい、PR を作って即監視したい、PR を自動でシップしたい際に使用する。
---

# PR 自動化ワークフロー

現在のブランチから ready for review の Pull Request を作成し、open 後はそのまま `pr-watch` スキルに監視を引き継ぐ統合ワークフロー。`pr-create` と `pr-watch` を 1 コマンドで連結し、コミット・PR 本文生成・PR 公開・監視を全てユーザー確認なしで自律実行する。引数で `--base <branch>` が指定された場合はそれをベースブランチとして使用する。

## 重要な原則

1. **全工程ユーザー確認なし** - コミット、PR タイトル・description、ラベルの全てを自動生成し、確認をスキップして実行する
2. **PR は ready for review として作成する** - `pr-create` の Draft 作成と異なり、`--draft` を付けずに作成する。既に Draft の PR が存在する場合は `gh pr ready` で ready 化する
3. **PR タイトル・description の言語は対象リポジトリに従う** - 既存の PR やコミット履歴を確認し、リポジトリで使用されている言語 (日本語/英語等) に合わせる
4. **日本語で記述する場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用する
5. **PR タイトルは Conventional Commits に準拠する** - コミットが 1 つの場合はそのメッセージをそのまま使用し、2 つ以上の場合は commit-proposer subagent で生成する
6. **コミットメッセージは commit-proposer subagent で生成する** - Conventional Commits / commitlint 設定に準拠
7. **PR テンプレートがある場合は必ず準拠する**
8. **ラベルはリポジトリに存在するもののみ使用する**
9. **open 後は pr-watch スキルに監視を委譲する** - 監視ロジックは `../pr-watch/SKILL.md` を read して従うことで再利用し、本スキル内に重複定義しない

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "事前確認: ブランチ状態、リモート差分、対象リポジトリの言語を確認" })
todo({ action: "add", text: "未コミット変更の自動コミット: 変更がある場合のみ commit-proposer で生成して確認なしでコミット" })
todo({ action: "add", text: "リモートへのプッシュ: 未プッシュコミットがある場合のみ git push を実行" })
todo({ action: "add", text: "PR テンプレートの確認: PULL_REQUEST_TEMPLATE.md を探索・読み込み" })
todo({ action: "add", text: "PR タイトル・description の生成: コミットが 1 つならそのメッセージを使用、2 つ以上なら commit-proposer subagent で生成" })
todo({ action: "add", text: "ラベルの選択: リポジトリのラベル一覧から適切なものを選択" })
todo({ action: "add", text: "ready for review PR の作成: gh pr create で Draft を付けずに PR を作成 (既存 Draft があれば ready 化)" })
todo({ action: "add", text: "pr-watch への引き継ぎ: pr-watch の SKILL.md を read して PR 番号を対象に監視を開始" })
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
- 未プッシュコミットがあるか (ステップ 3 で対応)
- ベースブランチとの差分があること (差分ゼロの場合は PR 作成不可。エラー報告して終了)

### 2. 未コミット変更の自動コミット

`git status` の結果から unstaged / untracked / staged の変更がある場合のみ実行する。変更がない場合はこのステップをスキップする。

**全工程ユーザー確認なし**で進める。`commit` スキルは内部でユーザー承認を取るため呼び出さず、commit-proposer subagent を直接使用する。

#### 2-1. 全変更のステージング

```bash
# 機密情報 (.env, credentials.json 等) が含まれていないことを git status で確認
git status

# 全変更をステージング
git add -A
```

`.env` / `*.pem` / `credentials*` / `*secret*` などの機密候補ファイルが含まれている場合は、ステージングを取り消して報告し、以降の処理を中止する:

```bash
git reset HEAD <該当ファイル>
```

中止時の報告例:

```
機密情報を含む可能性のあるファイル (.env) が変更に含まれています。
PR 作成を中止しました。コミット対象から除外してから再実行してください。
```

#### 2-2. コミットメッセージ候補の生成

commit-proposer subagent を subagent tool で呼び出す:

```
subagent({
  subagent_type: "commit-proposer",
  description: "コミットメッセージ候補の生成",
  prompt: "ステージング済みの変更に対してコミットメッセージ候補を提案してください。コンテキスト: PR 作成前の自動コミットです。subject には実際に何を変更したかを具体的に記述してください。"
})
```

subagent がエラーを返した場合は、`git diff --cached --stat` と `git log --oneline -5` を参考に Conventional Commits 形式のメッセージを自前で生成する。

#### 2-3. 確認なしでコミット実行

推奨メッセージ (候補 1) でそのままコミットする。ユーザーへの承認確認は行わない:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
)"
```

pre-commit hook が失敗した場合は、`--no-verify` でバイパスせずに失敗内容を確認し、修正可能であれば修正後に再コミットする。修正不可能な場合はユーザーに通知して中止する。

### 3. リモートへのプッシュ

未プッシュコミット (ステップ 2 で追加したコミットを含む) がある場合、または現在のブランチが未追跡の場合にプッシュする:

```bash
# 上流が未設定の場合
git push -u origin "$(git branch --show-current)"

# 上流が設定済みの場合
git push
```

プッシュ失敗時は `git pull --rebase` を試行し、コンフリクトが発生した場合は `git rebase --abort` でリベース状態を元に戻してから、ユーザーに通知して中止する。

### 4. PR テンプレートの確認

```bash
# テンプレートファイルを探す
ls -la .github/PULL_REQUEST_TEMPLATE.md 2>/dev/null || \
ls -la .github/PULL_REQUEST_TEMPLATE/ 2>/dev/null || \
ls -la docs/PULL_REQUEST_TEMPLATE.md 2>/dev/null
```

テンプレートが存在する場合は read tool で内容を確認し、そのフォーマットに準拠した description を生成する。

### 5. PR タイトル・description の生成

`git log origin/<base>..HEAD --oneline` を実行し、コミット数を確認する (ステップ 2 で新規コミットが追加された可能性があるため、必ずここで再取得する)。

**コミットが 1 つの場合:** `git log origin/<base>..HEAD -1 --format='%s'` で subject のみ取得し、そのまま PR タイトルとして使用する。commit-proposer subagent の呼び出しはスキップする。

**コミットが 2 つ以上の場合:** commit-proposer subagent を subagent tool で呼び出す:

```
subagent({
  subagent_type: "commit-proposer",
  description: "PR タイトル候補の生成",
  prompt: "PR のコミット履歴から PR タイトル候補を提案してください。ベースブランチ: <base>。PR タイトルとして Conventional Commits 形式で提案してください。"
})
```

**description の生成:**

- ステップ 4 で取得したテンプレートに準拠する
- テンプレートがない場合は変更内容のサマリ・テスト手順・関連 Issue の項目を含む簡潔な構成で生成する
- 複数コミットの場合は `git log origin/<base>..HEAD --pretty=format:"- %s"` の結果を要約して反映する
- ユーザー確認は行わずに採用する

### 6. ラベルの選択

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

### 7. ready for review PR の作成

**新規 PR 作成 (現在のブランチに紐づく PR が存在しない場合):**

```bash
gh pr create \
  --title "<タイトル>" \
  --body "<説明>" \
  --base <ベースブランチ> \
  --label "<ラベル1>,<ラベル2>" \
  --assignee @me
```

`--draft` は **付けない**。これにより PR は ready for review 状態で open される。

**既存の Draft PR を ready 化する場合 (現在のブランチに紐づく PR が既にある場合):**

`gh pr view --json state,isDraft` で状態を確認し、Draft の場合は ready 化する:

```bash
gh pr ready <pr-number>
```

ready 化に伴ってタイトル・description・ラベルを更新する場合は `gh pr edit` を併用する。

**PR 番号の取得:** 作成・ready 化の完了後、PR 番号と URL を取得する:

```bash
gh pr view --json number,url --jq '{number, url}'
```

取得した PR 番号を `PR_NUMBER` 変数として保持する (次ステップで pr-watch に渡す)。

### 8. pr-watch への引き継ぎ

`pr-watch` スキルの SKILL.md (`../pr-watch/SKILL.md`) を read し、ステップ 7 で取得した PR 番号を対象として、その手順に従って監視を開始する。監視ロジック (バックグラウンド監視、レビュー対応、CI 修正、コンフリクト解消、再リクエスト等) は pr-watch 側に全て委譲する。

引き継ぎ前にユーザーへ一行で報告する:

```
PR #<number> を ready for review で作成しました: <url>
続けて pr-watch スキルで監視を開始します。
```

pr-watch の完了報告 (ステップ 4 相当) がそのまま本スキルの最終報告となる。本スキルの呼び出し側で追加の完了報告を作成する必要はない。

## エラーハンドリング

### gh CLI が使用できない場合

GitHub MCP server が構成されていれば `mcp` tool 経由でフォールバックする (PR 作成は draft: false、ラベル一覧の取得、Draft → Ready 化も同様)。構成されていない場合はユーザーに gh CLI のセットアップを案内する。

### 認証エラー

```bash
gh auth status
gh auth login
```

### ブランチが存在しない / 未プッシュ

```bash
git push -u origin "$(git branch --show-current)"
```

### commit-proposer subagent エラー

`git diff --cached --stat` と `git log --oneline -5` を参考に、Conventional Commits 形式のメッセージを自前生成して使用する。

### pr-watch スキルが利用できない場合

`../pr-watch/SKILL.md` の read に失敗した場合、ユーザーに以下を報告して終了する:

```
PR #<number> を ready for review で作成しましたが、pr-watch スキルの読み込みに失敗しました。
監視が必要な場合は以下を手動で実行してください:

  /skill:pr-watch <number>

PR URL: <url>
```

PR 作成自体は成功しているため、本スキルとしては部分成功扱いとする。
