
#### 部署 CoreDNS

```
~]# git clone https://github.com/coredns/deployment.git
~]# cd deployment/kubernetes/
kubernetes]#  ./deploy.sh -i 10.0.0.10 -r "10.0.0.10/12" -s -t coredns.yaml.sed | kubectl apply  -f -
```
```
coredns]# bash deploy.sh -i 10.96.0.10 -r "10.96.0.0/12" -s -t coredns.yaml.sed | kubectl apply -f -
```
```
[root@k8s-master coredns]# kubectl  get pod  -n kube-system
NAME                          READY   STATUS    RESTARTS   AGE
coredns-f9cdf5b99-44hhc       1/1     Running   0          33s
kube-flannel-ds-amd64-hkhfw   1/1     Running   1          12m
```
```
kubectl run -it --rm dns-test --image=busybox:1.28.4 sh
If you don't see a command prompt, try pressing enter.

/ # nslookup kubernetes
Server:    10.0.0.2
Address 1: 10.0.0.2 kube-dns.kube-system.svc.cluster.local

Name:      kubernetes
Address 1: 10.0.0.1 kubernetes.default.svc.cluster.local
```
#### 部署 DashBoard

```
~]# wget https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta8/aio/deploy/recommended.yaml
```

默认Dashboard只能集群内部访问，修改Service为NodePort类型，暴露到外部

```
~]# vim recommended.yaml
...
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30443
  selector:
    k8s-app: kubernetes-dashboard
...
```
```
~]# kubectl get pods,svc -n kubernetes-dashboard
NAME                                             READY   STATUS    RESTARTS   AGE
pod/dashboard-metrics-scraper-694557449d-cmqhw   1/1     Running   0          53s
pod/kubernetes-dashboard-9774cc786-xt4x7         1/1     Running   0          53s

NAME                                TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)          AGE
service/dashboard-metrics-scraper   NodePort    10.0.0.136   <none>        8000:30001/TCP   53s
service/kubernetes-dashboard        ClusterIP   10.0.0.222   <none>        443/TCP          53s
```
