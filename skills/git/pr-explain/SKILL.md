---
name: pr-explain
description: PR やローカル変更の内容を包括的に収集・分析し、対応概要・背景・直感・実装の順で丁寧に解説する。引数なしの場合はローカル変更 → PR → 直近コミットの優先度で解説対象を決定する。--html 指定時は docs-html に委譲して HTML ドキュメントを生成する。Use when PR の解説、変更内容の説明、コミットの概要把握を求められた際に使用する。
---

# PR 解説ワークフロー

PR の変更内容を包括的に収集・分析し、対応概要・背景説明・直感・実装説明の順で丁寧に解説する。

## 重要な原則

1. **情報を包括的に収集する** - diff だけでなく、PR description、レビューコメント、作成者のコメント、コミット履歴すべてから情報を集める
2. **diff の外を読む** - 変更されていない周辺コードこそが背景説明の材料になる。呼び出し元、依存先、類似実装を能動的に探索する (Step 3)
3. **本質を先に、コードを後に** - 「コードがどう変わったか」の前に「何が本質的に変わったか」を具体例で示す (直感セクション)
4. **解説は常に日本語で行う** - 技術用語や固有名詞は原文のまま維持
5. **日本語の書式は japanese-text-style スキルに従う** - `../japanese-text-style/SKILL.md` を read して、スペース、句読点、括弧のルールを適用
6. **文体・図示は `references/writing-guide.md` に従う** - 読み物としての流れ、トイデータの作り方、図の指針
7. **理解しやすい順序で解説する** - コードの変更順ではなく、処理の流れやアーキテクチャの観点から説明
8. **レビュー議論は背景・実装に自然に統合する** - 独立セクションにせず、関連する文脈に織り込む

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "解説対象と出力形態の特定: 引数、ローカル変更、PR、直近コミットの優先度で対象を特定し、--html の有無を判定" })
todo({ action: "add", text: "PR 情報の収集: メタデータ、diff、レビューコメント、PR コメントを収集" })
todo({ action: "add", text: "背景の探索: 変更対象の呼び出し元・依存先・類似実装を探索し既存システムを把握" })
todo({ action: "add", text: "解説の作成: 対応概要・背景・直感・実装の 4 セクションで解説を執筆" })
todo({ action: "add", text: "出力: コマンドラインに出力、または --html 指定時は docs-html へ委譲" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. 解説対象と出力形態の特定

#### 出力形態の判定

引数に `--html` が含まれるかを最初に判定し、残りを解説対象の指定として扱う。

| 指定              | 出力形態                                                                          |
| ----------------- | --------------------------------------------------------------------------------- |
| なし (デフォルト) | コマンドラインに Markdown で出力する。ファイルは作成しない                        |
| `--html`          | 解説内容を docs-html skill (`/skill:docs-html`) に委譲し HTML ドキュメントを生成 |

出力形態は Step 4 の執筆内容にも影響する (図の指示コメントの有無)。Step 1 の時点で確定させる。

#### 解説対象の特定

引数が渡された場合、PR として特定する。URL 形式と番号形式の両方に対応する:

```bash
# URL 形式: https://<host>/owner/repo/pull/123
# host, owner, repo, number をすべて URL から抽出する。
# gh の --repo は [HOST/]OWNER/REPO 形式を受け付けるため、host も含めて渡す。
# host を落とすと gh はチェックアウトのホスト (または GH_HOST) で解決するため、
# GitHub Enterprise の PR を別ホストのクローンから解説したときに、
# 参照が失敗するか別ホストの同名リポジトリを掴んでしまう。
gh pr view <number> -R <host>/<owner>/<repo> --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'

# URL をそのまま渡す形でもよい (host の解決を gh に任せられる)
gh pr view <pr-url> --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'

# 番号形式: 123 または #123
# 現在のリポジトリの PR として扱う (この時点では host/owner/repo が未確定のため -R を付けない)
gh pr view <number> --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'

# host/owner/repo を取得 (番号形式の場合。以降の全 gh コマンドとサブクエリで使用する)
gh repo view --json owner,name,url --jq '"\(.url | sub("^https?://"; "") | split("/")[0])/\(.owner.login)/\(.name)"'
```

以降の本ドキュメントでは、この 3 つを合わせた値を `<host>/<owner>/<repo>` と表記する。GraphQL クエリや REST パスに埋め込む際は `<owner>` と `<repo>` のみを使う (ホストは `gh` の接続先設定で決まる)。

**確定した `<owner>/<repo>` は以降のすべての `gh pr` / `gh issue` 呼び出しで `-R` に渡す** (`gh api` は `-R` を受け付けないため、クエリまたはパスでリポジトリを指定する)。fork と upstream の両方をリモートに持つクローンでは、`-R` を省略した番号解決が Step 1 で選んだリポジトリとは別のリポジトリに当たりうる。特に fork 側に同番号の PR が存在すると、Step 3 のリビジョン比較が誤って「一致」と判定し、意図した upstream の PR ではなく現在のワークツリーを分析してしまう。

URL から owner/repo を抽出できない場合は、現在のリポジトリの owner/repo を使用する。

引数に PR 指定が含まれていない場合 (`--html` のみの場合を含む)、以下の優先度で解説対象を決定する:

1. **staged/unstaged の変更がある場合** → ローカル変更を解説対象とする

   ```bash
   git diff --stat
   git diff --cached --stat
   ```

   いずれかに変更があれば、`git diff` と `git diff --cached` の内容を解説する。この場合、Step 2 の PR 情報収集はスキップする。

2. **現在のブランチに PR が紐づいている場合** → その PR を解説対象とする

   ```bash
   gh pr view --json number,title,url,baseRefName,headRefName --jq '{number, title, url, baseRefName, headRefName}'
   ```

3. **上記いずれでもない場合** → 直近のコミットを解説対象とする

   ```bash
   git show HEAD
   ```

   この場合も Step 2 の PR 情報収集はスキップする。

**Step 2 をスキップした場合でも Step 3 の背景探索は必ず実行する。** PR description がない分、コードから背景を読み取る必要性はむしろ高い。

### 2. PR 情報の収集

**`gh pr` / `gh issue` サブコマンドには必ず `-R <host>/<owner>/<repo>` を渡す** (`--repo` は `[HOST/]OWNER/REPO` 形式を受け付ける。ホストを省くと `gh` がチェックアウトのホストや `GH_HOST` で解決してしまう)。 これらはリポジトリを省略すると現在のチェックアウトから解決するため、外部リポジトリの PR や、別リポジトリのクローン内で実行した場合に **無関係なリポジトリの情報を取得してしまう**。Step 1 で確定した `<owner>/<repo>` を必ず明示する。

**`gh api` は `-R` / `--repo` フラグを持たない** (`gh api <endpoint> [flags]`)。付けると unknown flag で失敗するため、リポジトリとホストは次のように指定する:

- リポジトリ — GraphQL: クエリ内の `repository(owner: "<owner>", name: "<repo>")` 引数 / REST: エンドポイントのパス (`repos/<owner>/<repo>/...`)
- ホスト — `--hostname <host>` フラグ。`<host>` が既定の接続先 (通常 `github.com`) と異なる場合、GitHub Enterprise では必ず指定する

以下の情報を **並列で** 収集する:

**PR メタデータと description:**

```bash
gh pr view <number> -R <host>/<owner>/<repo> --json title,body,author,baseRefName,headRefName,labels,additions,deletions,changedFiles,createdAt
```

**コミット履歴:**

```bash
gh pr view <number> -R <host>/<owner>/<repo> --json commits --jq '.commits[] | "\(.oid[0:7]) \(.messageHeadline)\n\(.messageBody)"'
```

**コード差分:**

```bash
# 変更ファイル一覧
gh pr diff <number> -R <host>/<owner>/<repo> --name-only

# 全差分
gh pr diff <number> -R <host>/<owner>/<repo>
```

**レビューコメント (インライン):**

```bash
# レビュースレッドのコメントを取得 (GraphQL で完全な議論を取得)
# <owner>, <repo>, <number> は実際の値に置き換える
gh api graphql -F query='
query {
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <number>) {
      reviews(first: 50) {
        nodes {
          body
          state
          author { login }
        }
      }
      reviewThreads(first: 100) {
        nodes {
          isResolved
          comments(first: 20) {
            nodes {
              body
              path
              line
              author { login }
              createdAt
            }
          }
        }
      }
    }
  }
}'
```

**PR コメント (一般):**

```bash
gh pr view <number> -R <host>/<owner>/<repo> --json comments --jq '.comments[] | "\(.author.login): \(.body)"'
```

**関連 Issue:** PR description に Issue 参照があれば取得する。参照の書き方は 3 種類あり、**それぞれ解決先のリポジトリが異なる**。番号だけを見て一律に PR のリポジトリへ問い合わせると、無関係な Issue を取得したり意図した背景を取りこぼしたりする:

| 参照の形式                          | 解決先                  | `-R` に渡す値            |
| ----------------------------------- | ----------------------- | ------------------------ |
| `#123` / `Closes #123`              | PR を所有するリポジトリ | `<owner>/<repo>`         |
| `other-owner/other-repo#123`        | 修飾子が指すリポジトリ  | `other-owner/other-repo` |
| `https://github.com/o/r/issues/123` | URL が指すリポジトリ    | URL をそのまま渡す       |

```bash
# 同一リポジトリの参照 (#123)
gh issue view <number> -R <host>/<owner>/<repo> --json title,body,labels

# クロスリポジトリ短縮形 (other-owner/other-repo#123) は修飾子をパースして -R に渡す
gh issue view <number> -R <other-host>/<other-owner>/<other-repo> --json title,body,labels

# フル URL はそのまま渡せる
gh issue view <issue-url> --json title,body,labels
```

`-R` を省略すると `gh` が現在のチェックアウトからリポジトリを解決するため、いずれの形式でも必ず解決先を明示する。

### 3. 背景の探索

**このステップが解説の質を決める。** diff は「何が変わったか」しか語らない。「何が変わったか」を意味のある情報にするには、変わっていない部分の理解が要る。

#### 探索リビジョンの整合 (必須の事前確認)

**read / grep / find による探索はワークスペースの現在の状態に対して動作する。** 解説対象が PR で、現在のチェックアウトがその head と異なる場合、`gh pr diff` は PR の内容を返す一方で探索は別リビジョンのコードを参照するため、**新規・リネームされたシンボルが見つからない / 参照先が誤ったリビジョンから返る**という不整合が起きる。しかもこれは静かに起きるため、誤った背景・影響範囲の分析がそのまま解説になる。

探索を始める前に必ずリビジョンの一致を確認する。**SHA の比較だけでは不十分で、未コミットの変更の有無も併せて見る**。SHA が一致していても作業ツリーが dirty なら、探索はその編集内容を読むため、クリーンな PR を記述する `gh pr diff` との間に同じ不整合が生じる:

```bash
# PR の head SHA と現在の HEAD を比較し、作業ツリーの汚れも確認する
PR_SHA=$(gh pr view <number> -R <host>/<owner>/<repo> --json headRefOid --jq '.headRefOid')
CUR_SHA=$(git rev-parse HEAD)
DIRTY=$(git status --porcelain)
```

| 状況                                             | 対応                                          |
| ------------------------------------------------ | --------------------------------------------- |
| `PR_SHA` == `CUR_SHA` かつ `DIRTY` が空          | そのまま現在のワークスペースを探索する        |
| `PR_SHA` == `CUR_SHA` だが未コミットの変更がある | 一時 worktree を作り、その中で探索する (下記) |
| SHA が異なる (別ブランチをチェックアウト中)      | 一時 worktree を作り、その中で探索する (下記) |
| ローカル変更 / 直近コミットが対象                | 現在のワークスペースが対象そのもの。確認不要  |
| 外部リポジトリ (ローカルに無い)                  | コードベース探索は不可。後述のフォールバックに従う |

**一時 worktree での探索:**

`pull/<number>/head` は **その PR を所有するリポジトリにしか存在しない ref** である。`origin` をハードコードすると、base remote を `upstream` と名付けているクローン、`origin` が無いクローン、`origin` がフォークを指すクローンでフェッチが解決できず、不要にフォールバックしてしまう。Step 1 で確定した `<owner>/<repo>` からフェッチ先を導出する:

リモートの照合は **ホストを含めて正規化したうえで完全一致**させる。判断を誤る例が 2 つある:

- パスの部分一致で探すと、対象が `acme/app` のときに `acme/app-fork` を拾ってしまう
- ホストを捨てて `owner/repo` だけで比べると、`gitlab.com/acme/app` が `github.com/acme/app` と等価になり、`pull/<number>/head` を持たないリモートを選んでフェッチが失敗する

そこで `<host>/<owner>/<repo>` の形に揃えて比較する。`<host>` は Step 1 の PR URL から取得する (GitHub Enterprise では `github.com` にならない):

```bash
# PR URL からホストを取り出す (例: https://github.com/acme/app/pull/1 -> github.com)
PR_HOST=$(gh pr view <number> -R <host>/<owner>/<repo> --json url --jq '.url' | sed -E 's#^https?://([^/]+)/.*#\1#')

# <host>/<owner>/<repo> を指すリモートを、URL を正規化したうえで完全一致で探す
REMOTE=""
for r in $(git remote); do
  # 以下をすべて host/owner/repo に正規化する:
  #   ssh://git@host:2222/owner/repo.git  (スキーム付き SSH。ポート付きも可)
  #   https://user@host/owner/repo.git    (HTTP(S)。user@ 付きも可)
  #   git://host/owner/repo.git           (その他スキーム)
  #   git@host:owner/repo.git             (SCP 形式)
  # スキーム -> user@ -> ホストのポート -> SCP 形式の : -> 末尾の / と .git
  # の順に落として host/owner/repo に揃える
  NORM=$(git remote get-url "$r" | sed -E 's#^[a-zA-Z][a-zA-Z0-9+.-]*://##; s#^[^@/]+@##; s#^([^/:]+):([0-9]+)/#\1/#; s#^([^/:]+):#\1/#; s#/*$##; s#\.git$##')
  if [ "$NORM" = "$PR_HOST/<owner>/<repo>" ]; then REMOTE="$r"; break; fi
done
FETCH_TARGET="${REMOTE:-https://$PR_HOST/<owner>/<repo>.git}"

# PR の head をフェッチする
git fetch "$FETCH_TARGET" "pull/<number>/head"

# フェッチした head が目的の PR のものか必ず照合する (リモート選択を誤っていた場合の最後の砦)
FETCHED_SHA=$(git rev-parse FETCH_HEAD)
if [ "$FETCHED_SHA" != "$PR_SHA" ]; then
  # 無関係な head を掴んでいる。worktree は作らずフォールバックへ
  echo "fetched head ($FETCHED_SHA) != PR head ($PR_SHA)" >&2
fi

# 照合できた場合のみ一時 worktree を作る
WORKTREE_DIR=$(mktemp -d)/pr-<number>
git worktree add --detach "$WORKTREE_DIR" "$PR_SHA"

# 探索は $WORKTREE_DIR 配下のファイルに対して行う

# 探索完了後に必ず後始末する
git worktree remove "$WORKTREE_DIR" --force
```

**削除を含む変更では base リビジョンの worktree も作る。** PR がファイルやシンボルを削除している場合、head 側の worktree には削除後のコードしかないため、**消えた定義そのものを読むことも、その定義を起点に呼び出し元を辿ることもできない**。「なぜ現設計だったか」「削除されたコードの呼び出し元がどう処理されたか」は base 側でしか調べられない:

`gh pr diff` に `--name-status` は無い (対応フラグは `--name-only` / `--patch` / `--web` / `--color` / `--exclude` など)。削除の有無は REST API の `files` エンドポイントで判定する:

```bash
# 削除されたファイルを列挙する (status は added / removed / modified / renamed など)
DELETED=$(gh api --hostname <host> repos/<owner>/<repo>/pulls/<number>/files --paginate --jq '.[] | select(.status == "removed") | .filename')

# $DELETED が空でない場合のみ、base リビジョンの worktree も作る
BASE_SHA=$(gh pr view <number> -R <host>/<owner>/<repo> --json baseRefOid --jq '.baseRefOid')
BASE_REF=$(gh pr view <number> -R <host>/<owner>/<repo> --json baseRefName --jq '.baseRefName')
git fetch "$FETCH_TARGET" "$BASE_REF"
BASE_WORKTREE_DIR=$(mktemp -d)/pr-<number>-base
git worktree add --detach "$BASE_WORKTREE_DIR" "$BASE_SHA"

# 削除されたシンボルの調査は $BASE_WORKTREE_DIR 側で行う
# 探索完了後に head 側と同様に後始末する
git worktree remove "$BASE_WORKTREE_DIR" --force
```

**`FETCH_HEAD` をそのまま worktree に渡さず、照合済みの `$PR_SHA` を渡す。** リモート選択やフェッチが意図とずれていた場合、`FETCH_HEAD` は無関係なコミットを指しうるが、`$PR_SHA` は Step 3 冒頭で PR から直接取得した値なので取り違えが起きない。

**フェッチした SHA の照合に失敗した場合、または worktree の作成に失敗した場合** (shallow clone、権限、ディスク等) は、**worktree での探索を諦めて現在のチェックアウトに対する read / grep 探索に切り替え、解説の「前提となる仕組み」に「現在のチェックアウトを基準に読んでいるため、PR head との差異がある可能性がある」と明示する**。誤った分析を黙って出さないことを優先する。

#### 探索対象

変更されたファイルを読むだけで終わらせず、builtin tools (read / grep / find) で以下を能動的に探索する。

| 探索対象                 | 手段                                    | 得られるもの                     |
| ------------------------ | --------------------------------------- | -------------------------------- |
| 変更対象の呼び出し元     | grep でシンボル名・関数名を検索         | 変更の影響範囲、利用文脈         |
| 変更対象の依存先         | read で import・参照先の定義を辿る      | 前提となるデータ構造・契約       |
| 型定義・インターフェース | grep で定義を検索し read で読む         | ドメインモデルの形               |
| 同ディレクトリの類似実装 | find / read                             | 設計パターン、慣習               |
| 変更対象のテスト         | find (`*test*`, `*spec*`)               | 期待される振る舞い、エッジケース |

探索範囲が広い場合や、呼び出し元の追跡・類似実装の調査など独立した問いを並行して調べたい場合は、Explore subagent (`subagent({ subagent_type: "Explore", ... })`) に委譲する。委譲時は探索対象のディレクトリ (worktree を作った場合はそのパス) と、答えてほしい問いを明示する。

#### 探索の停止条件

以下の 3 つの問いに答えられるようになったら探索を止める。

1. この変更対象は**何を担うモジュールか**。システム全体のどこに位置するか
2. **なぜ現在の設計になっていたか**。変更前の実装が解こうとしていた問題は何か
3. この変更で**何が壊れうるか**。呼び出し元にどんな影響が及ぶか

答えられない問いが残っている間は探索を続ける。逆に、3 つに答えられたなら深追いしない。

#### ファイル種別ごとの追加確認

| 状況             | 対応                                                                                 |
| ---------------- | ------------------------------------------------------------------------------------ |
| 新規ファイル追加 | 周辺の類似ファイルを読み、命名・構造の慣習を把握                                     |
| 既存ファイル変更 | 変更箇所の前後に加え、ファイル全体の責務を把握                                       |
| 型定義の変更     | grep で全使用箇所を確認                                                              |
| 設定変更         | その設定を読み込むコードまで辿る                                                     |
| 削除             | base リビジョンの worktree で、削除されたコードの呼び出し元がどう処理されたかを確認 |

### 4. 解説の作成

収集した情報を以下の 4 セクション構造で執筆する。**文体・トイデータ・図示の詳細は `references/writing-guide.md` を参照し、その品質チェックリストを満たすこと。**

```markdown
## 対応概要

[変更内容を 2-3 文で簡潔に説明する。何が変わったのかを端的に伝える。]

## 背景説明

### 前提となる仕組み

[この領域に馴染みがあれば読み飛ばしてよい旨を明記した上で、Step 3 で把握した既存システムを説明する。
diff に映っていないコードの説明が中心になる。変更対象の責務、呼び出し関係、現在の設計の理由。]

### この変更の動機

[なぜこの変更が必要だったのか。以下から構成する:]

- PR の description に記載された動機や課題
- 関連する Issue の内容
- レビューでの議論から明らかになった設計判断の理由
- コミットメッセージに含まれる意図の説明

## 直感

[一言でいえば何が変わったのかを 1 文で示す。
続けて、実在するフィールド名を使ったトイデータで Before/After を対比する。
最後にその変化の含意 (責務の移動、不変条件の追加、性能特性の変化など) を述べる。]

## 実装説明

[変更内容を処理の流れが理解しやすい順序で丁寧に解説する:]

- コードの変更順 (diff 順) ではなく、論理的な理解の順序で説明する
- 新しい概念やデータ構造を先に説明してから、それを使う処理を説明する
- ファイルパスと行番号を含めて具体的に参照する
- レビューで議論になった設計判断は、該当箇所の説明に自然に織り込む
```

#### 実装説明の順序決定方法

1. アーキテクチャレベルの変更 (新しいモジュール、ディレクトリ構成の変更) → 先に説明
2. データモデル・型定義の変更 → 次に説明
3. コアロジックの変更 → 処理の流れに沿って説明
4. UI・表示の変更 → 最後に説明
5. 設定・インフラの変更 → 関連する箇所で説明

#### 大規模な差分 (目安: 変更ファイル 30 以上) の場合

コンテキストウィンドウの制約上、全ファイルの diff を一度に処理できないため、以下の戦略を取る:

- 全ファイルの diff を一度に読み込まず、変更ファイル一覧から主要な変更を特定する
- ディレクトリ単位で変更の傾向をまとめ、重要なファイルに絞って詳細解説する
- 自動生成やリネームのみの変更はまとめて言及し、個別解説は省略する

**ただし直感セクションは差分規模に関わらず必ず書く。** 差分が大きいほど本質の要約が読者にとって重要になる。個々のファイルを説明しきれない場合でも、「この PR 全体で何が変わったか」は 1 つの具体例で示せる。

#### `--html` 指定時の追加作業

図があると理解が進む箇所に、HTML コメントで図の指示を残す。

```markdown
<!-- FIGURE: <図の種類>。<登場要素>。<Before/After の違い>。<流れる例示データ> -->
```

指示には **図の種類・登場要素・例示データ** を必ず含める。書き方は `references/writing-guide.md` の「図示の指針」を参照。コマンドライン出力時はこのコメントを出力しない。

### 5. 出力

#### デフォルト (コマンドライン)

Step 4 で作成した解説をそのままコマンドラインに出力して終了する。ファイルは作成しない。

出力の末尾に以下の案内を添える:

```markdown
> 図解付きの HTML ドキュメントが必要な場合は `--html` を付けて再実行してください。
```

#### `--html` 指定時

1. 解説本文を write tool で Markdown として書き出す。書き出し先のディレクトリが不明な場合は `mktemp -d` で作成する

   ファイル名は `pr-explain-<slug>.md` とし、`<slug>` は **必ずサニタイズしてから使う**。ブランチ名には `/` が含まれうるため (`feature/foo` 等)、そのまま埋めると存在しない中間ディレクトリを指すパスになり書き込みに失敗する:

   ```bash
   # PR 番号がある場合はそれを、無ければブランチ名、それも無ければ "local"
   RAW="${PR_NUMBER:-$(git branch --show-current 2>/dev/null || echo local)}"
   # パス区切りと不正な文字を - に潰し、先頭末尾の - を削る
   SLUG=$(printf '%s' "$RAW" | tr '/' '-' | tr -c '[:alnum:]._-' '-' | sed 's/^-*//; s/-*$//')
   SLUG=${SLUG:-local}
   MD_PATH="$SCRATCH_DIR/pr-explain-${SLUG}.md"
   ```

2. `../../dev/docs-html/SKILL.md` (この skill のディレクトリからの相対パス) を read し、その手順に従って HTML を生成する。書き出した md のパスを入力として渡し、以下の指示を添える:

   ```
   <md のパス> (PR 解説。explainer カテゴリで、review のコンポーネント (差分ビュー、行注釈) を組み合わせて生成してください。md 内の <!-- FIGURE: --> コメントは SVG/HTML 図として具現化してください)
   ```

3. docs-html の手順に従い、出力先の確認・カテゴリ判定・HTML 生成を行う。出力先の決定は docs-html の手順に任せる
4. 生成完了後、HTML のパスと解説の主要セクションを報告する

**docs-html の手順を経ずに HTML を自前で生成しない。** docs-html はテーマ切替・多言語切替・レスポンシブの実装パターンを持っており、それを再実装すると品質基準を満たせない中途半端な出力になる。

## エラーハンドリング

### docs-html が利用できない場合

docs-html skill が未導入などで `../../dev/docs-html/SKILL.md` の read が失敗した場合、コマンドライン出力にフォールバックし、その旨を報告する:

```
docs-html skill を読み込めなかったため、コマンドラインに解説を出力しました。
HTML 出力を利用するには docs-html skill を導入してください。
```

### gh CLI が使用できない場合

`gh api graphql` でメインワークフローと同じ GraphQL クエリを実行する:

```bash
# ステップ 2 の GraphQL クエリを gh api graphql で実行
# <owner>, <repo>, <number> は実際の値に置き換える
gh api graphql -F query='...'

# PR メタデータは REST API で取得
gh api repos/<owner>/<repo>/pulls/<number>
```

### 外部リポジトリの PR の場合

URL から owner/repo を抽出し、`gh api` で情報を取得する:

```bash
# リポジトリをクローンせずに PR 情報を取得
gh api repos/<owner>/<repo>/pulls/<number>
gh api repos/<owner>/<repo>/pulls/<number>/comments
gh api repos/<owner>/<repo>/pulls/<number>/reviews
```

外部リポジトリはローカルにコードがないため Step 3 のコードベース探索ができない。この場合は diff と PR description から読み取れる範囲で背景を構成し、「前提となる仕組み」の説明が推測を含むことを明示する。

### PR が見つからない場合

```
指定された PR が見つかりません。URL または PR 番号を確認してください。
例: /skill:pr-explain https://github.com/owner/repo/pull/123
```

## Additional Resources

- **`references/writing-guide.md`** - 文体の原則、背景説明の 2 層構造、直感セクションの書き方 (トイデータの粒度)、図示の指針、レビュー議論の織り込み方、出力前の品質チェックリスト
