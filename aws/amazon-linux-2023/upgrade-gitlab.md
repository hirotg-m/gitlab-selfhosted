# GitLab バージョンアップ手順（Amazon Linux 2023 / ネイティブインストール版）

Amazon Linux 2023 に rpm パッケージでインストールした GitLab EE のバージョンアップ手順です。

## 前提条件

| 項目 | 内容 |
|------|------|
| OS | Amazon Linux 2023 |
| GitLab | gitlab-ee（rpm パッケージ） |
| パッケージ管理 | dnf |

## 想定所要時間

約 15〜30 分（パッケージダウンロード + DB マイグレーション含む）

---

## 1. アップグレードパスの確認

GitLab はバージョンを飛ばしてアップグレードできない場合があります。**必ず公式のアップグレードパスを確認**してください。

👉 [GitLab Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)

### パッチバージョン（例: 18.8.2 → 18.8.10）

- 直接アップグレード可能

### マイナーバージョン（例: 18.8.x → 18.11.x）

- 直接アップグレード可能（通常）

### メジャーバージョン（例: 18.x → 19.0）

- 中間バージョンを経由する必要がある場合あり
- 必ず Upgrade Path Tool で確認すること

---

## 2. バックアップ取得

**バージョンアップ前に必ずバックアップを取得してください。**

```bash
# GitLab データのバックアップ
sudo gitlab-backup create

# 設定ファイルのバックアップ
sudo cp /etc/gitlab/gitlab.rb /var/opt/gitlab/backups/
sudo cp /etc/gitlab/gitlab-secrets.json /var/opt/gitlab/backups/

# 確認
sudo ls -lh /var/opt/gitlab/backups/
```

---

## 3. 現在のバージョン確認

```bash
sudo gitlab-rake gitlab:env:info
# または
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

---

## 4. バージョンアップ実行

### 4.1 利用可能なバージョンの確認

```bash
sudo dnf list available gitlab-ee --showduplicates | sort -V
```

### 4.2 特定バージョンへのアップグレード

```bash
sudo dnf install -y gitlab-ee-<version>
```

例:

```bash
sudo dnf install -y gitlab-ee-18.8.10-ee.0.amazon2023
```

> 💡 バージョン文字列は `dnf list` の出力から確認してください。

### 4.3 最新バージョンへのアップグレード

```bash
sudo dnf upgrade -y gitlab-ee
```

> ⚠️ メジャーバージョンが上がる可能性があります。本番環境ではバージョンを明示的に指定することを推奨します。

### 4.4 reconfigure の実行

パッケージのインストール時に自動で `gitlab-ctl reconfigure` が実行されますが、完了しない場合は手動で実行します。

```bash
sudo gitlab-ctl reconfigure
```

---

## 5. アップグレード後の確認

### 5.1 サービス状態確認

```bash
sudo gitlab-ctl status
```

すべてのサービスが `run` になっていることを確認します。

### 5.2 バージョン確認

```bash
cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

### 5.3 DB マイグレーション状態確認

```bash
sudo gitlab-rake db:migrate:status | tail -20
```

すべて `up` になっていれば正常です。

### 5.4 ブラウザで動作確認

- ログインできること
- リポジトリが閲覧できること
- CI/CD パイプラインが動作すること

---

## 6. ロールバック（アップグレード失敗時）

GitLab は基本的にダウングレード非対応です。失敗時はバックアップからリストアします。

### 6.1 旧バージョンのパッケージに戻す

```bash
sudo dnf downgrade -y gitlab-ee-<previous-version>
```

### 6.2 バックアップからリストア

```bash
# DB 接続プロセスを停止
sudo gitlab-ctl stop puma
sudo gitlab-ctl stop sidekiq

# 停止確認
sudo gitlab-ctl status

# 設定ファイルの復元
sudo cp /var/opt/gitlab/backups/gitlab.rb /etc/gitlab/gitlab.rb
sudo cp /var/opt/gitlab/backups/gitlab-secrets.json /etc/gitlab/gitlab-secrets.json

# reconfigure
sudo gitlab-ctl reconfigure

# リストア実行（タイムスタンプ部分を指定）
sudo gitlab-backup restore BACKUP=<タイムスタンプ>

# 再起動
sudo gitlab-ctl restart
```

`<タイムスタンプ>` はバックアップファイル名から `_gitlab_backup.tar` を除いた部分です。

---

## 7. 段階的アップグレード（メジャーバージョン跨ぎ）

メジャーバージョンを跨ぐ場合、中間バージョンを1つずつ経由する必要があります。

例: 18.8.2 → 18.8.10 → 18.11.4 → 19.0.1

```bash
# Step 1: パッチバージョンを最新に
sudo dnf install -y gitlab-ee-18.8.10-ee.0.amazon2023

# Step 2: マイナーバージョンアップ
sudo dnf install -y gitlab-ee-18.11.4-ee.0.amazon2023

# Step 3: メジャーバージョンアップ
sudo dnf install -y gitlab-ee-19.0.1-ee.0.amazon2023
```

各ステップの間で以下を確認してください：

```bash
sudo gitlab-ctl status
sudo gitlab-rake db:migrate:status | tail -5
```

---

## 8. 注意事項

| 項目 | 内容 |
|------|------|
| アップグレードパス | メジャーバージョンアップ時は中間バージョンが必要な場合あり |
| バックアップ必須 | アップグレード前に必ず取得する |
| ダウングレード非対応 | 失敗時はバックアップからリストアが基本 |
| DB マイグレーション | バージョンアップ時に自動実行される。完了まで数分かかる場合あり |
| ディスク容量 | 新旧パッケージが一時的に共存するため、十分な空き容量を確保する |
| バックグラウンドマイグレーション | 完了前に次のアップグレードを行うと失敗する場合あり |

### バックグラウンドマイグレーションの確認

```bash
sudo gitlab-rake gitlab:background_migrations:status
```

すべて `finished` になってから次のアップグレードに進んでください。

---

## トラブルシューティング

### GPG 署名検証エラー

```
Failed to download metadata for repo 'gitlab_gitlab-ee-source': repomd.xml GPG signature verification error
```

リポジトリメタデータの署名検証を無効化して解決します：

```bash
sudo vi /etc/yum.repos.d/gitlab_gitlab-ee.repo
```

`repo_gpgcheck=1` を `repo_gpgcheck=0` に変更（`gitlab_gitlab-ee` と `gitlab_gitlab-ee-source` の2箇所）：

```ini
[gitlab_gitlab-ee]
repo_gpgcheck=0

[gitlab_gitlab-ee-source]
repo_gpgcheck=0
```

```bash
sudo dnf clean all
sudo dnf makecache
```

> 💡 `repo_gpgcheck` はリポジトリメタデータの署名検証です。`gpgcheck`（パッケージ自体の署名検証）は `1` のまま残してください。

---

## 参考リンク

- [GitLab Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)
- [GitLab アップグレード公式ドキュメント](https://docs.gitlab.com/ee/update/index.html)
- [GitLab パッケージ情報（packagecloud）](https://packages.gitlab.com/gitlab/gitlab-ee)
