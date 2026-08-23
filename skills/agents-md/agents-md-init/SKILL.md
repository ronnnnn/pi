---
name: agents-md-init
description: AGENTS.md を新規作成する。プロジェクト構造を分析してテンプレートを生成し、ユーザー承認後に書き込む。Use when プロジェクトに AGENTS.md がない、または新規作成を求められた際に使用する。
---

# AGENTS.md 作成ワークフロー

プロジェクト構造を分析し、AGENTS.md ファイルを新規作成する。

## 重要な原則

1. **プロジェクト固有の情報のみ記載** - agent が既知の一般的なベストプラクティスは含めない
2. **簡潔さ優先** - コンテキストウィンドウを消費するため、必要最小限の情報のみ
3. **必須セクションを含める** - 日本語スタイリング、コード参照、技術調査優先順位
4. **ユーザー承認を得てから書き込む**

## 作業開始前の準備

**必須:** 作業開始前に `todo({ action: "list" })` で残存タスクを確認し、存在する場合は `todo({ action: "clear" })` で全て削除する。その後、`todo` tool で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "プロジェクト構造の分析 (パッケージマネージャー/ビルドツールを検出)" })
todo({ action: "add", text: "技術スタックの検出 (設定ファイルからスタックを検出)" })
todo({ action: "add", text: "既存 AGENTS.md の確認 (既存ファイルの有無を確認)" })
todo({ action: "add", text: "テンプレート生成 (必須セクションを含むテンプレートを生成)" })
todo({ action: "add", text: "ユーザー承認 (生成内容の承認を求める)" })
todo({ action: "add", text: "ファイル書き込み (承認後に AGENTS.md を作成)" })
todo({ action: "add", text: "完了報告 (作成結果を報告)" })
```

各ステップの完了時に `todo({ action: "toggle", id: <id> })` で完了済みに更新する。

## 実行手順

### 1. 作成場所の決定

- 引数 (パス) が指定された場合: `<path>/AGENTS.md`
- 引数がない場合: 現在のディレクトリの `AGENTS.md`

### 2. プロジェクト構造の分析

```bash
# パッケージマネージャー/ビルドツールを検出
ls -la package.json Cargo.toml go.mod pyproject.toml Makefile justfile 2>/dev/null
```

### 3. 技術スタックの検出

- `package.json`, `Cargo.toml`, `go.mod`, `pyproject.toml` 等からスタックを検出
- `Makefile`, `justfile`, `package.json scripts` からコマンドを抽出
- ディレクトリ構造を把握

### 4. 既存 AGENTS.md の確認

```bash
# 既存ファイルを確認 (CLAUDE.md 併存コピーの有無も確認)
ls -la AGENTS.md CLAUDE.md 2>/dev/null
```

既存ファイルがある場合は確認後マージ。

### 5. テンプレート生成

生成前に、スタイルガイド (`../agents-md-style/SKILL.md`) を `read` で読み込む。

以下のセクションを必須で含める:

<!-- prettier-ignore -->
```markdown
# [プロジェクト名]

[1-2 文のプロジェクト概要]

## コマンド

[検出したコマンド一覧]

## 日本語使用時のスタイリング

ドキュメントやコードコメントなど、特に指示がない限りは下記を厳守します。

- 技術用語や固有名詞は原文を維持
- スペース: 日本語と半角英数字記号間に半角スペース
- 文体: ですます調、句読点は「。」「、」
  - 箇条書きリストやチェックリストはこの限りではない
- 記号: 丸括弧は半角「()」、鉤括弧は全角「「」」

例:

- Terraform は、素晴らしい IaC (Infrastructure as Code) ツールです。
- Claude Code は、Anthropic 社が開発しているエージェント型 AI コーディングツールです。

## コード参照

参照元・参照先の調査は Grep ではなく LSP を使用 (goToDefinition, findReferences, incomingCalls, outgoingCalls)

## 技術調査

優先順位: LSP → deepwiki MCP → context7 MCP → ax による Web 取得
```

### 6. 最適化

- agent が既知の一般的な内容は含めない
- 目標: 500-1,500 words

### 7. ユーザー承認

生成した内容を表示し、`question` tool で承認を得る。

### 8. ファイル書き込み

承認後、`write` tool で AGENTS.md を作成。

**CLAUDE.md 併存コピーとの同期:** プロジェクトに CLAUDE.md が AGENTS.md の実ファイルコピーとして併存する場合は、書き込み後にバイト単位で一致するよう同期する (プロジェクトに同期タスクがあればそれを使用)。

### 9. 完了報告

```
## AGENTS.md 作成完了

- **ファイル:** <path>
- **文字数:** X words
- **セクション数:** N
```

## エラーハンドリング

### 既存ファイルがある場合

1. 既存内容を読み込み
2. マージ方法をユーザーに確認
3. 承認後に上書き or マージ
