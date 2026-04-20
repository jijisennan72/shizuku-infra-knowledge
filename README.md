# shizuku-infra-knowledge

**しずくネット配下プロジェクト共通のインフラナレッジベース**

さくらVPS (49.212.136.193) 上で稼働する複数プロジェクトが共有するインフラ知識の集約リポジトリです。

稼働プロジェクト: はまアプリ / gikai / くらさんチャレンジ / せんなん政治マップ / GDriveバックアップ監視ツール

---

## 🚨 トラブル発生時：まずここから

症状別の診断フローは [`runbooks/docker-network-diagnostics.md`](./runbooks/docker-network-diagnostics.md) を参照してください。

---

## 📁 ディレクトリ構成

| ディレクトリ | 役割 |
|---|---|
| [`incidents/`](./incidents/) | 過去に発生した障害の詳細記録（時系列・原因・復旧手順） |
| [`runbooks/`](./runbooks/) | 診断・作業手順書（再利用可能なコマンドセット） |
| [`reference/`](./reference/) | 環境構成・設計思想のリファレンス |
| [`templates/`](./templates/) | AIアシスタント向けプロンプトテンプレート |

---

## 📚 主要ドキュメント

### 障害記録（incidents/）

| ファイル | 概要 |
|---|---|
| [`2026-04-15_vps-reboot-cascade.md`](./incidents/2026-04-15_vps-reboot-cascade.md) | VPS再起動後の複合障害（UFW IPズレ + acme.json + systemd未設定 + ServiceWorkerキャッシュ）|
| [`2026-04-20_kong-ip-shift.md`](./incidents/2026-04-20_kong-ip-shift.md) | gikai移植時のKong IP再配置による504（コンテナ名参照化で恒久解決）|

### 手順書（runbooks/）

| ファイル | 概要 |
|---|---|
| [`docker-network-diagnostics.md`](./runbooks/docker-network-diagnostics.md) | 502/504発生時の診断フロー |
| [`vps-reboot-checklist.md`](./runbooks/vps-reboot-checklist.md) | VPS再起動前後の標準チェックリスト |

### リファレンス（reference/）

| ファイル | 概要 |
|---|---|
| [`infrastructure-overview.md`](./reference/infrastructure-overview.md) | VPS・ネットワーク・コンテナの全体像 |

### テンプレート（templates/）

| ファイル | 概要 |
|---|---|
| [`alice-bootstrap-prompt.md`](./templates/alice-bootstrap-prompt.md) | 新規プロジェクトでアリスに貼るプロンプト |

---

## 🤖 AIアシスタントに参照させる方法

新規チャット開始時、冒頭に以下を含めてください：

```
【共通インフラナレッジ】
しずくネット配下のVPS共通知見は以下に集約されています。
Docker/Traefik/UFW/Coolify関連のトラブル対応時は、まず以下を参照してください。

📚 https://github.com/jijisennan72/shizuku-infra-knowledge

- トラブル発生時は runbooks/docker-network-diagnostics.md から診断
- 過去の類似事例は incidents/ 配下を確認
- 新しい教訓が出た場合、ドキュメント追加を提案してください
```

詳細は [`templates/alice-bootstrap-prompt.md`](./templates/alice-bootstrap-prompt.md) 参照。

---

## 🏗️ 環境の要点

| 項目 | 値 |
|---|---|
| VPS | さくらVPS 大阪リージョン (49.212.136.193) |
| OS | Ubuntu 24 LTS |
| コンテナ基盤 | Docker |
| リバースプロキシ | Traefik v3.6 (Coolify管理) |
| PaaS | Coolify |
| DB | Supabase OSS (self-hosted) |

詳細は [`reference/infrastructure-overview.md`](./reference/infrastructure-overview.md) へ。

---

## 📝 ナレッジ追加時のルール

1. **症状ではなく構造で書く** — 再現性のある教訓として残す
2. **コマンドはコピペで動く形で載せる** — AIが即実行できるように
3. **新しい障害が出たら即座に記録する** — `incidents/YYYY-MM-DD_短い件名.md` の形式で
4. **ジージが編集しやすい表現で書く** — 後から追記・修正しやすく

---

*管理: ジージ（浜地区町内会・はまアプリプロジェクト代表）*
*記録・整理: アリス（AIアシスタント）*
*助言: ジニー（インフラ設計担当）*
*実装: くらさん（Claude Code）*
