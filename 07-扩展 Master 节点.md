#### 复制 master01 的证书和目录

```
~]# scp -r k8s-master01:/opt/kubernetes/ /opt/
~]# scp -r k8s-master01:/opt/cni /opt/
~]# scp -r k8s-master01:/opt/etcd /opt/
~]# scp -r k8s-master01:/usr/lib/systemd/system/kube*  /usr/lib/systemd/system
~]# scp -r k8s-master01:/usr/bin/kubectl /usr/bin/kubectl
```

#### 删除kubelet证书和kubeconfig文件

```
~]# rm -f /opt/kubernetes/cfg/kubelet.kubeconfig
~]# rm -f /opt/kubernetes/ssl/kubelet*
```

#### 修改配置文件IP和主机名

```
~]# vi /opt/kubernetes/cfg/kube-apiserver.conf
--bind-address=172.18.0.112 \
--advertise-address=172.18.0.112 \

~]# vi /opt/kubernetes/cfg/kubelet.conf
--hostname-override=k8s-master02

~]# vi /opt/kubernetes/cfg/kube-proxy-config.yml
hostnameOverride: k8s-master02
```

#### 启动设置开机启动

```
systemctl daemon-reload
systemctl start kube-apiserver
systemctl start kube-controller-manager
systemctl start kube-scheduler
systemctl start kubelet
systemctl start kube-proxy
systemctl enable kube-apiserver
systemctl enable kube-controller-manager
systemctl enable kube-scheduler
systemctl enable kubelet
systemctl enable kube-proxy
```

#### 批准kubelet证书申请

```
~]# kubectl get csr
NAME                                                   AGE   SIGNERNAME                                    REQUESTOR           CONDITION
node-csr-Y9j3KMX-J8c1q9Y12Eqf7JqDOO1XwNiImsBHiGKSpmI   61s   kubernetes.io/kube-apiserver-client-kubelet   kubelet-bootstrap   Pending
~]# kubectl certificate approve node-csr-Y9j3KMX-J8c1q9Y12Eqf7JqDOO1XwNiImsBHiGKSpmI
certificatesigningrequest.certificates.k8s.io/node-csr-Y9j3KMX-J8c1q9Y12Eqf7JqDOO1XwNiImsBHiGKSpmI approved
~]# kubectl get nodes
NAME           STATUS     ROLES    AGE     VERSION
k8s-master01   Ready      <none>   3h59m   v1.18.4
k8s-master02   NotReady   <none>   9s      v1.18.4
k8s-node01     Ready      <none>   172m    v1.18.4
```
