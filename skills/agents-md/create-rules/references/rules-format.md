# .claude/rules ファイルフォーマット

## 概要

`.claude/rules/` ディレクトリ内の `.md` ファイルは、プロジェクト固有のルールを agent に提供する。サブディレクトリも含めて自動検出される (pi では claude-rules extension が読み込む)。

## ファイル構造

```
.claude/rules/
├── general.md              # paths なし: 全ファイルに適用
├── api-patterns.md         # paths あり: API 関連ファイルのみ
├── frontend/
│   ├── components.md       # paths あり: コンポーネントファイルのみ
│   └── styling.md          # paths あり: スタイルファイルのみ
└── testing/
    └── e2e.md              # paths あり: E2E テストファイルのみ
```

## YAML Frontmatter

### paths あり (スコープ付き)

```markdown
---
paths:
  - 'src/api/**/*.ts'
  - 'src/services/**/*.ts'
---

# API Layer Rules

- エラーは必ず AppError クラスでラップする
- レスポンスは ResponseBuilder を使用して構築する
```

`paths` が指定されている場合、agent がマッチするファイルを操作しているときのみルールがロードされる。

### paths なし (グローバル)

```markdown
---
---

# General Rules

- 環境変数は env.ts 経由でアクセスする
- ログは logger モジュールを使用する
```

`paths` が省略されている場合、すべてのファイル操作時にルールがロードされる。コンテキスト節約のため、本当にプロジェクト全体に適用されるルールのみ `paths` なしにする。

## paths Glob パターン

### 基本パターン

| パターン               | マッチ対象                               |
| ---------------------- | ---------------------------------------- |
| `**/*.ts`              | 全ディレクトリの `.ts` ファイル          |
| `src/**/*`             | `src/` 以下の全ファイル                  |
| `*.md`                 | プロジェクトルートの `.md` ファイル      |
| `src/components/*.tsx` | `src/components/` 直下の `.tsx` ファイル |

### ブレース展開

| パターン                     | マッチ対象                       |
| ---------------------------- | -------------------------------- |
| `src/**/*.{ts,tsx}`          | `.ts` と `.tsx` の両方           |
| `{src,lib}/**/*.ts`          | `src/` と `lib/` 両方の `.ts`    |
| `src/{api,services}/**/*.ts` | `api/` と `services/` 内の `.ts` |

### スコープを狭くする例

広い:

```yaml
paths:
  - '**/*.ts'
```

狭い (推奨):

```yaml
paths:
  - 'src/api/**/*.ts'
  - 'src/services/**/*.ts'
```

可能な限り狭いスコープを使用し、不要なコンテキスト消費を防ぐ。

## サンプルルールファイル

### API エラーハンドリング

```markdown
---
paths:
  - 'src/api/**/*.ts'
  - 'src/controllers/**/*.ts'
---

# API Error Handling

- 全てのエラーは AppError クラスでラップする
- HTTP ステータスコードは ErrorCode enum を使用
- バリデーションエラーは ValidationError を使用し、フィールド単位のエラーを返す
- 外部 API 呼び出しのエラーは ExternalServiceError でラップし、リトライ情報を含める
```

### コンポーネント設計

```markdown
---
paths:
  - 'src/components/**/*.tsx'
  - 'src/features/**/*.tsx'
---

# Component Design

- Presentational コンポーネントは components/ に配置、ビジネスロジックは含めない
- Feature コンポーネントは features/ に配置、hooks でロジックを分離
- Props の型定義はコンポーネントと同じファイルに配置
- スタイルは CSS Modules を使用、グローバルスタイルは styles/ に配置
```

### テスト規約

```markdown
---
paths:
  - 'src/**/*.test.ts'
  - 'src/**/*.spec.ts'
  - 'tests/**/*'
---

# Testing Conventions

- テストファイルはテスト対象と同じディレクトリに配置
- テスト名は日本語で記述 (例: `it("ユーザーが存在しない場合は 404 を返す")`)
- モックは **mocks**/ ディレクトリに配置、テストファイル内のインラインモックは避ける
- E2E テストは tests/e2e/ に配置
```

### データベース操作

```markdown
---
paths:
  - 'src/repositories/**/*.ts'
  - 'src/db/**/*.ts'
  - 'prisma/**/*'
---

# Database Operations

- DB アクセスは Repository パターンで抽象化
- 複数テーブルの更新はトランザクションを使用
- マイグレーションファイル名は YYYYMMDDHHMMSS_description 形式
- Seed データは db/seeds/ に配置
```

## ファイル管理のベストプラクティス

### 命名規則

- ケバブケース: `api-error-handling.md`, `component-structure.md`
- 内容を端的に表す名前
- プレフィックスでカテゴリを示す場合もあり: `test-conventions.md`, `test-e2e.md`

### ファイルサイズの目安

- 1 ファイルあたり 10-50 行 (ルール部分)
- 50 行を超える場合はトピック分割を検討
- 短すぎる (5 行未満) 場合は関連ファイルとの統合を検討

### ディレクトリ構成パターン

シンプルなプロジェクト:

```
.claude/rules/
├── general.md
├── api.md
└── testing.md
```

中規模プロジェクト:

```
.claude/rules/
├── general.md
├── frontend/
│   ├── components.md
│   └── state-management.md
├── backend/
│   ├── api.md
│   └── database.md
└── testing.md
```

大規模プロジェクト:

```
.claude/rules/
├── general.md
├── frontend/
│   ├── components.md
│   ├── routing.md
│   ├── state-management.md
│   └── styling.md
├── backend/
│   ├── api.md
│   ├── database.md
│   ├── auth.md
│   └── messaging.md
├── infrastructure/
│   ├── terraform.md
│   └── ci-cd.md
└── testing/
    ├── unit.md
    ├── integration.md
    └── e2e.md
```
