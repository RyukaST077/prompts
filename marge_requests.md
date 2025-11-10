あなたはローカルのシェルと git にアクセスできるエンジニア用エージェントです。
目的：三点リーダ（merge-base 起点）の差分から **要約付き DIFF.md** を生成し、その内容を説明として GitLab の MR を作成する。
glab があれば glab を使用し、無ければ git の push オプション、どちらも使えない場合は REST API を利用する。

# ====== ユーザー入力（必要に応じて変更）======
HEADREF="HEAD"            # 例: HEAD / feature/xyz / origin/feature/xyz
BASE="origin/main"        # 取り込み先（origin/付き推奨）
PATHSPEC=""               # 差分対象の絞り込み（例: "", "docs/", "src/**/*.ts"）
OUT="DIFF.md"             # 出力 Markdown
TITLE="chore: open MR with DIFF.md"  # MR タイトル
DRAFT="true"              # "true" でドラフト MR
LABELS=""                 # "frontend,ready" など（glab 時のみ）
REVIEWERS=""              # "alice,bob"（glab 時のみ）
ASSIGNEES=""              # "alice"（glab 時のみ）
REMOVE_SOURCE="false"     # "true" でマージ後に source ブランチ削除

# ====== ここから自動計算・安全策 ======
set -euo pipefail
: "${HEADREF:=HEAD}"
: "${BASE:=origin/main}"
: "${OUT:=DIFF.md}"
: "${TITLE:=chore: open MR with DIFF.md}"
: "${DRAFT:=true}"
: "${REMOVE_SOURCE:=false}"

command -v git >/dev/null || { echo "git が見つかりません"; exit 1; }
git fetch --all --prune || true

# HEAD ブランチ名を決定
if [ "$HEADREF" = "HEAD" ]; then
  CUR=$(git rev-parse --abbrev-ref HEAD)
  if [ "$CUR" = "HEAD" ]; then
    NEWBR="mr/$(date +%Y%m%d-%H%M%S)-$(git rev-parse --short HEAD)"
    git checkout -b "$NEWBR"
    HEADBR="$NEWBR"
  else
    HEADBR="$CUR"
  fi
else
  HEADBR="${HEADREF##origin/}"
  git rev-parse --verify "$HEADBR" >/dev/null 2>&1 || git checkout -b "$HEADBR" "origin/$HEADBR" || git checkout "$HEADBR"
fi

TARGETBR="${BASE##origin/}"      # MR で使う target は origin を外した名前

# ====== 差分データ収集 ======
MB=$(git merge-base "$BASE" "$HEADBR")
SHORTSTAT=$(git diff --shortstat "$BASE...$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)
NUMSTAT=$(git diff --numstat "$BASE...$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)
NAMESTATUS=$(git diff --name-status "$BASE...$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)
CHFILES=$(git diff --name-only "$BASE...$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)
COMMITS=$(git log --no-decorate --oneline "$BASE..$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)
SUMMARY=$(git diff --summary "$BASE...$HEADBR" -- ${PATHSPEC:+"--" "$PATHSPEC"} || true)

# 追加の集計（要約の材料）
FILES_CHANGED=$(printf "%s\n" "$NAMESTATUS" | sed '/^$/d' | wc -l | tr -d ' ')
INSERTIONS=$(printf "%s\n" "$SHORTSTAT" | sed -n 's/.* \([0-9][0-9]*\) insertions*.*/\1/p' | tail -n1)
DELETIONS=$(printf "%s\n" "$SHORTSTAT" | sed -n 's/.* \([0-9][0-9]*\) deletions*.*/\1/p' | tail -n1)
: "${INSERTIONS:=0}"; : "${DELETIONS:=0}"
TOTAL_CHURN=$(( INSERTIONS + DELETIONS ))

RENAMES=$(printf "%s\n" "$SUMMARY" | grep -c '^ rename ' || true)
DELETEDS=$(printf "%s\n" "$NAMESTATUS" | awk '$1=="D"{print}' | wc -l | tr -d ' ')
ADDEDS=$(printf "%s\n" "$NAMESTATUS" | awk '$1=="A"{print}' | wc -l | tr -d ' ')

# 拡張子別の件数トップ5
EXT_STATS=$(printf "%s\n" "$CHFILES" | awk -F. '
  NF>1 {ext=tolower($NF)} NF==1 {ext="(noext)"} {c[ext]++}
  END{for(k in c) printf "%s %d\n",k,c[k]}' | sort -k2,2nr | head -5)

# ルート直下ディレクトリ別トップ5
DIR_STATS=$(printf "%s\n" "$CHFILES" | awk -F/ '{d=$1; if(d=="") d="/"; c[d]++} END{for(k in c) printf "%s %d\n",k,c[k]}' \
  | sort -k2,2nr | head -5)

# テスト関連変更
TEST_FILES=$(printf "%s\n" "$CHFILES" | grep -E '(^|/)(test|tests|__tests__)/|(_test\.[a-zA-Z0-9]+$)|(\.spec\.)' || true)

# 依存関係っぽいファイル
DEPS_CAND=$(printf "%s\n" "$CHFILES" | grep -E '(^|/)(package\.json|pnpm-lock\.yaml|yarn\.lock|package-lock\.json|requirements\.txt|poetry\.lock|Pipfile|Pipfile\.lock|go\.mod|go\.sum|Cargo\.toml|Cargo\.lock|Gemfile|Gemfile\.lock)$' || true)

# DB/スキーマ・API っぽい変更
DB_CAND=$(printf "%s\n" "$CHFILES" | grep -Ei '(migrations?/|schema\.(sql|rb)|db/|prisma/|migration)' || true)
API_CAND=$(printf "%s\n" "$CHFILES" | grep -Ei '(openapi|swagger|proto/|graphql|public.*api|api.*v[0-9])' || true)

# リスクレベルの簡易推定
RISK="Low"
if [ "$FILES_CHANGED" -ge 50 ] || [ "$TOTAL_CHURN" -ge 1500 ]; then
  RISK="High"
elif [ "$FILES_CHANGED" -ge 20 ] || [ "$TOTAL_CHURN" -ge 300 ]; then
  RISK="Medium"
fi

# 変更点の「見出し」候補（コミットログ先頭3件）
HEADLINES=$(printf "%s\n" "$COMMITS" | head -3 | sed 's/^/- /')

# ====== 要約付き DIFF.md を生成 ======
ISO=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
{
  echo "# Diff Report (summary-first): $BASE...$HEADBR"
  echo "- Generated at: $ISO"
  echo "- Merge base: $MB"
  echo "- Files changed: $FILES_CHANGED, +$INSERTIONS / -$DELETIONS (total $TOTAL_CHURN), renames: $RENAMES, adds: $ADDEDS, deletes: $DELETEDS"
  echo
  echo "## エグゼクティブサマリ"
  if [ "$FILES_CHANGED" -eq 0 ]; then
    echo "- No changes detected."
  else
    echo "- リスク評価: **$RISK**（ファイル数/変更行数からの概算）"
    [ -n "$HEADLINES" ] && echo "$HEADLINES"
    [ -n "$DEPS_CAND" ] && echo "- 依存関係に変更あり（要確認）" || echo "- 依存関係の変更は検出されませんでした"
    [ -n "$DB_CAND" ] && echo "- データベース/スキーマ関連の変更あり（要マイグレーション確認）"
    [ -n "$API_CAND" ] && echo "- API/スキーマ定義に変更あり（互換性に注意）"
    echo "- 主要ディレクトリ影響範囲: $(printf "%s" "$DIR_STATS" | awk '{printf "%s(%s) ",$1,$2}')"
    echo "- 主な拡張子: $(printf "%s" "$EXT_STATS" | awk '{printf "%s(%s) ",$1,$2}')"
  fi
  echo
  echo "## 変更サマリ"
  echo "- $SHORTSTAT"
  echo
  echo "### 変更ファイル（上位統計）"
  echo "- ディレクトリ別: "
  printf "%s\n" "$DIR_STATS" | awk '{printf "  - %s: %s\n",$1,$2}'
  echo "- 拡張子別: "
  printf "%s\n" "$EXT_STATS" | awk '{printf "  - %s: %s\n",$1,$2}'
  echo
  echo "## 依存関係に関する気配"
  if [ -n "$DEPS_CAND" ]; then
    printf "%s\n" "$DEPS_CAND" | sed 's/^/- /'
  else
    echo "- （検出なし）"
  fi
  echo
  echo "## DB / スキーマ / API の気配"
  [ -n "$DB_CAND" ] && { echo "### DB/Schema"; printf "%s\n" "$DB_CAND" | sed 's/^/- /'; } || echo "- DB/Schema: 検出なし"
  [ -n "$API_CAND" ] && { echo "### API"; printf "%s\n" "$API_CAND" | sed 's/^/- /'; } || echo "- API: 検出なし"
  echo
  echo "## テスト影響"
  if [ -n "$TEST_FILES" ]; then
    echo "- テスト関連の変更が含まれます："
    printf "%s\n" "$TEST_FILES" | sed 's/^/- /'
  else
    echo "- 明示的なテストファイルの変更は検出されませんでした"
  fi
  echo
  echo "## コミット一覧 (${BASE}..${HEADBR})"
  if [ -n "$COMMITS" ]; then
    printf "%s\n" "$COMMITS" | sed 's/^/- /'
  else
    echo "_(none)_"
  fi
  echo
  echo "## 変更ファイル一覧（numstat）"
  if [ -n "$NUMSTAT" ]; then
    echo "| file | + | - |"
    echo "|---|---:|---:|"
    printf "%s\n" "$NUMSTAT" | awk '{adds=$1; dels=$2; $1=$2=""; sub(/^  */,""); print "|" $0 " | " adds " | " dels " |"}'
  else
    echo "_(none)_"
  fi
  echo
  echo "## 差分抜粋"
  if [ -n "$CHFILES" ]; then
    printf "%s\n" "$CHFILES" | while read -r f; do
      if git diff -- "$BASE...$HEADBR" -- "$f" --summary | grep -q "Binary file"; then
        echo "<details><summary>$f (binary)</summary>"
        echo
        echo "\`\`\`"
        git diff -- "$BASE...$HEADBR" -- "$f" --summary || true
        echo "\`\`\`"
        echo "</details>"
      else
        echo "<details><summary>$f</summary>"
        echo
        echo "\`\`\`diff"
        git diff -U3 --find-renames -- "$BASE...$HEADBR" -- "$f" || true
        echo "\`\`\`"
        echo "</details>"
      fi
      echo
    done
  else
    echo "_(none)_"
  fi
} > "$OUT"

echo "✅ 要約付き $OUT を生成しました → $(realpath "$OUT")"

# ====== リモートへ push（MR 用）======
git ls-remote --exit-code --heads origin "$HEADBR" >/dev/null 2>&1 || git push -u origin "$HEADBR"

# ====== MR 作成（glab → pushオプション → REST API の順にフォールバック）======
if command -v glab >/dev/null 2>&1; then
  # glab
  FLAGS=()
  [ "$DRAFT" = "true" ] && FLAGS+=("--draft")
  [ -n "$LABELS" ] && FLAGS+=("-l" "$LABELS")
  [ -n "$REVIEWERS" ] && FLAGS+=("-r" "$REVIEWERS")
  [ -n "$ASSIGNEES" ] && FLAGS+=("-a" "$ASSIGNEES")
  glab mr create --source-branch "$HEADBR" --target-branch "$TARGETBR" \
    --title "$TITLE" --description-file "$OUT" "${FLAGS[@]}"
  URL=$(glab mr list --source "$HEADBR" --target "$TARGETBR" --state opened --json web_url --jq '.[0].web_url' 2>/dev/null || true)

elif git config --get remote.origin.url | grep -qi 'gitlab'; then
  # git push オプション
  git push -u origin "$HEADBR" \
    -o merge_request.create \
    -o merge_request.target="$TARGETBR" \
    -o merge_request.title="$TITLE" \
    ${DRAFT:+-o merge_request.draft} \
    ${REMOVE_SOURCE:+-o merge_request.remove_source_branch} \
    $( [ -s "$OUT" ] && printf %s "-o merge_request.description=$(tr '\n' ' ' < "$OUT")" )
  URL=""  # 場合により取得不可。Web UI で MR が開けます。

else
  # REST API（必要変数: PROJECT_PATH, API_TOKEN, HOST）
  : "${PROJECT_PATH:?set group/subgroup/repo to PROJECT_PATH}"
  : "${API_TOKEN:?set API_TOKEN (Personal Access Token with api scope)}"
  : "${HOST:=https://gitlab.com}"
  ENC_PATH=$(python - <<'PY'
import urllib.parse, os
print(urllib.parse.quote(os.environ["PROJECT_PATH"], safe=''))
PY
)
  URL=$(curl -sS -X POST -H "PRIVATE-TOKEN: $API_TOKEN" \
    --data-urlencode "source_branch=$HEADBR" \
    --data-urlencode "target_branch=$TARGETBR" \
    --data-urlencode "title=$TITLE" \
    --data-urlencode "description=$(cat "$OUT")" \
    "$HOST/api/v4/projects/$ENC_PATH/merge_requests" | jq -r '.web_url' 2>/dev/null || true)
fi

[ -n "${URL:-}" ] && echo "✅ マージリクエスト: $URL" || echo "✅ マージリクエストを作成しました（URL は取得できない場合があります）"
