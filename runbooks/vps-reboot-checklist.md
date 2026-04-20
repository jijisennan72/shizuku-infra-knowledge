# VPS再起動 チェックリスト

**目的**: VPS再起動前後の確認作業を標準化し、2026-04-15のような複合障害を防ぐ。

---

## 🔴 再起動前（必須）

### 1. アリスへの相談

パッケージ更新内容（特にカーネル・Docker・ネットワーク関連）を確認する。

### 2. 現状記録（IPとルール）

```bash
# TraefikのIPを記録
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}' \
  | tee ~/pre-reboot-traefik-ip.txt

# UFW FORWARDルールを記録
sudo iptables -L ufw-user-forward -n | tee ~/pre-reboot-ufw.txt

# 稼働中コンテナ一覧を記録
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}" | tee ~/pre-reboot-containers.txt
```

### 3. 恒久対策が入っているか確認

- [ ] UFWがサブネット許可（`10.0.x.0/24`）になっている（IP個別指定ではない）
- [ ] 主要コンテナの自動起動（systemd またはrestart policy）が設定済み
- [ ] `/data/coolify/proxy/acme.json` のパーミッションが `600`
- [ ] Traefik動的設定（`.yaml`）がコンテナ名参照になっている（IP直指定ではない）

チェックコマンド：

```bash
# UFW確認
sudo iptables -L ufw-user-forward -n | grep -E "/24"  # サブネット指定があればOK

# systemd確認
systemctl list-unit-files | grep -E "enabled" | grep -iE "hama|app"

# acme.json確認
ls -l /data/coolify/proxy/acme.json  # -rw------- であればOK

# Traefik動的設定確認（IP直指定が残ってないか）
grep -rE "http://10\.0\.[0-9]+\.[0-9]+:" /data/coolify/proxy/dynamic/
# 結果が空ならOK。何か出たらIPをコンテナ名に置換する
```

### 4. 再起動タイミング

- [ ] 深夜帯（住民への影響最小）
- [ ] 作業時間の目安として2時間を確保
- [ ] スマホから SSH 接続できる状態を確認

---

## 🟡 再起動後（必須確認）

### 1. 基本疎通確認

```bash
# HTTPS 応答
curl -sI https://hama.shizuku.net | head -3
curl -skI https://supabase.hama02.shizuku.net/rest/v1/ | head -3

# UFW状態
sudo iptables -L ufw-user-forward -n | grep -E "/24|ACCEPT"
```

### 2. IPズレ確認（再起動後の状態を記録と比較）

```bash
# 現在のTraefik IP
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

# 事前記録と比較
diff ~/pre-reboot-traefik-ip.txt <(docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}')
```

### 3. 全サービス動作確認

- [ ] はまアプリ ログイン動作
- [ ] Supabase Studio アクセス
- [ ] Coolify UI アクセス（SSHトンネル経由）
- [ ] その他の稼働サービス

### 4. docker network connect の再適用

`docker network connect` で追加した接続は再起動で消える可能性があるため確認：

```bash
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'
```

期待値：`coolify` と `y12thilic710z4ni7c44g9ro` の両方が出ていること。

もし `y12...` が消えていたら即時実行：

```bash
sudo docker network connect y12thilic710z4ni7c44g9ro coolify-proxy
```

---

## 🔴 復旧時間の目安

| 状況 | 復旧時間 |
|---|---|
| 恒久対策が全て入っている状態での再起動 | 0〜5分（ダウンタイムほぼなし） |
| 一部対策が欠けている状態 | 10〜60分 |
| 対策ゼロの状態での複合障害 | **数時間（実績：約4時間）** |

---

## 📎 関連

- 初出の事例: `incidents/2026-04-15_vps-reboot-cascade.md`
- Docker内部障害の事例: `incidents/2026-04-20_kong-ip-shift.md`
