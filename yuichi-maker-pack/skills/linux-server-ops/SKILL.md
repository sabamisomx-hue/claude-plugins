---
name: linux-server-ops
description: Raspberry Piなどの自宅サーバーの構築・常時稼働運用を支援する汎用スキル。ユーザーが「サーバーを立てたい」「ラズパイで常時動かしたい」「cronで定期実行したい」「外から自宅のサーバーにアクセスしたい」「systemdで常駐させたい」「SDカードのバックアップ」「サーバーが落ちてた」など、Linuxサーバーの構築・自動化・リモートアクセス・保守に関する相談をしたら必ず使うこと。OS書き込みからSSH設定、Node.js/Python環境構築、定期実行、常駐化、Tailscale等でのリモートアクセス、バックアップ、監視、セキュリティの基本まで適用する。個別アプリの中身（株分析ロジック→portfolio-review、ハード不調の切り分け→maker-troubleshoot）は各スキルの担当。ユーザーは電気の実務者だがLinuxは独学で、我流の遠回りを減らして定石の運用を身につけたい。
---

# 自宅サーバー運用スキル（Linux / Raspberry Pi）

## ユーザーの環境（回答の前提）

- **サーバー機**: Raspberry Pi 3（WiFi内蔵、旧OctoPrint機を転用）。用途は (1) レシピアプリ等のAPIサーバー役、(2) 株ポートフォリオの朝の定期処理（yfinance取得→Claude Code分析→LINE通知）。
- **他のPi**: Pi 2（予備）、Pi 3B+（エミュレータゲーム機、触らない）。
- **方針**: PCの24時間稼働を避け、常時稼働はPiに集約する。消費電力・静音・省スペース重視。
- **スキルレベル**: 電気・ハードは実務者レベル。Linuxコマンドは使えるが体系的な知識は独学。コピペで動く具体的なコマンドと、「なぜそうするか」の一言があると定着する。

## 大原則

1. **ヘッドレス前提** — モニタ・キーボードは繋がず、最初からSSHで運用する。Raspberry Pi Imagerの事前設定（SSH有効化・WiFi・ユーザー名・ホスト名）を使えば初回起動から画面なしでいける。
2. **手作業を残さない** — 「手で打ったコマンドは再構築時に消える」。セットアップ手順はメモ（Markdown）に残すか、スクリプト化する。SDカードは壊れる前提で、いつでも作り直せる状態を保つのが自宅サーバーの定石。
3. **定時実行はcron、常駐はsystemd** — 「朝1回動かす」はcron、「ずっと待ち受ける（APIサーバー等）」はsystemdサービス。混同すると管理が崩れる。
4. **外部公開はしない** — ルーターのポート開放は使わず、Tailscale等のVPNで自分の端末からだけ届くようにする。自宅サーバーのセキュリティ問題の大半はこれで回避できる。

## 初期構築の標準手順（Pi）

1. **OS書き込み** — Raspberry Pi Imagerで「Raspberry Pi OS Lite (64-bit)」。書き込み前の歯車/カスタマイズ設定で、ホスト名・ユーザー名/パスワード・WiFi（SSID/パスワード/国=JP）・SSH有効化を必ず設定する。ここを飛ばすとモニタが必要になる。
   - 注意: Pi 3（無印）は64-bit対応だがRAM 1GBなので、メモリ重視なら32-bit版も選択肢。Node.jsの新しめのバージョンはARMv7(32bit)向け公式ビルドが細る傾向があるため、基本は64-bitを推奨。
2. **初回接続** — `ssh ユーザー名@ホスト名.local`。繋がらなければルーターの管理画面でIPを確認してIP直打ち。
3. **最初の更新** — `sudo apt update && sudo apt full-upgrade -y` → 再起動。
4. **基本設定** — `sudo raspi-config` でタイムゾーン（Asia/Tokyo）とロケール。cronの実行時刻が狂う原因になるのでタイムゾーンは最初に確認する。
5. **固定IP or ホスト名運用** — ルーター側のDHCP固定割当（MACアドレス紐付け）が最も簡単で確実。Pi側での固定IP設定より、ルーター側でやるのが管理しやすい。

## 環境構築の定石

- **Python** — システムのpipに直接入れず、プロジェクトごとにvenvを作る（`python3 -m venv ~/venvs/株分析` → `source ~/venvs/株分析/bin/activate`）。requirements.txtで再現可能にしておく。Raspberry Pi OS (Bookworm以降)はvenv必須になっているのでこれが標準。
- **Node.js** — aptの標準リポジトリは古いことが多い。NodeSourceのセットアップスクリプトかnvmでLTS版を入れる。Claude Code CLIを動かす場合はその要求バージョンに合わせる。
- **Claude Code CLI** — npmでインストール後、`claude login`でサブスク認証。非対話モード（`claude -p "指示"`）でスクリプトから呼べる。cronから呼ぶ場合は認証情報とPATHが引き継がれるかを最初にテストする（cron環境の罠、下記）。

## cronの罠と定石

cronで「手で動くのにcronだと動かない」は定番中の定番。原因はほぼ環境の違い。

- **PATHが最小限** — コマンドは必ずフルパスで書くか、スクリプト冒頭でPATHを設定する。venvのPythonは `~/venvs/xxx/bin/python` を直接指定すればactivate不要。
- **ホームディレクトリ前提の相対パスが壊れる** — スクリプト内は絶対パスで書く。
- **ログがないと死因がわからない** — 必ずリダイレクトを付ける: `0 8 * * 1-5 /home/user/bin/朝の処理.sh >> /home/user/logs/asa.log 2>&1`
- **編集は `crontab -e`**、確認は `crontab -l`。平日朝8時なら `0 8 * * 1-5`。
- 電源断で実行時刻を逃した場合に備えたいなら anacron や systemd timer（Persistent=true）も選択肢として提示する。

## systemd常駐化のテンプレート

APIサーバー等の常駐プロセスは、以下の型で `/etc/systemd/system/名前.service` を作る:

```ini
[Unit]
Description=レシピAPIサーバー
After=network-online.target
Wants=network-online.target

[Service]
User=ユーザー名
WorkingDirectory=/home/ユーザー名/apps/recipe
ExecStart=/home/ユーザー名/venvs/recipe/bin/python server.py
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

有効化の流れ: `sudo systemctl daemon-reload` → `sudo systemctl enable --now 名前` → 状態確認 `systemctl status 名前` → ログ `journalctl -u 名前 -f`。落ちても自動再起動・起動時に自動開始・ログ統合が手に入る。

## リモートアクセス（外出先から）

- **Tailscale推奨** — 各端末にインストールして同一アカウントでログインするだけでVPNが張れる。無料枠で個人用途は十分。ポート開放不要・グローバルIP不要・CGNAT環境でも動く。スマホアプリからPiのTailscale IPで直接アクセスできる。
- SSHもTailscale経由に限定すれば、パスワード認証のままでも実用上のリスクは大きく下がる（それでも鍵認証化を推奨）。
- **公開が必要になったら**（家族以外にアプリを配る等）はCloudflare Tunnel等を検討するが、それは必要になってから。

## バックアップと SDカード対策

常時稼働のPiで最も壊れるのはSDカード。対策の優先順:

1. **セットアップの再現手順を文書化** — 最強のバックアップは「いつでも作り直せること」。
2. **設定・データの定期バックアップ** — crontab、~/apps、~/venvsのrequirements.txt、systemdユニット、収集データを、定期的に別マシンやクラウドへ（rsyncやrcloneをcronで）。
3. **SDカードの丸ごとイメージ** — 安定稼働に入った時点で一度取得（Win32DiskImager等でPCから）。
4. **書き込み削減** — ログ書き込みを減らすlog2ram導入や、頻繁に書くデータをtmpfsに置くのは、SD延命に有効。
5. **本格運用ならUSB-SSDブート** — Pi 3はUSBブート可能。信頼性を上げたくなったら移行を提案する。

## 監視・ヘルスチェックの基本

- 温度: `vcgencmd measure_temp` / 電源不足検出: `vcgencmd get_throttled`（0x0以外なら電源を疑う→maker-troubleshootへ）
- ディスク: `df -h` / メモリ: `free -h` / 負荷: `htop`
- 「朝の処理が動いたか」を確認する仕組みを最初から入れる — 処理の最後にLINE通知（成功/失敗どちらも）を送れば、通知が来ない=異常と気づける。沈黙は検知できないので「成功も知らせる」のが定石。

## セキュリティの基本線（自宅・VPN前提の現実解）

- パスワードは初期値から変更、可能ならSSH鍵認証化
- `sudo apt update && sudo apt full-upgrade` を月1程度、または unattended-upgrades で自動化
- 使わないサービスは入れない・止める
- ポート開放はしない（Tailscaleで代替）
- APIキー等の秘密情報はコードに直書きせず、環境変数か権限を絞った設定ファイル（chmod 600）に置く

## 他スキルとの連携

- サーバーが不調（落ちる・再起動する・不安定）の原因切り分け → **maker-troubleshoot**（電源・SD・熱の順で疑う）
- Pi上で動かす株処理の中身 → **portfolio-review / stock-analysis**
- Pi上で動かす電子工作系コード → **maker-coding**
