---
name: wt
description: 既存の Git リポジトリを bare + worktree の並列構成に変換する。引数のディレクトリを bare.git + ブランチディレクトリ構成にする。Use when worktree 構成への変換、bare リポジトリ化を求められた際に使用する。
disable-model-invocation: true
---

# Worktree 構成への変換ワークフロー

引数に指定されたディレクトリ内の Git リポジトリを、bare リポジトリ + worktree の並列構成に変換する。以下、`<directory>` は引数で指定されたディレクトリパスを指す。

## 重要な原則

1. **変換対象は既存の Git リポジトリのみ** - `.git` ディレクトリが存在するディレクトリのみ変換可能
2. **現在のブランチのみ worktree として作成する** - 他のブランチは変換後に手動で追加
3. **元のファイルは全てブランチディレクトリに移動する** - 未コミットの作業ツリー上の変更はコピーで引き継がれるが、ステージ済み (index) の状態は引き継がれないため、必要であれば事前にコミット/スタッシュしておく
4. **変換は一括実行する** - 途中失敗時の中途半端な状態を防ぐ
5. **`bare.git/config` にカスタム設定を追加する** - `[wt]` セクションを付与

## 変換後の構成

```
<directory>/
├── bare.git/       # bare リポジトリ
└── <branch>/       # 現在のブランチの worktree (元のファイルを含む)
```

例: `hoge/` が `main` ブランチの場合:

```
hoge/
├── bare.git/
└── main/
```

## 作業開始前の準備

**必須:** 作業開始前に todo tool の `list` で残存タスクを確認し、存在する場合は `clear` で全て削除する。その後、`add` で以下のステップをタスクとして登録する:

```
todo({ action: "add", text: "対象ディレクトリの検証: Git リポジトリであること、未変換であることを確認" })
todo({ action: "add", text: "現在のブランチ名を取得: branch --show-current でブランチ名を取得" })
todo({ action: "add", text: "未コミットの変更を確認: 変更がある場合はユーザーに警告" })
todo({ action: "add", text: "worktree 構成への変換: bare.git 作成、ファイル退避、worktree 追加、ファイル復元を一括実行" })
todo({ action: "add", text: "結果の確認: worktree list とディレクトリ構成を確認" })
todo({ action: "add", text: "完了報告: 変換結果を報告" })
```

各ステップの完了時に `toggle` (対象の id を指定) で完了済みに更新する。

## 実行手順

### 1. 対象ディレクトリの検証

`<directory>` は引数で指定された実際のパスに置き換えて実行する:

```bash
# Git リポジトリであることを確認
git -C "<directory>" rev-parse --git-dir

# 既に bare リポジトリでないことを確認
git -C "<directory>" rev-parse --is-bare-repository

# .git がディレクトリ (通常リポジトリ) であることを確認
# .git がファイルの場合は既に worktree の可能性がある
test -d "<directory>/.git"
```

- Git リポジトリでない場合 → エラーを報告して終了
- 既に bare リポジトリの場合 → 「既に bare リポジトリです」と報告して終了
- `.git` がファイルの場合 → 「既に worktree 構成の可能性があります」と報告して終了

### 2. 現在のブランチ名を取得

```bash
BRANCH=$(git -C "<directory>" branch --show-current)
```

detached HEAD (空文字列が返る) の場合はエラーを報告して終了する。

### 3. 未コミットの変更を確認

```bash
git -C "<directory>" status --porcelain
```

未コミットの変更がある場合、ユーザーに警告する。作業ツリー上の変更は worktree ディレクトリに引き継がれるが、ステージ済み (index) の状態は失われる。ステージ済み変更がある場合は事前にコミットまたはスタッシュを推奨する。

### 4. worktree 構成への変換

以下の手順を **1 つの bash コマンド** (`set -euo pipefail` 付き) で実行する。途中で失敗した場合に中途半端な状態にならないよう、一括で実行すること。

`<実際のディレクトリパス>` は引数で指定された値に置換して使用する。

```bash
set -euo pipefail

DIR="<実際のディレクトリパス>"
BRANCH=$(git -C "$DIR" branch --show-current)

# BRANCH が空の場合 (detached HEAD) はエラー終了
if [ -z "$BRANCH" ]; then
  echo "Error: detached HEAD 状態では変換できません" >&2
  exit 1
fi

# 事前検証
if [ "$(git -C "$DIR" rev-parse --is-bare-repository 2>/dev/null)" = "true" ]; then
  echo "Error: 既に bare リポジトリです" >&2
  exit 1
fi
if [ ! -d "$DIR/.git" ]; then
  echo "Error: .git ディレクトリが見つかりません (既に worktree 構成の可能性があります)" >&2
  exit 1
fi
if [ -e "$DIR/bare.git" ]; then
  echo "Error: bare.git が既に存在します" >&2
  exit 1
fi
if [ "$BRANCH" = "bare.git" ]; then
  echo "Error: ブランチ名 'bare.git' は bare リポジトリのディレクトリ名と衝突します" >&2
  exit 1
fi
if [ -f "$DIR/.gitmodules" ]; then
  echo "Error: submodule を含むリポジトリは非対応です" >&2
  exit 1
fi

# 変換前の remote 設定を記録
REMOTE_BEFORE=$(git -C "$DIR" remote -v)

# 未コミットの削除 (staged / unstaged) を記録 (.git 移動前に取得する必要がある)。
# worktree add が HEAD 版のファイルを再作成し、退避ファイルの上書きコピーでは
# 削除が反映されないため、変換後に再適用する
DELETED_PATHS=$(git -C "$DIR" status --porcelain | awk 'substr($0,1,1)=="D" || substr($0,2,1)=="D" {print substr($0,4)}')

# 復旧用 trap を .git 移動前に設定 (WORK_TMPDIR は後で設定されるため条件付き)
WORK_TMPDIR=""
trap '
  echo "Error: 変換に失敗しました。復旧を試みます..." >&2
  # 1. worktree ディレクトリを先に削除 (ファイル復元時の衝突を防ぐ)
  if [ -d "$DIR/$BRANCH" ]; then
    rm -rf "$DIR/$BRANCH" 2>/dev/null || true
    echo "worktree ディレクトリを削除しました" >&2
  fi
  # 2. bare 変換を元に戻す
  if [ -d "$DIR/bare.git" ] && [ ! -d "$DIR/.git" ]; then
    git -C "$DIR/bare.git" worktree prune 2>/dev/null || true
    mv "$DIR/bare.git" "$DIR/.git" 2>/dev/null || true
    git -C "$DIR" config core.bare false 2>/dev/null || true
    git -C "$DIR" config --remove-section wt 2>/dev/null || true
    echo ".git を復元しました" >&2
  fi
  # 3. 退避ファイルを復元
  if [ -n "$WORK_TMPDIR" ] && [ -d "$WORK_TMPDIR" ]; then
    if [ "$(ls -A "$WORK_TMPDIR" 2>/dev/null)" ]; then
      find "$WORK_TMPDIR" -maxdepth 1 -mindepth 1 -exec mv {} "$DIR/" \; 2>/dev/null || true
      echo "退避ファイルを復元しました" >&2
    fi
    rm -rf "$WORK_TMPDIR" 2>/dev/null || true
  fi
' ERR

# .git を bare.git に変換
mv "$DIR/.git" "$DIR/bare.git"
git -C "$DIR/bare.git" config core.bare true

# bare.git/config にカスタム設定を追加 (キー単位で設定/上書き)
git -C "$DIR/bare.git" config wt.copyignored false
git -C "$DIR/bare.git" config wt.basedir ..

# 元のファイルを一時ディレクトリに退避 (bare.git 以外)
# 同一ファイルシステム上に作成し mv が rename で完了するようにする
WORK_TMPDIR=$(mktemp -d "$DIR/.wt-tmp-XXXXXXXX")

find "$DIR" -maxdepth 1 -mindepth 1 \
  ! -name "bare.git" ! -name "$(basename "$WORK_TMPDIR")" \
  -exec mv {} "$WORK_TMPDIR/" \;

# worktree を追加
git -C "$DIR/bare.git" worktree add "../$BRANCH" "$BRANCH"

# 退避したファイルを worktree にコピー (checkout 済みファイルを上書き)
cp -a "$WORK_TMPDIR/." "$DIR/$BRANCH/"

# 未コミットの削除を worktree に再適用 (worktree add が HEAD 版を再作成するため)
if [ -n "$DELETED_PATHS" ]; then
  while IFS= read -r p; do
    [ -n "$p" ] && rm -f "$DIR/$BRANCH/$p"
  done <<< "$DELETED_PATHS"
  echo "未コミットの削除 $(printf '%s\n' "$DELETED_PATHS" | wc -l | tr -d ' ') 件を再適用しました" >&2
fi

# 変換成功 — 復旧用 trap を解除
trap - ERR

# 一時ディレクトリを削除
rm -rf "$WORK_TMPDIR"

# remote 設定が変換前と同一であることを検証
REMOTE_AFTER=$(git -C "$DIR/bare.git" remote -v)
if [ "$REMOTE_BEFORE" != "$REMOTE_AFTER" ]; then
  echo "Warning: remote 設定が変換前と異なります" >&2
  echo "変換前: $REMOTE_BEFORE" >&2
  echo "変換後: $REMOTE_AFTER" >&2
fi
```

### 5. 結果の確認

```bash
# worktree 一覧を確認
git -C "$DIR/bare.git" worktree list

# remote 設定が変換前と変わっていないことを確認
git -C "$DIR/bare.git" remote -v

# ディレクトリ構成を確認
ls -la "$DIR"
```

remote 設定が変換前と異なる場合は、ユーザーに警告し修正を提案する。

### 6. 完了報告

```
## Worktree 構成への変換完了

- **ディレクトリ:** <directory>
- **bare リポジトリ:** <directory>/bare.git
- **worktree:** <directory>/<branch>

新しい worktree を追加するには:
git -C <directory>/bare.git worktree add ../<new-branch> <new-branch>
```

## エラーハンドリング

### Git リポジトリでない場合

```
指定されたディレクトリは Git リポジトリではありません。
```

### detached HEAD の場合

```
detached HEAD 状態では変換できません。ブランチをチェックアウトしてから再実行してください。
```

### 既に bare リポジトリ / worktree 構成の場合

```
既に bare リポジトリです。
```

または:

```
.git ディレクトリが見つかりません (既に worktree 構成の可能性があります)。
```

### 変換途中で失敗した場合

`set -e` により即座に停止する。`trap` (ERR のみ) により自動復旧を試みる: 退避ファイルの復元、`.git` の復元、worktree ディレクトリの削除。自動復旧が失敗した場合は、以下の手順で手動復旧する:

1. 一時ディレクトリ (`.wt-tmp-*`) にファイルが残っている場合 → ファイルを元のディレクトリに戻す
2. `bare.git` が存在し `.git` がない場合 → `mv "$DIR/bare.git" "$DIR/.git"` と `git config core.bare false` で元に戻す
3. worktree ディレクトリが残っている場合 → 削除する

## 注意事項

- ブランチ名に `/` を含む場合 (例: `feature/foo`)、worktree ディレクトリがネストされる (`<directory>/feature/foo/`)。`git worktree add` が中間ディレクトリを自動作成するため動作上の問題はないが、フラットな並列構成にはならない
- submodule を含むリポジトリは非対応。`.git/modules/` 配下の submodule 状態が worktree 構成と整合しなくなるため、変換前に submodule の有無を確認すること
