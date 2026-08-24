---
name: tech-research
description: 技術・ツール・フレームワーク・ライブラリの使い方や最新情報を、優先順位に基づく複数ソースから構造的に調査する。Use when 技術調査、ドキュメント確認、API 仕様の調査、ライブラリの使い方を知りたい際に使用する。
---

# Tech Research Skill

技術やツール、フレームワークの使い方や最新情報を、信頼できるソースから優先順位に基づいて調査するためのガイダンス。

## 概要

技術調査は subagent tool を起動して行う。調査対象に応じて適切なソースを優先順位に基づいて選択し、正確な情報を取得する。

## 調査ソースの優先順位

以下の優先順位でソースを使い分ける:

| 優先度 | ソース           | 用途                                                | ツール                                                                        |
| ------ | ---------------- | ---------------------------------------------------- | ------------------------------------------------------------------------------ |
| 1      | コードベース探索 | コードベース内の定義・参照・実装の調査               | read / grep / find / ls (規模が大きい場合は Explore subagent)                 |
| 2      | deepwiki MCP     | OSS リポジトリの Wiki・ドキュメント                  | `mcp` tool 経由 (read_wiki_structure / read_wiki_contents / ask_question)     |
| 3      | context7 MCP     | ライブラリの公式ドキュメントとコード例               | `mcp` tool 経由 (resolve-library-id / query-docs)                              |
| 4      | ax CLI           | 公式サイト・GitHub・特定 URL のコンテンツ取得        | bash + `ax <url> --md --budget <tokens>` / `ax <url> --outline`               |
| 5      | gh CLI           | GitHub のリリース・Issues・API 情報の取得            | bash + `gh api` / `gh release view`                                            |

**例外 (上記の優先順位より優先):**

- terraform に関する内容は terraform MCP (`mcp` tool 経由) が最優先
- Google Cloud に関する内容は google-developer-knowledge MCP (`mcp` tool 経由) が最優先
- pi (coding agent) に関する内容は pi 同梱ドキュメント (mise 管理の pi 本体の `docs/`・README) を read するのが最優先

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存 todo を確認し、存在する場合は全て `clear` で削除する。その後、`add` で以下のステップを todo 登録する:

- 調査対象の分類: リクエストをカテゴリに分類し、使用するソースを決定
- subagent の起動: subagent tool で調査用 subagent を起動
- ソースからの情報取得: 優先順位に基づいてソースから情報を取得
- 結果の構造化: 調査結果を構造化してまとめる

各ステップの完了時に `toggle` で完了状態へ更新する。

## 調査手順

### 1. 調査対象の分類

調査リクエストを以下のカテゴリに分類する:

- **コードベース内の調査**: 関数の定義、参照元、呼び出し関係 → read / grep / find を使用
- **OSS ライブラリの仕組み**: 内部実装、アーキテクチャ → deepwiki MCP を使用
- **ライブラリの使い方**: API、設定方法、コード例 → context7 MCP を使用
- **特定ページの情報**: 公式ドキュメント、Changelog → ax CLI を使用
- **最新リリース情報**: GitHub Releases、バージョン確認 → gh CLI (または latest-version agent) を使用

### 2. subagent の起動

subagent tool で調査用の subagent を起動する。subagent には以下を指定する:

- `subagent_type`: 調査の性質に応じて選択
  - コードベース探索: `Explore`
  - 汎用調査: `general-purpose`
  - 最新バージョン確認: `latest-version`
- `prompt`: 調査内容を具体的に記述
- `model`: 軽量な調査は `haiku`、複雑な調査は `sonnet`

**並列調査**: 独立した複数の調査対象がある場合、`run_in_background: true` で複数の subagent を同時に起動して並列実行し、`get_subagent_result` で結果を回収する。

### 3. 各ソースの使い方

#### コードベース探索 (優先度 1)

コードベース内の調査に使用する。

```
grep: シンボル・文字列の出現箇所を横断検索
read: 定義・実装の読解
find: ファイル配置の把握
```

規模が大きい・調査範囲が広い場合は `Explore` subagent に委譲してコンテキストを節約する。

#### deepwiki MCP (優先度 2)

OSS リポジトリの内部ドキュメントを `mcp` tool 経由で調査する。

```
1. read_wiki_structure: リポジトリの Wiki 構造を確認
2. read_wiki_contents: 特定ページの内容を取得
3. ask_question: 特定の質問に回答を得る
```

リポジトリの指定形式: `owner/repo` (例: `facebook/react`, `vercel/next.js`)

#### context7 MCP (優先度 3)

ライブラリの公式ドキュメントとコード例を `mcp` tool 経由で取得する。

```
1. resolve-library-id: ライブラリ ID を解決
2. query-docs: ドキュメントを検索・取得
```

#### ax CLI (優先度 4)

特定の URL からコンテンツを bash + `ax` (curl 互換 CLI) で取得する。

```bash
ax <url> --md --budget <tokens>   # ページを Markdown 化して取得
ax <url> --outline                # ページ構造の探索
ax <url> --locate '<text>'        # 特定テキストの位置を特定
```

主な用途:

- 公式ドキュメントページの特定セクション
- GitHub Releases / Changelog
- API リファレンス

#### gh CLI (優先度 5)

GitHub の情報を bash + `gh` で取得する。

```bash
gh release view --repo <owner>/<repo> --json tagName,name,publishedAt
gh release list --repo <owner>/<repo> --limit 5
gh api repos/<owner>/<repo>/releases/latest
```

主な用途:

- 最新リリース情報・バージョン確認
- Issues / PR / Discussions の調査

### 4. 結果の構造化

調査結果を以下の形式でまとめる:

```markdown
## 調査結果: <対象名>

### 概要

<1-2 文で要約>

### 詳細

<調査で得られた具体的な情報>

### ソース

- [ソース名](URL) - 取得した情報の概要
```

## ソース選択のフローチャート

```
調査対象は何か？
├── コードベース内の定義・参照 → read / grep / find (または Explore subagent)
├── OSS の内部実装・アーキテクチャ → deepwiki MCP
├── ライブラリの使い方・API → context7 MCP
├── 特定 URL のコンテンツ → ax CLI
└── GitHub のリリース・Issues → gh CLI
```

上位ソースで情報が不足する場合、次の優先度のソースにフォールバックする。

## Additional Resources

### Reference Files

詳細なプロンプトテンプレートやソースごとの使い分けガイド:

- **`references/source-guide.md`** - 各ソースの詳細な使い方と具体例
