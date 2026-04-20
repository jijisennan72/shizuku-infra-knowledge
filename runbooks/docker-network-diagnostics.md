# Dockerネットワーク診断手順

**目的**: Traefik経由でのサービス疎通問題（特に 502/504 エラー）を体系的に診断する。

---

## 🎯 このドキュメントを使うべき症状

- `HTTP/2 504 Gateway Timeout` が返る
- `HTTP/2 502 Bad Gateway` が返る
- Traefikログにエラーが出てない／出てても本質的でない
- ピンポイントではなく一部のサービスだけ落ちている

---

## 🔍 診断フロー（上から順に実行）

### STEP 1: 症状の正確な把握

```bash
# 外部経由
curl -skI https://[ドメイン] | head -5

# VPS内から直接
curl -sI http://localhost:[ポート] | head -3  # アプリ本体
```

### STEP 2: コンテナ稼働状況

```bash
echo "=== 全コンテナ稼働状況 ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"

echo ""
echo "=== 停止してるコンテナ ==="
docker ps -a --filter "status=exited"
```

### STEP 3: Traefikの設定確認

```bash
echo "=== Traefik動的設定ファイル一覧 ==="
ls -la /data/coolify/proxy/dynamic/

echo ""
echo "=== 各設定ファイルの内容 ==="
for f in /data/coolify/proxy/dynamic/*.yaml; do
  echo "--- $f ---"
  cat "$f"
  echo ""
done
```

**チェックポイント**: `url:` に `http://10.0.x.y:port` のようなIP直指定がないか？あればコンテナ名に置換すべき。

### STEP 4: 該当コンテナの現在IPと整合性

```bash
# 対象コンテナ名を環境変数に
TARGET_CONTAINER=supabase-kong-y12thilic710z4ni7c44g9ro  # 例

echo "=== 対象コンテナの実IP ==="
docker inspect $TARGET_CONTAINER \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

echo ""
echo "=== Traefikが参加してるネットワーク ==="
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'
```

**チェックポイント**:
- Traefikが対象のネットワークに参加しているか？
- 設定ファイルのIPが実IPと一致しているか？

### STEP 5: Traefik→対象への疎通テスト

```bash
# コンテナ名で疎通テスト（推奨される方式）
docker exec coolify-proxy wget -qO- --timeout=3 http://$TARGET_CONTAINER:[ポート] 2>&1 | head -3

# IPで疎通テスト（移行前の方式）
docker exec coolify-proxy wget -qO- --timeout=3 http://[IP]:[ポート] 2>&1 | head -3
```

**結果の解釈**:

| 応答 | 判定 |
|---|---|
| HTTPステータス（200/401/404など） | ✅ 疎通OK（200番台ならサービス正常、401/404ならサービスは動いてるが該当パスでの応答） |
| `download timed out` | ❌ 疎通NG（ネットワーク未参加 or IPズレ or コンテナ停止） |
| `Connection refused` | ❌ コンテナ停止 or ポート違い |
| `Name or service not known` | ❌ コンテナ名で名前解決できない（ネットワーク未参加） |

---

## 🛠️ 典型的な問題と対処

### 問題A: IPズレ

**症状**: Traefik → 指定IP で timeout、でも実IPに対しては疎通OK

**対処**: 設定ファイルを **コンテナ名参照** に書き換え

```yaml
# Before
servers:
  - url: "http://10.0.2.7:8000"

# After
servers:
  - url: "http://supabase-kong-y12thilic710z4ni7c44g9ro:8000"
```

Traefik は `providers.file.watch=true` で自動リロード。

### 問題B: Traefikが対象ネットワークに不参加

**症状**: コンテナ名で `Name or service not known`

**対処**: ネットワークに参加させる

```bash
sudo docker network connect [ネットワーク名] coolify-proxy
```

### 問題C: コンテナ停止

**症状**: `docker ps` に出てこない、または `Status: Exited`

**対処**: ログを確認して原因特定してから起動

```bash
docker logs [コンテナ名] --tail 50
docker start [コンテナ名]
```

---

## 📋 診断コマンド一括実行スクリプト

新しい障害に遭遇したとき、まずこれを叩いて出力をアリスに貼る：

```bash
echo "=== 1. 全コンテナ稼働状況 ==="
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Networks}}"
echo ""
echo "=== 2. 停止中コンテナ ==="
docker ps -a --filter "status=exited" --format "table {{.Names}}\t{{.Status}}"
echo ""
echo "=== 3. Traefik設定ファイル ==="
ls -la /data/coolify/proxy/dynamic/
echo ""
for f in /data/coolify/proxy/dynamic/*.yaml; do
  echo "--- $f ---"
  cat "$f"
  echo ""
done
echo "=== 4. Traefikのネットワーク ==="
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'
echo ""
echo "=== 5. Traefikログ（最新50行のエラー系）==="
docker logs coolify-proxy --tail 50 2>&1 | grep -iE "error|gateway|refused|timeout"
```

---

## 📎 関連事例

- `incidents/2026-04-20_kong-ip-shift.md` — 実際にこの手順で10分で復旧した事例
