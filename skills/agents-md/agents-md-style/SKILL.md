---
name: agents-md-style
description: AGENTS.md のスタイリングガイドライン、構造の推奨事項、コンテンツ最適化戦略を提供する。Use when AGENTS.md ファイルを作成・更新する際、または AGENTS.md の書き方やベストプラクティスを確認する際に使用する。
---

# AGENTS.md Style Guide

AGENTS.md はプロジェクト固有の指示を agent に提供するファイルです。コンテキストウィンドウを消費するため、簡潔かつ効果的な内容にすることが重要です。

## Core Principles

1. **簡潔さ優先**: コンテキストを消費するため、必要最小限の情報のみ記載
2. **具体性**: 抽象的な説明より具体例を優先
3. **実用性**: 実際の摩擦点を解決する内容のみ含める
4. **境界設定**: Always/Ask first/Never の明確な区分

## Required Sections

### 日本語使用時のスタイリング

必須で含める内容:

```markdown
# 日本語使用時のスタイリング

ドキュメントやコードコメントなど、特に指示がない限りは下記を厳守します。

- 技術用語や固有名詞は原文を維持
- スペース: 日本語と半角英数字記号間に半角スペース
- 文体: ですます調、句読点は「。」「、」
  - 箇条書きリストやチェックリストはこの限りではない
- 記号: 丸括弧は半角「()」、鉤括弧は全角「「」」

例:

- Terraform は、素晴らしい IaC (Infrastructure as Code) ツールです。
- Claude Code は、Anthropic 社が開発しているエージェント型 AI コーディングツールです。
```

### コード参照

```markdown
# コード参照

参照元・参照先の調査は Grep ではなく LSP を使用 (goToDefinition, findReferences, incomingCalls, outgoingCalls)
```

### 技術調査

<!-- prettier-ignore -->
```markdown
# 技術調査

優先順位: LSP → deepwiki MCP → context7 MCP → ax による Web 取得

### 例外

- terraform に関する内容は terraform MCP が最優先
- Google Cloud に関する内容は google-developer-knowledge MCP が最優先
```

## Content Guidelines

### Include (効果的な内容)

- 繰り返し入力するコマンド
- アーキテクチャコンテキスト (コンポーネント間の関係)
- ドメイン固有のパターン・規約
- プロジェクト固有のワークフロー
- ツール統合情報 (MCP サーバー等)

### Exclude (含めるべきでない内容)

- センシティブ情報 (API キー、認証情報、DB 接続文字列)
- agent が既に知っている一般的なベストプラクティス
- IDE やエディタの使い方
- 言語やフレームワークの基本的な構文
- 冗長な説明文

## Structure Recommendations

### Optimal Length

- 目標: 500-1,500 words
- 最大: 3,000 words
- 超過時: 内容を精査し、不要な情報を削除

### Section Priority

1. 必須: プロジェクト概要 (1-2 文)
2. 必須: 主要コマンド
3. 必須: スタイリングルール
4. 推奨: アーキテクチャ概要
5. 推奨: ワークフロー
6. オプション: 詳細な規約 (別ファイル参照)

## Boundary Setting Pattern

3 層システムで行動境界を設定:

```markdown
# 行動ルール

## 即座実行 (確認不要)

- コード操作、ファイル管理
- テスト実行
- ドキュメント更新

## 確認必須

- アーキテクチャ変更
- 新 API・外部ライブラリ導入
- DB スキーマ変更
- Git コミット・プッシュ

## 禁止

- 本番環境への直接操作
- 認証情報のハードコード
```

## Optimization Strategies

### Redundancy Detection

以下は agent が既知のため省略可能:

- 一般的なコーディング規約 (PEP8, ESLint 等)
- 標準的な Git ワークフロー
- 言語固有のベストプラクティス
- フレームワークの基本的な使い方

### Consolidation Techniques

- 類似ルールを統合
- 例外は別途記載せず、ルールに組み込む
- 詳細は外部ファイル参照 (`docs/` 等)

### Update Checklist

AGENTS.md 更新時の確認項目:

1. 重複コンテンツはないか
2. agent が既知の内容が含まれていないか
3. 具体例 > 説明文の原則を守っているか
4. 実際に使用されている情報か
5. 境界設定は明確か

## Quick Reference

### Good Example

```markdown
# MyProject

React + TypeScript のダッシュボードアプリ。

## コマンド

- `bun dev`: 開発サーバー起動
- `bun test`: テスト実行
- `bun lint`: Lint チェック

## 日本語スタイリング

[Required section content]

## アーキテクチャ

- `src/components/`: UI コンポーネント
- `src/hooks/`: カスタム Hooks
- `src/api/`: API クライアント
```

### Bad Example

```markdown
# MyProject

これは React と TypeScript で作られたダッシュボードアプリケーションです。
React は Facebook が開発した JavaScript ライブラリで...
[冗長な説明が続く]

## 開発方法

開発を始めるには、まず依存関係をインストールします。
npm install または yarn install を実行してください...
[agent が既知の内容]
```

## Implementation Workflow

AGENTS.md 作成時:

1. プロジェクト構造を分析
2. 必須セクションを追加
3. プロジェクト固有の情報を追加
4. 冗長性を確認・削除
5. 文字数を確認 (1,500 words 以下目標)

AGENTS.md 更新時:

1. 現在の内容を読み込み
2. 各セクションの必要性を評価
3. 冗長・重複コンテンツを特定
4. 必須セクションの準拠確認
5. 最適化・統合
