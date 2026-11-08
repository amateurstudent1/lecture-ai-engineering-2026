# 第2回 解答

## 演習

- 使ったモデル・設定（`result.json` の `config` から）：
    - base_model: Qwen/Qwen3.5-0.8B, revision: 2fc06364715b967f1860aea9cf38778875588b17
    - seed: 42, num_train_epochs: 20, warmup_steps: 6, lr: 2e-4, lora_alpha: 32, max_seq_len: 512
    - 元: lora_r 16 / 変更後: lora_r 8, method: LoRA（量子化なし）, transformers 5.18.0 / peft 0.21.1

- 自分が学習を実行したか（`executed_training_myself`）：はい（実行できた範囲：Google Colab T4 GPUで2回学習・評価まで実行）

- 学習前: `format_rate` = 0.095 (2/21) / `content_rate` = 0.0
- 学習後（元 r=16）: `format_rate` = 1.0 (21/21) / `content_rate` = 0.857 (18/21)
- 学習後（変更 r=8）: `format_rate` = 1.0 (21/21) / `content_rate` = 0.904 (19/21) / trainable 0.4228% (3,194,880 params), 最終loss 0.01325, 学習222秒 / 評価42.8秒

- 退行した質問：[] なし（両モデルとも）

- 学習で残った失敗と、その型（根拠なしの質問に断定した / 根拠ありの質問に確認要求で逃げた / 形式の崩れ）：
    - 根拠なしの質問に断定した型が残存。FAQに無いのに needs_review=false で断定。

- 知識で問う質問（セクション 6）の結果（学習前の分類 → 学習後の分類）と気づいたこと：
    - 「この講座の最終課題の締切はいつですか」：format_error → asserted（「9月30日 23:59です」と誤断定）※FAQに無い
    - 「担当教員の名前は何ですか」：format_error → needs_review（「このFAQには記載がありません」）正しく回避
    - 「次回の講義のテーマは何ですか」：format_error → needs_review 正しく回避
    - 「課題の提出期限はいつですか」：format_error → asserted 誤断定
    - → 4問中2問は正しく needs_review=true にできたが、2問は知らないのに断定してしまう。形式は学べても事実の判断は学べていない。

- 学習で変わったこと・変わらなかったこと：
    - 変わった：JSONのキーが揃うようになり format_rate 0.095→1.0。content_rateも0.0→0.857/0.904へ。r=8の方が評価時間54.4秒→42.8秒に短縮。
    - 変わらなかった：FAQ外質問へのハルシネーション。rを半分にしてもその傾向は変わらず。

## 宿題

- 取り組んだレベル：基礎
- 変えた設定やデータ：CONFIGの lora_r を 16 → 8 に変更。他は全て固定し、変更の度に ランタイム → セッションを再起動 して上から実行。結果は result_original.json (r=16) と result_r8.json (r=8) に保存。
- 結果（比較表や数字）：
  |  | r=16 (元) | r=8 (変更) |
  | :--- | :--- | :--- |
  | trainable | 約0.84% | 0.4228% |
  | after format_rate | 1.0 | 1.0 |
  | after content_rate | 0.857 | 0.904 |
  | after both_rate | 0.857 | 0.904 |
  | 評価時間 | 54.4秒 | 42.8秒 / 学習222秒 |
    - rを半分にしても形式は維持でき、内容はむしろ向上。少量データでは小さいランクの方が過学習しにくい可能性。

- ファインチューニングが向く場面・向かない場面についての考察：
    - 向く場面：出力形式の統一。今回のように {"answer":..., "needs_review":...} 型を100%守らせる用途にはLoRAが非常に有効。14件×20epoch・4分で達成できた。
    - 向かない場面：事実の正確性や「知らない」と言う判断。FAQに無い締切を「9月30日」と断定する失敗が残った。こういった用途はRAGで根拠を渡す、needs_review=trueのデータを増やす、システムプロンプトで制約する、などの併用が必要。
