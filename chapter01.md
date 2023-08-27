<h1>环境配置</h1>

```
1、master 2c4g
2、3node 2c4g
3、1register ***
```



<h1>主机名规划</h1>
```
10.211.55.3   kubernetes-master.lg.com  kubernetes-master
10.211.55.24  kubernetes-node1.lg.com  kubernetes-node1
10.211.55.25  kubernetes-node2.lg.com  kubernetes-node2
10.211.55.26  kubernetes-node3.lg.com  kubernetes-node3
10.211.55.27  kubernetes-register.lg.com  kubernetes-register
```

修改主机名
> ssh root@10.211.55.3 "hostnamectl set-hostname kubernetes-master"
> ssh root@10.211.55.24 "hostnamectl set-hostname kubernetes-node1"
> ssh root@10.211.55.25 "hostnamectl set-hostname kubernetes-node2"
> ssh root@10.211.55.26 "hostnamectl set-hostname kubernetes-node3"
> ssh root@10.211.55.27 "hostnamectl set-hostname kubernetes-register"

主机名检测

```
for i in 3 24 25 26 27;do ssh root@10.211.55.$i "hostname";done
```



<h1>环境前置条件</h1>
```
跨主机免密码认证
ssh-keygen -t rsa

跨主机免密码认证
ssh-copy-id root@远程主机ip
```



<h1>swap环境配置(所有主机都需要操作)</h1>
```
swapoff -a 
永久禁用
sed -i "s/.*swap.*/#&/" /etc/fstab
内核参数调整
echo "vm.swappiness = 0">> /etc/sysctl.conf

重启生效配置
sysctl -p
```




<h1>网络参数调整（所有主机操作）</h1>
配置iptables参数，使得流经网桥的流量也经过iptables/netfilter防火墙

```
cat >> /etc/sysctl.d/k8s.conf << EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
```

配置生效

```
modprobe br_netfilter
modprobe overplay
sysctl -p /etc/sysctl.d/k8s.conf
```



<h1>cri环境操作</h1>
```
mkdir -p /data/softs && cd /data/softs
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd-0.3.4.amd64.tgz
tar xf cri-dockerd-0.3.4.amd64.tgz
mv cri-dockerd/cri-dockerd /usr/local/bin
#检查效果
c
```

定制配置文件1:

```
cat > /etc/systemd/system/cri-dockerd.service<<-EOF
[Unit]
Description=CRI interface for Docker Application Container Engine
Documentation=https://docs.mirantis.com
After=network-online.target firewalld.service docker.service
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/local/bin/cri-dockerd --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com/google_containers/pause:3.9 --network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin --container-runtime-endpoint=unix:///var/run/cri-dockerd.sock 
--cri-dockerd-root-directory=/var/lib/dockershim --docker-endpoint=unix:///var/run/docker.sock --cri-dockerd-root-directory=/var/lib/docker
ExecReload=/bin/kill -s HUP $MAINPID
TimeoutSec=0
RestartSec=2
Restart=always
StartLimitBurst=3
StartLimitInterval=60s
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
TasksMax=infinity
Delegate=yes
KillMode=process
[Install]
WantedBy=multi-user.target
EOF

定制配置文件2：
cat > /etc/systemd/system/cri-dockerd.socket<<-EOF
[Unit]
Description=CRI Docker Socket for the API
PartOf=cri-docker.service

[Socket]
ListenStream=/var/run/cri-dockerd.sock
SocketMode=0660
SocketUser=root
SocketGroup=docker

[Install]
WantedBy=sockets.target
EOF
```



<h1>设置开机自启动</h1>
```
systemctl daemon-reload
systemctl enable cri-dockerd.service
systemctl restart cri-dockerd.service
```

```
批量执行： for i in 24 25 26; do ssh root@10.211.55.$i "systemctl daemon-reload ;systemctl start cri-dockerd"; done
```




<h1>部署harbor服务</h1>
安装docker-compose

```
yum install -y docker-compose
```

获取软件

```
下载软件
mkdir /data/{softs,server} -p && cd /data/softs
wget https://github.com/goharbor/harbor/releases/download/v2.8.4/harbor-offline-installer-v2.8.4.tgz

解压软件
tar -zxvf harbor-offline-installer-v2.8.4.tgz -C /data/server/
cd /data/server/harbor/

加载镜像
docker load < harbor.v2.8.4.tar.gz
docker images

备份配置
cp harbor.yml.tmpl harbor.yml
```

修改配置

```
[root@kubernetes-register harbor]# vim harbor.yml
...
hostname: kubernetes-register

...
http:
  # port for http, default is 80. If https enabled, this port will redirect to https port
  port: 80
  
...
https:
  # https port for harbor, default is 443
  #port: 443
  # The path of cert and key files for nginx
  #certificate: /your/certificate/path
  #private_key: /your/private/key/path

...
harbor_admin_password: 123456

...
data_volume: /data/server/harbor/data
```

生成配置文件

```
[root@kubernetes-register harbor]# pwd
/data/server/harbor
[root@kubernetes-register harbor]# ls
common     data                harbor.v2.8.4.tar.gz  harbor.yml.tmpl  LICENSEcommon.sh  docker-compose.yml  harbor.yml            install.sh       prepare
[root@kubernetes-register harbor]# ./prepare 
```

运行harbor

```
[root@kubernetes-register harbor]# ./install.sh 
```

检查效果，或浏览器直接访问ip即可查看效果：

```
[root@kubernetes-register harbor]# docker-compose ps      
Name                 Command                 State                  Ports        ---------------------------------------------------------------------------------------harbor-core         /harbor/entrypoint.sh   Up (health:                                                                            starting)                                  harbor-db           /docker-entrypoint.sh   Up (health:                                                    13                      starting)                                  harbor-jobservice   /harbor/entrypoint.sh   Up (health:                                                                            starting)                                  harbor-log          /bin/sh -c              Up (health:            127.0.0.1:1514->1051                    /usr/local/bin/ ...     starting)              4/tcp               harbor-portal       nginx -g daemon off;    Up (health:                                                                            starting)                                  nginx               nginx -g daemon off;    Up (health:            0.0.0.0:80->8080/tcp                                            starting)              ,:::80->8080/tcp    redis               redis-server            Up (health:                                                    /etc/redis.conf         starting)                                  registry            /home/harbor/entrypoi   Up (health:                                                    nt.sh                   starting)                                  registryctl         /home/harbor/start.sh   Up (health:                                                                            starting)
```

ps：定制服务启动文件

```U
定制服务启动文件 /etc/systemd/system/harbor.service
[Unit]
Description=Harbor
After=docker.service systemd-networkd.service systemd-resolved.service
Requires=docker.service
Documentation=http://github.com/vmware/harbor

[Service]
Type=simple
Restart=on-failure
RestartSec=5
#需要注意harbor的安装位置
ExecStart=/usr/bin/docker-compose --file /data/server/harbor/docker-compose.yml up
ExecStop=/usr/bin/docker-compose --file /data/server/harbor/docker-compose.yml down

[Install]
WantedBy=multi-user.target
```

```
加载服务配置文件
systemctl daemon-reload
启动服务
systemctl start harbor
检查状态
systemctl status harbor
设置开机自启动
systemctl enable harbor
```

<h2>harbor仓库定制</h2>

```
浏览器访问域名，用户名：admin，密码：123456
创建lg用户专用的项目仓库，名称为：lg，权限为公开的
```

<h2>harbor仓库测试</h2>

```
登陆仓库：http://10.211.55.27
账号、密码为上述部署harbor服务中设置的。

下载镜像
docker pull busybox

定制镜像标签
docker tag busybox kubernetes-register.lg.com/lg/busybox:v0.1

登陆到自建的私有仓库harbor
docker login kubernetes-register -u admin -p123456

会发生错误，如：Error response from daemon: Get https://kubernetes-register/v2/: dial tcp 10.211.55.27:443: connect: connection refused
需要在docker的配置文件里面加上配置：
[root@kubernetes-master docker]# cat /etc/docker/daemon.json 
{  "registry-mirrors": [],  "insecure-registries": ["kubernetes-register"]}
添加完成之后，重启docker即可

推送镜像
docker push busybox kubernetes-register.lg.com/lg/busybox:v0.1
```

<h1>k8s集群初始化</h1>

<h2>软件源定制 -> 安装软件 -> 镜像获取 -> 主节点初始 -> 工作节点加入集群</h2>

软件部署

```
定制阿里云的关于k8s的软件源
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]k
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF
```

更新软件源

```
yum makecache fast
```

软件部署

```
master环境：yum install kubeadm kubectl kubelet -y

node环境：yum install kubeadm kubectl kubelet -y
```

