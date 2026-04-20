# しずくネット インフラ全体像

**最終更新**: 2026年4月20日

---

## 🏗️ 物理・論理構成

### VPS

| 項目 | 値 |
|---|---|
| プロバイダ | さくらインターネット |
| リージョン | 大阪 |
| IP | `49.212.136.193` |
| ホスト名 | `os3-322-50689` |
| OS | Ubuntu 24.04 LTS |
| Swap | 8GB（`/etc/fstab` 永続化済み） |

### ドメイン

- `shizuku.net`（管理ドメイン）
- `hama.shizuku.net` — はまアプリ
- `hama02.shizuku.net` — はまアプリ新環境
- `supabase.hama02.shizuku.net` — Supabase Studio

---

## 🐳 Dockerネットワーク構成

### ネットワーク一覧

| ネットワーク名 | サブネット | 主な参加コンテナ |
|---|---|---|
| `coolify` | `10.0.1.0/24` | Traefik、Coolify本体、各アプリコンテナ |
| `y12thilic710z4ni7c44g9ro` | `10.0.2.0/24` | Supabase OSS 全コンテナ |
| `bridge` | デフォルト | coolify-sentinel |

### Traefikの所属ネットワーク

`coolify-proxy`（Traefik）は以下の両方に参加：

- `coolify` — 各アプリへのルーティング用
- `y12thilic710z4ni7c44g9ro` — Supabaseへのルーティング用（2026-04-20に追加）

**⚠️ `docker network connect` で追加した接続は永続化されていない。** 再起動後にチェック必須。

---

## 🚏 リバースプロキシ（Traefik v3.6）

### 設定ファイル

- 静的設定: `/data/coolify/proxy/docker-compose.yml`
- 動的設定: `/data/coolify/proxy/dynamic/*.yaml`（`providers.file.watch=true` で自動リロード）
- 証明書: `/data/coolify/proxy/acme.json`（**パーミッション 600 必須**）

### 動的設定ファイル一覧（2026-04-20時点）

| ファイル | 対応ドメイン | ルーティング先 |
|---|---|---|
| `hama-app.yaml` | `hama02.shizuku.net` | `http://host.docker.internal:3000` |
| `supabase-kong.yaml` | `supabase.hama02.shizuku.net` | `http://supabase-kong-y12thilic710z4ni7c44g9ro:8000` ✅ コンテナ名参照 |
| `coolify-temp.yaml` | （内部用） | - |
| `hama-body-limit.yaml` | - | ミドルウェア（リクエストサイズ制限） |
| `hama-shinbun.yaml` | - | - |
| `default_redirect_503.yaml` | - | 503エラー時のデフォルトリダイレクト |

---

## 🔥 ファイアウォール（UFW）

### FORWARDルール（2026-04-15以降の構成）

```bash
ufw route allow to 10.0.1.0/24   # coolifyネットワーク全体
ufw route allow to 10.0.2.0/24   # supabase ネットワーク全体
```

**重要**: `user.rules` に記録。`after.rules` では効かない。

### ポート

| ポート | 用途 |
|---|---|
| 22 | SSH |
| 80 | HTTP → HTTPSリダイレクト |
| 443 | HTTPS |
| 5433 | PostgreSQL（socat経由で5432にフォワード、systemd永続化） |
| 10000 | Webmin |

---

## 🗄️ Supabase OSS

### コンテナ一覧（y12thilic710z4ni7c44g9ro ネットワーク）

| コンテナ | 役割 |
|---|---|
| `supabase-kong-*` | APIゲートウェイ |
| `supabase-db-*` | PostgreSQL |
| `supabase-auth-*` | 認証 |
| `supabase-rest-*` | PostgREST |
| `supabase-storage-*` | ファイルストレージ |
| `supabase-studio-*` | 管理UI |
| `supabase-meta-*` | メタデータ |
| `supabase-realtime-*` | リアルタイム通信 |
| `supabase-edge-functions-*` | Edge Functions |
| `supabase-minio-*` | オブジェクトストレージ |
| `supabase-supavisor-*` | プーリング |
| `imgproxy-*` | 画像処理 |

**注意**: これらのコンテナのIPは固定されておらず、Docker側の都合で変動する。ルーティングは必ずコンテナ名で行うこと。

---

## 💼 稼働サービス一覧

| サービス | URL | 主な技術スタック |
|---|---|---|
| はまアプリ | `hama.shizuku.net` | Next.js 14 + Supabase OSS |
| gikai | （要確認） | Next.js + Supabase OSS（2026-04-20 移植） |
| くらさんチャレンジ | （要確認） | - |
| GDriveバックアップ監視 | （要確認） | - |
| せんなん政治マップ | （要確認） | - |

---

## 🔐 管理アクセス

| 項目 | 方法 |
|---|---|
| Coolify UI | SSH tunnel → `localhost:18000` |
| Webmin | port 10000 |
| Supabase Studio | `https://supabase.hama02.shizuku.net` |
| SSH | 標準鍵認証（詳細はジージの手元） |

---

## ⚙️ systemd管理下のサービス

| unit名 | 用途 | 状態（2026-04-20時点） |
|---|---|---|
| `socat-pg.service` | 5433→5432ポートフォワード | enabled |
| `hama-app.service` | はまアプリコンテナ自動起動 | enabled（ただし現状 dead 状態で別途Next.jsが稼働中・要調査） |

---

## 📎 関連

- 過去障害: `incidents/` 配下を参照
- 標準手順: `runbooks/` 配下を参照
