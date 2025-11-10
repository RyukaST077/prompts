# Diff → Markdown レポート

目的：
$1（省略時 origin/main）を基準、$2（省略時 HEAD）を比較対象として、
三点リーダ（$BASE...$HEAD）の差分を取り、読みやすい Markdown を $3（省略時 DIFF.md）に生成する。
必要ならパス指定 $4 を付けて差分対象を絞る。

手順（エージェントが実行すること）：
- BASE="${1:-origin/main}"; HEAD="${2:-HEAD}"; OUT="${3:-DIFF.md}"; PATHSPEC="${4}"
- git fetch --all --prune || true
- MB=$(git merge-base "$BASE" "$HEAD")
- SHORTSTAT=$(git diff --shortstat "$BASE...$HEAD" -- ${PATHSPEC:+-- "$PATHSPEC"})
- NUMSTAT=$(git diff --numstat "$BASE...$HEAD" -- ${PATHSPEC:+-- "$PATHSPEC"})
- COMMITS=$(git log --no-decorate --oneline "$BASE..$HEAD" -- ${PATHSPEC:+-- "$PATHSPEC"} || true)

- Markdown OUT を以下の章立てで生成（日本語で簡潔に）：
  # Diff Report: $BASE...$HEAD
  - Generated at: <ISO8601>
  - Merge base: $MB
  ## 概要
  - 変更点の要約（5行以内）
  ## 変更サマリ
  - $SHORTSTAT
  - 変更ファイル一覧（NUMSTAT を表に整形：ファイル/追加/削除）
  ## コミット一覧
  - $COMMITS
  ## ファイル別差分
  - 変更ごとに <details> 折りたたみ。コードフェンス言語は拡張子から推定。
    各項目は `git diff -U3 --find-renames "$BASE...$HEAD" -- <file>` の該当 hunk を掲載。
    バイナリは「(binary)」と明記しパッチは省略。

- 差分が無ければ "No changes." とだけ書いて保存。
- 完了後、OUT の絶対パスを出力し、Markdown の体裁崩れがあれば自動で整形して保存し直す。
