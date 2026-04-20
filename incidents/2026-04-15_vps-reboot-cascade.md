# 2026-04-15 — VPS再起動による複合障害

**発生日**: 2026年4月（深夜帯）
**復旧完了**: 翌日 02:17頃
**影響範囲**: hama.shizuku.net 全断、Supabase全断、Coolify UI全断
**カテゴリ**: `vps-reboot` `ufw` `traefik` `service-worker` `acme-tls`

---

## 0. エグゼクティブサマリー

- **直接のトリガー**: さくらVPSコンパネでのパッケージ更新 → VPS再起動
- **本質**: 単一障害ではなく、**潜在的に存在していた「5つの爆弾」が再起動で同時に爆発した複合障害**
- **復旧が遅れた最大の要因**: ServiceWorkerの古いキャッシュがサイトを「生きているように見せていた」ため、障害検知と原因特定が遅れた

---

## 1. 発端と初期症状

さくらVPSコンパネでパッケージ更新を実施 → VPS再起動 → 全サービス停止。

| 観測された症状 | 実際の原因 |
|---|---|
| `https://hama.shizuku.net` が開かない | UFW FORWARDルール不一致＋acme.json不正 |
| でも一部ブラウザでは開く（ように見える） | ServiceWorkerが2週間前のキャッシュを返していた |
| `ping` は通る | L3は到達している |
| TCP 443にSYNは届くがSYN-ACKが返らない | iptables/UFWでブロック |
| Traefikログが `acme.json` エラーで埋まる | パーミッション644（正しくは600） |

---

## 2. 複合障害の構造

再起動前から仕込まれていた「爆弾」:

| # | 潜在リスク | 単独では無症状だった理由 |
|---|---|---|
| ① | `hama-app` コンテナに自動起動設定なし | 再起動しなければ問題化しない |
| ② | `/data/coolify/proxy/acme.json` が `644` | Let's Encrypt証明書の更新タイミングまでは機能する |
| ③ | UFWのFORWARDルールが **TraefikのIP `10.0.1.6` を個別指定** | Dockerコンテナの起動順が変わらなければIPも変わらない |
| ④ | `coolify.hama.shizuku.net` のDNS/証明書未設定 | Coolify UIに外部アクセスしなければ気づかない |
| ⑤ | ServiceWorkerが長期キャッシュを保持 | サーバー側が健康なら問題化しない |

**再起動というたった一つのイベントで、①②③が同時顕在化。④⑤が診断を妨害した。**

### なぜIPがズレたか（③の詳細）

Coolifyセットアップ時、TraefikのIPは自動で `10.0.1.6` に割り当てられ、UFWのFORWARDルールもその値で固定された。

通常のVPS再起動では Docker が**同じ順序でコンテナを起動**するのでIPが変わらない。しかし今回は直前の作業で `hama-app.service`（systemd）と `coolify-temp` ルーターが追加されていたため、Dockerネットワークへの参加順が変わり、TraefikのIPが `10.0.1.6 → 10.0.1.3` にズレた。

結果：外部からTCP 443のSYNは VPS に届くが、UFWが旧IP宛てしか許可していないため SYN-ACKが返らない → 接続タイムアウト。

---

## 3. 診断が難しかった理由

1. **ServiceWorkerキャッシュの罠**: 一部ユーザーのブラウザでは「サイトが動いているように見える」状態。障害検知が遅れた。
2. **Traefikログがacme.jsonエラーで埋まる**: 本質（UFWのIPズレ）が見えにくかった。
3. **L3は生きている**: `ping` が通るのでネットワーク疎通の問題と結論づけにくい。
4. **`tcpdump` でSYNが届いているのにSYN-ACKが返っていない** ことを確認して、ようやく iptables/UFW 層の問題と確定できた。

---

## 4. 当夜の復旧手順（時系列）

```
1. docker compose up で hama-app コンテナを手動起動
2. /etc/systemd/system/hama-app.service 作成 → systemctl enable
3. acme.json のパーミッションを 644 → 600 に修正
4. docker restart coolify-proxy（Traefik再起動・証明書リゾルバー復活）
5. tcpdump で SYN 到達・SYN-ACK 不達を確認
6. UFW FORWARDルールに新IP（10.0.1.3）を一時追加
7. サイト復旧確認(02:17頃)
```

---

## 5. 恒久対策

### ① UFWルールのサブネット化

IPアドレス個別指定はコンテナIP変動で壊れる。サブネット全体許可に変更：

```bash
# /etc/ufw/user.rules に永続化（ufw route コマンドで登録）
ufw route allow to 10.0.1.0/24   # coolifyネットワーク全体
ufw route allow to 10.0.2.0/24   # supabase-metaネットワーク全体
```

**⚠️ 重要な罠**: `/etc/ufw/after.rules` に書いても効かない。`ufw route allow` で `user.rules` に書くこと。

### ② コンテナ自動起動（systemd）

```
/etc/systemd/system/hama-app.service を作成 → systemctl enable
```

### ③ acme.jsonパーミッション修正

```bash
chmod 600 /data/coolify/proxy/acme.json
```

---

## 6. 教訓

1. **「パッケージ更新 → 再起動」は複合障害のトリガー**。更新前に既存の爆弾の存在を点検する。
2. **IP個別指定は爆弾**。コンテナIPは起動順で変わりうる。サブネット許可が鉄則。
3. **ServiceWorkerは諸刃の剣**。障害時の「見かけ上の正常」を生む。PWAでは特にキャッシュバスト手段を持っておく。
4. **`ping` が通ることはL7の健全性を一切保証しない**。TCP 3-way handshake まで確認する。
5. **`after.rules` と `user.rules` の使い分け**。`ufw route allow` は `user.rules` に書かないと効かない。

---

## 📎 関連

- フォロー対応: `runbooks/vps-reboot-checklist.md`
- 続編の障害: `incidents/2026-04-20_kong-ip-shift.md`（今回の対策後も別レイヤーで起きた類似事例）
