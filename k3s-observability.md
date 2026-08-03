# k3s 運用監視ガイド — どこを見たら何がわかるか

## 1. Pod の稼働状況

```bash
sudo k3s kubectl get pods -n plogger
```

| 表示 | 意味 |
|---|---|
| `1/1 Running` | 正常稼働 |
| `0/1 ContainerCreating` | 起動中（長ければ `describe` で原因確認） |
| `CrashLoopBackOff` | 繰り返しクラッシュ中 |
| `Error` | 直近で落ちた |

期待する正常状態：

| Pod | 正常 | 備考 |
|---|---|---|
| mosquitto | `1/1 Running` | |
| oracle-jdbc | `1/1 Running` | |
| bridge | `1/1 Running` | |
| detector | `1/1 Running` | USB カメラ接続時のみ。未接続は `ContainerCreating` のまま |

---

## 2. Oracle へのデータ登録確認（bridge ログ）

**Oracle に直接接続しなくても確認できる最も確実な方法。**

```bash
sudo k3s kubectl logs -n plogger deploy/bridge | grep -E "merge_committed|merge_failed"
```

### 登録成功

```json
{
  "event": "merge_committed",
  "event_type": "EXIT",
  "mk_date": "20260624230512",
  "rows_affected": 1,
  "profile": "taden-ot-ap"
}
```

- `rows_affected: 1` → Oracle の `HF1RCM01` に INSERT された
- `rows_affected: 0` → 同一キーが既に存在（重複。エラーではない）

### 登録失敗

```json
{
  "event": "merge_failed",
  "ora_code": 17820,
  "retry_count": 11
}
```

- `ora_code` で Oracle 側のエラー原因が分かる（後述）

---

## 3. Oracle 接続エラーの原因（ora_code）

| ora_code | 意味 | 対処 |
|---|---|---|
| `12170` | TCP タイムアウト | taden-ot-ap に未接続 |
| `17820` | ネットワークアダプタ接続不可 | taden-ot-ap 切断直後・ルート未確立 |
| `12899` | カラムサイズ超過 | `mk_date` のフォーマット誤り（`YYYYMMDDHHMMSS` 14文字が正しい） |
| `1017` | 認証失敗 | ユーザー名/パスワード誤り |

---

## 4. 送信待ちデータ（inbox）の確認

```bash
sudo k3s kubectl logs -n plogger deploy/bridge | grep "inbox_count" | tail -5
```

- `inbox_count: N` → taden-ot-ap に接続するまで N 件が送信待ち
- taden-ot-ap 接続後に `inbox_count` が減れば送信成功

---

## 5. SSID 判定ログ（bridge がどのネットワーク上にいるか）

```bash
sudo k3s kubectl logs -n plogger deploy/bridge | grep "periodic" | tail -10
```

```json
{ "event": "periodic", "current_ssid": "taden-ot-ap", "ntp_synced": true, "inbox_count": 0 }
```

- `current_ssid: "taden-ot-ap"` → 工場ネット接続中。Oracle 送信が有効
- `current_ssid: それ以外` → Oracle 送信停止中（`drop_unknown_ssid` で破棄）

---

## 6. oracle-jdbc の詳細エラー

```bash
sudo k3s kubectl logs -n plogger deploy/oracle-jdbc | grep -E "WARNING|ERROR" | tail -20
```

- Java スタックトレースと ORA コードが出る
- 成功時はログなし（`/healthz` の応答のみ）

oracle-jdbc のヘルスチェック：

```bash
JDBC_IP=$(sudo k3s kubectl get svc oracle-jdbc -n plogger -o jsonpath='{.spec.clusterIP}')
curl -s http://$JDBC_IP:8086/healthz
# → "ok" なら起動中
```

---

## 7. 時刻同期の確認

```bash
timedatectl status | grep -E "synchronized|NTP|Local time"
```

- `System clock synchronized: yes` → NTP 同期済み
- taden-ot-ap 接続時は `192.168.250.1` が NTP サーバーとして使われる
  （`/etc/systemd/timesyncd.conf.d/factory.conf` に自動生成）

---

## 8. k3s ノードと CoreDNS の状態

```bash
# ノード状態
sudo k3s kubectl get nodes

# CoreDNS（Pod 間 DNS 解決に必要）
sudo k3s kubectl get pod -n kube-system | grep coredns

# kubernetes サービスエンドポイント（10.42.0.1 であること）
sudo k3s kubectl get endpointslice kubernetes -n default \
  -o jsonpath='{.endpoints[0].addresses[0]}'
```

- エンドポイントが `172.29.1.4`（taden-ot-ap の IP）になっている場合は CoreDNS が機能しない
  → `10.42.0.1` になっていること（`/etc/rancher/k3s/config.yaml` で固定済み）

---

## 9. 全体ログ一括確認

```bash
# 直近のすべての bridge ログ
sudo k3s kubectl logs -n plogger deploy/bridge --tail=50

# リアルタイム監視
sudo k3s kubectl logs -n plogger deploy/bridge -f
```

---

## まとめ：よく使うコマンド早見表

| 知りたいこと | コマンド |
|---|---|
| Pod が全部動いているか | `sudo k3s kubectl get pods -n plogger` |
| Oracle に書き込めたか | `... logs deploy/bridge \| grep merge_committed` |
| 送信待ちは何件か | `... logs deploy/bridge \| grep inbox_count \| tail -3` |
| 今どのネットワークか | `... logs deploy/bridge \| grep periodic \| tail -3` |
| oracle-jdbc は生きているか | `curl -s http://$JDBC_IP:8086/healthz` |
| 時刻同期されているか | `timedatectl status` |
| k3s DNS が壊れていないか | `k3s kubectl get endpointslice kubernetes -n default -o jsonpath='{.endpoints[0].addresses[0]}'` |
