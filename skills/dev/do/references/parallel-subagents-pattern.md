# Background subagent 並列実行パターン詳細手順

各タスクを background subagent に割り当て、独立したセッションで真に並列実行する。起動・監視・介入・回収・再実行の手順を示す。

## 1. 起動

各タスクに対して `subagent` tool を **単一メッセージ内で並列に呼び出す**。`run_in_background: true` により各呼び出しは即座に `agent_id` を返す。

```
subagent({
  subagent_type: "general-purpose",
  description: "<タスク 1 の要約>",
  run_in_background: true,
  prompt: `以下のタスクを実行してください:

<タスクの詳細な指示>

完了したら結果をマークダウン形式で報告してください。`
})

// ... 残りのタスクも同じメッセージ内で同様に起動
```

- 返却された `agent_id` をタスク名と対にして控える
- 同時実行数の上限 (デフォルト 4) を超えた分は自動でキューイングされ、空きが出ると順に実行される

## 2. 監視と介入

- **進捗確認**: `get_subagent_result({ agent_id })` (wait なし) で走行中のステータスを確認できる
- **方向修正**: 走行中のタスクにスコープの追加・修正指示を出す場合は `steer_subagent({ agent_id, message })` を使う。メッセージは現在の tool 実行の完了後に割り込まれる
- background subagent の完了は通知として届くため、通常はポーリング不要

## 3. 回収

全タスクの起動後、各 `agent_id` に対して完了を待って結果を回収する:

```
get_subagent_result({ agent_id: "<agent_id>", wait: true })
```

- 依存関係のあるタスクは、依存先の回収結果を踏まえて次のウェーブの prompt を構成する
- 詳細な経過が必要な場合のみ `verbose: true` を付ける (通常は最終報告のみで十分)

## 4. 失敗タスクの再実行

- 失敗原因が prompt の不足にある場合: 前回の失敗内容と回避策を含めた改善 prompt で**新しい subagent** を起動する
- 完了済みタスクへの軽微な追加指示・確認: `subagent({ subagent_type: "general-purpose", resume: "<agent_id>", prompt: "<追加指示>" })` で同じセッションをコンテキスト保持のまま再開する
- 同一タスクの再実行は 2 回まで。それでも失敗する場合は失敗として報告に含める

## フォールバック

- background 起動が失敗した場合 → foreground (`run_in_background` なし) の順次実行にフォールバック
- 一部のタスクが失敗した場合 → 残りのタスクの結果で続行
- 全タスクが失敗した場合 → 共通原因を分析してユーザーに報告
