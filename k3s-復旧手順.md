# RaspberryPi 起動後の通常フロー

k3s / docker は systemd で自動起動する設定になっている (`systemctl is-enabled k3s` = enabled)。

### 電源ON後の流れ

1. **Pi を起動** → 自動で元WiFi(172.29.1.0/24)に接続
2. **k3s が自動起動** → 1〜2分でクラスタが Ready
3. **circle-detector Namespace のPodも自動復帰** (Deployment/PVCは永続化済み)
4. ブラウザで `http://localhost:30500/` を開くだけでFlask UIが使える

### 起動後の健全性チェック（任意）

```bash
sudo kubectl get nodes                       # Ready
sudo kubectl get pods -n circle-detector     # すべて Running (1/1)
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:30500/   # 200
```

### 注意

- 違うWiFiに接続して起動すると、下の「復旧手順」が必要になる。
- デスクトップの `Circle Detector` ショートカット(docker版)は**使わない**。k3s と /dev/video0 を取り合うため。ブラウザで `http://localhost:30500/` を直接開く。

---

# k3s 復旧手順（元のWiFiに戻した後）

## 背景
- k3s は以前のWiFi (ノードIP `172.29.1.4`) で構築された。
- 別WiFi に接続したことでノード実IPが `10.89.14.24` に変わり、flannelオーバーレイが機能しなくなりPod/PVCが全てPendingになった。
- **元のWiFiに戻す**ことで復旧するはず。

---

## 手順

### 1. WiFiを元のネットワークに戻す

タスクバーのネットワークアイコンから接続し直す。

### 2. ホストIPが戻ったか確認

```bash
ip -4 addr show wlan0 | grep inet
```

→ `inet 172.29.1.4/...` が表示されればOK。

### 3. k3s クラスタの回復を確認（1〜2分待つ）

```bash
sudo kubectl get nodes
# STATUS=Ready であること

sudo kubectl get pods -A
# kube-system 配下 (coredns, local-path-provisioner, metrics-server) がすべて Running
```

provisioner のエラーが止まるまで少し時間がかかる。止まらないときは:

```bash
sudo kubectl logs -n kube-system -l app=local-path-provisioner --tail=20
# "dial tcp 10.43.0.1:443: i/o timeout" が止まっていればOK
```

### 4. アプリ(circle-detector)のPodが上がるのを待つ

マニフェストは既に apply 済み(`kubectl apply -f` 実行済み)なので、ネットワーク回復後は自動で進むはず。

```bash
sudo kubectl get pods,pvc -n circle-detector -w
```

期待される遷移:
- PVC: Pending → Bound
- Pod: Pending → ContainerCreating → Running (1/1)

Ctrl+Cで watch 終了。

### 5. Flask UI 動作確認

```bash
curl -s -o /dev/null -w "HTTP %{http_code}\n" http://localhost:30500/
# → HTTP 200 なら成功
```

ブラウザで開く:

```
http://localhost:30500/
```

設定保存ボタンの動作も確認する。

---

## トラブルシューティング

### Podが Running にならない

```bash
# pod毎の詳細
sudo kubectl describe pod -n circle-detector <pod名>

# イベント確認
sudo kubectl get events -n circle-detector --sort-by=.lastTimestamp | tail -20
```

### PVCが Bound にならない

```bash
sudo kubectl logs -n kube-system -l app=local-path-provisioner --tail=40
```

API server到達エラーが続く場合はk3s再起動:

```bash
sudo systemctl restart k3s
# 30秒ほど待ってから再確認
```

### それでもダメな場合（k3s側が固まっている）

一度全削除して再apply:

```bash
sudo kubectl delete ns circle-detector
cd /home/pi/Desktop/k3s-server-raspi/color-detector-app/k3s
sudo bash xxx-startup-k3s.sh
```

---

## 注意事項

- **docker compose の config-ui は起動しないこと**
  - `/dev/video0` がk3sのdetector Podと競合するため
  - 既に起動していたら:
    ```bash
    cd /home/pi/Apps/color-detector-app && docker compose --profile ui down
    ```

- **デスクトップアプリ (circle-detector.desktop) を使うとdockerが起動するので注意**
  - k3s運用中はこのショートカットから起動しない
  - 代わりに `http://localhost:30500/` をブラウザで直接開く

- **再びWiFi変更する予定がある場合**
  - k3sを固定IPで再構築するのが恒久対策
  - `--node-ip=<固定IP> --flannel-iface=wlan0` を指定して再インストール
  - 詳細は別途相談

---

## 参考情報

| 項目 | 値 |
|------|----|
| namespace | circle-detector |
| Flask UI NodePort | 30500 |
| MQTT (内部) | mqtt-service:1883 |
| 元WiFiのノードIP | 172.29.1.4 |
| アプリ設定ファイル(k3s) | settings-configmap.yaml (ConfigMap) |
| マニフェスト置き場 | /home/pi/Desktop/k3s-server-raspi/color-detector-app/k3s/ |
| docker用設定(k3sとは別物) | /home/pi/Apps/color-detector-app/config/settings.json |
