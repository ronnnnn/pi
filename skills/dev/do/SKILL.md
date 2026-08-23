---
name: do
description: 複数の独立したタスクを background subagent で並列実行する。Use when 複数タスクの並列実行、同時作業、swarm 実行を求められた際に使用する。引数はタスクの列挙。
---

# 並列タスク実行ワークフロー

引数で渡された複数タスクを pi-subagents の background subagent で並列実行し、結果を集約して報告する。

## 重要な原則

1. **background subagent で真に並列実行する** - 各タスクを `run_in_background: true` で起動し、`get_subagent_result` で回収する
2. **各タスクは独立して実行** - タスク間の依存関係がある場合は順序を考慮
3. **結果を集約して報告** - 全タスクの完了後にサマリを提示
4. **失敗したタスクは明示的に報告** - 成功・失敗を区別して報告

## 作業開始前の準備

**必須:** 作業開始前に `todo({ action: "list" })` で残存タスクを確認し、存在する場合は `todo({ action: "clear" })` で全て削除する。その後、todo tool で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "タスクの分析と分割 (引数からタスクを抽出・分類)" })
todo({ action: "add", text: "タスクの並列実行 (background subagent で並列実行)" })
todo({ action: "add", text: "結果の集約と報告 (全タスクの結果をまとめて報告)" })
```

各ステップの完了時に `todo({ action: "toggle", id: <id> })` で完了にする。

## 実行手順

### 1. タスクの分析と分割

引数で渡されたテキストからタスクを抽出する:

- 番号付きリスト (1. 2. 3.)、箇条書き (- / \*)、「と」「、」区切り等のパターンを認識
- 各タスクを独立した作業単位に分割
- タスク間の依存関係を検出 (「A の結果を使って B」等)
- **競合回避**: 同一ファイルを変更するタスクは 1 つの subagent にまとめるか、順次実行にする

**分割結果を提示:**

```markdown
## 検出されたタスク

1. **タスク名** - 概要
2. **タスク名** - 概要
3. **タスク名** - 概要

依存関係: なし / タスク 2 はタスク 1 の完了後に実行
```

### 2. タスクの並列実行

各タスクを `subagent` tool で `general-purpose` subagent として **`run_in_background: true` 付きで単一メッセージ内で並列に起動**する。

```
subagent({
  subagent_type: "general-purpose",
  description: "<タスクの要約>",
  run_in_background: true,
  prompt: `以下のタスクを実行してください:

<タスクの詳細な指示>

完了したら結果をマークダウン形式で報告してください。`
})
```

**重要:**

- 全 subagent を単一メッセージ内で起動することで真の並列実行を実現 (同時実行数の上限を超えた分は自動でキューイングされる)
- 各起動の返却値に含まれる `agent_id` を控える (回収・介入の宛先になる)
- 全タスクの起動後、`get_subagent_result({ agent_id, wait: true })` を各タスクに対して呼び、完了を待って結果を回収する
- 走行中のタスクへの方向修正は `steer_subagent({ agent_id, message })` で行う

**詳細な手順 (監視・介入・再実行を含む) は `references/parallel-subagents-pattern.md` を参照。**

### 3. 結果の集約と報告

全タスクの結果を集約してユーザーに報告する。**報告テンプレートは `references/report-format.md` を参照。**

報告に含める情報:

- タスク数と成功・失敗の内訳
- 各タスクの結果要約
- 変更ファイル一覧 (あれば)
- 失敗したタスクのエラー内容と推奨対応

## 依存関係のあるタスクの処理

タスク間に依存関係がある場合:

1. 依存関係のないタスクを先に並列実行
2. 依存先タスクの完了を `get_subagent_result({ agent_id, wait: true })` で待機
3. 依存タスクを実行 (必要なら再度並列)

## エラーハンドリング

### タスクの一部が失敗した場合

成功したタスクの結果は保持し、失敗したタスクのみを報告する。ユーザーに再実行するか確認:

```
question({
  question: "失敗したタスクを再実行しますか？",
  options: [
    { label: "再実行する", description: "失敗したタスクのみを再実行します" },
    { label: "スキップ", description: "失敗したタスクをスキップして完了します" }
  ]
})
```

### 全タスクが失敗した場合

- ブリーフの前提 (環境、権限、タスクの実現性) に共通の問題がある可能性が高い。エラー内容の共通点を分析して報告し、ユーザーに個別実行を提案する

### background 起動が失敗する場合

`run_in_background` なしの foreground 実行に切り替え、タスクを 1 つずつ順次実行する。

## Additional Resources

### Reference Files

- **`references/parallel-subagents-pattern.md`** - background subagent の起動・監視・介入・回収・再実行の詳細手順
- **`references/report-format.md`** - 結果報告のテンプレート
