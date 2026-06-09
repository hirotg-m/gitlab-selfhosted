# GitLab 移行手順（移行元: Amazon Linux 2023 / ネイティブインストール）

Amazon Linux 2023 にネイティブインストールされた GitLab EE のデータを、Docker コンテナ版 GitLab へ移行するための、**移行元側の作業手順**です。

## 前提条件

| 項目 | 内容 |
|------|------|
| 移行元 | Amazon Linux 2023 / GitLab EE 19.0.1（rpm） |
| 移行先 | Rocky Linux 9 / GitLab EE 19.0.1（Docker） |
| バックアップ保存先 | `/var/opt/gitlab/backups` |

> ⚠️ **移行元と移行先の GitLab バージョンは一致させてください。**

---

## 1. バージョン確認

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

移行先の Docker コンテナも同じバージョン（19.0.1-ee）で起動してください。

---

## 2. バックアップ取得

### 2.1 GitLab データのバックアップ

```bash
sudo gitlab-backup create
```

バックアップファイルは以下に作成されます：

```
/var/opt/gitlab/backups/<タイムスタンプ>_gitlab_backup.tar
```

### 2.2 設定ファイルのコピー

```bash
sudo cp /etc/gitlab/gitlab.rb /var/opt/gitlab/backups/
sudo cp /etc/gitlab/gitlab-secrets.json /var/opt/gitlab/backups/
```

> ⚠️ `gitlab-secrets.json` は必須です。これがないと DB の暗号化データが復号できません。

### 2.3 確認

```bash
sudo ls -lh /var/opt/gitlab/backups/
```

必要なファイル：

| ファイル | 用途 |
|----------|------|
| `<タイムスタンプ>_gitlab_backup.tar` | リポジトリ・DB・アップロード等 |
| `gitlab.rb` | GitLab 設定（参考用） |
| `gitlab-secrets.json` | 暗号化キー（必須） |

---

## 3. バックアップファイルの転送

移行先サーバへバックアップファイルを転送します。

### 方法 A: scp で直接転送

```bash
scp /var/opt/gitlab/backups/<タイムスタンプ>_gitlab_backup.tar user@<移行先IP>:/data/gitlab/data/backups/
scp /var/opt/gitlab/backups/gitlab-secrets.json user@<移行先IP>:/data/gitlab/data/backups/
scp /var/opt/gitlab/backups/gitlab.rb user@<移行先IP>:/data/gitlab/data/backups/
```

### 方法 B: S3 経由で転送

```bash
# アップロード
aws s3 cp /var/opt/gitlab/backups/<タイムスタンプ>_gitlab_backup.tar s3://<bucket-name>/gitlab-migration/
aws s3 cp /var/opt/gitlab/backups/gitlab-secrets.json s3://<bucket-name>/gitlab-migration/
aws s3 cp /var/opt/gitlab/backups/gitlab.rb s3://<bucket-name>/gitlab-migration/
```

移行先でダウンロード：

```bash
aws s3 cp s3://<bucket-name>/gitlab-migration/<タイムスタンプ>_gitlab_backup.tar /data/gitlab/data/backups/
aws s3 cp s3://<bucket-name>/gitlab-migration/gitlab-secrets.json /data/gitlab/data/backups/
aws s3 cp s3://<bucket-name>/gitlab-migration/gitlab.rb /data/gitlab/data/backups/
```

---

## 4. 移行元での確認事項

移行先でリストアする前に、以下を確認・記録しておきます。

### 4.1 GitLab のバージョン

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

### 4.2 external_url の確認

```bash
grep "^external_url" /etc/gitlab/gitlab.rb
```

> 移行先では URL が変わるため、`gitlab.rb` はそのままリストアせず、`gitlab-secrets.json` のみ使用します。

### 4.3 ユーザー・プロジェクト数の確認（移行後の検証用）

```bash
sudo gitlab-rails runner "puts 'Users: ' + User.count.to_s"
sudo gitlab-rails runner "puts 'Projects: ' + Project.count.to_s"
```

---

## 5. 移行元の停止（任意）

データの整合性を完全に担保したい場合、バックアップ取得前に GitLab を停止します。

```bash
sudo gitlab-ctl stop
sudo gitlab-backup create
sudo gitlab-ctl start
```

通常運用中のバックアップでも問題ありませんが、移行中に書き込みがあると差分が失われます。

---

## 次のステップ

バックアップファイルの転送が完了したら、移行先（Docker 版）でリストアを実行します。

👉 [GitLab バックアップ・リストア手順（Docker コンテナ版）](../rocky9-docker/backup-restore.md) のセクション「3. リストア手順」を参照

移行先でのリストア手順の概要：

1. `gitlab-secrets.json` を `/data/gitlab/config/` に配置
2. `gitlab-ctl reconfigure` を実行
3. `gitlab-backup restore BACKUP=<タイムスタンプ>` を実行
4. ブラウザでログイン・データ確認
