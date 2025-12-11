## 目的
- タスクの実装を完了すること
  
## 手順
1. $TASK_FILE を参照
2. 前提条件確認コマンドを実行
3. clearが出力されていれば実装を開始
4. 実装が完了したら受け入れ基準確認コマンドを実行
5. clearが帰ってきたらコミットをして終了。（ユーザーの操作が必要な場合は要求）
6. clearが帰ってこなければ出力をもとに修正をして、再度受け入れ基準確認コマンドを実行。
7. $PR_NUM が完了するまで4~6を繰り返す。

## 前提条件確認コマンド
```bash
codex exec --full-auto --model gpt-5.1-codex-mini --cd $REPO_PATH - << 'EOF'
## Role

* Your role is to verify that all prerequisites for implementation are satisfied **before** any code is written.

## Procedure

1. Check the prerequisites for `$PR_NUM` in `$TASK_FILE`.
2. Run any necessary tests and inspect relevant files in the repository to confirm whether the prerequisites are satisfied.
3. If the prerequisites are satisfied, mark the prerequisites for `$PR_NUM` as completed in `$TASK_FILE` and output `clear`.
4. If the prerequisites are **not** satisfied, output a message indicating that they are not met using `echo`.

EOF
```

## 受け入れ基準確認コマンド
```bash
codex exec --full-auto --model gpt-5.1-codex-mini --cd $REPO_PATH - << 'EOF'
## Role

* Your role is to verify whether the current repository meets the acceptance criteria.

## Procedure

1. Check the acceptance criteria for `$PR_NUM` in `$TASK_FILE`.
2. Run any necessary tests and inspect the repository files to confirm whether the acceptance criteria are met.
3. If the acceptance criteria are met, mark the acceptance criteria for `$PR_NUM` as completed in `$TASK_FILE` and output `clear`.
4. If the acceptance criteria are **not** met, use `echo` to output that they are not satisfied.

EOF
```

## 制限
- タスクリストの最後のPRの場合はタスクリストのmdファイルを `_task/archive` に移動させる
- $PR_NUM が完了したら必ず日本語でメッセージをつけてコミットする