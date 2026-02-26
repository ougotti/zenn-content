---
title: "Raspberry Pi上のOpenClawをアップデートしてChatGPT Plusに切り替えた"
emoji: "🦞"
type: "tech"
topics: ["openclaw", "raspberrypi", "chatgpt", "ssh", "ai"]
published: true
---

## はじめに

[OpenClaw](https://docs.openclaw.ai/) は、Discord・Telegram・WhatsApp などのチャットアプリとAIを繋ぐOSSのボットフレームワークです。元はClawdbot → Moltbot → OpenClawと改名を繰り返しながら急成長し、2026年2月にはOpenAIに開発者が参加したことでも話題になりました。

自宅のRaspberry Pi上でずっと動かしていたのですが、アップデートとChatGPT Plusへの切り替えでいくつかハマったので実録として残しておきます。

## 環境

- Raspberry Pi（Raspberry Pi OS、ヘッドレス運用）
- Windows PC からSSHで管理
- OpenClaw: 2026.2.15 → 2026.2.22-2

## インストール（参考）

まだ入れていない方向けに。

```bash
sudo npm install -g openclaw
openclaw onboard   # 初期設定ウィザード
```

ウィザードに沿って進めると systemd user サービスとして登録され、再起動後も自動起動します。

```bash
systemctl --user enable --now openclaw-gateway
```

## SSHで手軽に状態確認

ヘッドレス機なので、普段は Windows のターミナルから `ssh <ホスト名> "コマンド"` で確認しています。

```bash
# プロセス確認
ssh raspberrypi "ps aux | grep openclaw"

# アップデートがあるか確認
ssh raspberrypi "openclaw update status"
```

実行結果：

```
┌──────────┬──────────────────────────────────────────┐
│ Item     │ Value                                    │
├──────────┼──────────────────────────────────────────┤
│ Install  │ pnpm                                     │
│ Channel  │ stable (default)                         │
│ Update   │ available · pnpm · npm update 2026.2.22-2│
└──────────┴──────────────────────────────────────────┘
```

アップデートがありました。

## アップデート：sudo が必要だった

SSH越しにそのまま実行しようとしたところ権限エラー。

```bash
ssh raspberrypi "openclaw update --yes"
```

```
npm error code EACCES
npm error syscall rename
npm error path /usr/lib/node_modules/openclaw
npm error errno -13
```

グローバルインストール（`/usr/lib/node_modules/`）なので `sudo` が必要でした。

```bash
# RPに直接SSHして実行
ssh raspberrypi
sudo openclaw update --yes
```

206秒かかりましたが無事完了。

```
Update Result: OK
  Before: 2026.2.15
  After:  2026.2.22-2
```

## ClaudeからChatGPT Plus（Codex）に切り替え

もともとClaudeのAPIキーで動かしていましたが、ChatGPT Plus のサブスクで Codex が使えると知ったので切り替えることにしました。

なおAnthropicはサブスク（Claude Pro/Max）のOAuth認証を Claude Code と Claude.ai 専用とし、それ以外のツールへの利用を規約違反としています（APIキーは別途OK）。

```bash
openclaw onboard --auth-choice openai-codex
```

ウィザードを進めると、こんなURLが表示されます：

```
http://localhost:1455/auth/callback?code=...&scope=...&state=...
```

## ヘッドレス機でのOAuth認証：SSHポートフォワードが鍵

ここで詰まりました。`localhost:1455` はRaspberry Pi側のポートなので、Windows のブラウザからそのまま開いても接続拒否になります。

「URLは別ホストマシンで開いてもいいの？」と思ったのが解決のきっかけ。**SSHポートフォワード**を使えばOKです。

**別ターミナルで**以下を実行してポートフォワードを張ります：

```bash
ssh -L 1455:localhost:1455 raspberrypi
```

この状態で Windows のブラウザから以下を開きます：

```
http://localhost:1455/auth/callback?code=...（ウィザードで表示されたURLをそのままコピー）
```

ブラウザに「Authentication successful. Return to your terminal to continue.」と表示されれば認証成功です。

## モデルを切り替えて再起動

認証後、デフォルトモデルを手動で切り替えます。

```bash
openclaw config set agents.defaults.model.primary openai-codex/gpt-5.3-codex
systemctl --user restart openclaw-gateway
```

設定確認：

```bash
openclaw config get agents.defaults.model
# => { "primary": "openai-codex/gpt-5.3-codex" }
```

## まとめ

今回のポイントをまとめます。

| ポイント | 対処 |
|---|---|
| グローバルnpmのアップデートが権限エラー | `sudo openclaw update --yes` |
| ヘッドレス機でOAuth認証のURLが開けない | SSHポートフォワード `ssh -L 1455:localhost:1455 <ホスト>` |
| モデルがウィザード後も変わらない | `openclaw config set` で手動切り替え |

ヘッドレスのRaspberry Piでもリモートでの認証は **SSHポートフォワード** を使えばスムーズに解決できます。同じ状況で困っている方の参考になれば。

---

この記事は [Claude Code](https://claude.ai/claude-code) との会話をもとに書きました。
