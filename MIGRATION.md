# Apple Silicon Mac（Colima + LaunchDaemon）での Immich サーバ構築

この文書は Apple Silicon Mac 上での Immich サーバ構築・データ復元・電源投入時自動起動の設計と実施手順をまとめたものである。旧環境は撤去済みで、Linux環境は存在しない。Mac上への新規構築と写真データの復元を主眼とする。

## 構成の決定事項

- **コンテナランタイムは Colima を採用する。** OrbStack は採用しない。理由: OrbStack はGUIアプリがユーザセッションで動くことが前提で、ログインしていない状態でのデーモン起動は unsupported（[Issue #194](https://github.com/orbstack/orbstack/issues/194) で要望のみ）。電源投入後の無操作起動と両立できない。
- **FileVault は内蔵SSDについて ON を維持する。** 起動手順は「電源ON → FileVaultの解除パスワードを1回入力 → 以降はログイン操作なしでサーバが起動」。GUI自動ログイン（kcpassword 追記など）は採用しない。セキュリティを落としたくないため。
- **自動起動は `/Library/LaunchDaemons` の LaunchDaemon（system domain）で行う。** LaunchDaemon はログイン前に起動するため、GUIログイン・自動ログインは不要。
- **Colima/Lima は root では実行できない。** Lima は root 実行を拒否し、Colima も root でのサービス化はサポート対象外（[Discussion #974](https://github.com/abiosoft/colima/discussions/974)）。そこで「LaunchDaemon自体をrootで管理しつつ、Colima起動コマンドは sudo -u で非rootのサービスユーザへ降りる」構成とする。**LaunchDaemon=root管理 / Colima・Dockerプロセス=非rootサービスユーザ（`immich`）**。
- **サービスユーザは `immich`（標準ユーザ、管理者ではない）を作成済み。** システム設定 > ユーザとグループ のGUIから作成してよい。GUIログインはしない。ログインウィンドウに表示したくなければ `sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add immich` で隠せる（任意）。
- Immichの機械学習・動画変換はCPU実行。Apple SiliconのGPU加速は前提にしない。
- FTPサービスは使わない（旧環境で使っていたが廃止）。Immichのホスト公開ポート `930` は維持する。

## 実行環境の実績値（2026-09-26 構築済み）

- Mac: M5 Max（18コア）/ 128GB RAM / macOS 27.0。
- Colima 0.10.3 / docker 29.8.1 / docker-compose 5.5.1（Homebrew）。VMスペックは **4 CPU / 8 GiB / ディスク100 GiB**（`colima list` で確認）。
- VM作成コマンド（外部SSDマウント済みの状態で実行済み）:

  ```
  sudo mkdir -p /opt/immich/run /opt/immich/bin /opt/immich/postgres /opt/immich/secrets
  sudo chown -R immich /opt/immich/run /opt/immich/postgres
  sudo -u immich -H env COLIMA_HOME=/Users/immich/.colima \
    /opt/homebrew/bin/colima start \
    --runtime docker --vm-type vz --cpu 4 --memory 8 --disk 100 \
    --mount /Volumes/database4t:w --mount /opt/immich:w
  ```

- 外付けSSD: `/Volumes/database4t`（APFS、disk5s1、**Volume UUID 852C2D25-3042-48BB-8AB9-AA1D407BF8CE**）、写真フォルダ `immich`。
- Docker socket: `/Users/immich/.colima/default/docker.sock`（サービスユーザHOME配下、管理者からは到達不能=意図通り）。
- Compose実行コピー: `/opt/immich/run/{compose.yml,.env}`（.env は `immich:staff` の `600`）。リポジトリの `compose.yml` と `.env` が原本。
- 書き込み検証済み: コンテナroot・uid999（postgres相当）の両方で `/opt/immich/postgres`、`/opt/immich/run`、`/Volumes/database4t` への書き込みとbind mount動作を確認。
- virtiofsの既知挙動: bind mount上では `chown` がエラーを出さずに無視される（所有者はホスト側のまま）。プレースホルダ遂行のためには問題ない。投入データのアクセスはVM内utilisateurのuid直接書き込みでも通るので問題なく使用できる。
  - 注意: `mount` で `/Volumes/database4t` が `noowners` になっている。所有権の無視が機能している状態。写真データを本置換した際に `sudo vsdbutil -d /Volumes/database4t` で所有権有効化し、データ配下を Immich コンテナから書き込み可能な所有者へ chown するのを推奨。
- DBダンプ: `database/immich-db-backup-20260924T095825-v3.0.2-pg14.19.sql.gz`（旧環境 v3.0.2 時点のもの）。**これを復元に使う。**

## 現在の構成

- `.env`（Git追跡対象外）: `IMMICH_VERSION=v3.0.2`、`UPLOAD_LOCATION=/Volumes/database4t/immich`、`DB_DATA_LOCATION=/opt/immich/postgres`。
- `compose.yml`: Mac用に修正済み（NVIDIA/CUDA/NVENC/ftpd_server 削除、`immich-server` は `immich-machine-learning` ともにCPU実行、Port 930→2283）。
- PostgreSQLイメージは `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`。
- `.env.example`: この compose のパラメータを値のみ置換したテンプレート（DBパスワード除外、実値はローカル.swift)で設定する。

## 手順

### 1. Mac用のComposeを用意する（完了）

1. `compose.yml`（Mac用）と `.env.example` はブランチ `mac-orbstack-migration` にコミット済み。
2. MacBookの（旧Linux用の）`.env` ファイルは削除済み。新Mac側の `.env` は `.env.example` を手元で複製し、DBパスワードを指定する（済み）。

### 2. モバイル・WebアクセスなどのImmichへの書き込みを止めてから始める

アプリ・ブラウザ経由のアップロードはない（既に旧環境停止済み）。以降、写真・DBダンプともにMacからは読み取り専用として扱う。

### 3. 外付けSSDをAPFSの保存先として用意する

1. 写真・動画保存先 `/Volumes/database4t` は APFS でマウント済み。**フォルダ `immich` が保存先**（`.env` の `UPLOAD_LOCATION=/Volumes/database4t/immich`）。
2. 所有権が無視されているので、本データ投入後 `sudo vsdbutil -d /Volumes/database4t` で所有権を有効化し、写真データ配下の所有権・モードを設定する。
3. Spotlightのインデックス走査をOFFにするとサーバ運転が軽い（任意）: `sudo mdutil -i off /Volumes/database4t`。
4. Compose の bind mount から読み書きできることをテストしてから本番データを置く。稼働中に取り外さない。

### 4. Immichサーバを起動してDBダンプを復元する

**DB新規構築**（/opt/immich/postgres は空のまま、Immichが新規起動する）:

```bash
# サービスユーザでColima VMに接続したまま compose up
sudo -u immich -H env COLIMA_HOME=/Users/immich/.colima \
  /opt/homebrew/bin/docker compose --project-directory /opt/immich/run up -d --wait
```

**DBダンプ復元**（Immichの初回復元画面ではなく、Postgres コンテナから直接実行）:

DBダンプは `database/immich-db-backup-20260924T095825-v3.0.2-pg14.19.sql.gz`。これを `/opt/immich/db-restore/` にコピーし、以下の手順で復元する。

```bash
# 空のDBへダンプを投入
gunzip -c /opt/immich/db-restore/immich-db-backup-20260924T095825-v3.0.2-pg14.19.sql.gz | \
  docker exec -i immich_postgres psql -U postgres -d immich
```

そのうえで Immich コンテナのみを再起動して最新DBへ接続させる:

```bash
sudo -u immich -H env COLIMA_HOME=/Users/immich/.colima \
  /opt/homebrew/bin/docker restart immich_server
```

### 5. 動作確認

1. 起動状態とログ（`docker compose logs -f immich-server`）でエラーを確認。
2. ブラウザで `http://<Mac>:930` を開き、管理ユーザは復元したDBに含まれるので、そのアカウントでログインできること。「ユーザー登録（新規ユーザ作成）」画面が出ていないこと（出ていたら、DBの復元が失敗している）。
3. 写真・動画の表示、アセット数、アルバム、検索（全文検索・スマート検索）、ML処理のジョブ（画像認識等）が止まっていないかを確認。
4. モバイルアプリからも `/Volumes/database4t/immich` 自体の写真ビューアとして扱えることを確認。

### 6. 念のためバックアップを作って退避SSDを初期化（任意だが推奨）

1. Immich管理画面 **Administration > Job Queues > Create job > Create Database Dump** で新ダンプを作成。macOS上に退避コピーを保存。
2. 旧環境データ（退避SSDやdump）は、全データ・全アルバムの整合性確認が完了するまで保持してから削除。保持期間を決めておく。

### 7. 電源投入後の自動起動を設置する（LaunchDaemon）

1. **起動スクリプト** `/opt/immich/bin/immich-autostart.sh`（`root:wheel`・`700`）。動作は冪等にする。

   ```bash
   #!/bin/bash
   set -u
   SERVICE_USER=immich
   UPLOAD_UUID=<外付けSSDのVolume UUID>
   UPLOAD_MOUNT=/Volumes/<外付けSSDボリューム>
   COLIMA_HOME=/Users/immich/.colima
   export PATH=/opt/homebrew/bin:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin

   log(){ echo "[$(date '+%F %T')] $*"; }

   while true; do
     # 1) 外付けSSDの存在とマウント
     if ! mount | grep -qF "on $UPLOAD_MOUNT ("; then
       DEV=$(diskutil info "$UPLOAD_UUID" 2>/dev/null | awk '/Device Identifier:/{print $3}')
       if [ -n "$DEV" ]; then
         if diskutil info "$DEV" | grep -q "Encrypted:.*Yes"; then
           cat /opt/immich/secrets/ext-ssd.pw | tr -d '\n' \
             | diskutil apfs unlockVolume "$DEV" -stdinpassphrase >/dev/null 2>&1 \
             || { log "unlock failed; retry"; sleep 30; continue; }
         else
           diskutil mount "$DEV" >/dev/null 2>&1 \
             || { log "mount failed; retry"; sleep 30; continue; }
         fi
         log "external SSD mounted"
       else
         log "external SSD not present; waiting"
         sleep 30; continue
       fi
     fi
     # 2) colima（サービスユーザで実行、rootのまま sudo -u で降りる）
     if ! sudo -u "$SERVICE_USER" -H env COLIMA_HOME="$COLIMA_HOME" \
        /opt/homebrew/bin/colima status >/dev/null 2>&1; then
       sudo -u "$SERVICE_USER" -H env COLIMA_HOME="$COLIMA_HOME" \
         /opt/homebrew/bin/colima start >/dev/null 2>&1 || { log "colima start failed; retry"; sleep 30; continue; }
       log "colima started"
     fi
     # 3) compose up（実行コピーから）
     cd /opt/immich/run
     if sudo -u "$SERVICE_USER" -H env COLIMA_HOME="$COLIMA_HOME" \
        /opt/homebrew/bin/docker compose up -d --wait >/dev/null 2>&1; then
       # 4) ヘルスチェック
       curl -fsS http://localhost:930 >/dev/null 2>&1 \
         && log "immich healthy" || log "warn: immich not answering"
     else
       log "compose up failed"
     fi
     sleep 60
   done
   ```

   - SSDのVolume UUIDは `diskutil info /Volumes/database4t | grep 'Volume UUID'` で取得する（実値: `852C2D25-3042-48BB-8AB9-AA1D407BF8CE`）。スクリプト内ではUUIDから `Device Identifier`（例 `disk5s1`）に解決してから `diskutil mount` / `diskutil apfs unlockVolume` に渡す。
   - 鍵ファイル `/opt/immich/secrets/ext-ssd.pw` は**末尾に改行を含めない**こと（`printf '%s' 'パスフレーズ' > file` で作成）。改行までパスフレーズの一部として扱われ、unlockに失敗する。暗号化しない運用の場合は鍵ファイル不要。
   - Disk Utilityで暗号化ボリュームのパスワードを「システムキーチェーンに保存」した場合、boot時の自動解除・マウントが可能になる（鍵ファイル不要）。挙動はmacOSバージョン依存のため、実機で確認してから鍵ファイル方式と置き換える。
2. **プロパティリスト** `/Library/LaunchDaemons/com.photoserver.immich-autostart.plist`（`root:wheel`・`644`）。

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
   <plist version="1.0">
   <dict>
     <key>Label</key><string>com.photoserver.immich-autostart</string>
     <key>ProgramArguments</key>
     <array><string>/opt/immich/bin/immich-autostart.sh</string></array>
     <key>RunAtLoad</key><true/>
     <key>KeepAlive</key><true/>
     <key>ThrottleInterval</key><integer>30</integer>
     <key>EnvironmentVariables</key>
     <dict>
       <key>PATH</key><string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
     </dict>
     <key>StandardOutPath</key><string>/var/log/immich-autostart.log</string>
     <key>StandardErrorPath</key><string>/var/log/immich-autostart.err.log</string>
   </dict>
   </plist>
   ```
3. 登録とテスト起動。

   ```
   sudo launchctl bootstrap system /Library/LaunchDaemons/com.photoserver.immich-autostart.plist
   sudo launchctl kickstart -k system/com.photoserver.immich-autostart
   tail -f /var/log/immich-autostart.log
   ```
4. 電喪失対策と再起動検証。

   ```
   sudo pmset -a autorestart 1
   sudo pmset -a sleep 0 disksleep 0
   ```

   再起動後、FileVault解除パスワード1回のみで、ログインしないまま サーバが `http://<Mac>:930` に応答することを確認する。ログインウィンドウのまま放置してよい。
5. 有効化解除は `sudo launchctl bootout system/com.photoserver.immich-autostart && sudo kill` 方式で。恒久運用ではlaunchctl項目は上書き実装のため `bootout` した上で plist作成し直し。
6. 不具合調査時は `/var/log/immich-autostart.log` と `.err.log`、`colima list`、`sudo launchctl print system/com.photoserver.immich-autostart` を見る。

## 秘密情報の扱い

`.env` と `/opt/immich/run/.env` を本リポジトリにコミットしない（`.gitignore` に登録済み）。DBパスワード等は過去のコミット履歴に残っている可能性があるため、リモートへpushする場合は認証情報の変更を検討すること。認証情報をこの文書や `.env.example` に記載しない。

外付けSSDを暗号化する場合のUnlock用パスフレーズファイル `/opt/immich/secrets/ext-ssd.pw` は `root:wheel`・`600` とする（現状は非暗号化運用のため未作成）。

## 参考資料

- [Immich: Backup and Restore](https://docs.immich.app/administration/backup-and-restore/)
- [Immich: Requirements](https://docs.immich.app/install/requirements/)
- [Colima: FAQ（自動起動・socket位置・COLIMA_HOME など）](https://colima.run/docs/faq/)
- [Colima Discussion #974: Run in system-space as root](https://github.com/abiosoft/colima/discussions/974)（rootでの起動がCLI側で拒否される件）
- [OrbStack Issue #194: Non-GUI daemon for unattended/headless setup](https://github.com/orbstack/orbstack/issues/194)（OrbStackのheadless起動が非対応であることの裏付け）
- [launchd.info](https://www.launchd.info/)（LaunchDaemon / LaunchAgent の違い・plist書式）
- [Use an External USB SSD before logon (daemon)](https://apple.stackexchange.com/questions/396506/)（ログイン前の外付けボリュームマウントに関する議論）
