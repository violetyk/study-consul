# study-consul


# インストール

- バイナリ落としてきて配置

```sh
wget https://dl.bintray.com/mitchellh/consul/0.5.2_linux_amd64.zip
unzip 0.5.2_linux_amd64.zip
sudo mv consul /usr/local/bin/
consul -v

Consul v0.5.2
Consul Protocol: 2 (Understands back to: 1)
```


# エージェントを起動

- consul01(server)

```sh
$ consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul -node consul01
==> WARNING: BootstrapExpect Mode is specified as 1; this is the same as Bootstrap mode.
==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
==> WARNING: It is highly recommended to set GOMAXPROCS higher than 1
==> Starting raft data migration...
==> Starting Consul agent...
==> Starting Consul agent RPC...
==> Consul agent running!
         Node name: 'consul01'
        Datacenter: 'dc1'
            Server: true (bootstrap: true)
       Client Addr: 127.0.0.1 (HTTP: 8500, HTTPS: -1, DNS: 8600, RPC: 8400)
      Cluster Addr: 10.0.2.15 (LAN: 8301, WAN: 8302)
    Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
             Atlas: <disabled>

==> Log data will now stream in as it occurs:

    2015/10/12 13:23:22 [INFO] serf: EventMemberJoin: consul01 10.0.2.15
    2015/10/12 13:23:22 [INFO] serf: EventMemberJoin: consul01.dc1 10.0.2.15
    2015/10/12 13:23:22 [INFO] raft: Node at 10.0.2.15:8300 [Follower] entering Follower state
    2015/10/12 13:23:22 [INFO] consul: adding server consul01 (Addr: 10.0.2.15:8300) (DC: dc1)
    2015/10/12 13:23:22 [INFO] consul: adding server consul01.dc1 (Addr: 10.0.2.15:8300) (DC: dc1)
    2015/10/12 13:23:22 [ERR] agent: failed to sync remote state: No cluster leader
    2015/10/12 13:23:23 [WARN] raft: Heartbeat timeout reached, starting election
    2015/10/12 13:23:23 [INFO] raft: Node at 10.0.2.15:8300 [Candidate] entering Candidate state
    2015/10/12 13:23:23 [INFO] raft: Election won. Tally: 1
    2015/10/12 13:23:23 [INFO] raft: Node at 10.0.2.15:8300 [Leader] entering Leader state
    2015/10/12 13:23:23 [INFO] consul: cluster leadership acquired
    2015/10/12 13:23:23 [INFO] consul: New leader elected: consul01
    2015/10/12 13:23:23 [INFO] raft: Disabling EnableSingleNode (bootstrap)
    2015/10/12 13:23:23 [INFO] consul: member 'consul01' joined, marking health alive
    2015/10/12 13:23:23 [INFO] consul: member 'ubuntu-1404' reaped, deregistering
    2015/10/12 13:23:26 [INFO] agent: Synced service 'consul'
```

# クラスタメンバの確認

- `consul members`
- クラスタメンバは自分だけ

```sh
$ consul members
Node      Address         Status  Type    Build  Protocol  DC
consul01  10.0.2.15:8301  alive   server  0.5.2  2         dc1
```

# HTTP APIインターフェース

デフォルトは8500番ポート

```sh 
$ curl localhost:8500/v1/catalog/nodes
[{"Node":"consul01","Address":"10.0.2.15"}]
```

# DNS インターフェース

- デフォルトは8600番ポートでDNSクエリを受ける
- .consulドメインをメンバー解決用に使う
- [.consulドメインは設定で変更可能](https://www.consul.io/docs/agent/options.html#domain)


## ノードの名前解決

- `<ホスト名>.node.<データセンター名>.consul`
- `<ホスト名>.node.consul` ※同一のデータセンタで動作している場合が明示的な場合

```
dig @127.0.0.1 -p 8600 consul01.node.consul. any
```

# エージェントの停止

Ctrl-C


# サービスの登録

- どちらかで行う
  - Service Difinition
  - HTTP API


## Service Difinitionによるサービス登録

consulの設定ファイルを置くディレクトリを作成

```sh
sudo mkdir /etc/consul.d
```

80番ポートで動くサービス名`web`、`rails`というタグをつけてJSON設定ファイルを作成

```sh
echo '{"service": {"name": "web", "tags": ["rails"], "port": 80}}' \
    >/etc/consul.d/web.json
```

設定ファイルディレクトリを指定してcunsul再起動すると、登録したサービス`web`を発見

```sh
consul agent -server -bootstrap-expect 1 -data-dir /tmp/consul -node consul01 -config-dir /etc/consul.d
...
[INFO] agent: Synced service 'web'
```

## DNS API

### <サービス名>.service.consul
```sh
dig @127.0.0.1 -p 8600 web.service.consul
```


### SRVレコードはアドレスとポートの両方を返す

```sh
$ dig @127.0.0.1 -p 8600 web.service.consul SRV

;; ANSWER SECTION:
web.service.consul.     0       IN      SRV     1 1 80 consul01.node.dc1.consul.

;; ADDITIONAL SECTION:
consul01.node.dc1.consul. 0     IN      A       10.0.2.15
```

### <タグ名>.<サービス名>.service.consul

```sh
dig @127.0.0.1 -p 8600 rails.web.service.consul
```

## HTTP API

HTTPでも問い合わせできる

```sh
$ curl http://localhost:8500/v1/catalog/service/web
[{"Node":"consul01","Address":"10.0.2.15","ServiceID":"web","ServiceName":"web","ServiceTags":["rails"],"ServiceAddress":"","ServicePort":80}]
```

## 設定ファイルの再読み込み

ほかのデーモン同様に、`SIGHUP`を送れば再読み込みする

```sh
kill -1 <プロセスID>
```


# Consul Cluster



