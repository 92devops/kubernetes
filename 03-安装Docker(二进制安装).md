在所有节点安装Docker

- 解压二进制包

```
~]# wget https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/docker-19.03.9.tgz
~]# tar -xf docker-19.03.9.tgz
~]# cp  docker/* /usr/bin/
```

- 创建 systemd 管理docker

```
~]# cat > /usr/lib/systemd/system/docker.service << EOF
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s

[Install]
WantedBy=multi-user.target
EOF
```

- 创建配置文件

```
~]# mkdir /etc/docker
~]# cat > /etc/docker/daemon.json << EOF
{
  "registry-mirrors": ["https://b9pmyelo.mirror.aliyuncs.com"]
}
EOF
```

- 启动并设置开机启动

```
~]# systemctl daemon-reload
~]# systemctl start docker
~]# systemctl enable docker
```
