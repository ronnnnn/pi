---
name: pr-ci
description: PR の CI/CD パイプライン失敗を調査・修正する。ci-analyzer subagent で失敗原因を分析し、コード修正とコミットまで行う。Use when CI が失敗している、CI を修正したい、PR のチェックが通らない際に使用する。
---

# PR CI 失敗の調査・修正ワークフロー

PR の CI が失敗した際に、原因を調査し修正を行う。引数で PR 番号が指定された場合はそれを対象とする。

## 重要な原則

1. **ci-analyzer subagent で失敗原因を調査する** - 直接ログを読むのではなく、subagent に委譲する
2. **修正前に分析結果をユーザーに提示する** - 修正方針の承認を得る
3. **コミット前に必ずユーザーの承認を取る** - 自動でコミットしない。承認確認には question tool を使用する
4. **コミットメッセージは commit-proposer subagent で Conventional Commits / commitlint 設定に準拠して生成する**
5. **コミットメッセージ・返信コメントの言語は対象リポジトリに従う** - 既存の PR やコミット履歴を確認し、リポジトリで使用されている言語に合わせる
6. **日本語でコミットメッセージを書く場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して適用する
7. **修正は CI を通すために必要な最小限に留める**

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "PR の特定: 引数または現在のブランチから PR を特定" })
todo({ action: "add", text: "CI 失敗の調査: ci-analyzer subagent で失敗原因を分析" })
todo({ action: "add", text: "分析結果の提示と修正方針の承認: ユーザーに分析結果を提示し修正方針の承認を得る" })
todo({ action: "add", text: "コード修正の実行: 承認された修正方針に基づいてコードを修正" })
todo({ action: "add", text: "修正の検証: 修正後にローカルで検証可能なコマンドを実行" })
todo({ action: "add", text: "変更のステージング: 修正したファイルを git add でステージング" })
todo({ action: "add", text: "コミットメッセージの生成: commit-proposer subagent でメッセージ候補を生成" })
todo({ action: "add", text: "コミット前の承認確認: ユーザーにコミットの承認を求める" })
todo({ action: "add", text: "コミットの実行: 承認されたメッセージでコミット" })
todo({ action: "add", text: "プッシュの実行: git push でリモートに反映" })
todo({ action: "add", text: "完了報告: 修正結果を報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. PR の特定

引数で PR 番号が指定されていない場合、現在のブランチから PR を特定する:

```bash
gh pr view --json number,title,headRefName --jq '{number, title, headRefName}'
```

### 2. CI 失敗の調査

**ci-analyzer subagent を subagent tool で呼び出す。**

```
subagent({
  subagent_type: "ci-analyzer",
  description: "CI 失敗原因の調査",
  prompt: "PR #<number> (ブランチ: <branch>) の CI 失敗を調査してください。失敗した job のログを取得し、エラーの根本原因を分析して、修正方針を提案してください。"
})
```

subagent から返却された分析レポートを次のステップで使用する。

### 3. 分析結果の提示と修正方針の承認

ci-analyzer の分析レポートをユーザーに提示し、question tool で承認を求める:

```
## CI 失敗分析結果

[ci-analyzer のレポートを要約]

### 修正計画

1. [修正内容 1] - [対象ファイル]
2. [修正内容 2] - [対象ファイル]
...

### 自動修正不可能な項目 (該当する場合)

- [手動対応が必要な内容とその理由]

---

この計画で修正を進めますか？
```

**自動修正の可否判断:**

| エラー種別             | 自動修正 | 対応                             |
| ---------------------- | -------- | -------------------------------- |
| Lint/フォーマット      | 可能     | プロジェクトのフォーマッタを実行 |
| 型エラー・ビルドエラー | 可能     | コードを修正                     |
| テスト失敗             | 要確認   | テストコードまたは実装を修正     |
| 依存関係               | 可能     | パッケージ更新・追加             |
| 環境変数・secret       | 不可能   | ユーザーに設定方法を案内         |
| 権限・認証             | 不可能   | ユーザーに対応方法を案内         |

### 4. コード修正の実行

承認後、修正を実行する:

1. 対象ファイルを read tool で読み込む
2. edit tool で修正を適用
3. フォーマッタやリンタがある場合は実行する:
   ```bash
   # プロジェクトの設定に応じて適切なコマンドを使用
   # 例: bun fmt, npm run lint --fix, etc.
   ```

### 5. 修正の検証

ローカルで検証可能な場合、CI と同等のチェックを実行する:

```bash
# プロジェクトの設定に応じて適切なコマンドを使用
# package.json の scripts や Makefile を確認
# 例: bun fmt, bun run lint, bun run build, bun run test
```

検証が失敗した場合は、エラー内容を確認して追加修正を行う。

### 6. 変更のステージング

修正したファイルをステージングする:

```bash
git add <修正したファイルのパス>
```

### 7. コミットメッセージの生成

**commit-proposer subagent を subagent tool で呼び出す。**

```
subagent({
  subagent_type: "commit-proposer",
  description: "コミットメッセージ候補の生成",
  prompt: "ステージング済みの変更に対してコミットメッセージ候補を提案してください。コンテキスト: CI 失敗の修正です。"
})
```

subagent が変更差分の分析、commitlint 設定の確認、メッセージ候補の生成を実行する。

### 8. コミット前の承認確認

**必須:** 修正内容とコミットメッセージをユーザーに提示し、question tool で承認を求める:

```
## コミット内容の確認

以下の変更をコミットします:

**変更ファイル:**
- path/to/file1.ts (+5, -3)
- path/to/file2.ts (+2, -1)

**コミットメッセージ:**

fix(<scope>): CI 失敗を修正

- [修正内容 1]
- [修正内容 2]

---

この内容でコミットしてよろしいですか？
```

**type の選択基準:**

- Lint/フォーマット修正 → `style`
- ビルド・設定修正 → `fix` または `build`
- テスト修正 → `fix` または `test`
- 依存関係修正 → `deps` または `fix`

### 9. コミットの実行

承認後、コミットする:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
)"
```

### 10. プッシュの実行

コミット完了後、リモートにプッシュする:

```bash
git push
```

### 11. 完了報告

```
## CI 修正完了

- 修正コミット: <commit_hash>
- 修正ファイル数: N
- 修正内容: [要約]

PR URL: <url>

CI の再実行結果を確認してください。
```

## エラーハンドリング

### gh CLI が使用できない場合

`gh api` コマンドで GitHub API に直接アクセスする:

```bash
HEAD_SHA=$(gh api repos/{owner}/{repo}/pulls/<number> --jq '.head.sha')
gh api repos/{owner}/{repo}/actions/runs?head_sha=$HEAD_SHA&status=failure&per_page=100
```

### PR が見つからない場合

現在のブランチに PR がない可能性がある。ユーザーに確認する:

```
現在のブランチ (<branch>) に紐づく PR が見つかりません。
PR 番号を指定してください。
```

### CI が実行中の場合

```
CI がまだ実行中です。完了後に再度実行してください。

実行中の job:
- <job 名> (ステータス: IN_PROGRESS)
```

### 全ての CI が成功している場合

```
全ての CI チェックが成功しています。修正は不要です。

チェック結果:
- <check 名>: SUCCESS
- ...
```
