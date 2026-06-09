# GitLab バージョンアップ手順（Docker コンテナ版）

Docker コンテナで稼働する GitLab EE のバージョンアップ手順です。

## 前提条件

| 項目 | 内容 |
|------|------|
| 現在のバージョン | 19.0.1-ee |
| コンテナ名 | `gitlab` |
| docker-compose.yml | `/data/gitlab/docker-compose.yml` |
| データディレクトリ | `/data/gitlab` |

## 想定所要時間

約 15〜30 分（イメージ pull + 再起動）

---

## 1. アップグレードパスの確認

GitLab はバージョンを飛ばしてアップグレードできない場合があります。**必ず公式のアップグレードパスを確認**してください。

👉 [GitLab Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)

### パッチバージョン（例: 19.0.1 → 19.0.2）

- 直接アップグレード可能

### マイナーバージョン（例: 19.0.x → 19.1.0）

- 直接アップグレード可能（通常）

### メジャーバージョン（例: 19.x → 20.0）

- 中間バージョンを経由する必要がある場合あり
- 必ず Upgrade Path Tool で確認すること

---

## 2. バックアップ取得

**バージョンアップ前に必ずバックアップを取得してください。**

```bash
# バックアップスクリプトを実行
sudo /usr/local/bin/gitlab-backup.sh

# 確認
sudo ls -lh /data/gitlab/data/backups/
```

詳細は [バックアップ・リストア手順](./backup-restore.md) を参照。

---

## 3. 現在のバージョン確認

```bash
docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

---

## 4. バージョンアップ実行

### 4.1 特定バージョンへのアップグレード

`docker-compose.yml` のイメージタグを変更します：

```bash
sudo vi /data/gitlab/docker-compose.yml
```

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ee:19.1.0-ee.0    # ← バージョンを指定
```

> 💡 利用可能なタグは [Docker Hub](https://hub.docker.com/r/gitlab/gitlab-ee/tags) で確認できます。

```bash
cd /data/gitlab
sudo docker compose pull
sudo docker compose up -d
```

### 4.2 最新バージョンへのアップグレード（latest）

```bash
cd /data/gitlab
sudo docker compose pull
sudo docker compose up -d
```

> ⚠️ `latest` タグを使用している場合、メジャーバージョンが上がる可能性があります。
> 本番環境では**バージョンを明示的に指定**することを推奨します。

---

## 5. アップグレード後の確認

### 5.1 コンテナ状態確認

```bash
# healthy になるまで数分待つ
docker ps --filter name=gitlab
```

### 5.2 バージョン確認

```bash
docker exec gitlab cat /opt/gitlab/embedded/service/gitlab-rails/VERSION
```

### 5.3 マイグレーション状態確認

```bash
docker exec gitlab gitlab-rake db:migrate:status | tail -20
```

すべて `up` になっていれば正常です。

### 5.4 ブラウザで動作確認

- ログインできること
- リポジトリが閲覧できること
- コンテナレジストリのタグが表示されること

---

## 6. ロールバック（アップグレード失敗時）

アップグレードに失敗した場合、バックアップからリストアします。

### 6.1 旧バージョンのイメージに戻す

```bash
sudo vi /data/gitlab/docker-compose.yml
```

```yaml
services:
  gitlab:
    image: gitlab/gitlab-ee:19.0.1-ee.0    # ← 元のバージョンに戻す
```

### 6.2 コンテナを再作成

```bash
cd /data/gitlab
sudo docker compose down
sudo docker compose up -d
```

### 6.3 バックアップからリストア

```bash
docker exec gitlab gitlab-ctl stop puma
docker exec gitlab gitlab-ctl stop sidekiq
docker exec -it gitlab gitlab-backup restore BACKUP=<タイムスタンプ>

# registry DB リストア
docker cp /data/gitlab/data/backups/registry_tags.csv gitlab:/tmp/
docker exec gitlab gitlab-psql -d registry -c "DELETE FROM tags;"
docker exec gitlab gitlab-psql -d registry -c "\copy tags FROM '/tmp/registry_tags.csv' CSV HEADER"

# 再起動
docker exec gitlab gitlab-ctl restart
```

詳細は [バックアップ・リストア手順](./backup-restore.md) を参照。

---

## 7. 注意事項

| 項目 | 内容 |
|------|------|
| アップグレードパス | メジャーバージョンアップ時は中間バージョンが必要な場合あり |
| バックアップ必須 | アップグレード前に必ず取得する |
| ダウングレード不可 | GitLab は基本的にダウングレード非対応。失敗時はバックアップからリストアする |
| DB マイグレーション | バージョンアップ時に自動実行される。完了まで数分かかる場合あり |
| ディスク容量 | 新旧イメージが一時的に共存するため、十分な空き容量を確保する |

---

## 8. 不要イメージの削除

アップグレード後、旧バージョンのイメージを削除してディスクを解放します：

```bash
docker image prune -f
```

---

## 参考リンク

- [GitLab Upgrade Path Tool](https://gitlab-com.gitlab.io/support/toolbox/upgrade-path/)
- [GitLab アップグレード公式ドキュメント](https://docs.gitlab.com/ee/update/index.html)
- [GitLab Docker イメージ タグ一覧](https://hub.docker.com/r/gitlab/gitlab-ee/tags)
