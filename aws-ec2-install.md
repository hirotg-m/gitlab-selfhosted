# AWS EC2 環境構築手順（練習環境）

GitLab EE および GitLab Runner をホストするための AWS EC2 環境の構築手順です。

## 前提条件

- AWS アカウントが作成済みであること
- AWS マネジメントコンソールへのアクセス権があること

## 構成

| インスタンス | 用途 | インスタンスタイプ | OS | ホスト名 |
|---|---|---|---|---|
| EC2-1 | GitLab 本体 | t3.large (2vCPU/8GB) | Rocky Linux 10 | `gitlab-` |
| EC2-2 | GitLab Runner | t3.medium (2vCPU/4GB) | Rocky Linux 10 | `gitlab-runner1-` |

---

## 1. EC2 インスタンスの起動

### 1.1 GitLab 本体（gitlab-）

1. EC2 コンソール → **インスタンスを起動**
2. 以下を設定:

| 項目 | 設定値 |
|------|--------|
| 名前 | `gitlab-` |
| AMI | Rocky Linux 9 |
| インスタンスタイプ | `t3.large` |
| キーペア | 新規作成 or 既存を選択 |
| VPC / サブネット | 環境に応じて選択 |

#### ストレージ

| ボリューム | サイズ | タイプ | 用途 |
|---|---|---|---|
| ルート | 20 GiB | gp3 | OS |
| 追加 | 20 GiB | gp3 | GitLab データ（/data） |

※ gp3 は後からオンラインで容量拡張可能です。

### 1.2 GitLab Runner（gitlab-runner1-）

| 項目 | 設定値 |
|------|--------|
| 名前 | `gitlab-runner1-` |
| AMI | Rocky Linux 10 |
| インスタンスタイプ | `t3.medium` |
| キーペア | GitLab 本体と同じ |
| VPC / サブネット | GitLab 本体と同じ |

#### ストレージ

| ボリューム | サイズ | タイプ | 用途 |
|---|---|---|---|
| ルート | 20 GiB | gp3 | OS + Runner |

---

## 2. セキュリティグループの設定

### GitLab 本体用（gitlab-ee-sg）

#### インバウンド

| タイプ | ポート | ソース | 説明 |
|--------|--------|--------|------|
| SSH | 22 | 管理者 IP / VPN | 管理用 |
| HTTP | 80 | 許可する IP 範囲 | HTTPS リダイレクト用 |
| HTTPS | 443 | 許可する IP 範囲 | GitLab Web UI |
| Custom TCP | 2222 | 許可する IP 範囲 | GitLab git SSH |

#### アウトバウンド

| タイプ | ポート | 送信先 | 説明 |
|--------|--------|--------|------|
| All | All | 0.0.0.0/0 | すべて許可 |

### GitLab Runner 用（gitlab-runner-sg）

#### インバウンド

| タイプ | ポート | ソース | 説明 |
|--------|--------|--------|------|
| SSH | 22 | 管理者 IP / VPN | 管理用 |

#### アウトバウンド

| タイプ | ポート | 送信先 | 説明 |
|--------|--------|--------|------|
| All | All | 0.0.0.0/0 | すべて許可 |

---

## 3. Elastic IP の割り当て

GitLab 本体に Elastic IP を割り当て、IP アドレスを固定します。

1. EC2 コンソール → Elastic IP → **Elastic IP アドレスを割り当てる**
2. 割り当て後、**アクション → 関連付け** で `gitlab-dev` に紐付け

⚠️ Elastic IP はインスタンス停止中も料金が発生します（$0.005/時間）。

---

## 4. ユーザー作成と公開鍵設定

デフォルトユーザー（rocky）で初回ログイン後、運用用ユーザーを作成します。

### 4.1 ユーザー作成

```bash
# ユーザー作成
sudo useradd user1

# sudo 権限を付与
sudo usermod -aG wheel user1

# パスワードなし sudo を設定
sudo sh -c 'echo "user1 ALL=(ALL) NOPASSWD: ALL" > /etc/sudoers.d/user1'
```

### 4.2 公開鍵の設定

```bash
# .ssh ディレクトリ作成
sudo mkdir -p /home/user1/.ssh
sudo chmod 700 /home/user1/.ssh

# 公開鍵を配置（ローカル PC の公開鍵を貼り付け）
sudo vi /home/user1/.ssh/authorized_keys

# 権限設定
sudo chmod 600 /home/user1/.ssh/authorized_keys
sudo chown -R user1:user1 /home/user1/.ssh
```

ローカル PC で公開鍵を確認（まだ鍵がない場合は作成）：

```bash
# 鍵がない場合は作成
ssh-keygen -t ed25519 -C "user1@"

# 公開鍵を表示→上記の authorized_keys に貼り付け
cat ~/.ssh/id_ed25519.pub
```

### 4.3 SSH 接続確認

`~/.ssh/config`:



---

## 5. データボリュームのマウント（GitLab 本体）

追加 EBS ボリュームを `/data` にマウントし、GitLab のデータ保存先として使用します。

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

GitLab のデータ保存先ディレクトリを作成します:

```bash
# Docker Compose で構築する場合
sudo mkdir -p /data/gitlab/{config,logs,data}

# Omnibus パッケージで直にインストールする場合
sudo mkdir -p /data/gitlab
```

Omnibus パッケージの場合は `/etc/gitlab/gitlab.rb` で以下を設定し、データ保存先を `/data/gitlab` に変更します:

```ruby
git_data_dirs({ "default" => { "path" => "/data/gitlab/git-data" } })
postgresql['dir'] = '/data/gitlab/postgresql'
```

---

## 6. EBS 容量拡張手順（必要になったとき）

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

## 7. 次のステップ

EC2 の起動とデータボリュームのマウントが完了したら、GitLab EE のインストールに進みます。

👉 [GitLab EE インストール手順](./gitlab-ee-install.md)

---

## 参考リンク

- [AWS EC2 ドキュメント](https://docs.aws.amazon.com/ec2/)
- [GitLab インストール要件](https://docs.gitlab.com/ee/install/requirements.html)
- [AWS 料金計算ツール](https://calculator.aws/)
