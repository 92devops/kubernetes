#### 安装程序包

```
~]# yum install epel-release -y
~]# yum install nginx keepalived -y
```

####  Nginx 配置文件

```
cat > /etc/nginx/nginx.conf << "EOF"
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

# 四层负载均衡，为两台Master apiserver组件提供负载均衡
stream {

    log_format  main  '$remote_addr $upstream_addr - [$time_local] $status $upstream_bytes_sent';

    access_log  /var/log/nginx/k8s-access.log  main;

    upstream k8s-apiserver {
       server 172.18.0.111:6443;   # Master1 APISERVER IP:PORT
       server 172.18.0.112:6443;   # Master2 APISERVER IP:PORT
    }

    server {
       listen 6443;
       proxy_pass k8s-apiserver;
    }
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    server {
        listen       80 default_server;
        server_name  _;

        location / {
        }
    }
}
EOF
```

#### keepalived 配置文件

- Master

```
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_MASTER
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0  # 修改为实际网卡名
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的
    priority 100    # 优先级，备服务器设置 90
    advert_int 1    # 指定VRRP 心跳包通告间隔时间，默认1秒
    authentication {
        auth_type PASS      
        auth_pass 1111
    }  
    # 虚拟IP
    virtual_ipaddress {
        172.18.0.100/16
    }
    track_script {
        check_nginx
    }
}
EOF
```

- backup

```
cat > /etc/keepalived/keepalived.conf << EOF
global_defs {
   notification_email {
     acassen@firewall.loc
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc  
   smtp_server 127.0.0.1
   smtp_connect_timeout 30
   router_id NGINX_BACKUP
}

vrrp_script check_nginx {
    script "/etc/keepalived/check_nginx.sh"
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51 # VRRP 路由 ID实例，每个实例是唯一的
    priority 90
    advert_int 1
    authentication {
        auth_type PASS      
        auth_pass 1111
    }  
    virtual_ipaddress {
        172.18.0.100/24
    }
    track_script {
        check_nginx
    }
}
EOF
```

配置文件中检查nginx运行状态脚本

```
cat > /etc/keepalived/check_nginx.sh  << "EOF"
#!/bin/bash
count=$(ps -ef |grep nginx |egrep -cv "grep|$$")

if [ "$count" -eq 0 ];then
    exit 1
else
    exit 0
fi
EOF
chmod +x /etc/keepalived/check_nginx.sh
```

验证

```
~]# curl -k https://172.18.0.100:6443/version
{
  "major": "1",
  "minor": "18",
  "gitVersion": "v1.18.4",
  "gitCommit": "c96aede7b5205121079932896c4ad89bb93260af",
  "gitTreeState": "clean",
  "buildDate": "2020-06-17T11:33:59Z",
  "goVersion": "go1.13.9",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

#### 修改所有Worker Node连接LB VIP

```
~]# sed -i 's@172.18.0.111:6443@172.18.0.100:6443@' /opt/kubernetes/cfg/*
~]# systemctl restart kubelet
~]# systemctl restart kube-proxy
```

至此，一套完整的 Kubernetes 高可用集群就部署完成了！
