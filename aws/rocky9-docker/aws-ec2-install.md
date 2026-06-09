# AWS EC2 環境構築手順（練習環境 / Docker コンテナ版）

GitLab EE を Docker コンテナで稼働させるための AWS EC2 環境の構築手順です。

## 前提条件

- AWS アカウントが作成済みであること
- AWS マネジメントコンソールへのアクセス権があること

## 構成

| インスタンス | 用途 | インスタンスタイプ | OS | ホスト名 |
|---|---|---|---|---|
| EC2 | GitLab 本体（Docker / EE） | t3.large (2vCPU/8GB) | Rocky Linux 9 | `gitlab-docker` |

---

## 1. EC2 インスタンスの起動

1. EC2 コンソール → **インスタンスを起動**
2. 以下を設定:

| 項目 | 設定値 |
|------|--------|
| 名前 | `gitlab-docker` |
| AMI | Rocky Linux 9 |
| インスタンスタイプ | `t3.large` |
| キーペア | 新規作成 or 既存を選択 |
| VPC / サブネット | 環境に応じて選択 |

### ストレージ

| ボリューム | サイズ | タイプ | 用途 |
|---|---|---|---|
| ルート | 20 GiB | gp3 | OS |
| 追加 | 50 GiB | gp3 | GitLab データ（/data） |

※ gp3 は後からオンラインで容量拡張可能です。  
※ Docker イメージ + GitLab データで容量を消費するため、追加ボリュームは 50 GiB 以上を推奨します。

---

## 2. セキュリティグループの設定

### セキュリティグループ（gitlab-docker-sg）

#### インバウンド

| タイプ | ポート | ソース | 説明 |
|--------|--------|--------|------|
| SSH | 22 | 管理者 IP / VPN | 管理用 |
| HTTP | 80 | 許可する IP 範囲 | GitLab Web UI |
| HTTPS | 443 | 許可する IP 範囲 | GitLab Web UI（SSL） |
| Custom TCP | 2222 | 許可する IP 範囲 | GitLab git SSH |
| Custom TCP | 5050 | 許可する IP 範囲 | GitLab コンテナレジストリ |

#### アウトバウンド

| タイプ | ポート | 送信先 | 説明 |
|--------|--------|--------|------|
| All | All | 0.0.0.0/0 | すべて許可 |

---

## 3. Elastic IP の割り当て

GitLab 本体に Elastic IP を割り当て、IP アドレスを固定します。

1. EC2 コンソール → Elastic IP → **Elastic IP アドレスを割り当てる**
2. 割り当て後、**アクション → 関連付け** で `gitlab-docker` に紐付け

⚠️ Elastic IP はインスタンス停止中も料金が発生します（$0.005/時間）。

---

## 4. DNS 設定

ドメインの DNS に以下の A レコードを追加します。

| レコードタイプ | 名前 | 値 |
|---|---|---|
| A | `gitlab.example.com` | （割り当てた Elastic IP） |

Route 53 を使用している場合：

1. Route 53 コンソール → ホストゾーン `example.com` を選択
2. **レコードを作成** をクリック
3. 以下を入力:
   - レコード名: `gitlab`
   - レコードタイプ: A
   - 値: Elastic IP アドレス
   - TTL: 300

確認：

```bash
nslookup gitlab.example.com
```

---

## 5. ユーザー作成と公開鍵設定

デフォルトユーザー（rocky）で初回ログイン後、運用用ユーザーを作成します。

### 5.1 ユーザー作成

```bash
# ユーザー作成
sudo useradd your-username

# sudo 権限を付与
sudo usermod -aG wheel your-username

# パスワードなし sudo を設定
sudo sh -c 'echo "your-username ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/your-username'
```

### 5.2 公開鍵の設定

```bash
# .ssh ディレクトリ作成
sudo mkdir -p /home/your-username/.ssh
sudo chmod 700 /home/your-username/.ssh

# 公開鍵を配置（ローカル PC の公開鍵を貼り付け）
sudo vi /home/your-username/.ssh/authorized_keys

# 権限設定
sudo chmod 600 /home/your-username/.ssh/authorized_keys
sudo chown -R your-username:your-username /home/your-username/.ssh
```

ローカル PC で公開鍵を確認（まだ鍵がない場合は作成）：

```bash
# 鍵がない場合は作成
ssh-keygen -t ed25519 -C "your-email@example.com"

# 公開鍵を表示→上記の authorized_keys に貼り付け
cat ~/.ssh/id_ed25519.pub
```

### 5.3 SSH 接続確認

`~/.ssh/config`:

```text
Host gitlab-docker
    HostName <Elastic IP or Private IP>
    Port 22
    User your-username
    IdentityFile ~/.ssh/id_ed25519
```

踏み台（Bastion）経由の場合：

```text
Host bastion
    HostName <Bastion Public IP>
    Port 22
    User your-username
    IdentityFile ~/.ssh/id_ed25519

Host gitlab-docker
    HostName <GitLab Private IP>
    Port 22
    User your-username
    IdentityFile ~/.ssh/id_ed25519
    ProxyJump bastion
```

```bash
ssh gitlab-docker
```

---

## 6. データボリュームのマウント

追加 EBS ボリュームを `/data` にマウントし、GitLab コンテナのデータ保存先として使用します。

```bash
# デバイス確認
lsblk

# フォーマット（初回のみ）
sudo mkfs.xfs /dev/nvme1n1

# マウントポイント作成
sudo mkdir -p /data

# マウント
sudo mount /dev/nvme1n1 /data

# 永続化（/etc/fstab に追記）
echo '/dev/nvme1n1 /data xfs defaults 0 0' | sudo tee -a /etc/fstab
```

GitLab コンテナ用のディレクトリを作成します:

```bash
sudo mkdir -p /data/gitlab/{config,logs,data}
```

| ディレクトリ | コンテナ内マウント先 | 用途 |
|---|---|---|
| `/data/gitlab/config` | `/etc/gitlab` | GitLab 設定ファイル |
| `/data/gitlab/logs` | `/var/log/gitlab` | ログ |
| `/data/gitlab/data` | `/var/opt/gitlab` | リポジトリ・DB 等 |

---

## 7. EBS 容量拡張手順（必要になったとき）

1. AWS コンソール → EC2 → ボリューム → 対象ボリュームを選択 → **変更**
2. サイズを増やして **変更** をクリック
3. EC2 内で以下を実行:

```bash
# パーティション拡張
sudo growpart /dev/nvme1n1 1

# ファイルシステム拡張（xfs の場合）
sudo xfs_growfs /data
```

---

## 8. 次のステップ

EC2 の起動とデータボリュームのマウントが完了したら、Docker のインストールと GitLab コンテナの起動に進みます。

👉 [GitLab Docker インストール手順](./install-gitlab-docker.md)

---

## 参考リンク

- [AWS EC2 ドキュメント](https://docs.aws.amazon.com/ec2/)
- [GitLab Docker インストール要件](https://docs.gitlab.com/ee/install/docker.html)
- [AWS 料金計算ツール](https://calculator.aws/)
