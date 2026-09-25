# Linux/NVIDIA から Apple Silicon Mac（Colima + LaunchDaemon）への移行

この文書は移行の準備・実施手順と、移行後の電源投入時自動起動の設計をまとめたものである。**この段階では `compose.yml` は変更していない。** 現在の `compose.yml` はLinux/NVIDIA向けのため、そのままMac上で起動しないこと。Mac向けComposeへの切り替えは、本手順を確認した後の別作業である。

## 決定事項（2026-09-25）

- **コンテナランタイムは Colima を採用する。** OrbStack は採用しない。理由: OrbStack はGUIアプリがユーザセッションで動くことが前提で、ログインしていない状態でのデーモン起動は unsupported（[Issue #194](https://github.com/orbstack/orbstack/issues/194) で要望のみ）。電源投入後の無操作起動と両立できない。
- **FileVault は内蔵SSDについて ON を維持する。** 起動手順は「電源ON → FileVaultの解除パスワードを1回入力 → 以降はログイン操作なしでサーバが起動」。GUI自動ログイン（kcpassword 追記など）は採用しない。セキュリティを落としたくないため。
- **自動起動は `/Library/LaunchDaemons` の LaunchDaemon（system domain）で行う。** LaunchDaemon はログイン前に起動するため、GUIログイン・自動ログインは不要。
- **Colima/Lima は root では実行できない。** Lima は root 実行を拒否し、Colima も root でのサービス化はサポート対象外（[Discussion #974](https://github.com/abiosoft/colima/discussions/974)）。そこで「LaunchDaemon自体をrootで管理しつつ、Colima起動コマンドは sudo -u で非rootのサービスユーザへ降りる」構成とする。「Immichをroot権限で動かす」という当初案は、この制約により **LaunchDaemon=root管理 / Colima・Dockerプロセス=非rootサービスユーザ** に修正する。
- Immichの機械学習・動画変換はCPU実行。Apple SiliconのGPU加速は前提にしない。
- FTPサービス（`ftpd_server`）は移行しない。Immichのホスト公開ポート `930` は維持する。

## 自動起動の全体設計

```
電源ON
 └ FileVault 解除画面（パスワード1回入力 = 唯一の操作）
    └ macOS 起動（GUIログインのまま放置してよい）
       └ LaunchDaemon (root, RunAtLoad) が /opt/immich/bin/immich-autostart.sh を起動
          1. 外付けSSDが現れるまで待機 → diskutil でマウント（未接続なら空ディレクトリで起動せず待機）
          2. colima start（サービスユーザで実行、冪等。既に起きていればスキップ）
          3. docker compose up -d --wait（/opt/immich/run の compose + .env）
          4. curl で localhost:930 を確認
          5. sleep して再チェック（KeepAlive と合わせ常時監視・復旧）
```

- 加電喪失後の自動復帰のため `sudo pmset -a autorestart 1` を設定しておく。
- サーバ用途のため `sudo pmset -a sleep 0 disksleep 0` でスリープを無効化する。

## 移行先の方針

- ComposeプロジェクトとGitリポジトリはMac内蔵SSDに置く。
- 自動起動用のランタイム配置（スクリプト・composeの実行コピー・.env実ファイル）は `/opt/immich/` 配下に置き、所有者はサービスユーザとする。Gitリポジトリは管理用の本体とする。
- Docker Engine は Colima を使い、操作は通常どおり `docker compose` で行う。Docker socket は `~/.colima/default/docker.sock`（サービスユーザのHOME配下）。
- Immichの機械学習・動画変換はCPU実行とする。Apple SiliconのGPU加速は前提にしない。
- PostgreSQLはMac内蔵SSD（`/opt/immich/postgres`）上に新規作成し、Linux側から作った論理DBダンプを復元する。
- 写真・動画は外付けSSDに置く。移行用SSDは一時退避用で、最終保存先とは分ける。
- 旧Linux SSDをext4のまま使う案は採用しない。macOSはext4をマウントできず、OrbStack固有のUSBパススルー（Linux側マウントをthinfs経由で共有する仕組み）はColima/Limaには同等機能がないため。旧Linux SSDは退避コピーの照合後にAPFSで初期化し、写真・動画を戻す。
- 外付けSSDはAPFSとする。exFATは権限・所有者が保持されないため最終保存先に使わない。
- 外付けSSDの暗号化: APFS暗号化でボリュームを作成し、Unlock用パスフレーズをrootのみ読める鍵ファイル（`/opt/immich/secrets/` 配下、mode 600）に置いて `diskutil apfs unlockVolume ... -stdinpassphrase` する案を基本とする。**未検証項目なので実機で必ず確認すること。** 確認できなければ、物理盗难リスクを受け入れて非暗号化とするか、運用方針を見直す（FileVaultの自動解除キーはGUIログインユーザのキーチェーンに紐づくため、ログイン前にカジュアルに解除できる仕組みは keycode を保持するしかない）。

## 現在の構成で確認できていること

- `.env` の設定値は `IMMICH_VERSION=v3.0.2`。
- `UPLOAD_LOCATION=/home/yo/immich-server/library`。
- `DB_DATA_LOCATION=/home/yo/immich-server/postgres`。
- PostgreSQLイメージは `ghcr.io/immich-app/postgres:14-vectorchord0.4.3-pgvectors0.2.0`。
- 現行ComposeはNVIDIA/NVENCとCUDAを指定している。`.env` にもNVENCの設定がある。
- 確認できる最新の既存DBバックアップは `immich-db-backup-20260621T020000-v2.7.5-pg14.19.sql.gz`。設定上の `v3.0.2` より古いため、移行用バックアップには使わず、移行直前に現行稼働版から作り直す。

これらはリポジトリとファイル名から分かる設定・記録であり、実際の稼働バージョンや最新のデータ状態は移行開始時にLinux側で確認する。

## 手順

### 1. Mac移行用の文書・設定をGitブランチへ用意する

1. この文書と `.env.example` を確認する。
2. Mac用Composeを作成する段階では、Immichの稼働バージョンに対応した公式Composeを基準にする。移行・復元が完了するまでは、移行元で実際に稼働しているバージョンに合わせる。
3. Mac用Composeでは少なくとも次を反映する。
   - `runtime: nvidia`、`NVIDIA_*`、NVENC設定、MLイメージの `-cuda` 指定を除く。
   - 機械学習・動画変換をCPU設定にする。
   - `ftpd_server` とFTP用の設定を除く。
   - 写真の `UPLOAD_LOCATION` は外付けSSD上、DBの `DB_DATA_LOCATION` は `/opt/immich/postgres`（内蔵SSD）とする。
   - ext4または外付けAPFSボリュームが未接続のとき、空のディレクトリを誤って保存先にしないようにする。保存先がマウント済みであることを確認してからComposeを起動する。
4. `.env` の実ファイルはコミットしない。移行先では `.env.example` を元にローカルの `.env` を作り、パスとDBパスワードを設定する。自動起動用に `/opt/immich/run/.env` としてコピーし、所有者・権限は `immich:immich`・`600` にする。

この文書の追加時点では、現行LinuxのComposeファイルをMac用に置き換えない。

### 2. Linux側で最終DBバックアップを作成する

1. Immichの実稼働バージョン、PostgreSQLの状態、`UPLOAD_LOCATION` が載っているパーティションを確認する。SSDのパーティション・ファイルシステムは `lsblk -f` や `findmnt` 等で記録する。LUKSやLVMを使っている場合は、Mac側で読み出すために必要な解除・有効化手順も確認する。
2. モバイルアップロードやWebからの変更を止める。
3. Immich管理画面の **Administration > Job Queues > Create job > Create Database Dump** からDBダンプを作成し、完了を確認する。
4. DBダンプの作成後にImmichへの書き込みを停止し、Immichのコンテナを正常停止する。Linuxはまだ起動したままにして、停止後にファイルを移行用SSDへコピーする。

### 3. DBダンプと写真データを別の移行用SSDへ複製する

1. 移行用SSDに、`UPLOAD_LOCATION` 全体とDBダンプを置ける空き容量があることを確認する。
2. `UPLOAD_LOCATION`（現在の設定では `/home/yo/immich-server/library`）の**中身全体**をコピーする。`backups`、`encoded-video`、`library`、`profile`、`thumbs`、`upload` などを含める。最終DBダンプがこの中に作成されている場合も、コピー先に含まれていることを確認する。
3. 移行用SSDは一時運搬用としてLinuxとMacの両方から扱える形式にする。exFATを使う場合は、Linuxの所有者・権限情報がそのまま保持される前提にせず、最終保存先でImmichが読み書きできることを確認する。FAT32は大容量ファイルの制限があるため使わない。
4. コピー後にファイル数・容量を照合し、DBダンプの圧縮ファイルが検査できることを確認する。移行用SSDのコピーは、旧Linux SSDの初期化が必要になった場合の退避データとして保持する。

**PostgreSQLの生データディレクトリ `/home/yo/immich-server/postgres` は、MacのDBとしてコピー・再利用しない。** DBは新しいMac側のPostgreSQLへ論理ダンプから復元する。

### 4. Linux SSDを外し、Macへ接続する

移行用SSDのコピーと検査が完了したら、Linuxを完全にシャットダウンする。その後、Linux SSDを外付けケースに入れてMacへ接続する。稼働中・サスペンド中にSSDを外さない。このSSDはデータ取り込み用として一時的に使うだけで、旧データが移行できたらAPFSで初期化する（手順6）。

### 5. MacにColima環境を用意する

手順5では環境の用意までを行い、Colima初回起動（手順5-4）は**手順6で外付けSSDをAPFS化・マウントした後に行う**こと。順序が逆になるとmounts設定が不正なままVMが作られる。

1. Homebrewで次をインストールする。

   ```
   brew install colima docker docker-compose
   ```

   `docker compose` サブコマンドを使えるようにするため、Homebrewのdocker-composeをサービスユーザのプラグインとしてリンクする（**下の3. でサービスユーザ `immich` を作成した後に実施**）。

   ```
   sudo -u immich -H mkdir -p /Users/immich/.docker/cli-plugins
   sudo -u immich -H ln -sfn /opt/homebrew/opt/docker-compose/bin/docker-compose \
     /Users/immich/.docker/cli-plugins/docker-compose
   ```

2. Docker Desktopが入っている場合、Docker context を Colima に切り替えるか、DOCKER_HOST を設定してColimaのsocketを使う。socketは `~/.colima/default/docker.sock`。
3. **サービスユーザ（例: `immich`）を作成する。** 通常（非管理者）ユーザでよい。自動ログインは設定しない。GUIログインもしない。
   - GUIで作成する場合は システム設定 > ユーザとグループ から。CLIの場合は `sudo sysadminctl -addUser immich -password <パスワード>`。
4. サービスユーザ配下にColimaのVM設定を固定するため、初回起動はサービスユーザとして実行してVMを作成する。**外付けSSDをAPFS化してマウントした状態（手順6が完了した状態）で実行すること。** Colimaは起動時にmountsへ未マウントのパスを登録できず、後から `colima stop && colima start --edit` で追加し直す必要がある。

   ```
   sudo mkdir -p /opt/immich/run /opt/immich/bin /opt/immich/postgres /opt/immich/secrets
   sudo chown -R immich /opt/immich/run /opt/immich/postgres
   sudo -u immich -H env COLIMA_HOME=/Users/immich/.colima \
     /opt/homebrew/bin/colima start \
     --runtime docker --vm-type vz --cpu 4 --memory 8 --disk 100 \
     --mount /Volumes/<外付けSSDボリューム>:w --mount /opt/immich:w
   ```

   - `/opt/immich`（compose実行コピー用・DB用）と外付けSSDボリュームをColimaのmountsに含めることが必須。含めていないパスはbind mountが空になる。
   - `--mount` には `:w` を付けて書き込み可能にする。既定は読み取り専用。
   - `Colima` のconfigはサービスユーザの `COLIMA_HOME`（`/Users/immich/.colima`）配下に作られる。**通常ユーザで `colima start` してVM設定が二重に作られないこと。**
   - `-H` と `COLIMA_HOME` を明示するのは、launchd から起動されたプロセスには HOME や設定ディレクトリが解決されないケースがあるため。以降の `sudo -u immich ... colima` 実行はすべてこの形で統一する。
5. 起動確認: `sudo -u immich -H env COLIMA_HOME=/Users/immich/.colima /opt/homebrew/bin/colima list` と、socket（`/Users/immich/.colima/default/docker.sock`）での `docker ps` が通ること。

### 6. 外付けSSDをAPFSの保存先として用意する

1. 旧Linux SSD（ext4）はmacOSから直接読めない。移行用SSDコピーから旧バックアップ相当の安全性を確認したうえで、旧Linux SSDをAPFSで初期化する。初期化はSSD上の全データを消去する。
2. APFS化したSSDに写真・動画などの `UPLOAD_LOCATION` 全体を戻す。`UPLOAD_LOCATION` を `/Volumes/...` パスに設定する。
3. 外付けSSDの暗号化を行う場合は、ボリュームのパスフレーズを新しい鍵ファイルに保存し、rootのみ参照可能にする（`/opt/immich/secrets/` 配下、`root:wheel`・`600`）。
4. Composeのbind mountから読み書きできることをテストしてから本番データを置く。稼働中に取り外さない。

### 7. 新しいDBへ復元し、移行を検証する

1. Mac側に空のDB保存先 `/opt/immich/postgres` を用意する。所有者はサービスユーザ。
2. 写真の保存先が正しくマウントされていることを確認してからComposeを起動する。通常ユーザで運用チェックする段階では、`.env` と compose を `/opt/immich/run` 配下に置いてサービスユーザから起動する。
3. Immichの初回復元画面から最終DBダンプを選び、復元する。旧 `postgres` ディレクトリをDB保存先として指定しない。
4. 起動状態とログ、写真・動画の表示、アセット数、アルバム、検索を確認する。問題がないことを確認してから通常の書き込みを再開する。
5. 移行用SSDは、新環境でバックアップが作成でき、動作確認が完了するまで保持する。

### 8. 電源投入後の自動起動を設置する（LaunchDaemon）

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

   - SSDのVolume UUIDは `diskutil info /Volumes/<ボリューム> | grep 'Volume UUID'` で取得する。スクリプト内ではUUIDから `Device Identifier`（例 `disk5s2`）に解決してから `diskutil mount` / `diskutil apfs unlockVolume` に渡す。
   - 鍵ファイル `/opt/immich/secrets/ext-ssd.pw` は**末尾に改行を含めない**こと（`printf '%s' 'パスフレーズ' > file` で作成）。改行までパスフレーズの一部として扱われ、unlockに失敗する。
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

このリポジトリの `.env` は以前からGit追跡対象です。この準備ブランチでは `.env` を追跡対象から外し、`.gitignore` に追加します。ただし、過去のコミットに含まれた値はこの変更では消えません。リモートにpush済みの場合は、DBなどの認証情報を変更してください。認証情報をこの文書や `.env.example` に記載しないでください。

自動起動用の `/opt/immich/run/.env` も本リポジトリにはコミットしない。外付けSSDのUnlock用パスフレーズファイル `/opt/immich/secrets/ext-ssd.pw` は `root:wheel`・`600` とする。

## 参考資料

- [Immich: Backup and Restore](https://docs.immich.app/administration/backup-and-restore/)
- [Immich: Requirements](https://docs.immich.app/install/requirements/)
- [Colima: FAQ（自動起動・socket位置・COLIMA_HOME など）](https://colima.run/docs/faq/)
- [Colima Discussion #974: Run in system-space as root](https://github.com/abiosoft/colima/discussions/974)（rootでの起動がCLI側で拒否される件）
- [OrbStack Issue #194: Non-GUI daemon for unattended/headless setup](https://github.com/orbstack/orbstack/issues/194)（OrbStackのheadless起動が非対応であることの裏付け）
- [launchd.info](https://www.launchd.info/)（LaunchDaemon / LaunchAgent の違い・plist書式）
- [Use an External USB SSD before logon (daemon)](https://apple.stackexchange.com/questions/396506/)（ログイン前の外付けボリュームマウントに関する議論）
