# cc からの移植ルール

[ronnnnn/cc](https://github.com/ronnnnn/cc) (Claude Code plugin marketplace) の skill / agent を pi 向けに移植する際のルール。cc 側の更新を再移植するときもこのルールに従う。

## 対象と除外

| cc plugin | pi での扱い |
| :--- | :--- |
| git | `skills/git/` + `agents/` に移植 |
| claude | AGENTS.md 管理に改変して `skills/agents-md/` に移植 (pi は CLAUDE.md ではなく AGENTS.md を読む) |
| catch-up | `skills/catch-up/` + `agents/` に移植 |
| dev | `skills/dev/` + `agents/` に移植 |
| hookify | 移植しない (pi-permission-system + permission-judge extension が担当) |
| caffeine | 移植しない (caffeinate.ts extension が担当) |

## 参照すべき pi ドキュメント

移植時は必ず最新の pi 本体のドキュメントを参照する (mise 管理の pi 本体に同梱。パスの `<version>` は `pi --version` で解決)。

- skills: `~/.local/share/mise/installs/npm-earendil-works-pi-coding-agent/<version>/lib/node_modules/@earendil-works/pi-coding-agent/docs/skills.md`
- prompt templates: 同 `docs/prompt-templates.md`
- packages: 同 `docs/packages.md`
- custom agents: `~/.pi/agent/npm/node_modules/@gotgenes/pi-subagents/docs/configuration.md`

## skill の変換ルール

### frontmatter

- `name` / `description`: 維持する。`description` 内の「Claude Code」への言及は pi に置き換える
- `allowed-tools`: **削除する**。pi では experimental な事前承認リストで意味が異なり、承認は pi-permission-system が担う
- `disable-model-invocation`: 維持する (pi も同じフィールドをサポート)
- skill 名は pi ではフラットな名前空間 (`/skill:<name>`) になるため、`init` / `update` など汎用的すぎる名前は `agents-md-init` のように接頭辞を付ける

### tool 参照の置き換え

| Claude Code | pi |
| :--- | :--- |
| `Bash` | `bash` (自作 background-bash extension により `run_in_background` / `notify_on` パラメーター、`bash_output` / `kill_shell` tool が使える) |
| `Read` / `Write` / `Edit` | `read` / `write` / `edit` |
| `Glob` / `Grep` / `LS` | `find` / `grep` / `ls` |
| `Task({ subagent_type: "plugin:name" })` | `subagent({ subagent_type: "name" })` (pi-subagents。namespace 接頭辞なし。background は `run_in_background: true` + `get_subagent_result` / `steer_subagent`) |
| `TaskCreate` / `TaskUpdate` / `TaskList` | `todo` tool (actions: `list` / `add` / `toggle` / `clear`。vendored todo.ts) |
| `AskUserQuestion` | `question` (単一質問) / `questionnaire` (複数質問) tool (vendored extension) |
| `WebFetch` | `bash` + `ax` (curl 互換 CLI。`ax <url> --md` 等) |
| `WebSearch` | 直接の代替なし。deepwiki / context7 MCP (`mcp` tool 経由) と `ax` によるリリースページ等の直接取得に書き換える |
| `SlashCommand` / `/plugin:skill` 参照 | skill ディレクトリからの相対パスで対象 SKILL.md を `read` する指示に書き換える (ユーザー向け表記は `/skill:<name>`) |
| `${CLAUDE_PLUGIN_ROOT}` | skill ディレクトリからの相対パス |
| モニタリング用の sleep ポーリング | `bash` の `run_in_background: true` + `notify_on` (regex) による push 通知に書き換える |

### 内容の置き換え

- `CLAUDE.md` → `AGENTS.md` (プロジェクトに CLAUDE.md が実コピーとして併存する場合は同期に言及する)
- `.claude/rules` はそのまま維持 (pi 側は claude-rules.ts extension が読む)
- Claude Code 固有の機能 (hooks、plugin 機構、`/reload-plugins` 等) への言及は削除または pi 相当に書き換える

## agent の変換ルール

pi-subagents の custom agent 形式 (`agents/<name>.md`) に変換する。**ファイル名が agent type 名になる**。

### frontmatter

| Claude Code | pi-subagents |
| :--- | :--- |
| `name` | 削除 (ファイル名で決まる) |
| `description` | 維持 (`<example>` ブロックも説明文としてそのまま有効) |
| `tools` | pi の小文字 tool 名に変換 (`read, bash, grep, find, ls` 等)。**完全な allowlist** なので、必要な extension tool (`question` 等) も明示する。`WebFetch` / `WebSearch` を使っていた agent は `bash` を与えて ax / gh に書き換える |
| `model` | fuzzy 名 (`sonnet` / `haiku` 等) を維持。`inherit` は省略 (デフォルトが inherit) |
| `context: fork` | `inherit_context: true` |
| `memory` | 削除 (pi-subagents に相当機能なし) |
| — | 必要に応じて `thinking` / `max_turns` / `prompt_mode` を追加できる |

### 本文

- tool 参照の置き換えは skill と同じ表を適用する
- agent は `subagent` / `get_subagent_result` / `steer_subagent` を持てない (再帰ガード)。agent 本文から他 agent の呼び出しを削除する

## 配布

- skills は pi package として `package.json` の `pi.skills` で宣言し、dotfiles の flake input + `home.file` で `~/.pi/agent/skills/` に配る
- agents は pi package の resource type にないため、dotfiles の `home.file` で `~/.pi/agent/agents/` に配る
