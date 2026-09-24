# Linux/NVIDIA から Apple Silicon Mac/OrbStack への移行

この文書は移行の準備・実施手順です。**この段階では `compose.yml` は変更していません。** 現在の `compose.yml` はLinux/NVIDIA向けのため、そのままMac上で起動しないでください。Mac向けComposeへの切り替えは、本手順を確認した後の別作業です。

## 移行先の方針

- ComposeプロジェクトとGitリポジトリはMac内蔵SSDに置く。
- OrbStackのDocker Engineを使い、操作は通常どおり `docker compose` で行う。
- Immichの機械学習・動画変換はCPU実行とする。Apple SiliconのGPU加速は前提にしない。
- FTPサービスは移行しない。Immichのホスト公開ポート `930` は維持する。
- PostgreSQLはMac内蔵SSD上に新規作成し、Linux側から作った論理DBダンプを復元する。
- 写真・動画は外付けSSDに置く。移行用SSDは一時退避用で、最終保存先とは分ける。
- 旧Linux SSDはext4のままOrbStack Linuxへ渡し、Immichの `UPLOAD_LOCATION` としてComposeから使えるかを先に試す。利用できなければ、退避データを使って旧Linux SSDをAPFSに初期化し、写真・動画を戻す。

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
   - 写真の `UPLOAD_LOCATION` は外付けSSD上、DBの `DB_DATA_LOCATION` はMac内蔵SSD上に設定する。
   - ext4または外付けAPFSボリュームが未接続のとき、空のディレクトリを誤って保存先にしないようにする。保存先がマウント済みであることを確認してからComposeを起動する。
4. `.env` の実ファイルはコミットしない。移行先では `.env.example` を元にローカルの `.env` を作り、パスとDBパスワードを設定する。

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

移行用SSDのコピーと検査が完了したら、Linuxを完全にシャットダウンする。その後、Linux SSDを外付けケースに入れてMacへ接続する。稼働中・サスペンド中にSSDを外さない。

### 5. Mac内蔵SSDにGitリポジトリを取得し、OrbStackを準備する

1. Mac内蔵SSDへ移行用ブランチをcloneまたはpullする。
2. `.env.example` を元にローカルの `.env` を作成する。`DB_DATA_LOCATION` はMac内蔵SSD上の新しい空ディレクトリ、`UPLOAD_LOCATION` は後述の確認結果に応じた外付けSSD上のパスにする。
3. OrbStackを起動し、OrbStackのDocker Engineが選ばれていることを確認する。Docker Desktopが別途ある場合は、誤ったDocker contextを使っていないことも確認する。

### 6. ext4を写真保存先として使えるか事前確認する

1. OrbStackのUSBパススルーで旧Linux SSDをLinux側に接続し、対象パーティションを確認してマウントする。
2. Linux側のマウントパスから、旧Linux上の `/home/yo/immich-server/library` に相当するディレクトリを特定する。旧Linuxの絶対パスを、そのままMacのパスとして指定しない。
3. まず内容を読み取りで確認し、その後、テスト用ディレクトリでComposeのbind mountから読み書きできることを確認する。所有者・権限、OrbStack再起動後の再接続・再マウントも確認する。
4. 確認できた場合だけ、そのLinux側マウントパスを `UPLOAD_LOCATION` に設定する。SSDを接続・マウントしてからImmichを起動し、稼働中に取り外さない。

OrbStackはUSB機器のLinux側パススルーとext4の利用を案内しているが、**ext4のマウントパスがComposeのホスト側bind mountとして使えるかは、実機で確認してから採用する**。

### 7. ext4のbind mountが使えない場合はAPFSへ切り替える

1. 移行用SSDのコピーに加え、`UPLOAD_LOCATION` 全体をMac内蔵SSDにも一時コピーし、容量と内容を確認する。
2. 2つの退避コピーを確認してから、旧Linux SSDをAPFSで初期化する。初期化はSSD上の全データを消去する。
3. 写真・動画などの `UPLOAD_LOCATION` をAPFS SSDへ戻し、`UPLOAD_LOCATION` をMacから見える `/Volumes/...` のパスに設定する。
4. PostgreSQLは外付けへ移さず、Mac内蔵SSD上に新規作成する。

### 8. 新しいDBへ復元し、移行を検証する

1. Mac側に空のDB保存先を用意する。DBイメージとImmichのバージョンは、まず移行元の実稼働版に合わせる。
2. 写真の保存先が正しくマウントされていることを確認してからComposeを起動する。
3. Immichの初回復元画面から最終DBダンプを選び、復元する。旧 `postgres` ディレクトリをDB保存先として指定しない。
4. 起動状態とログ、写真・動画の表示、アセット数、アルバム、検索を確認する。問題がないことを確認してから通常の書き込みを再開する。
5. 移行用SSDは、新環境でバックアップが作成でき、動作確認が完了するまで保持する。

## 秘密情報の扱い

このリポジトリの `.env` は以前からGit追跡対象です。この準備ブランチでは `.env` を追跡対象から外し、`.gitignore` に追加します。ただし、過去のコミットに含まれた値はこの変更では消えません。リモートにpush済みの場合は、DBなどの認証情報を変更してください。認証情報をこの文書や `.env.example` に記載しないでください。

## 参考資料

- [Immich: Backup and Restore](https://docs.immich.app/administration/backup-and-restore/)
- [Immich: Requirements](https://docs.immich.app/install/requirements/)
- [OrbStack: Docker containers and Compose](https://docs.orbstack.dev/docker/)
- [OrbStack: USB devices](https://docs.orbstack.dev/features/usb)
- [OrbStack: Volumes and mounts](https://docs.orbstack.dev/docker/file-sharing)
