# GitLab EE インストール手順（練習環境 / AWS / Omnibus / Rocky Linux 9）

AWS EC2（Rocky Linux 9）に Omnibus パッケージで
GitLab Enterprise Edition (EE) を直接インストールする手順です。

## 選定理由

- Rocky Linux 9 は GitLab 16.0.0 以降で公式サポート
- OS EOL: 2032年5月（GitLab 25.0.0 までサポート予定）
- SELinux Enforcing のまま問題なく動作する
- 本番環境・練習環境ともに推奨

## 前提条件

| 項目 | 内容 |
|------|------|
| インスタンス | t3.large (2 vCPU / 8 GB) |
| OS | Rocky Linux 9 |
| ホスト名 | `gitlab-dev` |
| ストレージ | ルート 20 GB + データ用 20 GB (gp3) |
| セキュリティグループ | SSH(22), HTTP(80), HTTPS(443), Custom TCP(2222) を許可 |
| Elastic IP | 割り当て済み |
| ドメイン名 | `ドメイン名` |
| データボリューム | `/data` にマウント済み（[EC2 構築手順](../aws-ec2-install.md) セクション5 参照） |

## 想定所要時間

約 30〜60 分

---

## 手順

### 1. SSH 接続

```bash
ssh aaa
```

### 2. SELinux の確認

```bash
# Enforcing であることを確認
getenforce
```

`Enforcing` と表示されること。Rocky Linux 9 では SELinux を有効のまま GitLab を運用します。

### 3. OS の基本設定

```bash
sudo dnf update -y

# ホスト名設定
sudo hostnamectl set-hostname gitlab-dxpf-dev
```

### 4. 必要パッケージのインストール

```bash
sudo dnf install -y curl policycoreutils policycoreutils-python-utils \
  openssh-server openssh-clients perl postfix
sudo systemctl enable --now sshd
sudo systemctl enable --now postfix
```

### 5. ファイアウォール設定

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --reload

# 確認
sudo firewall-cmd --list-all
```

### 6. データディレクトリの準備

データボリュームが `/data` にマウント済みであることを確認し、GitLab 用ディレクトリを作成します。

```bash
# マウント確認
df -h /data

# GitLab データディレクトリ作成
sudo mkdir -p /data/gitlab/git-data
sudo mkdir -p /data/gitlab/backups
```

### 7. GitLab リポジトリの追加とインストール

```bash
# GitLab 公式リポジトリ追加
curl -sS https://packages.gitlab.com/install/repositories/gitlab/gitlab-ee/script.rpm.sh | sudo bash

# GitLab EE インストール（external_url を指定）
sudo EXTERNAL_URL="https://ドメイン名" dnf install -y gitlab-ee
```

※ ドメイン名の DNS が未設定の場合は `EXTERNAL_URL="http://<Elastic IP>"` で仮運用できます。

インストール後、バージョン確認：

```bash
rpm -q gitlab-ee
```

### 8. GitLab の設定（/etc/gitlab/gitlab.rb）

```bash
sudo vi /etc/gitlab/gitlab.rb
```

以下を設定・変更します：

```ruby
# --- 基本設定 ---
external_url 'https://ドメイン名'
gitlab_rails['time_zone'] = 'Asia/Tokyo'

# --- HTTPS（Let's Encrypt） ---
letsencrypt['enable'] = true
letsencrypt['contact_emails'] = ['メールアドレス']
letsencrypt['auto_renew'] = true
letsencrypt['auto_renew_hour'] = 3
letsencrypt['auto_renew_minute'] = 30
nginx['redirect_http_to_https'] = true

# --- メール送信（使用しない） ---
gitlab_rails['smtp_enable'] = false
gitlab_rails['gitlab_email_enabled'] = false

# --- SSH ---
gitlab_rails['gitlab_shell_ssh_port'] = 2222

# --- データ保存先（データボリュームを使用） ---
git_data_dirs({ "default" => { "path" => "/data/gitlab/git-data" } })

# --- バックアップ ---
gitlab_rails['backup_path'] = '/data/gitlab/backups'
gitlab_rails['backup_keep_time'] = 604800

# --- パフォーマンス（8GB メモリ向け調整） ---
puma['worker_processes'] = 2
sidekiq['concurrency'] = 10
postgresql['shared_buffers'] = '512MB'
prometheus_monitoring['enable'] = false
```

### 9. reconfigure の実行

```bash
sudo gitlab-ctl reconfigure
```

※ Rocky Linux 9 では SELinux 関連のエラーは発生しません。

### 10. 初期パスワードの確認

```bash
sudo cat /etc/gitlab/initial_root_password | grep Password:
```

⚠️ このパスワードは 24 時間後に自動削除されます。必ず控えてください。

### 11. 動作確認

```bash
# 全サービスのステータス確認
sudo gitlab-ctl status

# SELinux が Enforcing のまま動作していることを確認
getenforce
```

ブラウザで `https://ドメイン名` にアクセスし、root / 初期パスワード でログインできることを確認します。

### 12. 初期セキュリティ設定（Web UI）

ログイン後、以下を設定します：

1. **root パスワード変更**: Admin Area → Password
2. **サインアップ無効化**: Admin Area → Settings → General → Sign-up restrictions → Sign-up enabled のチェックを外す
3. **2FA 推奨**: Admin Area → Settings → General → Sign-in restrictions → Two-factor authentication を推奨に設定

---

## バックアップ設定

### 手動バックアップ

```bash
sudo gitlab-backup create
```

バックアップは `/data/gitlab/backups/` に保存されます。

### 自動バックアップ（cron）

```bash
sudo crontab -e -u root
```

```cron
# 毎日 AM 2:00 にバックアップ
0 2 * * * /opt/gitlab/bin/gitlab-backup create CRON=1
```

### バックアップに含まれないファイル

以下は別途バックアップが必要です：

```bash
# 設定ファイル（機密情報を含む）
sudo cp /etc/gitlab/gitlab.rb /data/gitlab/backups/
sudo cp /etc/gitlab/gitlab-secrets.json /data/gitlab/backups/
```

---

## アップグレード

```bash
# バックアップ（必ず実行）
sudo gitlab-backup create
sudo cp /etc/gitlab/gitlab.rb /data/gitlab/backups/gitlab.rb.bak
sudo cp /etc/gitlab/gitlab-secrets.json /data/gitlab/backups/gitlab-secrets.json.bak

# アップグレード
sudo dnf update -y gitlab-ee

# 確認
sudo gitlab-ctl status
rpm -q gitlab-ee
```

⚠️ メジャーバージョンのアップグレードは段階的に行う必要があります。
公式の [アップグレードパス](https://docs.gitlab.com/ee/update/index.html#upgrade-paths) を参照。

---

## トラブルシューティング

### 502 エラーが出る

起動直後は内部サービスの初期化に数分かかります。数分待って再アクセスしてください。

```bash
sudo gitlab-ctl status puma
sudo gitlab-ctl tail puma
```

### メモリ不足

```bash
# メモリ使用状況確認
free -h

# スワップ追加（応急措置）
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Let's Encrypt 証明書の手動更新

```bash
sudo gitlab-ctl renew-le-certs
```

### reconfigure が途中で止まる

```bash
# ログ確認
sudo gitlab-ctl tail

# 全体ログ
sudo cat /var/log/gitlab/reconfigure/reconfigure.log
```

### サービス個別の再起動

```bash
sudo gitlab-ctl restart puma
sudo gitlab-ctl restart sidekiq
sudo gitlab-ctl restart postgresql
```

---

## 参考リンク

- [GitLab Omnibus インストール公式ドキュメント](https://docs.gitlab.com/ee/install/install_methods.html)
- [GitLab インストール要件](https://docs.gitlab.com/ee/install/requirements.html)
- [gitlab.rb 設定リファレンス](https://docs.gitlab.com/omnibus/settings/)
- [GitLab バックアップ・リストア](https://docs.gitlab.com/ee/administration/backup_restore/)
- [GitLab サポート OS 一覧](https://docs.gitlab.com/ee/administration/package_information/supported_os.html)
