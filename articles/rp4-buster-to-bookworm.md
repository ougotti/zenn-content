---
title: "Claudeと進めるRaspberry Pi 4アップグレード実践（Buster→Bookworm / 設定維持）"
emoji: "🍓"
type: "tech"
topics: ["raspberrypi", "linux", "debian", "raspbian", "docker"]
published: true
---

## はじめに

自宅のRaspberry Pi 4（以下、rp4）が Raspbian 10 Buster のまま放置されていました。Buster は EOL（2024年6月）を迎えており、`apt upgrade` を実行すると 404 エラーが大量発生して、セキュリティパッチすら適用できない状態でした。

しかもこの rp4 は、自宅外からアクセスする用途でも使っていました。古い OS のまま運用し続けるのは不安だったため、アップグレードを決意しました。

正直、私自身かなり面倒くさがりで、長年「そのうちやる」と先延ばしにしていました。クリーンインストールが一番確実ですが、長年積み上げた設定や Docker コンテナは引き継ぎたいところです。

それでも Claude と一緒に、手順を分解して一つずつ進めることで、最終的に in-place アップグレードを完了できました。

この記事は、次のような方に向けた内容です。

- Raspberry Pi を長年運用しており、クリーンインストールを避けたい
- Docker や SSH 設定を維持したまま OS を上げたい
- Buster の EOL により更新できず困っている

### Claude（AI）と一緒に作業しました

今回の作業はすべて **Claude Code（AIアシスタント）** と会話しながら進めました。私がざっくり指示を出して、Claudeがコマンドを提案・実行し、エラーが出たら一緒に対処する、というスタイルです。

```
私:   「不要なものをなくしてから軽くして、アップグレードできるかな。
        重要なのは、SSHの設定だけは残しておいてほしい」

Claude: 「Jenkins、Squid、BOINC、SSM Agentを削除してから
          アップグレードしましょう。まずバックアップを取ります」
```

この記事では、やりとりの中で発生したトラブルと解決方法を中心にまとめます。

### 最終的な成果

| 項目 | 変更前 | 変更後 |
|------|--------|--------|
| OS | Raspbian 10 (Buster) | **Raspbian 12 (Bookworm)** |
| カーネル | armhf (32bit) | **aarch64 (64bit)** |
| 起動時間 | 2分17秒 | **22秒** |
| SSH/fail2ban設定 | — | **維持** |

## 所要時間とリスク（今回の実績）

| 項目 | 値 |
|------|----|
| 所要時間 | 約2〜3時間 |
| 再起動回数 | 複数回（少なくとも6回） |
| 起動不能 | 1回（systemd循環依存） |
| 復旧可否 | 復旧済み（SDカード編集で対応） |

---

## 環境

- ハードウェア: Raspberry Pi 4
- 変更前OS: Raspbian GNU/Linux 10 (Buster)
- 接続: Wi-Fi（有線LANなし）
- SSH: 外部アクセス用途あり（ポート番号はマスク）
- fail2ban: 設定済み

:::message
公開記事のため、ホスト名・ポート番号などの接続情報は一部マスクしています。
:::

---

## 方針: 2段階アップグレード

Debian の in-place アップグレードは、1バージョンずつ進めるのが基本です。

```
Buster (10) → Bullseye (11) → Bookworm (12)
```

一気に進めると、依存関係が壊れるリスクが高まります。

## 最短手順（コピペ用）

詳細ログを読む前に、最短で全体像を把握したい方向けの手順です。

```bash
# 1) 重要ファイルのバックアップ
sudo mkdir -p /home/pi/backup
sudo cp /etc/ssh/sshd_config /home/pi/backup/
cp ~/.ssh/authorized_keys /home/pi/backup/
sudo cp /etc/fail2ban/jail.local /home/pi/backup/

# 2) 不要パッケージを整理
sudo apt-get remove --purge -y snapd
sudo apt-get autoremove --purge -y

# 3) Buster -> Bullseye
sudo sed -i 's/buster/bullseye/g' /etc/apt/sources.list
sudo sed -i 's/buster/bullseye/g' /etc/apt/sources.list.d/*.list
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y \
  -o Dpkg::Options::='--force-confdef' \
  -o Dpkg::Options::='--force-confold'

# 4) Bullseye -> Bookworm
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list.d/*.list
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y \
  -o Dpkg::Options::='--force-confdef' \
  -o Dpkg::Options::='--force-confold'
```

:::message
このセクションは最短版です。実運用では、後述の「ハマりポイント」と「復旧手順」も必ず確認してください。
:::

## やらない方がいいこと

- Buster から Bookworm へ一気にアップグレードする
- systemd サービスから別サービスを `restart` して依存関係を組む
- 物理コンソールなしで、公開中の本番機を SSH だけでアップグレードする

---

## Phase 1: 事前準備・バックアップ

まず重要な設定をバックアップします。SSH 設定と fail2ban 設定は特に重要です。

```bash
# rp4上で実行
sudo cp /etc/ssh/sshd_config /home/pi/sshd_config.bak
cp ~/.ssh/authorized_keys /home/pi/authorized_keys.bak
sudo cp /etc/fail2ban/jail.local /home/pi/jail.local.bak
```

```bash
# Windows側からコピー
scp "<user>@<raspi-host>:/home/pi/*.bak" C:/backup/rp4/
```

---

## Phase 2: 不要サービスの削除・軽量化

アップグレード前に不要なサービスを削除しました。依存関係が複雑になっているパッケージがあると、アップグレード時のトラブル要因になります。

削除したもの:

| サービス | 理由 |
|---------|------|
| Jenkins | 長年使っておらずほぼ放置 |
| Squid | リモートプロキシ目的だったが現在不要 |
| BOINC | 停止したまま放置 |
| Amazon SSM Agent | AWS側でも認識されていない |
| Snapd | 軽量化のため |
| Node-RED | Bookworm非対応で邪魔になった |
| wolfram-engine (Mathematica) | 依存関係の邪魔になるため |

```bash
sudo systemctl stop jenkins && sudo systemctl disable jenkins
sudo apt-get remove --purge -y jenkins squid squid-common \
  boinc-client boinc-manager amazon-ssm-agent snapd

sudo apt-get autoremove --purge -y
```

:::message
**ポイント**: `gcc`/`g++`/`libc6-dev`などのdev系パッケージも、アップグレード中に依存関係エラーの原因になります。使っていなければ事前に削除しておくのがおすすめです。
:::

---

## Phase 3: Buster → Bullseye アップグレード

### このフェーズの会話の流れ

作業途中で SSH が切断されたため、物理コンソールに切り替えて進めました。JIS キーボードなのに US キーマップで、パイプ文字が打てないというハプニングもありました。

```
私:   「続けましょう。なんか合図あるの？」

Claude: 「apt-listchangesの画面で止まっているかもしれません。
          Enterキーを押してください。
          設定ファイルのプロンプトが出たらNで既存を保持してください」

私:   「/etc/default/usraddの設定プロンプトで止まった」

Claude: 「Nを押して既存の設定を維持してください」

私:   「38.0 MBを39秒で取得しました。apt-listchangesで止まってるけど
        待ったほうがいい？」

Claude: 「はい、そのまま待ってください。正常な動作です」

私:   「まだアップグレード中。これアーキテクチャはなんだっけ？
        64bitになれるの？」  ← 待ち時間に雑談

Claude: 「今は32bit(armhf)です。/boot/config.txtに
          arm_64bit=1を追加するだけでカーネルだけ64bitにできます。
          クリーンインストール不要です」

私:   「dockerだけ64bitとかになんないの？」

Claude: 「arm_64bit=1でカーネルを64bitにすれば、
          OSはそのままでDockerのarm64コンテナが動きます」
```

### sources.listの更新

```bash
sudo sed -i 's/buster/bullseye/g' /etc/apt/sources.list
sudo sed -i 's/buster/bullseye/g' /etc/apt/sources.list.d/*.list
```

### アップグレード実行

SSHでの作業中にSSHデーモン自体がアップグレードされて切断される可能性があります。**物理コンソールでの作業を強く推奨します。**

```bash
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y \
  -o Dpkg::Options::='--force-confdef' \
  -o Dpkg::Options::='--force-confold'
```

`DEBIAN_FRONTEND=noninteractive` と `--force-confold` を組み合わせることで、設定ファイルの上書きプロンプトを自動的にスキップできます（既存設定を維持）。

### 発生したトラブル

**① libc6/libc-binのバージョン不一致**

アップグレード途中で libc6 が 2.31 に上がったのに、libc-bin が 2.28 のまま残ることがあります。

:::message alert
`--force-depends` はパッケージ管理を壊す可能性があります。通常は使用せず、どうしても進めない場合の最終手段として使ってください。
:::

```bash
# キャッシュにあるBullseye版を強制インストール
sudo dpkg --force-depends -i \
  /var/cache/apt/archives/libc-bin_2.31*.deb
```

**② dpkg処理中のエラー**

```bash
# エラーが出たパッケージを強制削除してから続行
sudo dpkg --remove --force-depends <パッケージ名>
sudo DEBIAN_FRONTEND=noninteractive dpkg --configure -a
sudo apt-get install -f -y
```

### 完了確認

```bash
cat /etc/os-release
# PRETTY_NAME="Raspbian GNU/Linux 11 (bullseye)" を確認
```

### 起動時間が2分17秒になった問題

Bullseye への更新後に再起動したところ、起動が異様に遅くなりました。

```bash
systemd-analyze blame | head -5
# 2min 603ms connman-wait-online.service  ← 犯人
```

`connman-wait-online.service` がネットワーク接続待ちで2分待っていました。

```bash
sudo systemctl disable connman-wait-online.service
sudo systemctl mask connman-wait-online.service
# 再起動後: 18秒に短縮
```

---

## Phase 3.5: 64bitカーネル化（オプション）

Bullseye が完了した時点で、クリーンインストールなしにカーネルだけ 64bit 化できます。

```bash
echo "arm_64bit=1" | sudo tee -a /boot/config.txt
sudo reboot
```

```bash
# 再起動後に確認
uname -m
# aarch64 → 64bitカーネル成功！
```

**なぜこれが嬉しいのか:**
- OSのユーザーランド（既存の設定・サービス）は32bitのまま維持
- カーネルが64bitになるので、Dockerでarm64コンテナが動く
- クリーンインストールなしで移行できる

---

## Phase 4: Bullseye → Bookworm アップグレード

### このフェーズの会話の流れ

Bookworm へのアップグレードはリモートのバックグラウンドで実行しましたが、途中で debconf の対話プロンプトが表示されて停止しました。

```
私:   「どうなってます？」

Claude: 「インストールが始まって、質問で止まっています。
          『パッケージのアップグレード中、質問することなく
          サービスを再起動しますか？[はい/いいえ]』
          →「はい」と入力してください」

私:   「コンソールが見えませんよ」

Claude: 「別のSSH接続から回答を送ります（試みるも失敗）
          ...debconfプロセスのPIDを調べて直接入力を送ります
          ...パイプの書き込み端が閉じています。
          debconfに直接答えを設定します」
          ← 結局、debconfプロセスをkillして
            DEBIAN_FRONTEND=noninteractiveで再実行
```

```
私:   「処理中にエラーが発生しました: raspi-firmware」

Claude: 「/boot/firmwareが存在しないのが原因です。
          Bookwormからマウントポイントが変わりました」
          ← fstabを更新して解決
```

### sources.listの更新

```bash
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list.d/*.list
sudo apt-get update
```

### アップグレード実行

```bash
sudo DEBIAN_FRONTEND=noninteractive apt-get full-upgrade -y \
  -o Dpkg::Options::='--force-confdef' \
  -o Dpkg::Options::='--force-confold'
```

### 発生したトラブル

**① libc6系の依存エラー（またも）**

```bash
# libc6が2.36になったのにlibc-bin/libc-l10nが古いまま
sudo dpkg --force-depends -i \
  /var/cache/apt/archives/libc-bin_2.36*.deb
sudo dpkg --force-depends -i \
  /var/cache/apt/archives/libc-l10n_2.36*.deb
```

**② dev系パッケージの依存エラー**

```bash
# gcc/g++/cpp系を強制削除
sudo dpkg --remove --force-depends \
  libc6-dev libc6-dbg libc-dev-bin \
  libstdc++-10-dev g++-10 gcc-10 cpp-10 gcc g++ cpp
```

**③ `/boot/firmware` が存在しない**

Bookworm からファームウェアのマウントポイントが `/boot` から `/boot/firmware` に変わりました。in-place アップグレードでは `fstab` が自動更新されません。

```bash
# fstabを更新
sudo sed -i 's|/boot           vfat|/boot/firmware  vfat|' /etc/fstab
sudo mkdir -p /boot/firmware

# 現在のセッションで即時反映
sudo mount --bind /boot /boot/firmware
sudo DEBIAN_FRONTEND=noninteractive dpkg --configure -a
```

### 完了確認

```bash
cat /etc/os-release
# PRETTY_NAME="Raspbian GNU/Linux 12 (bookworm)" を確認

uname -m
# aarch64 を確認
```

---

## Phase 5: Wi-Fi自動接続の修正

Bookworm へのアップグレード後、再起動のたびに Wi-Fi が接続されない問題が発生しました。ここが最も苦戦したポイントです。

### このフェーズの会話の流れ

```
私:   「rebootするとまたrfkill/iw/dhcpcdをやらないとSSH繋がらない」

Claude: 「永続化しましょう。
          sudo raspi-config nonint do_wifi_country JP」
          ← すでにcountry=JPは設定済みだった

私:   「毎回打つのが面倒なので、シェルスクリプトを作っておいて」

Claude: 「~/wifi-fix.sh を作成しました」

私:   「rebootするとipアドレスがない」（5回ほど繰り返す）

Claude: 「systemdサービスで対応」→「rc.localで対応」
          → 何度やっても再起動後に失敗

私:   「dhcpcdとwpa_supplicantをstop、startしたらipアドレスがついた」
      ← ユーザーが自分でデバッグして発見！

Claude: 「restartではなくstop→startが重要でした。
          wifi-fix.shとrc.localを更新します」
```

:::message
**ポイント**: `systemctl restart wpa_supplicant` ではINACTIVEになります。必ず `stop` → `start` の順番が必要でした。AIも最初はrestartで試みていましたが、ユーザーが実機でデバッグして発見した解決策です。
:::

### 症状

```bash
# 起動後にwlan0を確認するとIPがない
ip addr show wlan0
# inet が表示されない...
```

### 原因

in-place アップグレードにより、Bookworm の新しいハードウェア初期化順序と既存の wpa_supplicant/dhcpcd 設定が噛み合わなくなっていました。rfkill が Wi-Fi をブロックしたまま、wpa_supplicant が起動してしまう状態です。

### 失敗したアプローチ

- systemdサービスで `ExecStartPost=/bin/systemctl restart dhcpcd` → **循環依存でハング・起動不能に**
- `systemctl restart wpa_supplicant` → **INACTIVEのまま**（stopしてからstartが必要）
- NetworkManagerへの移行 → wpa_supplicantと競合してさらに悪化

### 解決策: rc.localで制御する

systemd の依存関係から外れた `rc.local` で、wpa_supplicant を stop→start し直す方法が安定しました。

今回 `rc.local` を選んだ理由は、systemd 側で厳密に依存関係を解決しようとすると循環依存に入りやすかったためです。`rc.local` には、次の利点がありました。

- 起動シーケンスの最終段で実行できる
- 既存サービスの依存関係に巻き込まれにくい
- 失敗時に手順を切り分けやすく、デバッグしやすい

今回のような「起動後に Wi-Fi 周りを強制的に再初期化する」用途では、systemd ユニットを複雑化するより `rc.local` の方が安全でした。

```bash
sudo tee /etc/rc.local << 'EOF'
#!/bin/bash
/bin/systemctl stop dhcpcd
/usr/bin/killall wpa_supplicant 2>/dev/null
/bin/sleep 1
/usr/sbin/rfkill unblock wifi
/sbin/iw reg set JP
/sbin/wpa_supplicant -B -i wlan0 -c /etc/wpa_supplicant/wpa_supplicant.conf
/bin/sleep 5
/bin/systemctl start dhcpcd
exit 0
EOF
sudo chmod +x /etc/rc.local
```

:::message alert
`restart` ではなく `stop` → `start` の順番が重要です。`restart` だとwpa_supplicantがINACTIVE状態になります。
:::

---

## 起動不能になった時の復旧方法

systemd サービスの設定ミスで起動不能になったときは、SD カードを直接編集して復旧しました。

### このトラブルの会話の流れ

```
Claude: 「wifi-countryサービスにdhcpcdの再起動を追加します」
          ← ExecStartPost=/bin/systemctl restart dhcpcd を追加

私:   「画面真っ黒、でなくなった」

私:   「コンソールは 'assuming drive cache: write through'
        でとまっている。カーソルも止まってだいぶたっている」

私:   「電源を抜いて入れ直したけど、起動しなくなっちゃった」

Claude: 「SDカードをWindowsに挿してください。
          cmdline.txtを編集して起動を復旧します」

私:   「D:\にあるよ」

Claude: 「cmdline.txtに init=/bin/bash rw を追加してください」

私:   「コンソールにroot@(none):/#  が出てるよ」
      ← rootシェル起動成功！

Claude: 「wifi-country.serviceを修正してください」
          → sync → SDカードをWindowsで元に戻す → 復旧完了
```

:::message alert
原因は `systemctl restart dhcpcd` を `ExecStartPost` に入れたことで循環依存が発生してハングしたことでした。systemdサービス内から他のサービスを再起動するのは危険です。
:::

### 手順

1. SDカードをWindowsに挿す（FATパーティションがDドライブなど）
2. `D:\cmdline.txt` をメモ帳で開き、末尾に追加:
   ```
   init=/bin/bash rw
   ```
3. SDカードをrp4に戻して起動 → rootシェルが起動する
4. ファイルを修正:
   ```bash
   # 問題のあるサービスを修正
   vi /etc/systemd/system/wifi-country.service
   sync
   ```
5. SDカードをWindowsに戻して `cmdline.txt` を元に戻す
6. rp4に戻して起動

:::message
`emergency.target` を使う方法もありますが、Raspberry Piはデフォルトでrootアカウントがロックされているため、`init=/bin/bash rw` の方が確実です。
:::

---

## 最終状態の確認

```bash
# OS確認
cat /etc/os-release
# PRETTY_NAME="Raspbian GNU/Linux 12 (bookworm)"

# カーネル確認
uname -m
# aarch64

# 起動時間
systemd-analyze
# Startup finished in 1.3s (kernel) + 21.0s (userspace) = 22.3s

# SSH設定確認
grep -E "Port|PasswordAuthentication|GatewayPorts" /etc/ssh/sshd_config
# Port <your-ssh-port>
# Port <your-secondary-port>
# PasswordAuthentication no
# GatewayPorts no

# サービス確認
sudo systemctl is-active docker fail2ban ssh
# active
# active
# active
```

---

## まとめ・ハマりポイント一覧

| ハマりポイント | 対処 |
|--------------|------|
| libc6/libc-binのバージョン不一致 | キャッシュから`dpkg --force-depends`で強制インストール |
| dev系パッケージの依存エラー | 使っていなければ事前に`--force-depends`で削除 |
| `/boot/firmware`が存在しない | fstabを`/boot/firmware`に更新 |
| Wi-Fi起動時にrfkillでブロック | rc.localでwpa_supplicantをstop→start |
| systemdサービスで循環依存ハング | rc.localを使う。`restart`ではなく`stop→start` |
| 起動時間が2分超え | `connman-wait-online.service`をmask |
| 起動不能 | SDカードをWindowsで直接編集、`init=/bin/bash rw`で復旧 |

---

## おわりに

in-place アップグレードは一筋縄ではいきませんでしたが、なんとか Buster→Bookworm（2世代分）のアップグレードを完了できました。

特に今回学んだのは:
- **`DEBIAN_FRONTEND=noninteractive` + `--force-confold`** の組み合わせが対話プロンプト回避に効果的
- **libc系のバージョン不一致**はin-placeアップグレードあるある。焦らずキャッシュから個別インストール
- **systemdの循環依存**は罠。複雑なことをしたいなら`rc.local`の方が安全
- **`init=/bin/bash rw`** をcmdline.txtに追加するとどんな状態でも復旧できる

クリーンインストールの方が確実ですが、設定を引き継ぎながらアップグレードしたい方の参考になれば幸いです。
