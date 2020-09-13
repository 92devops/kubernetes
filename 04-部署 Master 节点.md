
####  生成kube-apiserver证书

1.  自签证书颁发机构（CA）

```
~]# mkdir ~/tls/k8s && cd ~/tls/k8s
k8s]# cat > ca-config.json << EOF
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
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
k8s]# cat > ca-csr.json << EOF
{
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "Beijing",
            "ST": "Beijing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
```
k8s]# cfssl gencert -initca ca-csr.json | cfssljson -bare ca -
k8s]# ls *.pem
ca-key.pem  ca.pem
```

2. 使用自签CA签发kube-apiserver HTTPS证书

```
k8s]# cat > server-csr.json << EOF
{
    "CN": "kubernetes",
    "hosts": [
      "10.0.0.1",
      "127.0.0.1",
      "172.18.0.111",
      "172.18.0.112",
      "172.18.0.113",
      "172.18.0.211",
      "172.18.0.212",
      "172.18.0.213",
      "172.18.0.100",
      "kubernetes",
      "kubernetes.default",
      "kubernetes.default.svc",
      "kubernetes.default.svc.cluster",
      "kubernetes.default.svc.cluster.local"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "BeiJing",
            "ST": "BeiJing",
            "O": "k8s",
            "OU": "System"
        }
    ]
}
EOF
```
```
k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes server-csr.json | cfssljson -bare server
k8s]# ls server*pem
server-key.pem  server.pem
```

#### 解压二进制包

```
k8s]# wget https://dl.k8s.io/v1.18.4/kubernetes-server-linux-amd64.tar.gz
k8s]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
k8s]# tar -zxvf kubernetes-server-linux-amd64.tar.gz
k8s]# cd kubernetes/server/bin
k8s]# cp kube-apiserver kube-scheduler kube-controller-manager /opt/kubernetes/bin
k8s]# cp kubectl /usr/bin/
```

#### 部署 kube-apiserver

1. 创建配置文件

```
k8s]# cat > /opt/kubernetes/cfg/kube-apiserver.conf << EOF
KUBE_APISERVER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--etcd-servers=https://172.18.0.111:2379,https://172.18.0.112:2379,https://172.18.0.113:2379 \\
--bind-address=172.18.0.111 \\
--secure-port=6443 \\
--advertise-address=172.18.0.111 \\
--allow-privileged=true \\
--service-cluster-ip-range=10.0.0.0/24 \\
--enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,ResourceQuota,NodeRestriction \\
--authorization-mode=RBAC,Node \\
--enable-bootstrap-token-auth=true \\
--token-auth-file=/opt/kubernetes/cfg/token.csv \\
--service-node-port-range=30000-32767 \\
--kubelet-client-certificate=/opt/kubernetes/ssl/server.pem \\
--kubelet-client-key=/opt/kubernetes/ssl/server-key.pem \\
--tls-cert-file=/opt/kubernetes/ssl/server.pem  \\
--tls-private-key-file=/opt/kubernetes/ssl/server-key.pem \\
--client-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--etcd-cafile=/opt/etcd/ssl/ca.pem \\
--etcd-certfile=/opt/etcd/ssl/server.pem \\
--etcd-keyfile=/opt/etcd/ssl/server-key.pem \\
--audit-log-maxage=30 \\
--audit-log-maxbackup=3 \\
--audit-log-maxsize=100 \\
--audit-log-path=/opt/kubernetes/logs/k8s-audit.log"
EOF
```

2. 提供证书文件

```
~]# cp ~/tls/k8s/server*pem /opt/kubernetes/ssl/
~]#  cp ~/tls/k8s/ca*pem /opt/kubernetes/ssl/
```

3. 启用 TLS Bootstrapping 机制

```
~]# head -c 16 /dev/urandom | od -An -t x | tr -d ' '
e031c08e668453802822c4e95b32a488
~]# cat > /opt/kubernetes/cfg/token.csv << EOF
e031c08e668453802822c4e95b32a488,kubelet-bootstrap,10001,"system:node-bootstrapper"
EOF
```

4.  systemd管理apiserver

```
~]# cat > /usr/lib/systemd/system/kube-apiserver.service << EOF
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-apiserver.conf
ExecStart=/opt/kubernetes/bin/kube-apiserver \$KUBE_APISERVER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

4. 启动并设置开机启动

```
~]# systemctl daemon-reload
~]# systemctl start kube-apiserver
~]# systemctl enable kube-apiserver
```

5. 授权kubelet-bootstrap用户允许请求证书

```
~]# kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```

####  部署kube-controller-manager

1. 创建配置文件

```
~]# cat > /opt/kubernetes/cfg/kube-controller-manager.conf << EOF
KUBE_CONTROLLER_MANAGER_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--leader-elect=true \\
--master=127.0.0.1:8080 \\
--bind-address=127.0.0.1 \\
--allocate-node-cidrs=true \\
--cluster-cidr=10.244.0.0/16 \\
--service-cluster-ip-range=10.0.0.0/24 \\
--cluster-signing-cert-file=/opt/kubernetes/ssl/ca.pem \\
--cluster-signing-key-file=/opt/kubernetes/ssl/ca-key.pem  \\
--root-ca-file=/opt/kubernetes/ssl/ca.pem \\
--service-account-private-key-file=/opt/kubernetes/ssl/ca-key.pem \\
--experimental-cluster-signing-duration=87600h0m0s"
EOF
```

2. systemd管理controller-manager

```
~]# cat > /usr/lib/systemd/system/kube-controller-manager.service << EOF
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-controller-manager.conf
ExecStart=/opt/kubernetes/bin/kube-controller-manager \$KUBE_CONTROLLER_MANAGER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

3. 启动并设置开机启动

```
~]# systemctl daemon-reload
~]# systemctl start kube-controller-manager
~]# systemctl enable kube-controller-manager
```

#### 部署kube-scheduler

1. 创建配置文件

```
~]# cat > /opt/kubernetes/cfg/kube-scheduler.conf << EOF
KUBE_SCHEDULER_OPTS="--logtostderr=false \
--v=2 \
--log-dir=/opt/kubernetes/logs \
--leader-elect \
--master=127.0.0.1:8080 \
--bind-address=127.0.0.1"
EOF
```

2. systemd管理scheduler

```
~]# cat > /usr/lib/systemd/system/kube-scheduler.service << EOF
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-scheduler.conf
ExecStart=/opt/kubernetes/bin/kube-scheduler \$KUBE_SCHEDULER_OPTS
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

3. 启动并设置开机启动

```
~]# systemctl daemon-reload
~]# systemctl start kube-scheduler
~]# systemctl enable kube-scheduler
```

####  查看集群状态

```
~]# kubectl get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok                  
controller-manager   Healthy   ok                  
etcd-1               Healthy   {"health":"true"}   
etcd-2               Healthy   {"health":"true"}   
etcd-0               Healthy   {"health":"true"}  
```

如上输出说明Master节点组件运行正常
