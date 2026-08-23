---
name: pr-fix
description: PR のレビューコメントに基づいて修正を行う。未解決コメントのみを対象とし (自分のコメントは除外)、妥当性を判断して修正する。コミット・返信前にユーザー承認を取る。Use when PR のレビュー指摘を修正したい、レビューコメントに対応したい際に使用する。
---

# PR レビュー修正ワークフロー

PR のレビューコメントを確認し、必要な修正を行う。引数で PR 番号が指定された場合はそれを対象とする。

## 重要な原則

1. **未解決 (unresolved) のコメントのみを対象とする (自分のコメントは除外)**
2. **レビューの妥当性を判断し、修正が必要なもののみ修正する**
3. **コミット前に必ずユーザーの承認を取る** - 自動でコミットしない。承認確認には question tool を使用する
4. **返信コメント前に必ずユーザーの承認を取る** - 自動で返信を投稿しない
5. **コミットメッセージは commit-proposer subagent で Conventional Commits / commitlint 設定に準拠して生成する**
6. **コミットメッセージ・返信コメントの言語は対象リポジトリに従う** - 既存の PR やコミット履歴を確認し、リポジトリで使用されている言語 (日本語/英語等) に合わせる
7. **日本語でコミットメッセージ・返信コメントを書く場合は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用する
8. **修正は最小限に留める** - レビュー指摘以外の変更は含めない
9. **返信は丁寧かつ簡潔に**

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "PR の特定: 引数または現在のブランチから PR を特定" })
todo({ action: "add", text: "未解決のレビューコメントを取得: GraphQL で isResolved: false のスレッドを取得" })
todo({ action: "add", text: "レビューコメントの分析とファクトチェック: 各コメントの妥当性を判断し、技術的主張をファクトチェック" })
todo({ action: "add", text: "修正計画の提示: ユーザーに修正計画の承認を求める" })
todo({ action: "add", text: "コード修正の実行: 承認された修正を適用" })
todo({ action: "add", text: "変更のステージング: 修正したファイルを git add でステージング" })
todo({ action: "add", text: "コミットメッセージの生成: commit-proposer subagent でメッセージ候補を生成" })
todo({ action: "add", text: "コミット前の承認確認: ユーザーにコミットの承認を求める" })
todo({ action: "add", text: "コミットの実行: 承認されたメッセージでコミット" })
todo({ action: "add", text: "プッシュの実行: git push でリモートに反映" })
todo({ action: "add", text: "返信コメントの作成: 各レビューコメントへの返信を作成" })
todo({ action: "add", text: "返信・resolve の承認確認: ユーザーに返信と resolve の承認を求める" })
todo({ action: "add", text: "返信の投稿・スレッド resolve: 返信投稿とスレッド resolve を実行" })
todo({ action: "add", text: "レビュー再リクエスト: bot 以外・未 approve のレビュワーにレビュー再リクエストを送信" })
todo({ action: "add", text: "完了報告: 修正結果を報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. PR の特定

引数で PR 番号が指定されていない場合、現在のブランチから PR を特定する:

```bash
# 現在のブランチに関連する PR を取得
gh pr list --head $(git branch --show-current) --json number,title,state --jq '.[0]'

# または現在のブランチの PR 番号を取得
gh pr view --json number --jq '.number'
```

### 2. 未解決のレビューコメントを取得

```bash
# 自分の GitHub ユーザー名を取得 (自分のコメントを除外するため)
MY_LOGIN=$(gh api user --jq '.login')

# レビュースレッドの状態を確認 (GraphQL)
# 注意: id (スレッド resolve 用) と databaseId (リアクション API 用) の両方を取得する
# <owner>, <repo>, <number> は実際の値に置き換える
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <number>) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          comments(first: 10) {
            nodes {
              databaseId
              body
              path
              line
              author { login }
            }
          }
        }
      }
    }
  }
}'
```

**フィルタ条件:**

取得した `reviewThreads.nodes` に対して以下の条件でフィルタする:

1. `isResolved == false` のスレッドのみを対象とする
2. スレッドの最初のコメント (`comments.nodes[0].author.login`) が `MY_LOGIN` と一致するスレッドは除外する (自分によるコメントには返信・resolve しない)

### 3. レビューコメントの分析とファクトチェック

各未解決コメントについて、以下を判断する:

| 判断カテゴリ   | 対応                   |
| -------------- | ---------------------- |
| **修正が必要** | コードを修正する       |
| **議論が必要** | ユーザーに確認を求める |
| **対応不要**   | 理由を説明して resolve |

**妥当性判断の基準:**

- コードの正確性に関する指摘 → 修正が必要
- セキュリティに関する指摘 → 修正が必要
- パフォーマンスに関する指摘 → 検討が必要
- スタイルや好みの問題 (`nits:`) → 対応は任意
- 誤解に基づく指摘 → 説明で対応

**ファクトチェック (必須):**

レビューの指摘を鵜呑みにせず、技術的な主張や根拠が正しいか検証する。特に以下のケースでは必ずファクトチェックを行う:

- 言語仕様・ランタイムの挙動に関する指摘
- フレームワーク・ライブラリの API や推奨パターンに関する指摘
- セキュリティに関する指摘
- パフォーマンスに関する指摘
- 「〜すべき」「〜は非推奨」など規範的な主張

**ファクトチェックのソース優先順位:**

| 優先度 | ソース                         | 用途                                                    |
| ------ | ------------------------------ | ------------------------------------------------------- |
| 1      | LSP                            | コードベース内の定義・参照・型情報の確認                |
| 2      | deepwiki MCP (`mcp` tool 経由) | OSS リポジトリの Wiki・ドキュメント                     |
| 3      | context7 MCP (`mcp` tool 経由) | ライブラリの公式ドキュメントとコード例                  |
| 4      | `ax` (bash 経由)               | 公式サイト・GitHub・リリースノート等の URL の直接取得 (`ax <url> --md` 等) |

**例外 (上記の優先順位より優先):**

- terraform に関する内容は terraform MCP が最優先 (`mcp` tool 経由、構成されている場合)
- Google Cloud に関する内容は google-developer-knowledge MCP が最優先 (`mcp` tool 経由、構成されている場合)

ファクトチェックの結果は修正計画の提示 (ステップ 4) に含め、指摘が誤りだった場合はその根拠をソース付きで示す。

### 4. 修正計画の提示

分析結果をユーザーに提示し、question tool で承認を求める:

**レビューコメントの翻訳:** 引用するレビューコメントが英語の場合は、日本語に翻訳して表示する。原文を併記する必要はなく、翻訳後の日本語のみを表示する。

```
## レビューコメント分析結果

### 修正が必要なコメント (N 件)

1. **[path/to/file.ts:42]** @reviewer
   > コメント内容

   **対応:** [修正内容の説明]

2. ...

### 議論が必要なコメント (M 件)

1. **[path/to/file.ts:100]** @reviewer
   > コメント内容

   **判断:** [なぜ議論が必要か]

### 対応不要と判断したコメント (K 件)

1. **[path/to/file.ts:200]** @reviewer
   > コメント内容

   **理由:** [対応不要の理由]

---

この計画で修正を進めますか？
```

### 5. コード修正の実行

ユーザーの承認後、修正を実行する:

1. 対象ファイルを read tool で読み込む
2. edit tool で修正を適用
3. 修正内容を確認

```bash
# 修正後の差分を確認
git diff
```

### 6. 変更のステージング

修正したファイルをステージングする:

```bash
git add -A
```

### 7. コミットメッセージの生成

**commit-proposer subagent を subagent tool で呼び出す。**

```
subagent({
  subagent_type: "commit-proposer",
  description: "コミットメッセージ候補の生成",
  prompt: "ステージング済みの変更に対してコミットメッセージ候補を提案してください。コンテキスト: レビュー指摘に基づく修正です。subject には「レビュー指摘に基づく修正」のような汎用的な表現ではなく、実際に何を変更したかを具体的に記述してください。"
})
```

subagent が変更差分の分析、commitlint 設定の確認、メッセージ候補の生成を実行する。

### 8. コミット前の承認確認

**必須:** 修正内容をユーザーに提示し、question tool でコミットの承認を求める。

**コミットメッセージは commitlint 設定 (または Conventional Commits) に準拠する:**

```
## コミット内容の確認

以下の変更をコミットします:

**変更ファイル:**
- path/to/file1.ts (+5, -3)
- path/to/file2.ts (+2, -1)

**コミットメッセージ:** (commitlint 設定: <設定ファイル or デフォルト>)
```

<type>(<scope>): <実際の変更内容を具体的に記述>

- [修正内容 1]
- [修正内容 2]

```

例: `fix(auth): トークン検証に null チェックを追加`、`refactor(api): エラーレスポンスの型定義を厳格化`

この内容でコミットしてよろしいですか？
```

**type の選択基準:**

- レビュー指摘でバグを修正 → `fix`
- レビュー指摘でリファクタリング → `refactor`
- レビュー指摘でスタイル修正 → `style`
- レビュー指摘でドキュメント修正 → `docs`

### 9. コミットの実行

承認後、コミットを実行:

```bash
# subject には実際の変更内容を記述する (「レビュー指摘に基づく修正」のような汎用表現は使わない)
git commit -m "<type>(<scope>): <実際の変更内容>

- [修正内容 1]
- [修正内容 2]"
```

### 10. プッシュの実行

コミット完了後、リモートにプッシュ:

```bash
git push
```

### 11. 返信コメントの作成

各レビューコメントへの返信を作成する。

**返信テンプレート:**

| 対応タイプ     | 返信例                                                                        |
| -------------- | ----------------------------------------------------------------------------- |
| 修正完了       | `修正しました。ご指摘ありがとうございます。`                                  |
| 議論結果で修正 | `ご指摘の通り修正しました。[補足説明]`                                        |
| 対応しない     | `[理由] のため、現状のままとさせてください。ご意見があればお知らせください。` |

**ソース参照ルール:**

理由を添えて返信する場合 (対応しない、議論結果で修正など)、信頼できるソースの情報を参照できるときはコメントにも記載する。

- 公式ドキュメント (言語仕様、フレームワーク公式ドキュメント、API ドキュメント等) の URL
- プロジェクト内の既存コード・設定ファイルのパスと行番号
- lint ルールやコーディング規約の該当セクション
- RFC やセキュリティアドバイザリ等の公的な技術文書

**例:**

```
現在の実装は React 公式ドキュメントの推奨パターンに沿っています。
ref: https://react.dev/reference/react/useEffect#removing-unnecessary-object-dependencies

現状のままとさせてください。ご意見があればお知らせください。
```

### 12. 返信・resolve の承認確認

**必須:** 返信内容と resolve 対象をユーザーに提示し、question tool で承認を求める:

```
## 返信コメントと resolve の確認

以下の返信を投稿し、スレッドを resolve します:

### 1. [path/to/file.ts:42] への返信 ✅ resolve 予定
> 元のコメント: ...

**返信:** 修正しました。ご指摘ありがとうございます。

### 2. [path/to/file.ts:100] への返信 ✅ resolve 予定
> 元のコメント: ...

**返信:** [理由] のため、現状のままとさせてください。

---

**resolve 対象:** N 件 (修正: X 件、対応不要: Y 件)

これらの返信を投稿し、スレッドを resolve してよろしいですか？
- resolve しない場合は「返信のみ」と回答してください
```

**resolve 対象の判定基準:**

| 対応タイプ             | resolve 対象 |
| ---------------------- | ------------ |
| 修正が完了したコメント | ✅           |
| 対応不要と判断         | ✅           |
| 議論継続中             | ❌           |

### 13. 返信の投稿・スレッド resolve

承認後、リアクション追加・返信投稿・スレッド resolve を実行する。

**返信は GraphQL mutation を使用する** (REST API はエンドポイントの URL 構造が複雑でエラーを起こしやすいため):

```bash
# 元のコメントに 👍 リアクションを追加 (REST API)
# databaseId はステップ 2 の GraphQL クエリで取得した値を使用
gh api repos/{owner}/{repo}/pulls/comments/<databaseId>/reactions \
  -f content="+1"

# レビュースレッドへの返信 (GraphQL mutation)
# thread_id はステップ 2 で取得した reviewThreads の id を使用
# <thread_id>, <body> は実際の値に置き換える
gh api graphql -F query='
mutation {
  addPullRequestReviewThreadReply(input: {pullRequestReviewThreadId: "<thread_id>", body: "<body>"}) {
    comment {
      id
      body
    }
  }
}'
```

**ユーザーが resolve を承認した場合:**

```bash
# スレッドを resolve (GraphQL mutation)
# <thread_id> は実際の値に置き換える
gh api graphql -F query='
mutation {
  resolveReviewThread(input: {threadId: "<thread_id>"}) {
    thread {
      isResolved
    }
  }
}'
```

**処理順序:**

1. 元のコメントに 👍 リアクションを追加 (`databaseId` を使用)
2. スレッドに返信を投稿 (`id` (GraphQL node ID) を使用)
3. resolve を実行 (承認された場合のみ、`id` を使用)
4. エラーが発生した場合は続行し、完了報告で失敗したスレッドを報告

### 14. レビュー再リクエスト

返信・resolve が完了した後、対応したレビューコメントの投稿者に対してレビューの再リクエストを送信する。

**手順:**

1. ステップ 13 で返信・resolve したスレッドの投稿者 (最初のコメントの `author.login`) を重複なしで収集する
2. PR のレビュー一覧を取得し、再リクエスト対象の判定に必要な情報 (ユーザー種別・レビュー状態) を収集する:

   ```bash
   gh api repos/{owner}/{repo}/pulls/<number>/reviews \
     --jq '[.[] | {login: .user.login, type: .user.type, state: .state}]'
   ```

3. 取得したレビュー情報をもとに、以下の条件で再リクエスト対象を判定する:

   | 条件                                            | 再リクエスト |
   | ----------------------------------------------- | ------------ |
   | `user.type` が `Bot` (bot アカウント)           | スキップ     |
   | 同一ユーザーの最新レビューが `APPROVED`         | スキップ     |
   | 上記に該当しない (人間のレビュワーで未 approve) | **送信**     |

   **approve 判定:** 同一ユーザーが複数回レビューしている場合、最新のレビュー状態で判断する (古い approve の後に `CHANGES_REQUESTED` していればスキップしない)。

4. 対象ユーザーがいる場合、再リクエストを送信する:

   ```bash
   # <login1>, <login2> は対象ユーザーの login に置き換える
   gh api repos/{owner}/{repo}/pulls/<number>/requested_reviewers \
     -f "reviewers[]=<login1>" -f "reviewers[]=<login2>"
   ```

5. エラーが発生した場合は続行し、完了報告で失敗したユーザーを報告する

### 15. 完了報告

```
## 修正完了

- 修正コミット: <commit_hash>
- 修正ファイル数: N
- 返信済みコメント数: M
- resolve 済みスレッド数: K
- レビュー再リクエスト: L 人 (対象: @user1, @user2)

PR URL: <url>
```

**一部エラーがある場合:**

```
## 修正完了 (一部エラーあり)

- 修正コミット: <commit_hash>
- 修正ファイル数: N
- 返信済みコメント数: M
- resolve 済みスレッド数: K
- レビュー再リクエスト: L 人

**エラー:**
- resolve 失敗: [path/to/file.ts:42]: エラー内容
- 再リクエスト失敗: @user3: エラー内容

PR URL: <url>
```

## エラーハンドリング

### gh CLI が使用できない場合

GitHub MCP server が構成されていれば `mcp` tool 経由でフォールバックする (PR 情報取得、コメント取得)。構成されていない場合はユーザーに gh CLI のセットアップを案内する。

### 未解決コメントがない場合

```
未解決のレビューコメントはありません。
```

### コンフリクトがある場合

```bash
git fetch origin
git rebase origin/main
# コンフリクト解決後
git push --force-with-lease
```
