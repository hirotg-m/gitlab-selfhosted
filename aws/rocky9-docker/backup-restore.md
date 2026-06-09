# GitLab バックアップ・リストア手順（Docker コンテナ版）

Docker コンテナで稼働する GitLab EE のバックアップ取得とリストア手順です。

## 前提条件

| 項目 | 内容 |
|------|------|
| GitLab | Docker コンテナで稼働中（EE） |
| コンテナ名 | `gitlab` |
| データディレクトリ | `/data/gitlab` |
| バックアップ保存先 | `/data/gitlab/data/backups` |
| レジストリ blob 保存先 | `/data/gitlab/data/gitlab-rails/shared/registry/` |
| レジストリ メタデータ | PostgreSQL `registry` DB（database.enabled: prefer） |

## バックアップ対象

| 対象 | バックアップ方法 | `gitlab-backup` に含まれるか |
|------|------|:---:|
| リポジトリ・DB・アップロードファイル等 | `gitlab-backup create` | ✅ |
| レジストリ イメージ blob（実体） | `gitlab-backup create`（registry.tar.gz） | ✅ |
| レジストリ メタデータ（タグ情報等） | `gitlab-psql` で registry DB をダンプ | ❌ |
| 設定ファイル（gitlab.rb, gitlab-secrets.json） | 手動コピー | ❌ |

> ⚠️ **重要: `gitlab-backup create` に含まれないもの**
>
> | データ | 欠落時の影響 |
> |--------|-------------|
> | `gitlab.rb` / `gitlab-secrets.json` | DB の暗号化データが復号不可 |
> | `registry` DB（タグ等のメタデータ） | レジストリのタグが表示されない（blob は存在しても参照不可） |
>
> これらは**必ず別途バックアップ**してください。

---

## 1. 手動バックアップ

### 1.1 GitLab データのバックアップ

```bash
sudo docker exec gitlab gitlab-backup create
```

バックアップファイルは以下に作成されます：

```
/data/gitlab/data/backups/<タイムスタンプ>_gitlab_backup.tar
```

### 1.2 設定ファイルのバックアップ

```bash
sudo cp /data/gitlab/config/gitlab.rb /data/gitlab/data/backups/
sudo cp /data/gitlab/config/gitlab-secrets.json /data/gitlab/data/backups/
```

### 1.3 コンテナレジストリ DB のバックアップ

コンテナレジストリのタグ情報は PostgreSQL の `registry` DB に保存されており、`gitlab-backup` には含まれません。

```bash
# /tmp に出力後コピー（backups ディレクトリへの直接書き込みは権限エラーになるため）
sudo docker exec gitlab gitlab-psql -d registry -c "\copy (SELECT * FROM tags) TO '/tmp/registry_tags.csv' CSV HEADER"
sudo docker cp gitlab:/tmp/registry_tags.csv /data/gitlab/data/backups/
```

> 💡 上記は最低限（tags テーブル）のみです。全テーブルをバックアップする場合は
> `gitlab-psql -d registry -c "\dt"` でテーブル一覧を確認してください。

### 1.4 バックアップの確認

```bash
sudo ls -lh /data/gitlab/data/backups/
```

---

## 2. 定期バックアップ（cron）

### バックアップスクリプト

cron に長いコマンドを直接書くと保守しづらいため、スクリプト化します。

```bash
sudo vi /usr/local/bin/gitlab-backup.sh
```

```bash
#!/bin/bash
set -e

BACKUP_DIR="/data/gitlab/data/backups"

# GitLab データバックアップ
docker exec gitlab gitlab-backup create CRON=1

# 設定ファイル
cp /data/gitlab/config/gitlab.rb "$BACKUP_DIR/"
cp /data/gitlab/config/gitlab-secrets.json "$BACKUP_DIR/"

# レジストリ DB ダンプ
docker exec gitlab rm -f /tmp/registry_tags.csv
docker exec gitlab gitlab-psql -d registry -c "\copy (SELECT * FROM tags) TO '/tmp/registry_tags.csv' CSV HEADER"
docker cp gitlab:/tmp/registry_tags.csv "$BACKUP_DIR/"
```

```bash
sudo chmod +x /usr/local/bin/gitlab-backup.sh
```

### cron 設定

毎日 12:00 に実行：

```bash
sudo vi /etc/cron.d/gitlab-backup
```

```cron
0 12 * * * root /usr/local/bin/gitlab-backup.sh >> /var/log/gitlab-backup.log 2>&1
```

### バックアップの世代管理

`gitlab.rb` に以下を追加し、7日を超えたバックアップを自動削除します：

```bash
sudo vi /data/gitlab/config/gitlab.rb
```

```ruby
gitlab_rails['backup_keep_time'] = 604800
```

```bash
sudo docker exec gitlab gitlab-ctl reconfigure
```

---

## 3. リストア手順

### 3.1 前提

- リストア先の GitLab コンテナが起動済みであること
- リストア先の GitLab が**同じバージョン**であること

バージョン確認：

```bash
sudo docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

### 3.2 設定ファイルの復元

```bash
sudo cp /data/gitlab/data/backups/gitlab.rb /data/gitlab/config/gitlab.rb
sudo cp /data/gitlab/data/backups/gitlab-secrets.json /data/gitlab/config/gitlab-secrets.json

sudo docker exec gitlab gitlab-ctl reconfigure
```

### 3.3 リストアの実行

```bash
# DB 接続プロセスを停止
sudo docker exec gitlab gitlab-ctl stop puma
sudo docker exec gitlab gitlab-ctl stop sidekiq

# 停止確認
sudo docker exec gitlab gitlab-ctl status

# リストア実行（ファイル名からタイムスタンプ部分を指定）
# ※ -it 必須（確認プロンプトで yes を入力する必要があるため）
sudo docker exec -it gitlab gitlab-backup restore BACKUP=<タイムスタンプ>
```

`<タイムスタンプ>` はバックアップファイル名から `_gitlab_backup.tar` を除いた部分です。  
例: `1780969837_2026_06_09_19.0.1-ee` → `BACKUP=1780969837_2026_06_09_19.0.1-ee`

> ⚠️ `-it` を付けないと確認プロンプトへの入力ができず、エラーで中断します。

### 3.4 コンテナレジストリ DB のリストア

```bash
sudo docker cp /data/gitlab/data/backups/registry_tags.csv gitlab:/tmp/
sudo docker exec gitlab gitlab-psql -d registry -c "DELETE FROM tags;"
sudo docker exec gitlab gitlab-psql -d registry -c "\copy tags FROM '/tmp/registry_tags.csv' CSV HEADER"
```

### 3.5 リストア後の確認

```bash
# 全サービスの再起動
sudo docker exec gitlab gitlab-ctl restart

# レジストリ blob の復元確認
sudo du -sh /data/gitlab/data/gitlab-rails/shared/registry/

# タグ数の確認
sudo docker exec gitlab gitlab-psql -d registry -c "SELECT count(*) FROM tags;"
```

ブラウザでログインし、以下を確認します：
- リポジトリ・ユーザー・設定が復元されていること
- コンテナレジストリのタグが表示されていること

---

## 4. バックアップの外部保存（推奨）

ディスク障害に備え、バックアップを外部に退避します。

### S3 へ転送する例

```bash
aws s3 sync /data/gitlab/data/backups/ s3://<bucket-name>/gitlab-backups/
```

### 別サーバへ rsync する例

```bash
rsync -avz /data/gitlab/data/backups/ user@backup-server:/backup/gitlab/
```

---

## 5. 災害復旧（新規 EC2 からの復元）

EC2 が完全に失われた場合の復旧手順です。

1. [EC2 環境構築手順](./aws-ec2-install.md) に従い新規 EC2 を構築
2. [GitLab Docker インストール手順](./install-gitlab-docker.md) に従い GitLab コンテナを起動
3. 外部保存先からバックアップファイルを取得：
   ```bash
   aws s3 cp s3://<bucket-name>/gitlab-backups/<タイムスタンプ>_gitlab_backup.tar /data/gitlab/data/backups/
   aws s3 cp s3://<bucket-name>/gitlab-backups/gitlab.rb /data/gitlab/data/backups/
   aws s3 cp s3://<bucket-name>/gitlab-backups/gitlab-secrets.json /data/gitlab/data/backups/
   aws s3 cp s3://<bucket-name>/gitlab-backups/registry_tags.csv /data/gitlab/data/backups/
   ```
4. 本手順「3. リストア手順」を実行

---

## 6. 注意事項

| 項目 | 内容 |
|------|------|
| `-it` オプション | リストア時は `docker exec -it` が必須。付けないと確認プロンプトでエラー終了する |
| registry DB | `gitlab-backup` にはレジストリの blob のみ含まれ、タグ情報（メタデータ）は含まれない |
| バージョン一致 | リストア先の GitLab は**バックアップ元と同じバージョン**が必要 |
| GC（ガベージコレクション） | database モード（`database.enabled: prefer`）では従来の `registry garbage-collect` コマンドは使用不可。オンライン GC が自動実行される |
| バックアップ出力先 | registry DB のダンプは `/tmp` 経由でコピーする（`/var/opt/gitlab/backups/` は権限エラーになる） |

---

## 参考リンク

- [GitLab バックアップ公式ドキュメント](https://docs.gitlab.com/ee/administration/backup_restore/backup_gitlab.html)
- [GitLab リストア公式ドキュメント](https://docs.gitlab.com/ee/administration/backup_restore/restore_gitlab.html)
