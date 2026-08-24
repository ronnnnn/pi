# pi

[pi](https://pi.dev) (coding agent) 向けの skill / agent 集です。[ronnnnn/cc](https://github.com/ronnnnn/cc) (Claude Code plugin marketplace) をベースに、pi のツール群 (pi-subagents、background bash、question / questionnaire / todo extension 等) へ移植しています。

## 構成

| ディレクトリ | 内容 |
| :--- | :--- |
| [skills/git](./skills/git) | Git/GitHub ワークフロー (コミット、PR 作成・レビュー・修正・監視、CI 分析) |
| [skills/agents-md](./skills/agents-md) | AGENTS.md の作成・更新・品質管理、`.claude/rules` の作成 |
| [skills/catch-up](./skills/catch-up) | 技術・ツール・フレームワークの最新バージョン取得と技術調査 |
| [skills/dev](./skills/dev) | 開発支援 (コードコメント追加、行動計画作成、並列タスク実行、レビュー) |
| [agents/](./agents) | [pi-subagents](https://github.com/gotgenes/pi-packages/tree/main/packages/pi-subagents) 用の custom agent 定義 |

cc の hookify / caffeine は移植していません (pi 側では pi-permission-system と caffeinate extension が同等の役割を担うため)。

## インストール

### skills (pi package として)

このリポジトリは pi package です。`package.json` の `pi.skills` で skills を宣言しています。

```shell
pi install git:github.com/ronnnnn/pi
```

### agents

agent 定義は pi package の resource type に含まれないため、`agents/*.md` を手動または dotfiles 経由で `~/.pi/agent/agents/` (グローバル) か `.pi/agents/` (プロジェクト) に配置してください。

### Nix (dotfiles) 経由

[ronnnnn/dotfiles](https://github.com/ronnnnn/dotfiles) では、このリポジトリを flake input として取り込み、`home.file` で `~/.pi/agent/skills/` と `~/.pi/agent/agents/` に配布しています (リビジョンは flake.lock で固定)。

## 使い方

skill は `/skill:<name>` コマンドで明示的に起動するか、タスク内容に応じて agent が自動で読み込みます。

```shell
/skill:commit          # 変更をコミット
/skill:pr-create       # Draft PR を作成
/skill:pr-review       # PR をレビュー
/skill:agents-md-init  # AGENTS.md を新規作成
/skill:plan            # 行動計画を作成
/skill:do              # 複数タスクを並列実行
/skill:tech-research   # 技術調査を実行
```

agent は pi-subagents の `subagent` tool から呼び出されます (例: `subagent({ subagent_type: "commit-proposer", ... })`)。

## 前提

skill / agent の一部は以下の pi package / extension を前提としています。

- [@gotgenes/pi-subagents](https://www.npmjs.com/package/@gotgenes/pi-subagents) — `subagent` / `get_subagent_result` / `steer_subagent` tool
- background bash extension — `bash` の `run_in_background` / `notify_on` パラメーターと `bash_output` / `kill_shell` tool
- todo / question / questionnaire extension — タスク管理とユーザーへの選択式質問
- [ax](https://github.com/yusukebe/ax) — Web ページ・API 取得 (curl 代替)

## 開発

cc からの移植・再移植のルールは [docs/porting-from-cc.md](./docs/porting-from-cc.md) を参照してください。

### コミット規約

[Conventional Commits](https://www.conventionalcommits.org/) 形式に従います。

## ライセンス

[MIT](./LICENSE)
