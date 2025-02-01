### Install etcd cluster

```
wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz
tar xzvf etcd-v3.5.1-linux-amd64.tar.gz
sudo mv etcd-v3.5.1-linux-amd64/etcd* /usr/local/bin/
chmod +x /usr/local/bin/etcd*
echo 'export PATH=$PATH:/usr/local/bin' >> ~/.bashrc
source ~/.bashrc
```

### Node 1

```
cat <<EOF | sudo tee /etc/etcd/etcd.conf
name: "master01"
data-dir: "/var/lib/etcd/master01"
listen-client-urls: "http://192.168.30.142:2379"
advertise-client-urls: "http://192.168.30.142:2379"
listen-peer-urls: "http://192.168.30.142:2380"
initial-advertise-peer-urls: "http://192.168.30.142:2380"
initial-cluster: "master01=http://192.168.30.142:2380,master02=http://192.168.30.143:2380,master03=http://192.168.30.144:2380"
initial-cluster-token: "etcd-cluster-1"
initial-cluster-state: "new"
EOF
```

### Node 2

```
cat <<EOF | sudo tee /etc/etcd/etcd.conf
name: "master02"
data-dir: "/var/lib/etcd/master02"
listen-client-urls: "http://192.168.30.143:2379"
advertise-client-urls: "http://192.168.30.143:2379"
listen-peer-urls: "http://192.168.30.143:2380"
initial-advertise-peer-urls: "http://192.168.30.143:2380"
initial-cluster: "master01=http://192.168.30.142:2380,master02=http://192.168.30.143:2380,master03=http://192.168.30.144:2380"
initial-cluster-token: "etcd-cluster-1"
initial-cluster-state: "new"
EOF
```

### Node 3

```
cat <<EOF | sudo tee /etc/etcd/etcd.conf
name: "master03"
data-dir: "/var/lib/etcd/master03"
listen-client-urls: "http://192.168.30.144:2379"
advertise-client-urls: "http://192.168.30.144:2379"
listen-peer-urls: "http://192.168.30.144:2380"
initial-advertise-peer-urls: "http://192.168.30.144:2380"
initial-cluster: "master01=http://192.168.30.142:2380,master02=http://192.168.30.143:2380,master03=http://192.168.30.144:2380"
initial-cluster-token: "etcd-cluster-1"
initial-cluster-state: "new"
EOF
```

```
sudo mkdir -p /var/lib/etcd/
```

### Create file `/etc/systemd/system/etcd.service`

```
[Unit]
Description=etcd
Documentation=https://etcd.io/docs/
After=network.target

[Service]
User=etcd
ExecStart=etcd --config-file=/etc/etcd/etcd.conf
Restart=always
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

```
systemctl daemon-reload
systemctl start etcd
systemctl enable etcd 
```
```
ETCDCTL_API=3 etcdctl --endpoints=http://192.168.30.142:2379,http://192.168.30.143:2379,http://192.168.30.144:2379 member list -w table
+------------------+---------+----------+----------------------------+----------------------------+
|        ID        | STATUS  |   NAME   |         PEER ADDRS         |        CLIENT ADDRS        |
+------------------+---------+----------+----------------------------+----------------------------+
| 1e65736a44586a82 | started | master03 | http://192.168.30.144:2380 | http://192.168.30.144:2379 |
| 71f58412ca9015c1 | started | master02 | http://192.168.30.143:2380 | http://192.168.30.143:2379 |
| aeb401d37974f3a3 | started | master01 | http://192.168.30.142:2380 | http://192.168.30.142:2379 |
+------------------+---------+----------+----------------------------+----------------------------+

ETCDCTL_API=3 etcdctl --endpoints=https://192.168.30.142:2379 endpoint status -w table
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
| http://192.168.30.142:2379 | aeb401d37974f3a3 | 3.0.16  | 25 kB   | false     |       248 |       2026 |
| http://192.168.30.143:2379 | 71f58412ca9015c1 | 3.0.16  | 25 kB   | false     |       248 |       2026 |
| http://192.168.30.144:2379 | 1e65736a44586a82 | 3.0.16  | 25 kB   | true      |       248 |       2026 |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+

ETCDCTL_API=3 etcdctl --endpoints=http://192.168.30.142:2379,http://192.168.30.143:2379,http://192.168.30.144:2379 endpoint status -w table
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
|          ENDPOINT          |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
| http://192.168.30.142:2379 | aeb401d37974f3a3 | 3.0.16  | 25 kB   | false     |       248 |       2042 |
| http://192.168.30.143:2379 | 71f58412ca9015c1 | 3.0.16  | 25 kB   | false     |       248 |       2042 |
| http://192.168.30.144:2379 | 1e65736a44586a82 | 3.0.16  | 25 kB   | true      |       248 |       2042 |
+----------------------------+------------------+---------+---------+-----------+-----------+------------+
```

```
curl http://192.168.30.142:2379/health
```