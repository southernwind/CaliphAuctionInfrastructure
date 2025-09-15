<div align="center">
   <!-- TODO: 差し替え: ロゴ ライト/ダーク -->
   <img src="docs/images/logo.png" alt="Kalif Auction Logo" width="200" />
</div>

# CaliphAuctionInfrastructure

Caliph Auction プロジェクトのインフラ用リポジトリです。以下の他リポジトリと連携します:

- `CaliphAuctionFront` (Vue でビルドされたフロントエンド) – ビルド成果物のみこのサーバーへ配置
- `CaliphAuctionBackend` (ASP.NET Core / .NET 8) – 発行成果物のみこのサーバーへ配置
- `CaliphAuctionAssets` (画像・静的ファイル) – `/cdn` パスで参照される静的コンテンツ
- `CaliphAuctionInfrastructure` (本リポジトリ) – サーバープロビジョニング (Ansible), GitHub Actions

本リポジトリでは「実行環境の用意 (nginx, .NET 8 Runtime, PostgreSQL)」のみを行い、アプリケーションのデプロイ (ファイル配置や再起動) は各アプリ側リポジトリのパイプラインで行う想定です。

## 対象環境
- OS: Ubuntu 24.04 (VPS 単一サーバー)
- Web: nginx (リバースプロキシ)
- Backend: ASP.NET Core (.NET 8) Kestrel で 5000番ポートを listen
- DB: PostgreSQL (Ubuntu 標準リポジトリの meta package `postgresql` を使用 / バージョン固定しない)
- Frontend: 事前ビルド済みの `dist` を `/var/www/caliph-auction/front` に配置
- Assets: `/var/www/caliph-auction/cdn`

## ディレクトリ構成 (サーバー側)
```
/opt/caliph-auction/backend        # Backend 発行成果物 (dll など) を配置
/var/www/caliph-auction/front      # Vue ビルド成果物 (index.html 等)
/var/www/caliph-auction/cdn     # 画像など静的ファイル
```

## Ansible 構成
```
ansible/
  ansible.cfg
  site.yml
  inventory/hosts.ini
  group_vars/all.yml
  roles/
    common/            # 共通: ユーザー, UFW, 基本パッケージ
    dotnet/            # .NET 8 ランタイム導入
  postgres/          # PostgreSQL (Ubuntu 標準) 導入と初期設定 (バージョン自動検出は残す)
    nginx/             # nginx インストールとリバプロ設定
    backend_service/   # systemd ユニット (Kestrel) 作成
```

### 変数 (`group_vars/all.yml`)
主要な変数:
- `app_user`, `app_group` : 実行ユーザー/グループ
- `backend_port` : Kestrel ポート (デフォルト 5000)
- `postgres_user`, `postgres_db`
- `nginx_server_name` : ドメイン (未設定なら `_`)

PostgreSQL の実バージョン (`postgres_version_detected`) は `/etc/postgresql/*/main` を探索して自動検出します。PGDG リポジトリ追加は行わず、Ubuntu が提供する安定版を利用します。より新しいメジャーを即時利用したい場合のみ今後 PGDG 追加を再検討してください。

### PostgreSQL パスワードの扱い (機微情報非コミット方針)
`group_vars/all.yml` にはパスワードを保持しません。以下のいずれかの方法で注入します:
1. GitHub Actions: `Secrets.POSTGRES_PASSWORD` を環境変数経由で `-e postgres_password=...` として渡す (既に workflow 実装済)
2. ローカル実行: `ansible-playbook ... -e "postgres_password=YourStrongPassword"`
3. (将来) Ansible Vault / 外部 Secret Manager

Playbook は `postgres_password` が未指定の場合 `assert` で失敗します。

例 (ローカルから):
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/site.yml -e "postgres_password=StrongPass123" \
  -u deploy --private-key=~/.ssh/your_key
```

## nginx とルーティング / TLS
- `/` → フロントエンド (SPA, `index.html` フォールバック)
- `/assets/` → 静的ファイル (キャッシュ 30日)
- `/api/` → Backend へプロキシ (`http://127.0.0.1:5000/api/`)

### TLS / Let's Encrypt 対応 (常時有効)
本構成では TLS を前提としており、初回プロビジョニング時に以下を自動実行します:
- `roles/letsencrypt` で `certbot` (webroot) により証明書取得
- 80番ポート: ACME challenge 以外を 443 へリダイレクト
- 443番ポート: HSTS / セキュリティヘッダー適用済み

手順:
1. DNS を対象ドメイン → VPS IP に正しく向ける
2. 上記変数を編集 (メール/ドメイン)
3. Playbook 実行
4. 証明書は `/etc/letsencrypt/live/<ドメイン>/` 配下に配置され nginx が自動参照

証明書更新は `certbot renew` が systemd timer により自動実行 (Ubuntu デフォルト)。必要なら別ロールで明示的な更新タスク・再読み込みを追加可能。

## systemd ユニット
`roles/backend_service/templates/caliph-backend.service.j2` で作成。
`ExecStart=/usr/bin/dotnet /opt/caliph-auction/backend/CaliphAuctionBackend.dll` を想定。各 Backend リポジトリのデプロイ後に `systemctl restart caliph-backend` で再起動してください。

## GitHub Actions
`.github/workflows/provision.yml` で以下を実施:
1. リポジトリチェックアウト
2. Python + Ansible セットアップ
3. SSH 秘密鍵 ( `SSH_PRIVATE_KEY` シークレット ) をエージェント登録
4. `Secrets.POSTGRES_PASSWORD` を利用し `ansible-playbook` 実行 (workflow_dispatch 入力でパスワードを受け取らない)

### 必要な Secrets (想定)
- `SSH_PRIVATE_KEY` : 対象VPSへ接続できる秘密鍵
- `POSTGRES_PASSWORD` : PostgreSQL ログイン用アプリユーザーパスワード
- (将来) `SLACK_WEBHOOK` など通知系

### 手動実行方法
Actions → `Provision Infrastructure` → `Run workflow` で:
- `host` に `hosts.ini` に記載したホスト名 (例: `vps1`)
を入力し実行。パスワードは Secrets から自動注入されます。

## 他リポジトリのデプロイ戦略 (将来案)
- Backend リポジトリ: ビルド & 発行 → scp/rsync で `/opt/caliph-auction/backend` へ配置 → `systemctl restart caliph-backend`
- Frontend リポジトリ: `npm run build` 成果物を `/var/www/caliph-auction/front` へ同期
- Assets リポジトリ: 画像追加時に `/var/www/caliph-auction/cdn` へ同期
- これらはそれぞれのリポジトリの GitHub Actions から SSH or rsync で実行

## 今後の改善候補
- OCSP Stapling / 追加セキュリティヘッダー (Content-Security-Policy など)
- staging / production 分離 (`inventory/stg`, `inventory/prod`)
- 監視/ログ (Prometheus node exporter, fail2ban, logrotate 最適化)
- Secrets 管理を Ansible Vault / 外部 Secret Manager へ移行
- certbot 更新後の nginx reload hook (必要なら)

## ローカルテスト (Dry Run)
SSH 接続ができる状態で差分のみ確認:
```bash
ansible-playbook -i ansible/inventory/hosts.ini ansible/site.yml --check -e "postgres_password=Dummy"
```

## 注意
- PostgreSQL パスワードを平文でコミット/入力フォーム送信しない (Secrets 利用)
- 本番導入前に UFW / セキュリティグループで不要ポートを閉じる
PostgreSQL はデフォルト (ローカルのみ / 5432) のまま利用しています。リモート接続が必要になった場合のみ `postgresql.conf` と `pg_hba.conf` を別途手動/ロール拡張で調整してください。

## ライセンス

このリポジトリは「ソースコード閲覧・学習目的での公開」であり、一般的な OSS ライセンス (MIT / Apache など) ではありません。いわゆる _Source-Available_ ポリシーです。

### 許可される行為

- 個人的または社内での学習・参考・評価
- 自身の環境でのビルド・実行・検証
- 一部コード断片 (短い抜粋) を引用した技術記事等への掲載 (出典明記が条件)

### 禁止される行為 (明示的に許可しない)

- 本リポジトリ全体または実質的主要部分の再頒布 (フォークを含む公的再公開)
- コードの改変版を公衆に提供 / ホスティング / SaaS として提供
- ライセンス互換を前提とした他 OSS への組み込み
- 商用目的 (利用・販売・再販) での使用

### 追加注意

- 上記に該当しない利用 (教材化 / セミナー利用 / 研究引用 など) を希望する場合は事前に相談してください。
- いつでもライセンス/公開方針を変更・終了する可能性があります。
- Issue / PR は受け付けますが、マージ/反映は保証されません。

将来的に OSS ライセンスへ移行する場合は明示的に本節を置き換えます。それまでは本記述が優先します。

