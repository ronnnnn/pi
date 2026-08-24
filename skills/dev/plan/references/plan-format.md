# 行動計画ファイルフォーマット

## 全体概要ファイル (00-overview.md)

```markdown
# [計画タイトル]

## 概要

[何を達成するか 1-3 文で記述]

## 前提条件

- [環境、ツール、アクセス権等]

## フェーズ一覧

| #   | フェーズ | タスク数 | 依存 | 概要   |
| --- | -------- | -------- | ---- | ------ |
| 1   | [名前]   | N        | -    | [概要] |
| 2   | [名前]   | N        | 1    | [概要] |
| 3   | [名前]   | N        | 1    | [概要] |
| 4   | [名前]   | N        | 2, 3 | [概要] |

## タスクサマリー

- **総タスク数**: N
  - AI agent: X
  - ユーザー手動: Y
  - AI agent + ユーザー確認: Z
- **クリティカルパス**: Phase 1 → Phase 2 → Phase 4
- **並列実行可能**: Phase 2 と Phase 3

## 凡例

- 🤖 AI agent が実行
- 👤 ユーザーが手動で実行
- 🤝 AI agent が実行 + ユーザーが確認
```

## フェーズファイル (NN-phase-name.md)

```markdown
# Phase N: [フェーズ名]

## 目的

[このフェーズで達成すること]

## 依存関係

- **前提フェーズ**: Phase X (完了必須)
- **後続フェーズ**: Phase Y, Phase Z

## タスク一覧

| ID  | タスク | 実行主体 | 依存   | 並列        |
| --- | ------ | -------- | ------ | ----------- |
| T1  | [名前] | 🤖       | -      | T2 と並列可 |
| T2  | [名前] | 🤖       | -      | T1 と並列可 |
| T3  | [名前] | 👤       | T1     | -           |
| T4  | [名前] | 🤝       | T2, T3 | -           |

---

[以下、各タスクの詳細]
```

## タスク記述フォーマット

```markdown
### T1: [タスク名] 🤖

**概要**: [何をするか 1-2 文]

**依存**: なし (または TN)
**並列**: TN と並列実行可能 (または 逐次実行)

**手順**:

1. [具体的な手順 1]
2. [具体的な手順 2]
3. [具体的な手順 3]

**成果物**:

- `path/to/file.ts` (新規作成 or 変更)

**完了条件**:

- [ ] `bun test path/to/file.test.ts` が全て PASS
- [ ] `bun tsc --noEmit` でエラーなし
```

## 実行主体別テンプレート

### AI agent タスク (🤖)

```markdown
### T1: API エンドポイントの実装 🤖

**概要**: /api/users エンドポイントを実装する。

**依存**: T3 (DB スキーマ作成)
**並列**: T5 (フロントエンド実装) と並列実行可能

**手順**:

1. `src/api/users.ts` を作成
2. GET /api/users ハンドラを実装
3. バリデーションを追加
4. ユニットテストを作成

**成果物**:

- `src/api/users.ts` (新規作成)
- `src/api/users.test.ts` (新規作成)

**完了条件**:

- [ ] `bun test src/api/users.test.ts` が全て PASS
- [ ] `bun tsc --noEmit` でエラーなし
```

### ユーザー手動タスク (👤)

````markdown
### T2: 外部サービスの API キー取得 👤

**概要**: Stripe の API キーを取得し、環境変数に設定する。

**依存**: なし
**並列**: T1 と並列実行可能

**手順**:

1. [Stripe Dashboard](https://dashboard.stripe.com/) にログイン
2. 「Developers」→「API keys」に移動
3. 「Secret key」をコピー
4. プロジェクトルートに `.env.local` を作成
5. 以下の内容を記述:

```

STRIPE_SECRET_KEY=sk_test_xxxxx

```

**成果物**:

- `.env.local` に STRIPE_SECRET_KEY が設定されていること

**完了条件**:

- [ ] `.env.local` に STRIPE_SECRET_KEY が存在する
- [ ] `echo $STRIPE_SECRET_KEY` で値が表示される (source 後)
````

### AI agent + ユーザー確認タスク (🤝)

```markdown
### T4: ログイン画面の実装 🤝

**概要**: ログイン画面のコンポーネントを実装し、UI をユーザーが確認する。

**依存**: T1 (API 実装)
**並列**: 逐次実行

**手順** (AI agent):

1. `src/components/LoginForm.tsx` を作成
2. フォームバリデーションを実装
3. API 連携を実装
4. テストを作成

**手順** (ユーザー確認):

1. `bun dev` で開発サーバーを起動
2. ブラウザで `http://localhost:3000/login` を開く
3. 以下を確認:
   - フォームが正しく表示される
   - メールアドレスとパスワードを入力できる
   - バリデーションエラーが表示される (空欄で送信)
   - 正しい認証情報でログインが成功する

**成果物**:

- `src/components/LoginForm.tsx` (新規作成)
- `src/components/LoginForm.test.tsx` (新規作成)

**完了条件**:

- [ ] `bun test src/components/LoginForm.test.tsx` が全て PASS
- [ ] ブラウザでフォームが正しく表示される (手動確認)
- [ ] ログインフローが正常に動作する (手動確認)
```

## 依存関係の記述パターン

### 単純な逐次依存

```
T1 → T2 → T3
```

```markdown
### T2: [タスク名]

**依存**: T1
```

### 並列実行可能

```
T1 ──┐
     ├──→ T3
T2 ──┘
```

```markdown
### T1: [タスク名]

**並列**: T2 と並列実行可能

### T2: [タスク名]

**並列**: T1 と並列実行可能

### T3: [タスク名]

**依存**: T1, T2
```

### 部分依存

```markdown
### T3: [タスク名]

**依存**: T1 (成果物: `src/types.ts` のみ必要)
```

## 完了条件の記述パターン

### 自動検証

```markdown
**完了条件**:

- [ ] `bun test path/to/test.ts` が全て PASS
- [ ] `bun tsc --noEmit` でエラーなし
- [ ] `bun lint` でエラーなし
- [ ] `bun build` が成功
- [ ] `curl -s http://localhost:3000/api/health | jq .status` が `"ok"` を返す
```

### 手動検証

```markdown
**完了条件**:

- [ ] ブラウザで `http://localhost:3000/` を開き、ダッシュボードが表示される
  - 左サイドバーにナビゲーションが表示される
  - ヘッダーにユーザー名が表示される
  - メインエリアにグラフが表示される
- [ ] モバイル表示 (幅 375px) でレイアウトが崩れない
  - Chrome DevTools → Toggle device toolbar → iPhone SE を選択
```

### 混合検証

```markdown
**完了条件**:

- [ ] `bun test` が全て PASS (自動)
- [ ] `bun build` が成功 (自動)
- [ ] ブラウザで動作確認 (手動):
  - `bun dev` で起動後 `http://localhost:3000/` にアクセス
  - フォーム送信が正常に動作する
```
