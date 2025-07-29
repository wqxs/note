## 安装docker
```
# 安装yum工具
yum install yum-utils -y
 
# 配置yum源
yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
 
# 安装docker
yum install -y docker-ce-25.0.3 docker-ce-cli-25.0.3 containerd.io
 
# 加载镜像
systemctl daemon-reload
 
# 启动docker并且设置开机启动
systemctl enable docker && systemctl start docker
 
# 检查docker版本
docker -v
```

## 离线安装docker
[国外下载地址](https://docs.docker.com/engine/install/)

[国内下载地址](https://mirrors.aliyun.com/docker-ce/linux/static/stable/x86_64/)

```
# 解压包
tar -zxvf docker-25.0.3.tgz
 
# 复制文件到bin目录下
cp docker/* /usr/bin/
 
# 删除旧文件
rm -rf /data/docker
 
# 创建docker.service
vi /usr/lib/systemd/system/docker.service
 
# 录入到docker.service中
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network-online.target firewalld.service
Wants=network-online.target
 
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
 
TimeoutSec=0
 
RestartSec=2
 
ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT
Restart=always
 
TimeoutStartSec=0
 
 
LimitNOFILE=infinity
LimitNPROC=infinity
 
LimitCORE=infinity
 
Delegate=yes
KillMode=process
StartLimitBurst=3
StartLimitInterval=60s
 
[Install]
WantedBy=multi-user.target
 
# 赋权
chmod +x /usr/lib/systemd/system/docker.service
 
# 创建daemon.json
vi /etc/docker/daemon.json
 
# 录入到daemon.json中
{
        # docker安装目录，可自定义修改
        "data-root": "/data/docker",
        "registry-mirrors": [
                "https://hub-mirror.c.163.com/",
                "https://dockerproxy.com/",
                "https://yy28v837.mirror.aliyuncs.com"
        ]
}
 
# 加载docker配置文件
systemctl daemon-reload
 
# 设置自启动
systemctl enable docker.service
 
# 启动docker
systemctl start docker
 
# 查看docker版本
docker -v
```

## 安装docker-compose
[国外下载地址](https://docs.docker.com/desktop/setup/install/linux/)

```
# （在线文件）下载二进制文件
curl -L https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
 
# （在线文件）如果下载太慢用这个地址
curl -L https://get.daocloud.io/docker/compose/releases/download/v2.17.3/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
 
# 离线文件使用上面的
cp -f docker-compose-linux-x86_64 /usr/local/bin/docker-compose
 
# 添加可执行权限
chmod +x /usr/local/bin/docker-compose
 
# 检查docker-compose版本
docker-compose -v
```

## 安装Harbor
[下载地址](https://github.com/goharbor/harbor/releases)
```
# 解压后
tar -zxvf harbor-offline-installer-v2.10.3.tgz
 
# 修改配置文件vi harbor.yml
# 修改hostname、harbor_admin_password、data_volume
 
# 执行install.sh，会生成docker-compose.yml
./install.sh
 
# 如果执行失败，操作docker-compose.yml即可
# 关闭镜像
docker-compose down
# 运行镜像
docker-compose up -d
 
# 查看界面
http://hostname:5000
账号：admin
密码：admin
```

设置docker私服仓库
```
# 编辑docker配置文件
vi /etc/docker/daemon.json
 
# insecure-registries：私服仓库地址
{
    "data-root": "/data/docker",
    "insecure-registries": [
        "http://hostname:5000"
    ],
    "registry-mirrors": [
        "http://hub-mirror.c.163.com",
        "https://dockerproxy.com",
        "https://registry.docker-cn.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://registry.dockermirror.com"
    ]
}
 
# 重启docker
systemctl restart docker
```

开启docker远程连接
```
# 修改docker配置文件
vi /lib/systemd/system/docker.service 
 
# 在 ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock中加上 -H tcp://0.0.0.0:2375
 
# 重新加载配置文件
systemctl daemon-reload
systemctl restart docker
```