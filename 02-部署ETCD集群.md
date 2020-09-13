#### 部署Etcd集群

Etcd 是一个分布式键值存储系统，Kubernetes使用Etcd进行数据存储，所以先准备一个Etcd数据库，为解决Etcd单点故障，应采用集群方式部署，这里使用3台组建集群。

| 主机名称 | IP |
| --- | --- |
| etcd01 | 172.18.0.111 |
| etcd02 | 172.18.0.112 |
| etcd03 | 172.18.0.113 |


1.  生成 Etcd CA 证书

```
~]# cd ~/tls/etcd
etcd]# cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "www": {
         "expiry": "87600h",
         "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ]
      }
    }
  }
}
EOF

etcd]# cat > ca-csr.json << EOF
{
    "CN": "etcd CA",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing"
        }
    ]
}
EOF
```

```
etcd]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
etcd]# ls *.pem
ca-key.pem  ca.pem
```

2. 使用自签CA签发Etcd HTTPS证书

```
etcd]# cat > server-csr.json << EOF
{
    "CN": "etcd",
    "hosts": [
    "172.18.0.111",
    "172.18.0.112",
    "172.18.0.113"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing"
        }
    ]
}
EOF
```
```
etcd]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=www server-csr.json | cfssljson -bare server
etcd]# ls server*.pem
server-key.pem  server.pem
```

3. 部署 etcd 集群

- 获取etcd 程序包

```
etcd]# wget https://github.com/etcd-io/etcd/releases/download/v3.4.9/etcd-v3.4.9-linux-amd64.tar.gz
```

- 创建工作目录并解压二进制包

```
etcd]# mkdir /opt/etcd/{bin,cfg,ssl} -p
etcd]# tar -zxvf etcd-v3.4.9-linux-amd64.tar.gz
etcd]# mv etcd-v3.4.9-linux-amd64/{etcd,etcdctl} /opt/etcd/bin/
```

- 创建 etcd 配置文件

```
etcd]# cat > /opt/etcd/cfg/etcd.conf << EOF
#[Member]
ETCD_NAME="etcd01"
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.18.0.111:2380"
ETCD_LISTEN_CLIENT_URLS="https://172.18.0.111:2379"

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.18.0.111:2380"
ETCD_ADVERTISE_CLIENT_URLS="https://172.18.0.111:2379"
ETCD_INITIAL_CLUSTER="etcd01=https://172.18.0.111:2380,etcd02=https://172.18.0.112:2380,etcd03=https://172.18.0.113:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
EOF
```

- 创建 unit 文件

```
etcd]# cat > /usr/lib/systemd/system/etcd.service << EOF
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
EnvironmentFile=/opt/etcd/cfg/etcd.conf
ExecStart=/opt/etcd/bin/etcd \
--cert-file=/opt/etcd/ssl/server.pem \
--key-file=/opt/etcd/ssl/server-key.pem \
--peer-cert-file=/opt/etcd/ssl/server.pem \
--peer-key-file=/opt/etcd/ssl/server-key.pem \
--trusted-ca-file=/opt/etcd/ssl/ca.pem \
--peer-trusted-ca-file=/opt/etcd/ssl/ca.pem \
--logger=zap
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

- 拷贝证书

```
etcd]# cp ~/tls/etcd/{ca*pem,server*pem} /opt/etcd/ssl/
```

- 拷备 etcd01 节点上的文件到etcd02, etcd03 节点

```
~]# for i in etcd02 etcd03;do scp -r /opt/etcd/ $i:/opt/;done
 ~]# for i in etcd02 etcd03;do scp /usr/lib/systemd/system/etcd.service $i:/usr/lib/systemd/system/etcd.service;done
```
- 修改 /opt/etcd/cfg/etcd.conf 为各自节点独有的配置文件

```
~]# cat /opt/etcd/cfg/etcd.conf
#[Member]
ETCD_NAME="etcd02" # 修改此处 对应 etcd02、 etcd03
ETCD_DATA_DIR="/var/lib/etcd/default.etcd"
ETCD_LISTEN_PEER_URLS="https://172.18.0.112:2380" # 修改此处为当前服务器IP
ETCD_LISTEN_CLIENT_URLS="https://172.18.0.112:2379" # 修改此处为当前服务器IP

#[Clustering]
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://172.18.0.112:2380" # 修改此处为当前服务器IP
ETCD_ADVERTISE_CLIENT_URLS="https://172.18.0.112:2379" # 修改此处为当前服务器IP
ETCD_INITIAL_CLUSTER="etcd01=https://172.18.0.111:2380,etcd02=https://172.18.0.112:2380,etcd03=https://172.18.0.113:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
ETCD_INITIAL_CLUSTER_STATE="new"
```

- 启动并设置开机启动

```
~]# systemctl  daemon-reload
~]# systemctl  start etcd
~]# systemctl  enable etcd
```

- 验证集群

```
~]# ETCDCTL_API=3 /opt/etcd/bin/etcdctl --cacert=/opt/etcd/ssl/ca.pem --cert=/opt/etcd/ssl/server.pem --key=/opt/etcd/ssl/server-key.pem --endpoints="https://172.18.0.111:2379,https://172.18.0.112:2379,https://172.18.0.113:2379" endpoint health  
https://172.18.0.111:2379 is healthy: successfully committed proposal: took = 12.215159ms
https://172.18.0.113:2379 is healthy: successfully committed proposal: took = 12.639871ms
https://172.18.0.112:2379 is healthy: successfully committed proposal: took = 13.622818ms
```
