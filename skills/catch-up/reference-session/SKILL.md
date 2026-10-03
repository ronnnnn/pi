---
name: reference-session
description: ローカルリポジトリの path、ブランチ名、または自分の GitHub Pull Request に紐づく過去の pi セッションのログから、何を依頼し何を実施したかを抽出して経緯を把握し現在の作業に役立てる。Use when 過去セッションの参照、以前の作業内容の確認、PR やブランチの経緯把握を求められた際に使用する。引数はリポジトリ path、worktree path、ブランチ名、PR の URL・番号のいずれか (省略時は現在の作業ディレクトリ)。
---

# Reference Session

対象のブランチ・worktree で過去に行われた pi セッションのログを読み、依頼内容・実施内容・結論・未完了事項を抽出して現在のセッションに取り込む。

## 基本原則

- **読み取り専用**。セッションファイル (`*.jsonl`) の変更・削除・移動は一切行わない
- **他人の PR は探索しない**。PR が渡された場合は todo 登録やセッション探索より先に author を判定し、自分の PR だと確認できなければ何もせず終了する (同名ブランチの誤マッチを防ぐため)
- **全読みしない**。セッションファイルは数百 KB から数十 MB になるため、read tool で開かず jq で必要な要素だけを抽出する
- **コンテキストを汚さない**。抽出結果が大きい場合は worker subagent に蒸留を委譲し、要約だけを受け取る
- **ログにないことは補わない**。見つからなかったことを「存在しない」と断定せず、探索した範囲を明示して報告する

## 作業開始前の準備

引数が PR の場合は、先に手順 2 の author 判定だけを行う。自分の PR でない、または判定できない場合は todo を操作せずに終了する。

それ以外の場合は、todo tool の `list` で残存 todo を確認し、存在すれば全て `clear` で削除する。その後、`add` で次のステップを登録する。

```
todo({ action: "add", text: "引数の解釈: path / ブランチ名 / PR を判別し対象を特定" })
todo({ action: "add", text: "worktree の解決: ブランチ名や PR の headRefName から対象 path を特定" })
todo({ action: "add", text: "セッションディレクトリの解決: 対象 path をディレクトリ名に変換し存在を確認" })
todo({ action: "add", text: "セッションの選定と抽出: 直近のセッションから指示・要約・変更ファイル・最終報告を抽出" })
todo({ action: "add", text: "現在のセッションへの取り込み: 経緯を構造化して報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了状態へ更新する。セッションが見つからず途中で終了する場合は、残りの todo を `toggle` で完了にしてから報告する。

## pi セッションの保存形式

### 保存場所

セッションは cwd ごとに次のパスへ保存される。

```
~/.pi/agent/sessions/--<encoded-cwd>--/<timestamp>_<session-id>.jsonl
```

- `<encoded-cwd>` は cwd の先頭の path separator を 1 文字除去し、`/` `\` `:` を `-` に置換したもの。pi 本体の `getDefaultSessionDirPath()` と同じ規則
  - 例: `/Users/x/git/repo/feat-foo` → `--Users-x-git-repo-feat-foo--`
- 保存先は `PI_CODING_AGENT_SESSION_DIR` や settings の `sessionDir` で変更できるが、本 skill は既定の `~/.pi/agent/sessions/` を前提とする
- `<timestamp>_<session-id>/tasks/*.jsonl` は subagent のセッションなので対象外とし、セッションディレクトリ直下の `*.jsonl` のみを扱う
- pi は起動時にセッションディレクトリを作成するため、ディレクトリが存在しても `*.jsonl` が 0 件の場合がある
- ディレクトリ名は `--` で始まり、相対名で `ls` などに渡すとオプションと解釈される。常に絶対パスで扱う
- セッションは「pi を起動した cwd」に保存される。別の worktree (例: `main`) から PR を作成・監視した作業は、その worktree 側のセッションに残る

### JSONL の構造

各行は `type` フィールドを持つ JSON オブジェクトで、先頭行は `{"type":"session","version":3,"id":"...","timestamp":"...","cwd":"..."}` の header。本 skill で使う `type` は次のとおり。

| `type` | 内容 |
| :--- | :--- |
| `session` | header。`cwd` でセッションの作業ディレクトリを確認できる |
| `message` | `.message.role` が `user` / `assistant` / `toolResult` のメッセージ。assistant の `.message.content[]` には `text` / `toolCall` ブロックが含まれ、toolResult は `.message.toolCallId` と `.message.isError` を持つ |
| `compaction` | 長いセッションの要約。`.summary` に compaction 時点までの経緯がまとまっている |
| `context_edit` | 以前の entry (`targetId`) の内容を、以降のモデルのコンテキストでだけ差し替える。`replacement` が null なら除外、それ以外は置換後の内容。同じ entry への編集は系列上で最後のものが有効 |

- `/skill:<name>` で起動したユーザーメッセージには、SKILL.md 全文が `<skill name="...">...</skill>` として展開されている。抽出時は `[/skill:<name>]` に置換して圧縮する
- entry は `id` / `parentId` による木構造で、`/tree` で分岐したセッションには破棄された分岐も残る。pi は再開時にファイルの最後の entry を leaf とし、leaf から `parentId` をたどった系列を会話として使う。本 skill も手順 4-2 でこの系列だけを抽出し、破棄された分岐は `branch_summary` (分岐を離れた際の要約) で把握する
- header の `version` が 1 (または欠落) のセッションは、`id` / `parentId` を持たない直線的な形式。pi は読み込み時にファイルの順で `id` / `parentId` を振るため、本 skill もファイルの順のまま扱う

## 実行手順

### 1. 引数の解釈

引数は次の順で判別する。

| 判別順 | 引数の形 | 扱い |
| :--- | :--- | :--- |
| 1 | `https://github.com/<owner>/<repo>/pull/<n>` または `#<n>` | PR として手順 2 へ |
| 2 | 既存のディレクトリ (絶対 path / 相対 path) | `realpath` で正規化して対象 path とし、手順 3 へ |
| 3 | 数字のみ (`123`) | PR として手順 2 へ |
| 4 | 上記以外 | ブランチ名として worktree を解決する |
| - | 引数なし | 現在の作業ディレクトリ (`pwd -P`) を対象 path とし、手順 3 へ |

ブランチ名は次のように worktree を解決する。git が失敗した場合 (リポジトリ外での実行等) は、「一致する worktree なし」と区別して理由を報告し、終了する。

```bash
branch="<branch>"
worktrees=$(git worktree list --porcelain) || { echo "git worktree list に失敗"; exit 1; }
printf '%s\n' "$worktrees" | awk -v b="refs/heads/$branch" '/^worktree /{p=substr($0,10)} $0=="branch " b {print p}'
```

- 一致する worktree があれば、その path を対象 path として手順 3 へ進む
- 一致しなければ (worktree が削除済み等)、手順 2 の「worktree がない場合のフォールバック」で探索する

### 2. PR の場合の判定

**必ず todo 登録とセッション探索より先に author を判定する。** `#123` は `#` を除いた番号を変数に入れて渡す (shell に直接書くと `#` 以降がコメントになるため)。

```bash
ref="123"  # URL ならそのまま、#123 なら 123
pr=$(gh pr view "$ref" --json headRefName,author,url) || { echo "PR 情報の取得に失敗"; exit 1; }
me=$(gh api user --jq .login) || { echo "login の取得に失敗"; exit 1; }
printf '%s\n' "$pr" | jq -r --arg me "$me" '"head=\(.headRefName) author=\(.author.login) url=\(.url) mine=\(.author.login == $me)"'
```

次の 3 条件を全て満たした場合だけ、自分の PR として扱う。

- PR 情報と自分の login の取得がともに成功した
- `headRefName` / `author.login` / `$me` がいずれも空でも null でもない
- `author.login` が `$me` と一致する

- 条件を満たさない場合は、todo 操作・セッション探索・subagent 起動を一切せず、次のいずれか 1 行だけを報告して終了する
  - author が異なる場合: `PR <url> の author は <author> のため、セッション参照は行いません。`
  - 取得に失敗した場合: `PR <ref> の情報を取得できなかったため、セッション参照は行いません (<理由>)。`
- 自分の PR であれば、`headRefName` を手順 1 のブランチ名の解決と同じ方法で worktree と突き合わせ、一致した path を対象 path として手順 3 へ進む
- PR のリポジトリ (URL の `<owner>/<repo>`) が現在のリポジトリと異なる場合は、フォールバックで探索する。現在のリポジトリは `gh repo view --json nameWithOwner --jq .nameWithOwner` で確認し、その worktree とは突き合わせない

#### worktree がない場合のフォールバック

ブランチ名の `/` `\` `:` を `-` に置換した文字列で終わるセッションディレクトリを、次のように直接探索する。

```bash
encoded_branch=$(printf '%s' "$branch" | tr '/\\:' '---')
find ~/.pi/agent/sessions -mindepth 1 -maxdepth 1 -type d -name "*-${encoded_branch}--"
```

候補が見つかった場合は、直下の `*.jsonl` の header (`head -1 "$f" | jq -r .cwd`) を確認し、**対象リポジトリ**のものかを判断する。対象リポジトリとは、PR のリポジトリ、またはブランチ名を渡された時点の作業リポジトリのことで、skill を実行している現在のリポジトリとは限らない。

- cwd が対象リポジトリの worktree 群と同じ親ディレクトリ配下にある候補を優先する。ただし worktree は任意の場所に作れるため、親ディレクトリの一致は補助情報として扱う
- 対象リポジトリとの対応を裏付けられない候補は、1 件でも question tool で確認してから使う
- worktree のディレクトリ名をブランチ名と変える運用 (例: ブランチ `feat/review-body-detection` を `feat-review-body` に置く) もある。この場合、worktree を削除した後はフォールバックが当たらないため、その旨を報告に含める

#### PR URL による追加探索 (自分の PR のみ)

PR の作成・監視・修正を別の worktree (例: `main`) で起動した pi から行った場合、その作業はブランチの worktree ではなく起動元のセッションに残る。author を確認済みの PR に限り、対象リポジトリのセッションディレクトリ群から PR URL を含むファイルを追加の候補とする。

探索するセッションディレクトリは次の 2 つの和集合とする。

- `git worktree list --porcelain` に載っている各 worktree の path を encode したディレクトリ。通常の clone に `../main` などを linked worktree として追加した構成でも、起動元の worktree を拾える
- 共通 git ディレクトリの親を encode した prefix で始まるディレクトリ。bare + worktree 構成で、削除済みの worktree のセッションも拾える

```bash
set -o pipefail
url_re="github\.com/<owner>/<repo>/pull/<n>([^0-9]|$)"
worktrees=$(git worktree list --porcelain) || { echo "git worktree list に失敗"; exit 1; }
common_dir=$(git rev-parse --path-format=absolute --git-common-dir) || { echo "git rev-parse に失敗"; exit 1; }
repo_parent=$(dirname "$common_dir")
prefix="--$(printf '%s' "${repo_parent#/}" | tr '/\\:' '---')-"
# 現在のセッションは依頼文や gh の出力に PR URL を含むため、ここで除外する
url_candidates=$(
  {
    printf '%s\n' "$worktrees" | sed -n 's/^worktree //p' | while IFS= read -r p; do
      printf '%s\n' "$HOME/.pi/agent/sessions/--$(printf '%s' "${p#/}" | tr '/\\:' '---')--"
    done
    find "$HOME/.pi/agent/sessions" -mindepth 1 -maxdepth 1 -type d -name "${prefix}*"
  } | sort -u | while IFS= read -r d; do
    [ -d "$d" ] || continue
    find "$d" -mindepth 1 -maxdepth 1 -name '*.jsonl' -exec grep -lE "$url_re" {} +
  done | { grep -vxF "${PI_SESSION_FILE:-}" || true; }
)
printf '%s\n' "$url_candidates"
```

- `([^0-9]|$)` は `pull/4` が `pull/40` に誤一致するのを防ぐ
- path にスペースを含んでも分割されないよう、パスは 1 行 1 件で `while IFS= read -r` で読む
- PR が現在のリポジトリと異なる場合は、そのリポジトリのローカル clone を特定できるときだけ、clone 内で `repo_parent` を求めて実施する
- 出力されたパスを控え、手順 4-1 の選定に引き継ぐ。bash の呼び出しをまたぐと変数は消えるため、4-1 ではパスを `url_candidates` に代入し直す
- 追加候補はブランチの worktree のセッションと区別して扱い、報告で「PR URL を含む別 worktree のセッション」と明記する

### 3. セッションディレクトリの解決

対象 path を次のようにディレクトリ名に変換し、存在を確認する。

```bash
target="<対象 path>"
encoded="--$(printf '%s' "${target#/}" | tr '/\\:' '---')--"
session_dir="$HOME/.pi/agent/sessions/$encoded"
test -d "$session_dir" && find "$session_dir" -mindepth 1 -maxdepth 1 -name '*.jsonl' | head -1
```

- 直下の `*.jsonl` が 0 件 (ディレクトリ自体がない場合を含む) で、手順 2 の追加探索の候補もなければ終了する。報告は「`<対象 path>` に対応するセッションディレクトリは見つかりませんでした」とし、探索した path を添える
- `realpath` 後の path で見つからず、引数が symlink を含む path だった場合は、正規化前の絶対 path でも同じ変換を試す

### 4. セッションの選定と抽出

#### 4-1. セッションの選定

直下の `*.jsonl` を mtime 降順で列挙し、直近 3 件 (ユーザーが件数を指定した場合はその件数) を対象にする。手順 2 の追加探索で候補を得た場合は、それも別枠で同じ件数まで対象にする。選定はメインセッションで行い、現在のセッション (`$PI_SESSION_FILE`) を除外する。

```bash
session_dir="<手順 3 のセッションディレクトリ>"
url_candidates="<手順 2 の追加探索で得たパス (改行区切り。なければ空)>"

# ブランチの worktree のセッション (ディレクトリがなければ空)
ls -t "$session_dir"/*.jsonl 2>/dev/null | { grep -vxF "${PI_SESSION_FILE:-}" || true; } | head -3

# PR URL を含む別 worktree のセッション (上と重複するものを除き、mtime 降順)
# path のスペースで分割されないよう、NUL 区切りで ls に渡す
files=$(printf '%s\n' "$url_candidates" | { grep -vF "$session_dir/" || true; } | { grep -v '^$' || true; })
if [ -n "$files" ]; then printf '%s\n' "$files" | tr '\n' '\0' | xargs -0 ls -t | head -3; fi
```

- 両方を合わせて除外後に 0 件なら「参照可能な過去セッションなし」として正常に終了する
- `PI_SESSION_FILE` が未設定の場合は自己除外を保証できない。最新のファイルが現在のセッションでないか (header の `id` と timestamp) を確認する

各セッションのサイズと最終更新時刻は次のように把握しておく。

```bash
for f in <選定したファイル>; do
  printf '%s\t%s\t%s\n' "$(basename "$f")" "$(du -h "$f" | cut -f1)" "$(jq -r 'select(.type=="message") | .timestamp' "$f" | tail -1)"
done
```

#### 4-2. 抽出

各セッションから次の 4 要素を jq で抽出する。jq が失敗した場合 (書き込み途中の不完全な行等) は、その要素を「抽出失敗」として扱い、空の結果を正常結果とみなさない。

破棄された分岐の内容を結論と取り違えないよう、抽出は現在の系列に絞ってから行う。bash の呼び出しごとにシェルが変わるため、関数定義と `set -o pipefail` は各呼び出しに含める。

- `active`: 最後の entry から `parentId` をたどった系列だけを、ファイル順の JSONL として出力する (v1 はファイルの順のまま)
- `edited`: `active` の系列に `context_edit` を適用したもの。差し替え後の内容がモデルの見ていた依頼・報告なので、ユーザーの指示と最終報告に使う

`context_edit` が変えるのはモデルのコンテキストだけで、edit / write によるファイル変更は実際に起きている。そのため、変更ファイル、compaction、subagent の結果は `active` から抽出する。

```bash
set -o pipefail
f=<session>.jsonl

# 現在の系列 (pi が再開時に使う leaf から root までの経路) だけを出力する
active() {
  jq -cn '[inputs] as $all
    | (if $all[0].type == "session" then ($all[0].version // 1) else 1 end) as $v
    | [$all[] | select(.type != "session")] as $es
    | if $v < 2 then $es[]
      else reduce $es[] as $e ({m: {}, leaf: null}; .m[$e.id] = $e | .leaf = $e.id)
        | .m as $m | [.leaf | recurse($m[.].parentId; . != null) | $m[.]] | reverse[]
      end' "$1"
}

# active の系列に context_edit を適用する (null は除外、それ以外は内容を置換)
edited() {
  active "$1" | jq -cn '[inputs]
    | (reduce (.[] | select(.type == "context_edit")) as $c ({}; .[$c.targetId] = {r: $c.replacement})) as $ed
    | .[] | select(.type != "context_edit")
    | $ed[.id // ""] as $x
    | if $x == null then .
      elif $x.r == null then empty
      elif .type == "message" then
        .message.content = (if .message.role != "user" and ($x.r | type) == "string" then [{type: "text", text: $x.r}] else $x.r end)
      else .content = $x.r end'
}

# 分岐の有無 (1 以上なら /tree による分岐あり) と、破棄された分岐の要約
jq -rn '[inputs | select(.type != "session") | .parentId | select(. != null)] | group_by(.) | map(select(length > 1)) | length' "$f"
active "$f" | jq -r 'select(.type=="branch_summary") | .summary'

# ユーザーの指示 (何を依頼したか)。skill 展開は [/skill:<name>] に圧縮する
edited "$f" | jq -r 'select(.type=="message" and .message.role=="user")
  | .message.content
  | if type=="string" then . else (map(select(.type=="text") | .text) | join("\n")) end
  | gsub("<skill name=\"(?<n>[^\"]+)\"[^>]*>[\\s\\S]*?</skill>"; "[/skill:\(.n)]")'

# compaction summary (長いセッションの要約が保存されている場合がある)
active "$f" | jq -r 'select(.type=="compaction") | .summary'

# edit / write したファイル。toolResult の isError と突き合わせ、ok (1 回以上成功) / unknown (結果の記録なし) / failed (全て失敗) を付ける
active "$f" | jq -rn '
  reduce inputs as $e ({calls: [], result: {}};
    if $e.type == "message" and $e.message.role == "assistant" then
      .calls += [$e.message.content[]? | select(.type == "toolCall" and (.name == "edit" or .name == "write"))
                 | {id, path: (.arguments.path // .arguments.file_path)} | select(.path != null)]
    elif $e.type == "message" and $e.message.role == "toolResult" then
      .result[$e.message.toolCallId] = (if $e.message.isError == true then "failed" else "ok" end)
    else . end)
  | .result as $r
  | .calls | group_by(.path)[]
  | [.[] | $r[.id] // "unknown"] as $s
  | "\(if any($s[]; . == "ok") then "ok" elif any($s[]; . == "unknown") then "unknown" else "failed" end)\t\(.[0].path)"'

# 最終報告 (現在の系列で text を持つ最後の assistant メッセージ)
edited "$f" | jq -rn '
  reduce (inputs | select(.type == "message" and .message.role == "assistant")
          | {timestamp, text: ([.message.content[]? | select(.type == "text") | .text] | join("\n"))}
          | select(.text | test("\\S"))) as $m (null; $m)
  | if . == null then "(text を持つ assistant メッセージなし)" else "[\(.timestamp)]\n\(.text)" end'
```

抽出結果の扱いでは、次の点に注意する。

- 変更ファイル一覧は網羅的ではない。bash (`sed -i` や `git mv` 等) や MCP、subagent 経由の変更が含まれないため、必要なら `git log` で補う
- 分岐がある場合は、破棄された分岐の作業 (変更ファイルを含む) は抽出結果に含まれない。`branch_summary` がない分岐は内容を把握できないため、分岐があった事実だけを報告する
- 最終報告がセッションの結論とは限らない。作業途中で終わったセッションでは途中経過の報告になるため、報告ではその旨を明記する
- worker などに委譲した作業の結果は `toolResult` 側に残る。経緯の把握に必要な場合だけ、次の jq で subagent の結果を追加抽出する

```bash
active "$f" | jq -r 'select(.type=="message" and .message.role=="toolResult" and .message.toolName=="get_subagent_result")
  | [.message.content[]? | select(.type=="text") | .text] | join("\n")'
```

jq の実行自体は、`edited` による系列の抽出と編集の適用を含めても 60MB 程度のファイルなら約 2 秒で終わる。問題になるのは抽出結果の量なので、先に `| wc -c` で出力サイズを確認してから本文を取得する。

#### 4-3. worker subagent への委譲

次のいずれかに当てはまる場合は、抽出と蒸留を worker subagent に委譲し、メインセッションでは要約だけを受け取る。

- 対象セッションが 4 件以上
- 1 セッションあたりの抽出結果 (4 要素の合計) が 20KB を超える

複数セッションを委譲する場合は、セッションごとに `run_in_background: true` で並列起動し、`get_subagent_result({ agent_id, wait: true })` で回収する。

```
subagent({
  subagent_type: "worker",
  description: "セッションログの蒸留",
  run_in_background: true,
  prompt: <以下のブリーフ>
})
```

ブリーフには次の内容を含める。

- メインセッションで選定済みのセッションファイルの絶対パス (worker 側で選定し直させない)
- 手順 4-2 の jq コマンド一式と抽出結果の注意点。read tool で全読みしないこと、ファイルを変更しないことを明記する
- 現在のセッションの作業内容 (関連度の判断に使う)
- 返却フォーマットは手順 5 の「セッションごとの経緯」と同じ項目

### 5. 現在のセッションへの取り込み

抽出結果を次の形式で報告する。ログに書かれていない事実は推測で補わず、不明な点は「ログからは不明」と書く。

```markdown
## 参照したセッション

- 対象: <対象 path / ブランチ名 / PR URL>
- セッションディレクトリ: <session_dir>
- 参照件数: <n> 件 (全 <m> 件中、直近順)
- PR URL を含む別 worktree のセッション: <あればファイル名、なければ「なし」>

## セッションごとの経緯

### <timestamp> (<session-id の先頭 8 文字>)

- **目的**: <ユーザーの指示の要約>
- **実施内容**: <行ったこと。compaction summary があれば優先して使う>
- **変更されたファイル**: <ok のパス。多い場合はディレクトリ単位でまとめる。unknown は「結果不明」、failed は「失敗した変更」として分ける>
- **結論**: <最終報告の要点。途中で終わったセッションならその旨>
- **未完了の事項・既知の注意点**: <残課題、失敗したアプローチ、ユーザーが却下した方針など>
- **分岐**: </tree による分岐があれば、破棄された分岐の要約 (branch_summary)。なければ省略>

## 現在の作業との関連

- <過去の判断や注意点のうち、現在の作業に役立つもの>
- <同じ失敗を避けるためのポイント>
```

過去セッションの内容と現在のリポジトリの状態 (`git log`、ファイルの実体) が食い違う場合は、現在の状態を正とし、食い違いを報告に明記する。
