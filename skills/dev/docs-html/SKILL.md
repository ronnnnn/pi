---
name: docs-html
description: 引数として渡された内容や資料を分析し、用途に応じたレイアウト・スタイリングを選択して、dark/light モード切替と日/英 言語切替を備えた単一 HTML ドキュメントを生成する。Use when HTML ドキュメント作成、リッチな仕様書/レポート/レビュー資料作成、Markdown より表現力のある成果物を求められた際に使用する。引数は生成したい内容、資料のパス、URL、PR など。
---

# HTML ドキュメント生成ワークフロー

引数として渡された内容や資料 (ファイル、URL、PR、テキスト指示等) を分析し、用途に最適なカテゴリを判定して、専用テンプレートをもとに単一の自己完結型 HTML ファイルを生成する。出力するすべての HTML には **dark/light モード切替** と **日/英 言語切替** を必須で組み込む。

## 設計思想

Markdown より HTML を選ぶ理由は「情報密度」「視覚的明確さ」「共有のしやすさ」「双方向操作」「データ統合」の 5 点である (参考: [Using Claude Code: The unreasonable effectiveness of HTML](https://claude.com/blog/using-claude-code-the-unreasonable-effectiveness-of-html))。本 skill はこの思想に従い、Markdown では表現しきれない情報を 1 ファイルの HTML で表現する。

## 重要な原則

1. **単一ファイル・ビルド不要** - CSS・JS・SVG・データすべてを 1 つの `.html` に内包し、外部依存なしでブラウザで開けるようにする (CDN 含めて外部リソースに依存しない)
2. **カテゴリ判定が品質を決める** - 内容に応じたカテゴリを正しく選び、それに合ったレイアウト/スタイル/コンポーネントを採用する
3. **dark/light/言語切替は必須機能** - すべての生成 HTML に組み込む。実装パターンは `references/theme-toggle.md`, `references/i18n-toggle.md` を厳守する
4. **両言語を同時に内包する** - 日本語と英語の両方のテキストを HTML 内に保持し、切替ボタンで瞬時に切り替える (再ロード不要)
5. **モバイル対応** - `viewport` メタタグと CSS のレスポンシブ設計を必ず行う
6. **アクセシビリティ** - 適切な `lang` 属性、コントラスト比、フォーカススタイル、ARIA を確保する

## 作業開始前の準備

**必須:** `todo({ action: "list" })` で残存タスクを確認し、存在する場合は `todo({ action: "clear" })` で全て削除する。その後、以下のステップを todo tool で登録する:

```
todo({ action: "add", text: "入力の取得と分析 (引数の資料を読み込み内容を把握)" })
todo({ action: "add", text: "カテゴリ判定 (内容から適切な HTML カテゴリを決定)" })
todo({ action: "add", text: "出力先の確認 (出力先パスを確認)" })
todo({ action: "add", text: "コンテンツ構造化と多言語化 (日/英 両言語のコンテンツを準備しセクション構造に整理)" })
todo({ action: "add", text: "HTML 生成 (選定したテンプレートに基づき HTML を出力)" })
todo({ action: "add", text: "セルフチェック (必須機能 (テーマ/言語切替) とリンク/構文を検証)" })
todo({ action: "add", text: "完了報告 (出力ファイル、採用カテゴリ、特記事項を報告)" })
```

各ステップの完了時に `todo({ action: "toggle", id: <id> })` で完了にする。

## 処理フロー

### Step 1: 入力の取得と分析

引数の形式に応じてコンテンツを取得する:

- **ファイルパス**: read tool で読み込み
- **URL**: bash で `ax <url> --md` により取得
- **PR / Issue**: `gh pr view`, `gh pr diff`, `gh issue view` で取得
- **複数資料**: 全て取得し関連性を整理
- **テキスト指示のみ**: 引数本文をそのまま分析対象とする

抽出する情報:

- **目的** (何を伝えたい資料か)
- **主要素材** (コード、データ、図、設計、判断軸など)
- **想定読者** (本人 / チーム / 経営層 / 一般)
- **判定材料となるキーワード** (カテゴリ判定に使用)

### Step 2: カテゴリ判定

`references/categories.md` の分類表を参照し、最適なカテゴリを 1 つ選ぶ。

主要カテゴリ:

| カテゴリ ID | 用途                                               |
| ----------- | -------------------------------------------------- |
| `plan`      | 仕様書・実装計画・タスク分解・探索                 |
| `review`    | コードレビュー・PR 解説・差分注釈                  |
| `design`    | デザインシステム・モックアップ・コンポーネント比較 |
| `prototype` | アニメーション/インタラクション試作・スライダー UI |
| `report`    | 状況報告・インシデント・週次レポート               |
| `explainer` | 技術解説・チュートリアル・概念説明                 |
| `diagram`   | フローチャート・SVG 図・アーキテクチャ図           |
| `slide`     | スライドデッキ                                     |
| `editor`    | カスタム編集 UI・トリアージボード・設定エディタ    |

判定に迷う場合は question tool で確認する。複数カテゴリの要素がある場合は主要素材に最も近い 1 つを選び、補助要素は流用するコンポーネント (差分、図、表など) として組み込む。

### Step 3: 出力先の確認

question tool で出力先パスを確認する。選択肢の例:

- `.pi/docs/` (デフォルト推奨)
- `docs/` (リポジトリの公開ドキュメント置き場)
- 任意のカスタムパス

ファイル名は `<連番 or 日付>-<カテゴリ>-<内容>.html` 形式 (例: `20260521-plan-onboarding.html`)。ディレクトリが存在しない場合は作成する。

### Step 4: コンテンツ構造化と多言語化

#### 構造化

カテゴリのテンプレートに合わせてセクションを設計する。レイアウト/コンポーネントの選び方は `references/html-templates.md` を参照。

#### 多言語化

**全てのテキストノードを日/英 両方準備する**。手順:

1. 元コンテンツが片言語のみの場合、もう片方を翻訳して用意する (技術用語/固有名詞は原語維持)
2. テキストは `data-ja` / `data-en` 属性で両言語を保持
3. 細かい注釈・図のラベル・ボタン名も漏れなく対象とする

実装の詳細は `references/i18n-toggle.md` を厳守する。

### Step 5: HTML 生成

`examples/base-template.html` をベースに、選定したカテゴリのレイアウト・スタイル・コンポーネントを適用して 1 ファイルを生成する。

#### 必須要素 (すべてのカテゴリ共通)

- `<!doctype html>` と `<html lang="ja">` (初期言語は ja、切替で en)
- `<meta charset="utf-8">` と `<meta name="viewport" content="width=device-width, initial-scale=1">`
- `<title>` (日本語タイトル、言語切替時に英語に変える)
- CSS 変数による色管理 (light/dark の両セットを `:root` と `[data-theme="dark"]` に定義)
- `prefers-color-scheme` での初期値判定 + `localStorage` での状態保持
- 言語/テーマ切替ボタンを右上に固定配置 (モバイルでもタップ可)
- レスポンシブ設計 (max-width 1100px 程度のコンテンツ幅、モバイル時はパディング縮小)

#### カテゴリ別の追加要素

`references/html-templates.md` に各カテゴリの「使うべきコンポーネント」「レイアウト」「色のアクセント」を記載しているので必ず参照する。

#### スタイリング指針

- システムフォント優先 (`system-ui, -apple-system, "Segoe UI", Roboto, ...`)
- 等幅フォントは `ui-monospace, "SF Mono", Menlo, Consolas, monospace`
- 配色はカテゴリで雰囲気を変える (`references/html-templates.md` のパレット参照)
- 数式やコードハイライトが必要な場合も外部 CDN を使わず、`<pre><code>` + 軽量な JS 着色で完結させる

### Step 6: セルフチェック

生成後、以下を必ず bash でチェックする:

```bash
# 構文/構造の簡易チェック
grep -cE 'data-theme=["'"'"']dark["'"'"']' <出力ファイル>  # dark テーマの CSS が存在するか
grep -cE 'data-(lang|ja|en)' <出力ファイル>          # i18n 仕組みが存在するか
grep -c 'localStorage' <出力ファイル>                # 永続化が組み込まれているか
grep -c 'viewport' <出力ファイル>                    # viewport 設定
grep -c 'aria-' <出力ファイル>                       # ARIA 属性
```

検証項目:

- [ ] dark/light 切替ボタンが存在し、`localStorage` で状態保持される
- [ ] 日/英 切替ボタンが存在し、すべてのテキストが両言語で用意されている
- [ ] `prefers-color-scheme` で OS 設定を初期値に反映している
- [ ] ブラウザに直接ドラッグ&ドロップして開ける (外部依存なし)
- [ ] モバイル幅 (375px) でレイアウト崩れがない
- [ ] コードブロック、表、SVG など必要なコンポーネントが採用カテゴリに沿っている

問題があれば edit tool で修正する。

### Step 7: 完了報告

以下を報告する:

```markdown
## HTML ドキュメント生成完了

- 出力ファイル: `<path>`
- 採用カテゴリ: `<category>` (理由: ...)
- 組み込み機能: dark/light 切替, 日/英 切替, レスポンシブ, localStorage 永続化
- 主要セクション: ...
- 確認方法: `open <path>` (macOS) または対応ブラウザで開く
```

開いて確認したいかを聞き、ユーザーが希望すれば `open` コマンドを提案する。

## エラーハンドリング

- **入力が不明確**: questionnaire tool で「何を伝える HTML か」「想定読者は誰か」「主要素材は何か」を確認する
- **カテゴリが定まらない**: 候補を 2-3 提示し、question tool で選んでもらう
- **外部 URL 取得失敗**: ユーザーに代替手段 (ローカルコピー貼付など) を依頼する
- **コンテンツが巨大すぎる**: 主要セクションに絞り込み、続編 HTML への分割を提案する

## Additional Resources

### Reference Files

実装と判定時に参照:

- **`references/categories.md`** - カテゴリ分類表、判定キーワード、補助要素の組合せ方
- **`references/html-templates.md`** - カテゴリ別レイアウト・コンポーネント・配色パレット
- **`references/theme-toggle.md`** - dark/light モード切替の実装パターン (CSS 変数、`localStorage`, `prefers-color-scheme`)
- **`references/i18n-toggle.md`** - 日/英 言語切替の実装パターン (`data-ja` / `data-en` 属性方式、`<html lang>` 切替、`title` 切替)

### Examples

- **`examples/base-template.html`** - dark/light + 日/英 切替を含む最小完備のベーステンプレート。すべての生成 HTML はこのファイルを土台として、カテゴリ別のセクション・スタイルを追加していく
