---
name: review
description: ローカルの変更 (staged/unstaged) をレビューし、指摘箇所を自動修正する。指摘がなくなるまで最大 3 回繰り返す。Use when コミット前のコードレビュー、ローカル変更のセルフレビューを求められた際に使用する。
---

# ローカルレビューワークフロー

ローカルの変更を複数の視点で並列レビューし、指摘箇所を自動修正する。

## 重要な原則

1. **複数の視点で並列レビューする** - reviewer subagent (background 並列) と Codex MCP を同時に使用
2. **結果を統合・重複排除する** - 同じ指摘は 1 つにマージ
3. **修正が必要なものは承認なしで自動修正する**
4. **レビュー・修正を繰り返す** - 修正がなくなるまで
5. **最終結果のサマリを報告する**
6. **自動修正はメインセッションのみが実行する** - reviewer subagent はファイル編集しない (並行編集の競合回避)
7. **MCP の呼び出しはメインセッションが行う** - reviewer subagent は mcp tool を持たないため、Codex MCP へのレビュー依頼はメインセッションが mcp tool で実行する

## 作業開始前の準備

**必須:** 作業開始前に `todo({ action: "list" })` で残存タスクを確認し、存在する場合は `todo({ action: "clear" })` で全て削除する。その後、todo tool で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "ローカル差分の取得 (git diff HEAD で全変更を取得)" })
todo({ action: "add", text: "アプローチ判定 (変更量・内容に基づいて reviewer 構成を選択)" })
todo({ action: "add", text: "並列レビューの実行" })
todo({ action: "add", text: "自動修正の実行 (修正が必要な指摘を自動修正)" })
todo({ action: "add", text: "再レビュー (修正後に再度レビュー、最大 3 回)" })
todo({ action: "add", text: "完了報告 (修正サマリと残課題を報告)" })
```

各ステップの完了時に `todo({ action: "toggle", id: <id> })` で完了にする。

## 実行手順

### 1. ローカル差分の取得

staged と unstaged の両方の変更を取得する:

```bash
# staged 変更
git diff --cached

# unstaged 変更
git diff

# 全変更 (staged + unstaged)
git diff HEAD

# 変更ファイル一覧
git diff HEAD --name-only
```

変更がない場合は終了:

```markdown
レビュー対象の変更がありません。
```

### 2. アプローチ判定

差分の統計情報に基づいて、reviewer 構成を選択する。

#### 2-1. 統計情報の取得

```bash
# 変更ファイル数
git diff HEAD --name-only | wc -l

# 変更行数
git diff HEAD --stat | tail -1
# "N files changed, X insertions(+), Y deletions(-)" から X + Y を算出
```

#### 2-2. アプローチの選択

| 基準             | パターン A (単一 reviewer)   | パターン B (複数 lens reviewer)             |
| ---------------- | ---------------------------- | ------------------------------------------- |
| **変更の複雑度** | 単純な変更、少数ファイル     | 複数モジュール/レイヤーにまたがる複雑な変更 |
| **変更量**       | 小〜中規模                   | 大規模 (多数ファイル、大差分)               |
| **最適なケース** | 結果だけが重要な集中レビュー | 観点ごとの深い分析が必要な複雑変更          |

**目安:**

- ファイル数 15 未満 かつ 変更行数 500 未満 かつ 単一モジュール → パターン A 推奨
- ファイル数 15 以上 または 変更行数 500 以上 → パターン B 推奨
- 複数モジュール/レイヤーにまたがる変更 → パターン B 推奨
- セキュリティ関連の変更 → パターン B 推奨 (複数視点が重要)

### 3. 並列レビューの実行

reviewer subagent を **`run_in_background: true` で起動してから**、メインセッションが Codex MCP レビューを実行する (両者が並行に走る)。その後 `get_subagent_result({ agent_id, wait: true })` で reviewer の結果を回収する。

#### Codex MCP レビュー (メインセッションが実行)

mcp tool で `codex` server の利用可能性を確認し、利用可能なら `codex` tool を以下のパラメーターで呼び出す:

- `prompt: "/review"`
- `profile: "review"`
- `cwd: "<対象ディレクトリの絶対パス>"`

利用不可の場合はスキップし、reviewer subagent の結果のみで統合する。

**Codex 出力の severity マッピング:**

- critical, severe, security → CRITICAL
- bug, error, high → HIGH
- warning, medium → MEDIUM
- info, suggestion, nit → LOW

#### パターン A: 単一 reviewer

```
subagent({
  subagent_type: "general-purpose",
  description: "ローカル変更のレビュー",
  run_in_background: true,
  prompt: `あなたは code-reviewer です。ローカルの変更差分をレビューしてください。

## 手順

### 1. 差分の取得

\`\`\`bash
git diff HEAD
git diff HEAD --name-only
\`\`\`

### 2. レビュー

差分と関連ファイルを分析し、以下の観点でレビューする:
- バグ: 論理エラー、off-by-one、null 参照
- セキュリティ: インジェクション、認証、機密情報
- パフォーマンス: N+1、不要なループ、メモリリーク
- 可読性: 命名、複雑度、コメント
- テスト: カバレッジ、エッジケース

**ファイルの変更は一切行わないこと。**

## 出力形式 (最終メッセージ)

\`\`\`markdown
## Review Results

**Total Issues:** N

### Critical Issues (X)
1. **[CRITICAL]** [file:line] - 説明
   - 問題: ...
   - 推奨: ...

### High Priority Issues (Y)
...
### Medium Priority Issues (Z)
...
### Low Priority Issues (W)
...
\`\`\`

**severity 基準:**
- CRITICAL: セキュリティ脆弱性、データ損失リスク (即時修正必須)
- HIGH: バグ、重大なロジックエラー (修正推奨)
- MEDIUM: パフォーマンス問題、可読性 (検討推奨)
- LOW: スタイル、軽微な改善 (任意)

## 注意事項
- スタイルのみの指摘 (linter で対応すべき)、好みの問題、曖昧な指摘は除外する`
})
```

→ Codex MCP の結果と統合してステップ 4 (自動修正) へ進む

#### パターン B: 複数 lens reviewer

各 reviewer に異なるレンズ (観点) を割り当て、**単一メッセージ内で `run_in_background: true` 付きで並列に起動**する。各 reviewer は独立したセッションで深く分析し、最終メッセージで結果を報告する。

**重要:** reviewer はファイル編集しない。自動修正はメインセッションのみが実行する。

**security-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  description: "セキュリティレビュー",
  run_in_background: true,
  prompt: `あなたは security-reviewer です。ローカルの変更差分をセキュリティ観点でレビューしてください。

## 手順

### 1. 差分の取得
\`\`\`bash
git diff HEAD
\`\`\`

### 2. セキュリティ観点でのレビュー
以下に集中してレビューする:
- インジェクション (SQL, XSS, コマンド等)
- 認証・認可の欠陥
- 機密情報の漏洩 (ハードコードされたシークレット、ログへの出力)
- 入力バリデーションの不足
- 安全でないデシリアライゼーション
- アクセス制御の問題

**ファイルの変更は一切行わないこと。**

## 出力形式 (最終メッセージ)

\`\`\`markdown
## Security Review Results

**Reviewer:** security-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ...
\`\`\``
})
```

**logic-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  description: "ロジックレビュー",
  run_in_background: true,
  prompt: `あなたは logic-reviewer です。ローカルの変更差分をバグ・ロジック観点でレビューしてください。

## 手順

### 1. 差分の取得
\`\`\`bash
git diff HEAD
\`\`\`

### 2. バグ・ロジック観点でのレビュー
以下に集中してレビューする:
- 論理エラー、off-by-one エラー
- null/undefined 参照
- 境界条件の処理漏れ
- 競合状態、デッドロック
- エラーハンドリングの不足
- パフォーマンス問題 (N+1 クエリ、不要なループ、メモリリーク)

**ファイルの変更は一切行わないこと。**

## 出力形式 (最終メッセージ)

\`\`\`markdown
## Logic Review Results

**Reviewer:** logic-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ...
\`\`\``
})
```

**bestpractice-reviewer:**

```
subagent({
  subagent_type: "general-purpose",
  description: "ベストプラクティスレビュー",
  run_in_background: true,
  prompt: `あなたは bestpractice-reviewer です。ローカルの変更差分を使用ツール・FW・ライブラリ・言語のベストプラクティス観点でレビューしてください。

## 手順

### 1. 差分の取得
\`\`\`bash
git diff HEAD
\`\`\`

### 2. ベストプラクティス観点でのレビュー
以下に集中してレビューする:
- 使用言語のイディオムに従っているか
- フレームワーク・ライブラリの推奨パターンに従っているか
- API の正しい使用方法
- 非推奨 API・パターンの使用
- テストのベストプラクティス (カバレッジ、エッジケース)
- 可読性・命名規則

**ファイルの変更は一切行わないこと。**

## 出力形式 (最終メッセージ)

\`\`\`markdown
## Best Practice Review Results

**Reviewer:** bestpractice-reviewer
**Issues Found:** N

1. **[SEVERITY]** [file:line] - 説明
   - 問題: ...
   - 推奨: ...
\`\`\``
})
```

##### 結果の回収

全 reviewer の起動と Codex MCP レビューの完了後、各 `agent_id` に対して `get_subagent_result({ agent_id, wait: true })` で結果を回収する。

##### フォールバック

- 一部の reviewer が失敗した場合 → 残りの reviewer の結果で続行
- 全 reviewer が失敗した場合 → パターン A (単一 reviewer) にフォールバック。それも失敗する場合はメインセッションが直接レビューする

#### 結果の統合

全 reviewer と Codex MCP の結果を統合・重複排除する:

1. ファイルパスと行番号で指摘をグループ化
2. 同じ問題への指摘は最も詳細な説明を採用
3. severity は最も高いものを採用
4. 検出元 (security-reviewer, logic-reviewer, bestpractice-reviewer, Codex 等) を付記

**severity 統一:**

- CRITICAL: セキュリティ脆弱性、データ損失リスク (即時修正必須)
- HIGH: バグ、重大なロジックエラー (修正推奨)
- MEDIUM: パフォーマンス問題、可読性 (検討推奨)
- LOW: スタイル、軽微な改善 (任意)

統合結果に基づき、修正可能性を判断する:

**自動修正対象:**

- 具体的な修正案がある
- ファイル・行番号が明確
- 機械的に修正可能

**自動修正しない指摘:**

- 設計レベルの変更が必要
- 複数ファイルにまたがる修正
- 判断が必要な修正

### 4. 自動修正の実行

修正が必要な指摘に対して、承認なしで自動修正を行う:

1. 対象ファイルを read tool で読み込む
2. edit tool で修正を適用
3. 修正内容をログ

```markdown
### 修正ログ

1. **src/api/users.ts:42** - null チェック追加
   - Before: `return user.id;`
   - After: `return user?.id ?? null;`

2. **src/utils/format.ts:15** - 型アノテーション追加
   ...
```

### 5. 再レビュー (必要な場合)

修正後、再度レビューを実行する。**再レビューは常にパターン A (単一 reviewer) を使用する** (修正後の差分は小さいため複数 lens reviewer は不要)。

```bash
# 修正後の差分を確認
git diff HEAD
```

**繰り返し条件:**

- 新たな指摘がある場合 → ステップ 3 (パターン A) に戻る
- 指摘がない場合 → 完了報告へ

**最大繰り返し回数:** 3 回

3 回繰り返しても指摘がある場合:

```markdown
## 自動修正の限界

以下の指摘は手動での対応が必要です:

1. **[src/core/engine.ts:100-150]**
   - 問題: アーキテクチャレベルの変更が必要
   - 推奨: ...
```

### 6. 完了報告

```markdown
## ローカルレビュー完了

**レビュー方式:** 単一 reviewer / 複数 lens reviewer
**レビュー視点:** reviewer subagent, Codex (利用したもののみ記載)
**レビュー回数:** N 回

### 修正サマリ

| ファイル            | 修正数 | 内容                      |
| ------------------- | ------ | ------------------------- |
| src/api/users.ts    | 2      | null チェック追加、型修正 |
| src/utils/format.ts | 1      | 型アノテーション追加      |

**合計修正数:** X 件

### 残課題 (手動対応が必要)

なし / または以下:

- [file:line] - 説明
```

## エラーハンドリング

### 修正に失敗した場合

```markdown
## 修正失敗

以下のファイルの修正に失敗しました:

- **src/api/users.ts:42**
  - 理由: 該当行が見つかりません (ファイルが変更された可能性)
  - 対応: 手動での修正が必要
```
