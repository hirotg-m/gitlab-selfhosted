# GitLab インストール手順（Rocky Linux 9 / Docker コンテナ）

Rocky Linux 9 上に Docker コンテナで GitLab EE をインストールする手順です。

> ※ ライセンスキーを登録しない場合、EE は Free tier（CE と同等の機能）で動作します。

## 前提条件

| 項目 | 内容 |
|------|------|
| OS | Rocky Linux 9 |
| メモリ | 8 GB 以上推奨（最低 4 GB） |
| ストレージ | 50 GB 以上 |
| Docker | 本手順内でインストール |
| ネットワーク | ポート 80, 443, 2222, 5050 が使用可能であること（セキュリティグループで許可済み） |
| SELinux | enforcing（デフォルト） |
| データ領域 | `/data` に追加 EBS がマウント済みであること（[EC2 構築手順](./aws-ec2-install.md) 参照） |

## 想定所要時間

約 30〜60 分（初回イメージ pull 含む）

---

## 1. Docker のインストール

既に Docker がインストール済みの場合はスキップしてください。

```bash
# Podman の削除（競合防止）
sudo dnf remove -y podman podman-docker buildah skopeo

# Docker リポジトリ追加
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# Docker インストール
sudo dnf install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# 起動・自動起動有効化
sudo systemctl enable --now docker

# バージョン確認
docker --version
```

---

## 2. SELinux の設定

Rocky Linux 9 では SELinux が enforcing です。Docker ボリュームマウントで問題が発生しないよう、`container_file_t` コンテキストを設定します。

```bash
# 現在の SELinux 状態を確認
getenforce

# /data 配下全体に SELinux コンテキストを設定
sudo semanage fcontext -a -t container_file_t "/data(/.*)?"
sudo restorecon -Rv /data
```

> ※ `semanage` がない場合は以下でインストール:
>
> ```bash
> sudo dnf install -y policycoreutils-python-utils
> ```

---

## 3. Docker のデータルート変更

Docker イメージのキャッシュもルートボリューム（20 GiB）を圧迫するため、`/data` 配下に変更します。

```bash
sudo mkdir -p /data/docker
sudo vi /etc/docker/daemon.json
```

```json
{
  "data-root": "/data/docker"
}
```

```bash
sudo systemctl restart docker
```

---

## 4. データ永続化用ディレクトリの作成

[EC2 構築手順](./aws-ec2-install.md) で既に作成済みの場合はスキップしてください。

```bash
sudo mkdir -p /data/gitlab/{config,logs,data}
```

| ディレクトリ | 用途 |
|---|---|
| `/data/gitlab/config` | GitLab 設定ファイル（gitlab.rb 等） |
| `/data/gitlab/logs` | ログファイル |
| `/data/gitlab/data` | リポジトリ・DB 等のデータ |

---

## 5. GitLab コンテナの起動

### 5.1 docker compose ファイルの作成

```bash
sudo vi /data/gitlab/docker-compose.yml
```

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ee:latest
    container_name: gitlab
    restart: always
    hostname: gitlab.example.com
    environment:
      GITLAB_OMNIBUS_CONFIG: |
        # HTTPS (Let's Encrypt)
        external_url 'https://gitlab.example.com'
        letsencrypt['enable'] = true
        letsencrypt['contact_emails'] = ['admin@example.com']
        letsencrypt['auto_renew'] = true
        # SSH
        gitlab_rails['gitlab_shell_ssh_port'] = 2222
        # コンテナレジストリ（同一ドメイン・ポート 5050）
        registry_external_url 'https://gitlab.example.com:5050'
        registry_nginx['listen_port'] = 5050
        registry_nginx['listen_https'] = true
    ports:
      - "80:80"
      - "443:443"
      - "2222:22"
      - "5050:5050"
    volumes:
      - /data/gitlab/config:/etc/gitlab
      - /data/gitlab/logs:/var/log/gitlab
      - /data/gitlab/data:/var/opt/gitlab
    shm_size: "256m"
```

> **Let's Encrypt の前提条件:**
>
> - ポート 80 がインターネットからアクセス可能であること
> - `gitlab.example.com` が DNS で EC2 のパブリック IP（Elastic IP）に解決できること
> - コンテナレジストリの SSL 証明書は Let's Encrypt が自動取得・更新します

### 5.2 コンテナの起動

```bash
cd /data/gitlab
sudo docker compose up -d
```

初回はイメージの pull に数分かかります。

---

## 6. 起動確認

```bash
# コンテナの状態確認（healthy になるまで数分待つ）
sudo docker compose ps

# ログの確認
sudo docker compose logs -f gitlab
```

`healthy` になるまで 3〜5 分程度かかります。

ブラウザで `https://gitlab.example.com` にアクセスし、GitLab のログイン画面が表示されることを確認します。

> ※ HTTP (80) でアクセスした場合は HTTPS に自動リダイレクトされます。

---

## 7. 初期パスワードの確認

root ユーザーの初期パスワードはコンテナ内に自動生成されます。

```bash
sudo docker exec gitlab cat /etc/gitlab/initial_root_password
```

> ※ このファイルは初回起動後 24 時間で自動削除されます。早めに控えてください。

ブラウザで以下の情報でログインします：

- ユーザー名: `root`
- パスワード: 上記コマンドで確認した値

ログイン後、速やかにパスワードを変更してください。

---

## 8. GitLab の設定変更

設定を変更する場合は `/data/gitlab/config/gitlab.rb` を編集し、再設定を適用します。

```bash
# 設定ファイルの編集
sudo vi /data/gitlab/config/gitlab.rb

# 設定の適用（コンテナ内で reconfigure を実行）
sudo docker exec gitlab gitlab-ctl reconfigure
```

---

## 9. コンテナの管理

```bash
# 停止
sudo docker compose -f /data/gitlab/docker-compose.yml down

# 起動
sudo docker compose -f /data/gitlab/docker-compose.yml up -d

# 再起動
sudo docker compose -f /data/gitlab/docker-compose.yml restart

# GitLab のバージョン確認
sudo docker exec gitlab gitlab-rake gitlab:env:info
```

---

## 10. GitLab のアップデート

```bash
cd /data/gitlab

# 最新イメージの取得
sudo docker compose pull

# コンテナの再作成
sudo docker compose up -d
```

> ※ メジャーバージョンのアップグレードは段階的に行う必要があります。  
> 公式ドキュメントのアップグレードパスを確認してください。

---

## 11. トラブルシューティング

### コンテナが起動しない / unhealthy のまま

```bash
# 詳細ログの確認
sudo docker compose logs gitlab

# コンテナ内のサービス状態確認
sudo docker exec gitlab gitlab-ctl status
```

### ポート競合

ホスト側で既にポートが使用されている場合、`docker-compose.yml` のポートマッピングを変更してください。

```bash
# 使用中ポートの確認
sudo ss -tlnp | grep -E ':(80|443|22)\s'
```

### メモリ不足

GitLab は最低 4 GB のメモリが必要です。Swap の追加を検討してください。

```bash
sudo fallocate -l 4G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
```

---

## 12. 次のステップ

インストール完了後、バックアップの設定を行ってください。

👉 [GitLab バックアップ・リストア手順](./backup-restore.md)

---

## 参考リンク

- [GitLab Docker インストール公式ドキュメント](https://docs.gitlab.com/ee/install/docker.html)
- [GitLab Docker イメージ (Docker Hub)](https://hub.docker.com/r/gitlab/gitlab-ee/)
- [GitLab アップグレードパス](https://docs.gitlab.com/ee/update/index.html#upgrade-paths)
