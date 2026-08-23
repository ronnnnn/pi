---
name: pr-review
description: PR をレビューし、指摘箇所に GitHub コメントを投稿する。メインセッション・観点別 subagent・MCP (codex 等) を併用して並列レビューし結果を統合する。Use when PR のレビュー、コードレビュー、PR にフィードバックを投稿したい際に使用する。
---

# PR レビューワークフロー

PR の変更を複数の視点で並列レビューし、指摘箇所に PR コメントを投稿する。引数として PR の URL または番号を受け取れる。

## 重要な原則

1. **複数の視点で並列レビューする** - メインセッションのレビューに加え、観点別 subagent や MCP 経由の外部 AI (codex 等、構成されている場合) を併用する
2. **結果を統合・重複排除する** - 同じ指摘は 1 つにマージ
3. **有用な指摘のみコメントする** - メインセッションが最終判断
4. **インラインコメントを優先する** - ファイル・行番号が明確な場合
5. **コメント投稿前に必ずユーザー承認を取る** - question tool を使用する
6. **コメントの言語は対象リポジトリに従う** - 既存の PR やコメント履歴を確認
7. **日本語でコメントを書く場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して適用する

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "PR の特定: 引数または現在のブランチから PR を特定" })
todo({ action: "add", text: "PR 差分の取得: gh pr diff で差分を取得" })
todo({ action: "add", text: "アプローチ判定: 変更量・内容に基づいて単独レビュー / 並列 subagent レビューを選択" })
todo({ action: "add", text: "並列レビューの実行: 選択したアプローチで並列レビュー" })
todo({ action: "add", text: "コメント案の作成: インラインコメントと一般コメントを作成" })
todo({ action: "add", text: "ユーザー承認の取得: コメント案の承認を求める" })
todo({ action: "add", text: "コメントの投稿: GitHub API でコメントを投稿" })
todo({ action: "add", text: "完了報告: レビュー結果を報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. PR の特定

引数で PR 番号/URL が指定されていない場合、現在のブランチから PR を特定する:

```bash
gh pr view --json number --jq '.number'
```

引数が URL の場合は、URL から owner / repo / PR 番号を抽出し、**以降の全ての gh コマンド (`gh pr view` / `gh pr diff` / `gh api` 等) に `-R <owner>/<repo>` を付与する**。`-R` なしではカレントリポジトリの同一番号の無関係な PR をレビュー・コメントしてしまう可能性がある:

```bash
# URL 形式: https://github.com/<owner>/<repo>/pull/<number>
gh pr view <number> -R <owner>/<repo> --json number,title,url,baseRefName,headRefName,headRefOid
```

```bash
# PR 情報を取得 (headRefOid はコメント投稿時に必要)
gh pr view <number> --json number,title,url,baseRefName,headRefName,headRefOid
```

### 2. PR 差分の取得

```bash
# PR の差分を取得
gh pr diff <number>

# 変更ファイル一覧
gh pr diff <number> --name-only
```

### 3. アプローチ判定

差分の統計情報に基づいて、レビューアプローチを選択する。

#### 3-1. 統計情報の取得

```bash
# 変更ファイル数
gh pr diff <number> --name-only | wc -l

# 変更行数 (追加 + 削除)
gh pr view <number> --json additions,deletions --jq '.additions + .deletions'
```

#### 3-2. アプローチの選択

以下の基準で**メインセッションが総合判断**する:

| 基準             | パターン A (単独レビュー)  | パターン B (並列 subagent)                  |
| ---------------- | -------------------------- | ------------------------------------------- |
| **変更の複雑度** | 単純な変更、少数ファイル   | 複数モジュール/レイヤーにまたがる複雑な変更 |
| **変更量**       | 小〜中規模                 | 大規模 (多数ファイル、大差分)               |
| **最適なケース** | 結果だけが重要な集中タスク | 複数視点の深い分析が必要な複雑タスク        |

**目安:**

- ファイル数 15 未満 かつ 変更行数 500 未満 かつ 単一モジュール → パターン A 推奨
- ファイル数 15 以上 または 変更行数 500 以上 → パターン B 推奨
- 複数モジュール/レイヤーにまたがる変更 → パターン B 推奨
- セキュリティ関連の変更 → パターン B 推奨 (複数視点が重要)

### 4. 並列レビューの実行

どちらのパターンでも、MCP 経由の外部 AI レビュー (4-3) を並行して実行する。

#### パターン A: 単独レビュー

メインセッションが差分を直接分析し、以下の観点でレビューする:

- バグ: 論理エラー、off-by-one、null 参照
- セキュリティ: インジェクション、認証、機密情報
- パフォーマンス: N+1、不要なループ、メモリリーク
- 可読性: 命名、複雑度、コメント
- テスト: カバレッジ、エッジケース

→ 4-3 (MCP レビュー) の結果と合わせて 4-4 (結果の統合) へ進む。

#### パターン B: 観点別 subagent の並列レビュー

各 reviewer に異なるレンズ (観点) を割り当て、subagent tool の `run_in_background: true` で並列に起動する。以下の 3 つを**単一メッセージ内で並列に起動**する。

**security-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  run_in_background: true,
  description: "セキュリティレビュー",
  prompt: "あなたは security-reviewer です。PR #<number> をセキュリティ観点でレビューしてください。

## 手順

### 1. 差分の取得
gh pr diff <number>

### 2. セキュリティ観点でのレビュー
以下に集中してレビューする:
- インジェクション (SQL, XSS, コマンド等)
- 認証・認可の欠陥
- 機密情報の漏洩 (ハードコードされたシークレット、ログへの出力)
- 入力バリデーションの不足
- 安全でないデシリアライゼーション
- アクセス制御の問題

### 3. 結果の返却
最終結果を以下の形式で返す:

## Security Review Results

**Reviewer:** security-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ..."
})
```

**logic-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  run_in_background: true,
  description: "ロジックレビュー",
  prompt: "あなたは logic-reviewer です。PR #<number> をバグ・ロジック観点でレビューしてください。

## 手順

### 1. 差分の取得
gh pr diff <number>

### 2. バグ・ロジック観点でのレビュー
以下に集中してレビューする:
- 論理エラー、off-by-one エラー
- null/undefined 参照
- 境界条件の処理漏れ
- 競合状態、デッドロック
- エラーハンドリングの不足
- パフォーマンス問題 (N+1 クエリ、不要なループ、メモリリーク)

### 3. 結果の返却
最終結果を以下の形式で返す:

## Logic Review Results

**Reviewer:** logic-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ..."
})
```

**bestpractice-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  run_in_background: true,
  description: "ベストプラクティスレビュー",
  prompt: "あなたは bestpractice-reviewer です。PR #<number> を使用ツール・FW・ライブラリ・言語のベストプラクティス観点でレビューしてください。

## 手順

### 1. 差分の取得
gh pr diff <number>

### 2. ベストプラクティス観点でのレビュー
以下に集中してレビューする:
- 使用言語のイディオムに従っているか
- フレームワーク・ライブラリの推奨パターンに従っているか
- API の正しい使用方法
- 非推奨 API・パターンの使用
- テストのベストプラクティス (カバレッジ、エッジケース)
- 可読性・命名規則

### 3. 結果の返却
最終結果を以下の形式で返す:

## Best Practice Review Results

**Reviewer:** bestpractice-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ..."
})
```

**結果の収集:**

subagent の起動後、4-3 (MCP レビュー) をメインセッションで実行し、その後 `get_subagent_result` で各 reviewer の結果を回収する。長時間応答がない reviewer は `steer_subagent` で切り上げを指示し、結果なしで続行する (結果統合時に「結果なし」と記録する)。

**フォールバック:**

- 一部の reviewer が失敗した場合 → 残りの reviewer の結果で続行
- 全 reviewer が失敗した場合 → パターン A にフォールバック

#### 4-3. MCP レビュー (構成されている場合のみ)

`mcp` tool (pi-mcp-adapter) で利用可能な MCP server を確認し、以下が構成されていれば並列にレビューを依頼する。構成されていなければスキップする:

- **codex**: 「PR <PR の URL> の変更をレビューし、バグ・セキュリティ・パフォーマンス・可読性・テストの観点で file:line 付きの指摘を返してください」というプロンプトでレビューを依頼する

MCP が全て利用不可の場合は、パターン A / B のレビュー結果のみで続行する。

**MCP 出力の severity マッピング:**

- critical, severe, security → CRITICAL
- bug, error, high → HIGH
- warning, medium → MEDIUM
- info, suggestion, nit → LOW

#### 4-4. 結果の統合

全レビュー結果を統合・重複排除する:

1. ファイルパスと行番号で指摘をグループ化
2. 同じ問題への指摘は最も詳細な説明を採用
3. severity は最も高いものを採用
4. 検出元 (メインセッション, security-reviewer, logic-reviewer, bestpractice-reviewer, codex) を付記

**severity 統一:**

- CRITICAL: セキュリティ脆弱性、データ損失リスク (即時修正必須)
- HIGH: バグ、重大なロジックエラー (修正推奨)
- MEDIUM: パフォーマンス問題、可読性 (検討推奨)
- LOW: スタイル、軽微な改善 (任意)

**統合結果の形式:**

```markdown
## Aggregated Review Results

**Reviewed by:** <参加したレビュアーのみ記載>
**Total Issues:** N

### Critical Issues (X)

1. **[CRITICAL]** [file:line] - 説明
   - 問題: ...
   - 推奨: ...
   - 検出元: ...

### High Priority Issues (Y)

...

### Medium Priority Issues (Z)

...

### Low Priority Issues (W)

...
```

### 5. 指摘のフィルタリング

統合結果から、有用な指摘のみ採用する:

- バグや論理エラー
- セキュリティ脆弱性
- 明らかなパフォーマンス問題
- 重要な設計上の問題

**除外する指摘:**

- スタイルのみの指摘 (linter で対応すべき)
- 好みの問題
- 曖昧な指摘

### 6. コメント案の作成

統合結果から PR コメント案を作成する:

**インラインコメント** (ファイル・行番号が明確な場合):

```markdown
### コメント 1

- **ファイル:** src/api/users.ts
- **行:** 42
- **内容:** `user.id` が null の場合の処理が欠けています。null チェックを追加することを推奨します。
```

**一般コメント** (特定の行に紐付かない場合):

```markdown
### 一般コメント

- **内容:** エラーハンドリングが全体的に不足しています。try-catch ブロックの追加を検討してください。
```

### 7. ユーザー承認の取得

**必須:** コメント案をユーザーに提示し、question tool で投稿の承認を求める:

```markdown
## PR レビュー結果

**PR:** #<number> - <title>
**レビュアー:** <参加したレビュアーのみ記載>

### 投稿予定のコメント (N 件)

#### インラインコメント (X 件)

1. **[src/api/users.ts:42]**

   > `user.id` が null の場合の処理が欠けています。

2. ...

#### 一般コメント (Y 件)

1. エラーハンドリングが全体的に不足しています。

---

これらのコメントを PR に投稿してよろしいですか？

- 特定のコメントを除外する場合は番号を指定してください
```

### 8. コメントの投稿

承認後、GitHub API でコメントを投稿する:

**インラインコメント:**

```bash
# レビューコメントを作成
# commit_id にはステップ 1 で取得した headRefOid を使用
gh api repos/{owner}/{repo}/pulls/<number>/comments \
  -f body="コメント内容" \
  -f commit_id="<headRefOid>" \
  -f path="src/api/users.ts" \
  -F line=42 \
  -f side="RIGHT"
```

**一般コメント:**

```bash
# PR コメントを作成
gh pr comment <number> --body "コメント内容"
```

### 9. 完了報告

```markdown
## PR レビュー完了

- **PR:** #<number> - <title>
- **レビュー方式:** 単独レビュー / 並列 subagent
- **レビュアー:** <参加したレビュアー>
- **投稿コメント数:** N 件
  - インラインコメント: X 件
  - 一般コメント: Y 件

PR URL: <url>
```

## エラーハンドリング

### gh CLI が使用できない場合

`gh api` コマンドで GitHub API に直接アクセスする:

```bash
gh api repos/{owner}/{repo}/pulls/<number>
```
