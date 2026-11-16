# dc-dns-switch

DeltaChat 用の DNS 設定を自動的に切り替えるためのスクリプトです。

インターネット接続状態を監視し、`dnsmasq` の DHCP 配布 DNS を以下のように自動変更します。

* **オンライン時**

  * ローカル DNS + 外部 DNS（8.8.8.8、8.8.4.4）を配布
* **オフライン時**

  * ローカル DNS のみを配布

また、HaLow 側経路が利用可能な場合は、`eth0` に設定された IPv4 アドレスを削除し、不要な経路競合を防止します。

---

# 構成ファイル

```
dc-dns-switch/
├── dc-dns-switch.sh        # DNS切替スクリプト
├── dc-dns-switch.service   # systemdサービス
├── dc-dns-switch.timer     # 定期実行タイマー
└── install.sh              # インストールスクリプト
```

---

# 動作概要

## 1. MASTER ノードのみ実行

`/etc/mesh.conf`

```sh
NODE_TYPE=MASTER
```

以外のノードでは処理を行わず終了します。

---

## 2. HaLow 接続確認

`/etc/halow_mesh.conf`

から

```sh
CORE_RELAY_IP=xxx.xxx.xxx.xxx/xx
```

を取得し、

```sh
ping CORE_RELAY_IP
```

で到達可能か確認します。

到達可能で、かつ `eth0` に IPv4 アドレスが設定されている場合は、

```sh
ip addr flush dev eth0
```

を実行して IPv4 を削除します。

---

## 3. インターネット接続確認

以下へ ping を実行します。

```text
8.8.8.8
```

### オンライン時

`/etc/dnsmasq.conf`

の

```text
dhcp-option=6,<LOCALDNS>
```

を

```text
dhcp-option=6,<LOCALDNS>,8.8.8.8,8.8.4.4
```

へ変更します。

その後、

```sh
systemctl restart dnsmasq
```

を実行します。

---

### オフライン時

外部 DNS

```text
,8.8.8.8,8.8.4.4
```

を削除し、

```text
dhcp-option=6,<LOCALDNS>
```

のみに戻します。

その後、

```sh
systemctl restart dnsmasq
```

を実行します。

---

# 必要な設定

## `/etc/mesh.conf`

例

```sh
NODE_TYPE=MASTER
```

---

## `/etc/halow_mesh.conf`

例

```sh
CORE_RELAY_IP=192.168.10.254/24
```

---

## `/etc/dnsmasq.conf`

以下の行が存在している必要があります。

```text
dhcp-option=6,192.168.10.1
```

この IP がローカル DNS として利用されます。

---

# インストール

```sh
tar xzf dc-dns-switch.tar.gz
cd dc-dns-switch
chmod +x install.sh
./install.sh
```

インストールスクリプトは以下を実行します。

```sh
sudo cp dc-dns-switch.sh /usr/local/bin/
sudo cp dc-dns-switch.service /etc/systemd/system/
sudo cp dc-dns-switch.timer /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now dc-dns-switch.timer
```

---

# systemd タイマー

起動後 10 秒で開始し、その後 60 秒ごとに実行されます。

```ini
[Timer]
OnBootSec=10
OnUnitActiveSec=60
```

---

# 動作確認

### タイマー状態

```sh
systemctl status dc-dns-switch.timer
```

### サービスログ監視

```sh
journalctl -u dc-dns-switch.service -f
```

### 手動実行

```sh
sudo /usr/local/bin/dc-dns-switch.sh
```

---

# 想定用途

* DeltaChat メッシュネットワーク
* Raspberry Pi + dnsmasq 環境
* HaLow Wi-Fi メッシュ
* インターネット接続有無に応じた DNS 自動切替
* ローカル通信優先時の eth0 経路競合回避

---

# ライセンス

自由に改変・再利用可能。
運用環境に合わせて十分な検証を行った上で利用してください。

