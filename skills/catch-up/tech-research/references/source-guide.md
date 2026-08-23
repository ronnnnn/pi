# 調査ソース詳細ガイド

各ソースの詳細な使い方、具体的なプロンプト例、よくあるユースケースをまとめる。

## コードベース探索 (優先度 1)

### 対象ユースケース

- 関数の定義元を知りたい
- あるシンボルがどこから参照されているか調べたい
- 呼び出し関係を追跡したい
- 実装の詳細を確認したい

### 具体例

```
# シンボルの出現箇所を横断検索
grep("handleSubmit")
→ src/components/Form.tsx:45, ...

# 定義・実装を読解
read("src/components/Form.tsx")

# ファイル配置を把握
find("src/**/*.ts")
```

### Explore subagent に委譲する基準

- 調査範囲が複数モジュールにまたがる
- 呼び出し関係を何段も追跡する必要がある
- 親セッションのコンテキストを消費したくない

```
subagent({ subagent_type: "Explore", prompt: "<調査内容を具体的に記述>", description: "コードベース調査" })
```

## deepwiki MCP (優先度 2)

### 対象ユースケース

- OSS ライブラリの内部アーキテクチャを理解したい
- 特定の機能がどう実装されているか知りたい
- コントリビューションガイドを確認したい

### 使い方

`mcp` tool 経由で deepwiki MCP のツールを呼び出す。

```
# Step 1: Wiki 構造を確認
read_wiki_structure (repoName: "vercel/next.js")

# Step 2: 特定ページを読む
read_wiki_contents (repoName: "vercel/next.js", pagePath: "architecture/routing")

# Step 3: 特定の質問をする
ask_question (repoName: "vercel/next.js", question: "How does the App Router handle parallel routes?")
```

### よくあるリポジトリ名

| ツール     | リポジトリ           |
| ---------- | -------------------- |
| React      | facebook/react       |
| Next.js    | vercel/next.js       |
| Vue.js     | vuejs/core           |
| Svelte     | sveltejs/svelte      |
| Bun        | oven-sh/bun          |
| Deno       | denoland/deno        |
| Rust       | rust-lang/rust       |
| Go         | golang/go            |
| TypeScript | microsoft/TypeScript |
| Terraform  | hashicorp/terraform  |
| Docker     | moby/moby            |

## context7 MCP (優先度 3)

### 対象ユースケース

- ライブラリの API リファレンスを確認したい
- 公式ドキュメントのコード例を取得したい
- 設定方法やオプションを調べたい

### 使い方

`mcp` tool 経由で context7 MCP のツールを呼び出す。

```
# Step 1: ライブラリ ID を解決
resolve-library-id (libraryName: "react")

# Step 2: ドキュメントを検索
query-docs (libraryId: "<resolved-id>", query: "useEffect cleanup function")
```

### 効果的なクエリ例

| 目的             | クエリ例                          |
| ---------------- | --------------------------------- |
| API の使い方     | "useState hook usage"             |
| 設定方法         | "configuration options"           |
| マイグレーション | "migration guide v2 to v3"        |
| 特定機能         | "server components data fetching" |

## ax CLI (優先度 4)

### 対象ユースケース

- 公式ドキュメントの特定ページを参照したい
- GitHub Releases のリリースノートを確認したい
- Changelog を読みたい

### 使い方

bash で `ax` (curl 互換 CLI) を実行する。

```bash
# ページ全体を Markdown 化して取得 (トークン量を制御)
ax https://nodejs.org/en/blog/release/v22.0.0 --md --budget 4000

# ページ構造の探索 (どこに何があるか把握してから読む)
ax https://docs.example.com/guide --outline

# 特定テキストの位置を特定
ax https://docs.example.com/guide --locate 'configuration'

# 構造化データ抽出
ax https://example.com '<selector>' --table
```

### よく使う URL パターン

| 用途            | URL パターン                                 |
| --------------- | -------------------------------------------- |
| GitHub Releases | `https://github.com/{owner}/{repo}/releases` |
| npm パッケージ  | `https://www.npmjs.com/package/{name}`       |
| PyPI パッケージ | `https://pypi.org/project/{name}/`           |
| Go パッケージ   | `https://pkg.go.dev/{module}`                |
| Rust crate      | `https://crates.io/crates/{name}`            |

## gh CLI (優先度 5)

### 対象ユースケース

- 最新のリリース情報を確認したい
- GitHub Issues / PR / Discussions を調査したい
- GitHub API から構造化データを取得したい

### 使い方

```bash
# 最新リリースを取得
gh release view --repo <owner>/<repo> --json tagName,name,isPrerelease,publishedAt

# リリース一覧 (pre-release の判別込み)
gh release list --repo <owner>/<repo> --limit 5 --json tagName,isPrerelease,publishedAt

# GitHub API を直接叩く
gh api repos/<owner>/<repo>/releases/latest

# Issues を検索
gh api search/issues -f q='repo:<owner>/<repo> <keyword>'
```

## 例外的な優先順位

以下の内容については、通常の優先順位に関わらず専用ソースを最優先で使用する:

- **terraform に関する内容**: terraform MCP (`mcp` tool 経由) が最優先
- **Google Cloud に関する内容**: google-developer-knowledge MCP (`mcp` tool 経由) が最優先
- **pi (coding agent) に関する内容**: pi 同梱ドキュメント (mise 管理の pi 本体の `docs/`・README・examples) を read するのが最優先

## フォールバック戦略

上位ソースで情報が得られない場合のフォールバックパターン:

### パターン 1: ライブラリの使い方

```
context7 MCP で API ドキュメントを取得
  ↓ 不十分な場合
deepwiki MCP でリポジトリの内部ドキュメントを確認
  ↓ 不十分な場合
ax CLI で公式ドキュメントページを直接取得
```

### パターン 2: 最新バージョン情報

```
latest-version agent を起動 (gh CLI 優先)
  ↓ 不十分な場合
ax CLI で GitHub Releases ページを取得
  ↓ 不十分な場合
ax CLI で公式サイトのダウンロード・リリースページを取得
```

### パターン 3: エラー・問題解決

```
read / grep / find でコードベース内の関連コードを調査
  ↓ 不十分な場合
deepwiki MCP で関連リポジトリの Wiki に質問 (ask_question)
  ↓ 不十分な場合
gh api で関連リポジトリの Issues を検索
```
