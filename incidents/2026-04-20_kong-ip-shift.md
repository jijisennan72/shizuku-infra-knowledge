# 2026-04-20 — Supabase Kong IPズレによる 504 Bad Gateway

**発生日**: 2026年4月20日
**復旧完了**: 約10分で復旧（前回の教訓が活きた）
**影響範囲**: Supabase OSS (supabase.hama02.shizuku.net) が504で全断
**カテゴリ**: `docker-network` `traefik` `ip-shift` `container-name-resolution`

---

## 0. エグゼクティブサマリー

- **直接のトリガー**: gikaiプロジェクトのCoolifyへの移植作業
- **本質**: Supabase OSS の `y12thilic710z4ni7c44g9ro` ネットワーク内でコンテナIPが再配置され、**Traefikが設定ファイルで指定していたKongのIP（`10.0.2.7`）に、別コンテナ（imgproxy）が居座る状態**になった
- **恒久対策**: Traefikを該当ネットワークに参加させ、IP指定をコンテナ名参照に変更 → 今後のIPズレに完全免疫

---

## 1. 発端と症状

gikaiをGitHub+Vercel+Supabase Cloud構成から、GitHub+さくらVPS Coolify+Supabase OSS構成に移植した直後、Supabaseが 504 Bad Gateway に。

- hama.shizuku.net は内部的には `HTTP/2 200`（実は正常に動いていた）
- supabase.hama02.shizuku.net は `HTTP/2 504`
- Traefikログには関連エラー無し（これが罠）

---

## 2. 原因究明

診断コマンド：

```bash
# Kongコンテナの現在のIP
docker inspect supabase-kong-y12thilic710z4ni7c44g9ro \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'

# y12... ネットワークの全メンバー
docker network inspect y12thilic710z4ni7c44g9ro \
  --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}'

# Traefikから指定IPへの疎通テスト
docker exec coolify-proxy wget -qO- --timeout=3 http://10.0.2.7:8000
```

### 判明した事実

| 項目 | 値 |
|---|---|
| Traefik設定ファイルが指してるIP | `10.0.2.7` |
| Kongの実IP（現在） | `10.0.2.2` ← ズレてる |
| `10.0.2.7` に今いるコンテナ | **`imgproxy`**（Kongじゃない！） |

つまり **Traefikは「Kongやと思って imgproxy に転送してた」**。imgproxyはKongのREST APIを喋れないため、レスポンスが返らず504タイムアウト。

---

## 3. 復旧手順（10分で完了）

### ステップ1: TraefikをSupabaseのネットワークにも参加させる

```bash
sudo docker network connect y12thilic710z4ni7c44g9ro coolify-proxy

# 確認（2つのネットワークに所属することを確認）
docker inspect coolify-proxy \
  --format '{{range $net, $conf := .NetworkSettings.Networks}}{{$net}}: {{$conf.IPAddress}}{{"\n"}}{{end}}'
# coolify: 10.0.1.3
# y12thilic710z4ni7c44g9ro: 10.0.2.14
```

### ステップ2: Traefik動的設定をコンテナ名参照に書き換え

```bash
sudo tee /data/coolify/proxy/dynamic/supabase-kong.yaml > /dev/null << 'EOF'
http:
  routers:
    supabase-kong:
      rule: "Host(`supabase.hama02.shizuku.net`)"
      service: supabase-kong
      entryPoints:
        - https
      tls:
        certResolver: letsencrypt
  services:
    supabase-kong:
      loadBalancer:
        servers:
          - url: "http://supabase-kong-y12thilic710z4ni7c44g9ro:8000"
EOF
```

### ステップ3: 確認

Traefikは `providers.file.watch=true` のため自動リロード。5秒待ってから：

```bash
curl -skI https://supabase.hama02.shizuku.net/rest/v1/ | head -3
# HTTP/2 401  ← 401ならKongに到達している（APIキーなしでの正常応答）
```

---

## 4. 教訓・ベストプラクティス

### 🏆 Dockerネットワーク間通信はコンテナ名で参照

**IPアドレスではなく、Dockerの内蔵DNSによる名前解決を使う。** これでIPが何回動いても自動追従。

```yaml
# ❌ 悪い例
services:
  my-service:
    loadBalancer:
      servers:
        - url: "http://10.0.2.7:8000"

# ✅ 良い例
services:
  my-service:
    loadBalancer:
      servers:
        - url: "http://supabase-kong-y12thilic710z4ni7c44g9ro:8000"
```

### 🏆 異なるDockerネットワーク間で通信する場合

通信元コンテナ（例：Traefik）を、通信先のネットワークに **`docker network connect`** で追加参加させる。

### ⚠️ `docker network connect` の永続性

`docker network connect` コマンドで追加した接続は、**コンテナ再作成時に消える可能性がある**。永続化するには以下のいずれか：

1. コンテナの docker-compose.yml に networks を追加
2. systemd の ExecStartPost で `docker network connect` を実行
3. 運用ドキュメントに「再作成後に要実行」として明記

**現状はジージが運用ドキュメントに記録する方式を採用。**

---

## 5. 同種トラブルの兆候チェックリスト

他プロジェクトで同じ問題が起きてないか確認する：

```bash
# (A) Traefik動的設定にIP直指定が残っていないか
grep -rE "http://10\.0\.[0-9]+\.[0-9]+:" /data/coolify/proxy/dynamic/

# (B) 各コンテナの現在IPと、設定ファイルで指定してるIPの整合性
for f in /data/coolify/proxy/dynamic/*.yaml; do
  echo "=== $f ==="
  cat "$f"
  echo ""
done

# (C) Traefikから各コンテナへの疎通
docker exec coolify-proxy wget -qO- --timeout=3 http://[指定先]:[port]
```

---

## 📎 関連

- 前提となる事件: `incidents/2026-04-15_vps-reboot-cascade.md`（UFWサブネット化で外→内のIP問題は解決済み、今回はその後に発覚した Docker 内部ネットワーク層の同種問題）
- 運用ルール: `runbooks/docker-network-diagnostics.md`
