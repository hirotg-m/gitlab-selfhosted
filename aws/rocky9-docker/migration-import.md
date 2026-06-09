# GitLab 移行手順（移行先: Rocky Linux 9 / Docker コンテナ版）

Amazon Linux 2023 のネイティブインストール版 GitLab からバックアップを受け取り、Docker コンテナ版 GitLab にリストアする**移行先側の作業手順**です。

## 前提条件

| 項目 | 内容 |
|------|------|
| 移行元 | Amazon Linux 2023 / GitLab EE 19.0.1（rpm） |
| 移行先 | Rocky Linux 9 / GitLab EE 19.0.1（Docker） |
| コンテナ名 | `gitlab` |
| データディレクトリ | `/data/gitlab` |
| バックアップ配置先 | `/data/gitlab/data/backups` |

> ⚠️ **移行元と移行先の GitLab バージョンは一致している必要があります。**

---

## 1. 事前準備

### 1.1 GitLab コンテナが起動済みであること

```bash
sudo docker compose -f /data/gitlab/docker-compose.yml ps
```

まだ構築していない場合は [GitLab Docker インストール手順](./install-gitlab-docker.md) を先に実行してください。

### 1.2 バージョン確認

```bash
sudo docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

移行元と同じ `19.0.1` であることを確認します。

---

## 2. バックアップファイルの配置

移行元から転送されたファイルを所定のディレクトリに配置します。

### 2.1 S3 からダウンロードする場合

```bash
aws s3 cp s3://<bucket-name>/gitlab-migration/<タイムスタンプ>_gitlab_backup.tar /data/gitlab/data/backups/
aws s3 cp s3://<bucket-name>/gitlab-migration/gitlab-secrets.json /data/gitlab/data/backups/
```

### 2.2 権限の設定

バックアップファイルをコンテナ内の git ユーザーが読めるよう権限を設定します。

```bash
sudo chown git:git /data/gitlab/data/backups/<タイムスタンプ>_gitlab_backup.tar
sudo chmod 600 /data/gitlab/data/backups/<タイムスタンプ>_gitlab_backup.tar
```

> 💡 コンテナ内の git ユーザーの UID は通常 `998` です。`chown 998:998` でも可。

---

## 3. gitlab-secrets.json の復元

移行元の暗号化キーがないと DB 内の暗号化データ（CI/CD 変数、2FA キー等）が復号できません。

```bash
# 既存の secrets をバックアップ（念のため）
sudo cp /data/gitlab/config/gitlab-secrets.json /data/gitlab/config/gitlab-secrets.json.bak

# 移行元の secrets を配置
sudo cp /data/gitlab/data/backups/gitlab-secrets.json /data/gitlab/config/gitlab-secrets.json

# reconfigure で反映
sudo docker exec gitlab gitlab-ctl reconfigure
```

---

## 4. リストアの実行

### 4.1 DB 接続プロセスの停止

```bash
sudo docker exec gitlab gitlab-ctl stop puma
sudo docker exec gitlab gitlab-ctl stop sidekiq

# 停止確認
sudo docker exec gitlab gitlab-ctl status
```

### 4.2 リストア

```bash
sudo docker exec -it gitlab gitlab-backup restore BACKUP=<タイムスタンプ>
```

`<タイムスタンプ>` はバックアップファイル名から `_gitlab_backup.tar` を除いた部分です。

例: `1749456000_2026_06_09_19.0.1-ee_gitlab_backup.tar`  
→ `BACKUP=1749456000_2026_06_09_19.0.1-ee`

> ⚠️ `-it` 必須です。確認プロンプトで `yes` を2回入力する必要があります。

---

## 5. リストア後の作業

### 5.1 全サービスの再起動

```bash
sudo docker exec gitlab gitlab-ctl restart
```

### 5.2 ヘルスチェック

```bash
sudo docker compose -f /data/gitlab/docker-compose.yml ps
```

`healthy` になるまで数分待ちます。

---

## 6. 動作確認

### 6.1 ブラウザでの確認

- ログインできること（移行元と同じユーザー・パスワード）
- リポジトリが閲覧できること
- CI/CD 変数が表示できること（secrets 復元の確認）

### 6.2 データ件数の確認

移行元で記録した件数と一致するか確認します。

```bash
sudo docker exec gitlab gitlab-rails runner "puts 'Users: ' + User.count.to_s"
sudo docker exec gitlab gitlab-rails runner "puts 'Projects: ' + Project.count.to_s"
```

### 6.3 Git clone の確認

```bash
git clone ssh://git@<移行先ホスト名>:2222/<namespace>/<project>.git
```

---

## 7. external_url の変更

移行元と URL が異なる場合、`gitlab.rb` を編集します。

```bash
sudo vi /data/gitlab/config/gitlab.rb
```

```ruby
external_url 'https://gitlab.example.com'
```

```bash
sudo docker exec gitlab gitlab-ctl reconfigure
```

---

## 8. 移行後のクリーンアップ

```bash
# バックアップファイルの削除（不要になったら）
sudo rm /data/gitlab/data/backups/<タイムスタンプ>_gitlab_backup.tar
sudo rm /data/gitlab/data/backups/gitlab-secrets.json
```

---

## トラブルシューティング

### Permission denied でリストアが失敗する

バックアップファイルの所有者を確認：

```bash
sudo chown 998:998 /data/gitlab/data/backups/<タイムスタンプ>_gitlab_backup.tar
```

### リストア後にログインできない

`gitlab-secrets.json` が正しく復元されていない可能性があります。再度配置して reconfigure を実行してください。

### DB マイグレーションエラー

バージョンの不一致が原因です。移行元と移行先のバージョンが完全に一致していることを確認してください。

---

## 参考リンク

- [GitLab リストア公式ドキュメント](https://docs.gitlab.com/ee/administration/backup_restore/restore_gitlab.html)
- [GitLab Docker 環境への移行](https://docs.gitlab.com/ee/install/docker/upgrade.html)
