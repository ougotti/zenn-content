---
title: "Claude Codeに会話しながらZoom入退室をLINE通知するAWSサーバーレス基盤を作った"
emoji: "📹"
type: "tech"
topics: ["ClaudeCode", "AWS", "Lambda", "Zoom", "LINE"]
published: true
---

## はじめに

「Zoom会議に誰かが入ったら LINE に通知したい」というシンプルな要望を、**Claude Code との会話だけで実装**しました。

既存の「Google Calendar → LINE リマインド通知」システム（AWS CDK v2 / Python）があり、そこに Zoom 参加人数通知を**別スタックとして追加**する形で進めています。

この記事では「**どんな指示を出したか・どこを自分で判断したか**」にフォーカスします。コードはほぼ Claude Code が書きました。

---

## システム構成

```
Zoom Webhook
    ↓
AWS Lambda (Function URL)  ← 署名検証・CRC検証
    ↓
DynamoDB (MeetingsState) + SQS (delay 5s)
                                ↓
                        AWS Lambda (SQS Consumer)
                                ↓
                    フィルタ・デバウンスチェック
                                ↓
                          LINE Push 通知
```

CDK スタックは既存を汚さないよう**2スタック構成**にしました：

| スタック | 役割 |
|---------|------|
| `ZoomNotifyLineStack` | 既存: Google Calendar → LINE リマインド |
| `ZoomParticipantStack` | 新規: Zoom Webhook → 参加人数 → LINE |

---

## 実装の流れと指示の内容

### Step 1. 仕様書を渡して計画してもらう

まず要件をまとめた仕様書（Webhook受信 → DynamoDB → SQS → LINE通知の構成）をチャットに貼り付けて、こう指示しました。

> 「新しいプロジェクトの仕様だよ。いまのフォルダに追加修正するよ。まずは計画をしましょう」

Claude Code は既存コードを読み、追加すべきファイルと変更点を整理したプランを提示してくれました。

**ここで自分が確認・修正した点:**

プランの中で既存の API Gateway が「Zoom Webhook 用」として扱われていました。おかしいと思い確認を指示。

> 「現在の API Gateway は Zoom Webhook 受信専用（LINE用ではない）→ もう一度確認して」

Claude Code がコードを再確認したところ、`lineAccessTokenParam.grantRead(webhookFunction)` の記述から **LINE Webhook 用**だと判明。Zoom 専用には Lambda Function URL を別途用意する方針に修正されました。

また、スタックの分割も私から提案しました。

> 「`zoom_notify_line_stack.ts` とは分けて Zoom 用として、別スタックに分けたほうが保守性が良いと思います」

→ `ZoomParticipantStack` を新規作成する方針になりました。

**ポイント:** Claude Code は既存コードを正確に読みますが、設計の方向性（既存を変えるか・分割するか）は人間が判断したほうが良いです。

---

### Step 2. 実装を進める

プランを承認し、実装を依頼。指示はシンプルです。

> 「OK」
> 「つづけましょう」

Webhook受信Lambda、CDKスタック、SSM登録スクリプトへの追記、ドキュメント更新まで一気に実装してくれました。

**Zoom Webhook 受信 Lambda の主な実装内容:**
- HMAC-SHA256 署名検証（`x-zm-signature` ヘッダー）
- CRC検証（Zoom App Marketplace 登録時の疎通確認）
- `participant_joined` / `participant_left` → DynamoDB の `currentCount` を加減算
- DynamoDB 更新後に SQS へ送信（DelaySeconds=5）

---

### Step 3. Zoom App Marketplace の設定で詰まった

デプロイ後、Zoom App Marketplace で Webhook を登録する際に必須入力項目で詰まりました。

> 「これとか入力必須なの？大丈夫？」（Privacy Policy URL などの画面を見せながら）

Claude Code の回答:「プライベートアプリ（社内利用）であれば、Privacy Policy URL は社内のページや GitHub リポジトリの URL でも問題なく通過できます」

→ 実際に通過できました。このような**ツールの使い方に関する質問**にも答えてくれます。

---

### Step 4. エラーはログをそのまま貼る

デプロイ後のテスト中、CloudWatch Logs にエラーが出ました。

```
[ERROR] ClientError: An error occurred (ValidationException) when calling
the UpdateItem operation: Invalid ConditionExpression: Syntax error;
token: "+", near: "currentCount + :delta"
```

ログをそのままチャットに貼り付けただけで、Claude Code が原因と修正を提示してくれました。

**原因:** DynamoDB の `ConditionExpression` は算術演算子（`+`）が使えない。

**修正:** `currentCount + :delta >= 0` という条件を、退出時（delta=-1）のみ `currentCount >= :one` に変更。

> エラーログを貼るだけで修正まで完結します。原因調査の時間がゼロになります。

---

### Step 5. SQS Consumer Lambda（LINE通知）の追加

Phase 2 として通知 Lambda を依頼しました。

> 「つづけましょう」

デバウンスロジック込みで実装してくれました：

```python
# debounce チェック: 後続イベントが来ていたらスキップ
if debounce_until > now:
    return

# 重複チェック: 前回と同じ人数なら送らない
if current_count == last_sent_count:
    return
```

外部ライブラリは使わず `urllib.request` で実装。「既存スタックの Lambda Layer に依存しない設計にする」という判断を Claude Code が自律的に行いました。

---

### Step 6. テストは段階的に — 指示で制御できる

LINE グループに誤送信しないよう、段階的に有効化しました。

**最初の指示:**
> 「まだLINEグループに通知しないで」

私が想定した応答（NotifyFunction をコメントアウト）を提示してきましたが、それは違いました。

> 「ちがう、通知はしないけど、関数は実装までしておいて」

→ コードは実装済みの状態を維持し、LINE 送信部分だけコメントアウトする形（ドライラン）に修正されました。

さらに、特定の会議だけ通知対象にしたい場合も自然な言葉で指示できます。

> 「ミーティング ID: XXX XXXX XXXX だけ LINE 通知の対象としてください。ほかはログだけで」

→ `NOTIFY_MEETING_IDS = {'YOUR_MEETING_ID'}` というフィルタが追加されました。

**DynamoDB でのカウント動作を確認してから LINE 通知を有効化:**

> 「対象ルームでのLINEへの通知も有効にしてください」

→ ドライランのコメントを外して完了。

---

### Step 7. 機能追加も会話で

通知に参加者名を表示したくなりました。

> 「参加した人のユーザー名を表示することはできます？」

Claude Code が「できます。ただしデバウンスとの相性で複数人同時入室時は最後の人の名前だけになります」と説明し、2つの案を提示してくれました。

> 「頻度は少ないので、最後のイベント者のみ表示する案でいきましょう」

→ `zoom_webhook/handler.py` と `zoom_notify/handler.py` の2ファイルを修正して完了。

| 状況 | メッセージ |
|------|-----------|
| 入室 | `👥 現在 2 人が参加中（田中さんが入室）` |
| 退出（残あり） | `👥 現在 1 人が参加中（鈴木さんが退出）` |
| 全員退出 | `📴 全員退出しました（佐藤さんが退出）` |

---

### Step 8. ドキュメントも任せる

> 「ここまでの修正内容をドキュメントとして反映してください」

手順書（`.github/instructions/project-instructions.md`）を自動更新してくれました。実装が終わったらこの一言で済みます。

---

## Claude Code との会話で学んだこと

### 効果的な指示の出し方

| シーン | 指示例 |
|--------|--------|
| 計画してもらう | 仕様書を貼り付けて「まずは計画をしましょう」 |
| 実装を続けてもらう | 「OK」「つづけましょう」 |
| 修正を指示する | 誤った前提を指摘して「もう一度確認して」 |
| エラー対応 | CloudWatch Logs をそのまま貼る |
| 機能追加 | 「〇〇することはできますか？」と相談形式で |
| テスト制御 | 「まだ〇〇しないで」「〇〇だけ有効にして」 |
| ドキュメント | 「ここまでの内容をドキュメントに反映して」 |

### 自分で判断すべき点

- **設計の分割方針**（既存スタックに混ぜるか、別スタックにするか）
- **既存リソースの役割確認**（Claude Code の誤認を指摘する）
- **テストの段階的な進め方**（何をいつ有効化するか）
- **トレードオフの選択**（シンプルな実装 vs 完全な実装）

---

## まとめ

コードを書く作業は Claude Code に任せ、自分は**設計判断・動作確認・指示の調整**に集中できました。

特に効果的だったのは：
- **エラーログをそのまま貼る** → 原因調査がゼロになる
- **「まだ〇〇しないで」という否定形の指示** → 段階的な有効化が自然な言葉でできる
- **相談形式で機能追加** → トレードオフを説明してもらってから判断できる

AWS CDK (TypeScript) + Python Lambda の組み合わせでも、言語をまたいで一貫した実装をしてくれます。
