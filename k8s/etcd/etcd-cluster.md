etcd集群部署
1.下载二进程程序
https://github.com/etcd-io/etcd/releases/tag/v3.5.4
wget https://github.com/etcd-io/etcd/releases/download/v3.5.4/etcd-v3.5.4-linux-amd64.tar.gz

2.tar -xvf etcd-v3.5.4-linux-amd64.tar.gz -C /usr/local/etcd
  cp /usr/local/etcd/etcd-v3.5.4-linux-amd64/etcd /usr/bin
  cp /usr/local/etcd/etcd-v3.5.4-linux-amd64/etcdctl /usr/bin
  mkdir -p /data/etcd
3.download 证书配置文件模板
https://github.com/etcd-io/etcd/tree/main/hack/tls-setup/config

4.下载cfssl生成ca和etcd证书
mkdir /etc/etcd/ssl && cd /etc/etcd/ssl 
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd

5.https://etcd.io/docs/v3.5/op-guide/clustering/ 此为参考，操作见下一步
配置自签名证书 
etcd --name infra0 --initial-advertise-peer-urls https://10.0.1.10:2380 \
  --listen-peer-urls https://10.0.1.10:2380 \
  --listen-client-urls https://10.0.1.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --client-cert-auth --trusted-ca-file=/path/to/ca-client.crt \
  --cert-file=/path/to/infra0-client.crt --key-file=/path/to/infra0-client.key \
  --peer-client-cert-auth --peer-trusted-ca-file=ca-peer.crt \
  --peer-cert-file=/path/to/infra0-peer.crt --peer-key-file=/path/to/infra0-peer.key

配置自动证书
 etcd --name infra0 --initial-advertise-peer-urls https://10.0.1.10:2380 \
  --listen-peer-urls https://10.0.1.10:2380 \
  --listen-client-urls https://10.0.1.10:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://10.0.1.10:2379 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-cluster infra0=https://10.0.1.10:2380,infra1=https://10.0.1.11:2380,infra2=https://10.0.1.12:2380 \
  --initial-cluster-state new \
  --auto-tls \
  --peer-auto-tls

6. service文件参考
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos
[Service]
Type=notify
ExecStart=/usr/bin/etcd \
--name=etcd0 \
--data-dir=/data/etcd/data \
--wal-dir=/data/etcd/wal \
--cert-file=/etc/etcd/ssl/etcd.pem \
--key-file=/etc/etcd/ssl/etcd-key.pem \
--trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-cert-file=/etc/etcd/ssl/etcd.pem \
--peer-key-file=/etc/etcd/ssl/etcd-key.pem \
--peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
--peer-client-cert-auth \
--client-cert-auth \
--listen-peer-urls=https://172.17.1.231:2380 \
--initial-advertise-peer-urls=https://172.17.1.231:2380 \
--listen-client-urls=https://172.17.1.231:2379,http://127.0.0.1:2379 \
--advertise-client-urls=https://172.17.1.231:2379 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster="etcd0=https://172.17.1.231:2380,etcd1=https://172.17.1.168:2380,etcd2=https://172.17.1.17:2380" \
--initial-cluster-state=new \
--max-request-bytes=33554432 \
--quota-backend-bytes=6442450944 \
--heartbeat-interval=250 \
--election-timeout=2000
Restart=on-failure
RestartSec=5
LimitNOFILE=65536
[Install]
WantedBy=multi-user.target

7.验证
#无tls
etcdctl  --endpoints="http://172.30.200.98:2380,http://172.30.200.132:2380,http://172.30.200.58:2380" endpoint health
#tls验证
ETCDCTL_API=3  etcdctl --cacert=/etc/ssl/etcd/ca.pem --cert=/etc/ssl/etcd/cert.pem --key=/etc/ssl/etcd/key.pem --endpoints=https://10.211.55.24:2379,https://10.211.55.25:2379,https://10.211.55.26:2379 endpoint health

********有tls验证example***********
ETCDCTL_API=3 etcdctl \
--endpoints=https://172.17.1.231:2379 \
--endpoints=https://172.17.1.168:2379 \
--endpoints=https://172.17.1.17:2379 \
--cacert=/etc/etcd/ssl/ca.pem \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem  \
 endpoint health

endpoint status --write-out=table
验证方法二：
etcdctl put test "Hello, etcd"
etcdctl get test

8.删除节点
etcdctl member remove  13de95822d57af73

#添加节点
1)etcdctl member add  etcd-0 --peer-urls=http://172.30.200.98:2380
2)修改--initial-cluster-state 为 existing（原来为new），并清空data数据
