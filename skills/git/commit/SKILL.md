---
name: commit
description: 全変更をステージングし、Conventional Commits 形式でコミットする。commitlint 設定があれば準拠し、コミット前にユーザー承認を取る。Use when コミット、変更の保存、git commit を求められた際に使用する。
---

# コミットワークフロー

全変更 (staged + unstaged) をステージングし、Conventional Commits 形式でコミットする。

## 重要な原則

1. **コミットメッセージの言語は対象リポジトリに従う** - 既存のコミット履歴 (`git log`) を確認し、リポジトリで使用されている言語 (日本語/英語等) に合わせる
2. **日本語でコミットメッセージを書く場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用する
3. **全変更を一括でコミット** - unstaged も untracked も全てステージング
4. **コミットメッセージは commitlint 設定 / Conventional Commits に準拠する**
5. **pre-commit hook がある場合は、それに従う**
6. **機密情報 (.env, credentials 等) がステージングされていないか確認**

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "変更状態の確認: git status で現在の変更状態を確認" })
todo({ action: "add", text: "全変更のステージング: git add -A で全変更をステージング" })
todo({ action: "add", text: "コミットメッセージ候補の生成: commit-proposer subagent で差分分析・commitlint 確認・メッセージ候補生成" })
todo({ action: "add", text: "コミット前の承認確認: ユーザーにコミットメッセージの承認を求める" })
todo({ action: "add", text: "コミットの実行: 承認されたメッセージでコミットを実行" })
todo({ action: "add", text: "プッシュの判定: PR の有無を確認し、PR があれば自動プッシュ、なければスキップ" })
todo({ action: "add", text: "完了報告: コミット結果を報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. 変更状態の確認

```bash
# 現在の変更状態を確認
git status

# 変更がない場合は終了
```

**変更がない場合:**

```
コミットする変更がありません。
```

### 2. 全変更のステージング

```bash
# 全変更をステージング (unstaged + untracked)
git add -A

# ステージング結果を確認
git status
```

### 3. コミットメッセージ候補の生成

**commit-proposer subagent を subagent tool で呼び出す。**

```
subagent({
  subagent_type: "commit-proposer",
  description: "コミットメッセージ候補の生成",
  prompt: "ステージング済みの変更に対してコミットメッセージ候補を最大 3 つ提案してください。"
})
```

subagent が以下を自動で実行する:

- `git diff --cached` で変更差分を分析
- commitlint 設定ファイルを検索・解析
- `git log` で既存コミットの言語・スタイルを確認
- Conventional Commits 形式のメッセージ候補を最大 3 つ生成

subagent から返却された提案を次のステップで使用する。

### 4. コミット前の承認確認

**必須:** commit-proposer の提案をユーザーに提示し、承認を求める。

まず以下のフォーマットで候補を提示する:

````
## コミットメッセージの確認

以下の変更をコミットします:

**変更ファイル:**
- path/to/file1.ts (+10, -5)
- path/to/file2.ts (+3, -1)

---

**コミットメッセージ候補** (推奨度順):

### 1. (推奨)
```

<type>(<scope>): <subject>

<body>
```

### 2.

```
<type>(<scope>): <subject>

<body>
```

### 3.

```
<type>(<scope>): <subject>

<body>
```
````

提示後、question tool で選択を求める:

```
question({
  question: "どのメッセージでコミットしますか？",
  options: [
    { label: "1 (推奨)", description: "<候補 1 の subject>" },
    { label: "2", description: "<候補 2 の subject>" },
    { label: "3", description: "<候補 3 の subject>" }
  ]
})
```

**候補生成の考え方:**

- **候補 1 (推奨):** 変更内容を最も的確に表現するメッセージ
- **候補 2:** 別の観点 (異なる type や scope) からのメッセージ
- **候補 3:** より簡潔、または より詳細なメッセージ

ユーザーが選択肢以外の修正案を入力した場合は、その内容でコミットを実行する。

### 5. コミットの実行

承認されたメッセージでコミットを実行:

```bash
git commit -m "$(cat <<'EOF'
<type>(<scope>): <subject>

<body>
EOF
)"
```

**例:**

```bash
git commit -m "$(cat <<'EOF'
feat(auth): ログイン機能を追加

- メール/パスワード認証を実装
- セッション管理を追加
EOF
)"
```

### 6. プッシュの判定

コミット完了後、現在のブランチに紐づく PR の有無でプッシュを自動判定する。

```bash
# 現在のブランチに紐づく PR を確認
gh pr view --json number 2>/dev/null
```

**PR がある場合:** 確認なしで自動プッシュする。

```bash
# upstream が既に設定されているか確認
git rev-parse --abbrev-ref --symbolic-full-name @{u} 2>/dev/null || echo "no-upstream"

# upstream が既に設定されている場合
git push

# upstream が設定されていない場合は -u を付与して設定
git push -u origin HEAD
```

**PR がない場合:** プッシュをスキップし、次のステップへ進む。

### 7. 完了報告

```bash
# コミット結果を確認
git log -1 --oneline
```

**報告フォーマット:**

```
## コミット完了

- **コミット:** <hash>
- **メッセージ:** <type>(<scope>): <subject>
- **変更ファイル数:** N
- **追加行数:** +X
- **削除行数:** -Y
- **プッシュ:** 済 / スキップ
```

## エラーハンドリング

### 変更がない場合

```
コミットする変更がありません。
```

### pre-commit hook が失敗した場合

1. hook のエラーメッセージを確認
2. 問題を修正
3. 再度 `git add -A` でステージング
4. コミットを再実行

### コンフリクトがある場合

```bash
# コンフリクトを解決後
git add -A
git commit
```
