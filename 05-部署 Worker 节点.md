#### 创建工作目录并拷贝二进制文件

在 master01 节点操作

```
~]# mkdir -p /opt/kubernetes/{bin,cfg,ssl,logs}
~]# tar -xf kubernetes-server-linux-amd64.tar.gz
~]# cd kubernetes/server/bin
~]# cp kubelet kube-proxy /opt/kubernetes/bin
~]# scp k8s-master01:/root/tls/k8s/*pem /opt/kubernetes/ssl/
```

#### 部署 kubelet

1. 生成bootstrap.kubeconfig文件

```
~]# KUBE_APISERVER="https://172.18.0.111:6443"
~]# TOKEN="e031c08e668453802822c4e95b32a488"
~]# kubectl config set-cluster kubernetes   --certificate-authority=/opt/kubernetes/ssl/ca.pem   --embed-certs=true   --server=${KUBE_APISERVER}   --kubeconfig=bootstrap.kubeconfig
~]# kubectl config set-credentials "kubelet-bootstrap"   --token=${TOKEN}   --kubeconfig=bootstrap.kubeconfig
~]# kubectl config set-context default   --cluster=kubernetes   --user="kubelet-bootstrap"   --kubeconfig=bootstrap.kubeconfig
~]# kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```
```
~]# cp bootstrap.kubeconfig /opt/kubernetes/cfg
```

2. 创建配置文件

```
~]# cat > /opt/kubernetes/cfg/kubelet.conf << EOF
KUBELET_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--hostname-override=k8s-master01 \\
--network-plugin=cni \\
--kubeconfig=/opt/kubernetes/cfg/kubelet.kubeconfig \\
--bootstrap-kubeconfig=/opt/kubernetes/cfg/bootstrap.kubeconfig \\
--config=/opt/kubernetes/cfg/kubelet-config.yml \\
--cert-dir=/opt/kubernetes/ssl \\
--pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.2"
EOF
```

3. 配置参数文件

```
~]# cat > /opt/kubernetes/cfg/kubelet-config.yml << EOF
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
cgroupDriver: cgroupfs
clusterDNS:
- 10.0.0.2
clusterDomain: cluster.local
failSwapOn: false
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kubernetes/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
maxOpenFiles: 1000000
maxPods: 110
EOF
```

4. systemd管理kubelet

```
~]# cat > /usr/lib/systemd/system/kubelet.service << EOF
[Unit]
Description=Kubernetes Kubelet
After=docker.service

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kubelet.conf
ExecStart=/opt/kubernetes/bin/kubelet \$KUBELET_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

5. 启动并设置开机启动

```
~]# cat systemctl daemon-reload
~]# cat systemctl start kubelet
~]# cat systemctl enable kubelet
```

6.  批准kubelet证书申请并加入集群

```
~]# kubectl get csr
NAME                                                   AGE    SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-iZrUQAHLQxgAZ-sBJ12b3hjfCKaHd2mW3FjU_cL4OR0   9m7s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
[root@k8s-master01 ~]# kubectl certificate approve node-csr-iZrUQAHLQxgAZ-sBJ12b3hjfCKaHd2mW3FjU_cL4OR0
certificatesigningrequest.certificates.k8s.io/node-csr-iZrUQAHLQxgAZ-sBJ12b3hjfCKaHd2mW3FjU_cL4OR0 approved
[root@k8s-master01 ~]# kubectl get node
NAME           STATUS     ROLES    AGE   VERSION
k8s-master01   NotReady   <none>   11s   v1.18.4
```

#### 部署 kube-proxy

1. 创建配置文件

```
~]# cat > /opt/kubernetes/cfg/kube-proxy.conf << EOF
KUBE_PROXY_OPTS="--logtostderr=false \\
--v=2 \\
--log-dir=/opt/kubernetes/logs \\
--config=/opt/kubernetes/cfg/kube-proxy-config.yml"
EOF
```

2. 配置参数文件

```
~]# cat > /opt/kubernetes/cfg/kube-proxy-config.yml << EOF
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
metricsBindAddress: 0.0.0.0:10249
clientConnection:
  kubeconfig: /opt/kubernetes/cfg/kube-proxy.kubeconfig
hostnameOverride: k8s-master01
clusterCIDR: 10.0.0.0/24
EOF
```

3. 生成kube-proxy.kubeconfig文件

```
~]# cd ~/tls/k8s/
k8s]# cat > kube-proxy-csr.json << EOF
{
  "CN": "system:kube-proxy",
  "hosts": [],
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
k8s]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy
k8s]# ls kube-proxy*pem
kube-proxy-key.pem  kube-proxy.pem
```

4. 生成kubeconfig文件

```
k8s]# KUBE_APISERVER="https://172.18.0.111:6443"
k8s]# kubectl config set-cluster kubernetes   --certificate-authority=/opt/kubernetes/ssl/ca.pem   --embed-certs=true   --server=${KUBE_APISERVER}   --kubeconfig=kube-proxy.kubeconfig
k8s]# kubectl config set-credentials kube-proxy   --client-certificate=./kube-proxy.pem   --client-key=./kube-proxy-key.pem   --embed-certs=true   --kubeconfig=kube-proxy.kubeconfig
k8s]# kubectl config set-context default   --cluster=kubernetes   --user=kube-proxy   --kubeconfig=kube-proxy.kubeconfig
k8s]# kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```
```
k8s]# cp kube-proxy.kubeconfig /opt/kubernetes/cfg/
```

5. systemd管理kube-proxy

```
k8s]# cat > /usr/lib/systemd/system/kube-proxy.service << EOF
[Unit]
Description=Kubernetes Proxy
After=network.target

[Service]
EnvironmentFile=/opt/kubernetes/cfg/kube-proxy.conf
ExecStart=/opt/kubernetes/bin/kube-proxy \$KUBE_PROXY_OPTS
Restart=on-failure
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
EOF
```

6. 启动并设置开机自启

```
k8s]# systemctl daemon-reload
k8s]# systemctl  start kube-proxy
k8s]# systemctl  enable kube-proxy
```

#### 部署 CNI 网络

```
~]# wget https://github.com/containernetworking/plugins/releases/download/v0.8.6/cni-plugins-linux-amd64-v0.8.6.tgz
~]# mkdir -pv /opt/cni/bin
~]# tar -zxvf cni-plugins-linux-amd64-v0.8.6.tgz -C /opt/cni/bin
```
```
~]# wget https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
~]# sed -i 's@quay.io/coreos/flannel@registry.cn-beijing.aliyuncs.com/dengyou/flannel@g' kube-flannel.yml
~]# kubectl  apply -f kube-flannel.yml
~]# kubectl  get nodes
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   55m   v1.18.4
```

#### 授权apiserver访问kubelet

```
~]# cat > apiserver-to-kubelet-rbac.yaml << EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
      - pods/log
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes
EOF

~]# kubectl apply -f apiserver-to-kubelet-rbac.yaml
```

#### 新增加Worker Node

1. 拷贝已部署好的Node相关文件到新节点

```
~]# scp -r /opt/kubernetes/ node01:/opt
~]# scp /usr/lib/systemd/system/{kubelet.service,kube-proxy.service} node01:/usr/lib/systemd/system/
~]# scp -r /opt/cni/ node01:/opt/
~]# scp /opt/kubernetes/ssl/ca.pem  node01:/opt/kubernetes/ssl
```

2. 删除kubelet证书和kubeconfig文件

```
~]# rm /opt/kubernetes/cfg/kubelet.kubeconfig
~]# rm -f /opt/kubernetes/ssl/kubelet*
```

3. 修改主机名

```
~]# vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-node01

~]# vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-node01
```

4. 启动并设置开机启动

```
~]# systemctl daemon-reload
~]# for i in kubelet kube-proxy;do systemctl start $i;done
~]# for i in kubelet kube-proxy;do systemctl enable $i;done
```

5. 在Master上批准新Node kubelet证书申请

```
~]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-iZrUQAHLQxgAZ-sBJ12b3hjfCKaHd2mW3FjU_cL4OR0   76m   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Approved,Issued
node-csr-lCPAnDGG4A4mwqUe5bmcuz6cIdg6Al8DanCwynWvdIQ   55s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
~]# kubectl certificate approve node-csr-lCPAnDGG4A4mwqUe5bmcuz6cIdg6Al8DanCwynWvdIQ
```

6. 查看Node状态

```
~]# kubectl get node
NAME           STATUS   ROLES    AGE   VERSION
k8s-master01   Ready    <none>   67m   v1.18.4
k8s-node01     Ready    <none>   39s   v1.18.4
```

```
~]# kubectl  get pod -A
NAMESPACE     NAME                          READY   STATUS    RESTARTS   AGE
kube-system   kube-flannel-ds-amd64-2nx2w   1/1     Running   0          13m
kube-system   kube-flannel-ds-amd64-69hkb   1/1     Running   0          69s
```
