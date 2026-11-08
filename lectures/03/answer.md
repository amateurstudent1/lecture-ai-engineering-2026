# 第3回 解答

## 演習

- 第1回（素のLLM）とエージェントの回答の違い：
- `--trace` で見えたツール呼び出し（ツール名と引数）：
- 1つの質問での LLM 呼び出し回数（`requests`）：
- 追加したツールと、docstring に書いた説明：

## 宿題

- 取り組んだレベル：実装
- やったこと：
    - app/tools.py に save_learning_log(content, topic) を追加。learning_log/ に `YYYYMMDD-HHMMSS-topic.md` で保存し、同名なら番号を付けて上書きを避ける。
    - app/agent.py で `agent.tool_plain(tools.save_learning_log, requires_approval=True)` で登録し、`output_type=[str, DeferredToolRequests]` にした。
    - main に承認ループを追加。`DeferredToolRequests` が返ってきたら `requests.approvals` を表示して `y/n` で聞き、`DeferredToolResults` に `True` または `ToolDenied` を入れて再実行する。
    - INSTRUCTIONS に「必ず save_learning_log ツールを呼び出す。テキストだけで保存したと言わない。保存の前に確認が入る。」を追加。
        - tests/test_agent.py の30行目も承認待ちを数えるように直した。`DeferredToolRequests` の `approvals` を足して数える。
- 結果：
    - `uv run python -m app.agent --list-tools` で `save_learning_log` が表示され、3個のツールになった。
    - y の場合：`[tool call] save_learning_log({'topic': 'agent-loop', 'content': 'エージェントループはモデルがツール呼び出しを返したら実行して結果を戻すことを繰り返す仕組みです'})` → `[tool return] save_learning_log -> learning_log/ に保存しました: 20261108-155805-agent-loop.md`。ファイルの中身は `# agent-loop` と上記の1文で、保存が確認できた。
    - n の場合：`[tool call] save_learning_log({'topic': 'deny-test', 'content': 'これは断るテストです'})` → `[tool return] save_learning_log -> ユーザーが保存を断りました...`。モデルは「保存処理は実行されませんでした」と答え、`learning_log/` に新しいファイルは増えなかった。
        - `uv run pytest -q` で `11 passed` になった。直す前は `1 failed, 10 passed` で `assert 2 == 3` だった。
- 気づき・疑問：
    - あいまいな頼み方だとモデルがツールを呼ばずにテキストだけで「保存しました」と言ってしまうことがあった。具体的にツール名と中身を指定し、INSTRUCTIONS を強くしたら呼ぶようになった。
    - 最初は `requests.calls` を見ていて承認結果が空でエラーになった。調べたら `DeferredToolRequests` のフィールドは `['approvals', 'calls', 'metadata']` で、承認が必要なツールは `approvals` に入ることが分かり、`requests.approvals` に直したら動いた。
    - 承認を挟むことで、誤って保存されるのを防げることが実感できた。疑問：他のツールにも同じ承認を付ける場合、どこまで承認を求めるのが適切か。
