## 单机
参考文档：https://blog.csdn.net/BThinker/article/details/125412751

```
# 账号：miniouser
# 密码：miniopassword

# docker安装minio
docker run -d \
--restart=always \
--name minio \
-v /data/ememory/minio/data:/data \
-h minio \
-p 9000:9000 \
-p 9001:9001 \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server /data --address ":9000" --console-address ":9001"
```

## 联邦集群
### Etcd

```
# 节点1
docker run -d \
-p 2379:2379 \
-p 2380:2380 \
--name etcd1 \
--network=etcd \
--restart=always \
-v /data/etcd/data:/var/lib/etcd \
quay.io/coreos/etcd:v3.5.9 \
etcd \
-name etcd1 \
-advertise-client-urls http://192.168.0.11:2379 \
-initial-advertise-peer-urls http://192.168.0.11:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "etcd1=http://192.168.0.11:2380,etcd2=http://192.168.0.12:2380,etcd3=http://192.168.0.13:2380,etcd4=http://192.168.0.14:2380" \
-initial-cluster-state new
 
# 节点2
docker run -d \
-p 2379:2379 \
-p 2380:2380 \
--name etcd2 \
--network=etcd \
--restart=always \
-v /data/etcd/data:/var/lib/etcd \
quay.io/coreos/etcd:v3.5.9 \
etcd \
-name etcd2 \
-advertise-client-urls http://192.168.0.12:2379 \
-initial-advertise-peer-urls http://192.168.0.12:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "etcd1=http://192.168.0.11:2380,etcd2=http://192.168.0.12:2380,etcd3=http://192.168.0.13:2380,etcd4=http://192.168.0.14:2380" \
-initial-cluster-state new
 
# 节点3
docker run -d \
-p 2379:2379 \
-p 2380:2380 \
--name etcd3 \
--network=etcd \
--restart=always \
-v /data/etcd/data:/var/lib/etcd \
quay.io/coreos/etcd:v3.5.9 \
etcd \
-name etcd3 \
-advertise-client-urls http://192.168.0.13:2379 \
-initial-advertise-peer-urls http://192.168.0.13:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "etcd1=http://192.168.0.11:2380,etcd2=http://192.168.0.12:2380,etcd3=http://192.168.0.13:2380,etcd4=http://192.168.0.14:2380" \
-initial-cluster-state new
 
# 节点4
docker run -d \
-p 2379:2379 \
-p 2380:2380 \
--name etcd4 \
--network=etcd \
--restart=always \
quay.io/coreos/etcd:v3.5.9 \
etcd \
-name etcd4 \
-advertise-client-urls http://192.168.0.14:2379 \
-initial-advertise-peer-urls http://192.168.0.14:2380 \
-listen-client-urls http://0.0.0.0:2379 \
-listen-peer-urls http://0.0.0.0:2380 \
-initial-cluster-token etcd-cluster \
-initial-cluster "etcd1=http://192.168.0.11:2380,etcd2=http://192.168.0.12:2380,etcd3=http://192.168.0.13:2380,etcd4=http://192.168.0.14:2380" \
-initial-cluster-state new
 
# 查看集群列表
docker exec etcd1 etcdctl member list
 
# 查看leader
docker exec etcd1 etcdctl  endpoint status --write-out=table
```

### Minio

```
# 初始化 Swarm manager
# 如果有云服务占用了vxlan端口，所以需要重新设置端口
# docker swarm init --advertise-addr=192.168.0.11 --data-path-addr=192.168.0.11 --data-path-por=6666
docker swarm init --advertise-addr=192.168.0.11
 
# 结果如下
--------------------------------------------------------------------------------------------
Swarm initialized: current node (e6ub8sz10bepdy2gv3fa32unz) is now a manager.
 
To add a worker to this swarm, run the following command:
 
    docker swarm join --token SWMTKN-1-1lk8u55bvlr2xnkiztw78qbw56jzfm0lacphsiwo8gvkalydq2p5-ep378x98as8dy4skp8pv7o1ol 192.168.0.11:2377
 
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
--------------------------------------------------------------------------------------------
 
# 将其他主机加入 Swarm 集群，依次在其他主机执行下面命令
docker swarm join --token SWMTKN-1-1lk8u55bvlr2xnkiztw78qbw56jzfm0lacphsiwo8gvkalydq2p5-ep378x98as8dy4skp8pv7o1ol 192.168.0.11:2377
 
# 创建 Overlay 网络
docker network create --driver overlay --attachable minio-cluster-1
 
# 创建完network之后先看一下网段信息
docker network inspect minio-cluster-1
 
# 确定网段，下面需要自定义ip地址
{
    "Subnet": "10.0.2.0/24",
    "Gateway": "10.0.2.1"
}
 
# 所有swarm节点需要设置端口
# 7946:用于容器网络发现
# 4789:用于容器覆盖网络
# firewall-cmd --zone=public --add-port=7946/tcp --permanent && firewall-cmd --reload
# firewall-cmd --zone=public --add-port=7946/udp --permanent && firewall-cmd --reload
# firewall-cmd --zone=public --add-port=4789/udp --permanent && firewall-cmd --reload
 
# 搭建集群
# 主机192.168.0.11：minio-1
# 主机192.168.0.12：minio-2
# 主机192.168.0.13：minio-3
# 主机192.168.0.14：minio-4
# minio-1
docker run -d \
--restart=always \
--network minio-cluster-1 \
--name minio-1 \
-v /data/minio/cluster1/data:/data \
--ip 10.0.2.11 \
-h minio-1 \
-p 9000:9000 \
-p 9001:9001 \
--add-host="minio-1:10.0.2.11" \
--add-host="minio-2:10.0.2.12" \
--add-host="minio-3:10.0.2.13" \
--add-host="minio-4:10.0.2.14" \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
-e "MINIO_ETCD_ENDPOINTS=http://192.168.0.11:2379,http://192.168.0.12:2379,http://192.168.0.13:2379,http://192.168.0.14:2379" \
-e "MINIO_PUBLIC_IPS=minio-1,minio-2,minio-3,minio-4" \
-e "MINIO_DOMAIN=mydomain.com" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server http://minio-{1...4}/data --address ":9000" --console-address ":9001"
 
# minio-2
docker run -d \
--restart=always \
--network minio-cluster-1 \
--name minio-2 \
-v /data/minio/cluster1/data:/data \
--ip 10.0.2.12 \
-h minio-2 \
-p 9000:9000 \
-p 9001:9001 \
--add-host="minio-1:10.0.2.11" \
--add-host="minio-2:10.0.2.12" \
--add-host="minio-3:10.0.2.13" \
--add-host="minio-4:10.0.2.14" \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
-e "MINIO_ETCD_ENDPOINTS=http://192.168.0.11:2379,http://192.168.0.12:2379,http://192.168.0.13:2379,http://192.168.0.14:2379" \
-e "MINIO_PUBLIC_IPS=minio-1,minio-2,minio-3,minio-4" \
-e "MINIO_DOMAIN=mydomain.com" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server http://minio-{1...4}/data --address ":9000" --console-address ":9001"
 
# minio-3
docker run -d \
--restart=always \
--network minio-cluster-1 \
--name minio-3 \
-v /data/minio/cluster1/data:/data \
--ip 10.0.2.13 \
-h minio-3 \
-p 9000:9000 \
-p 9001:9001 \
--add-host="minio-1:10.0.2.11" \
--add-host="minio-2:10.0.2.12" \
--add-host="minio-3:10.0.2.13" \
--add-host="minio-4:10.0.2.14" \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
-e "MINIO_ETCD_ENDPOINTS=http://192.168.0.11:2379,http://192.168.0.12:2379,http://192.168.0.13:2379,http://192.168.0.14:2379" \
-e "MINIO_PUBLIC_IPS=minio-1,minio-2,minio-3,minio-4" \
-e "MINIO_DOMAIN=mydomain.com" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server http://minio-{1...4}/data --address ":9000" --console-address ":9001"
 
# minio-4
docker run -d \
--restart=always \
--network minio-cluster-1 \
--name minio-4 \
-v /data/minio/cluster1/data:/data \
--ip 10.0.2.14 \
-h minio-4 \
-p 9000:9000 \
-p 9001:9001 \
--add-host="minio-1:10.0.2.11" \
--add-host="minio-2:10.0.2.12" \
--add-host="minio-3:10.0.2.13" \
--add-host="minio-4:10.0.2.14" \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
-e "MINIO_ETCD_ENDPOINTS=http://192.168.0.11:2379,http://192.168.0.12:2379,http://192.168.0.13:2379,http://192.168.0.14:2379" \
-e "MINIO_PUBLIC_IPS=minio-1,minio-2,minio-3,minio-4" \
-e "MINIO_DOMAIN=mydomain.com" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server http://minio-{1...4}/data --address ":9000" --console-address ":9001"
 
 
# 搭建单机minio测试etcd是否可行，在控制台新增bucket看其他minio集群是否可以看到
docker run -d \
--restart=always \
--name minio-test \
-h minio-test \
-p 9100:9000 \
-p 9200:9001 \
-e "MINIO_ROOT_USER=miniouser" \
-e "MINIO_ROOT_PASSWORD=miniopassword" \
-e "MINIO_ETCD_ENDPOINTS=http://192.168.0.11:2379,http://192.168.0.12:2379,http://192.168.0.13:2379,http://192.168.0.14:2379" \
-e "MINIO_PUBLIC_IPS=minio-test" \
-e "MINIO_DOMAIN=mydomain.com" \
minio/minio:RELEASE.2024-04-18T19-09-19Z.fips \
server /data --address ":9000" --console-address ":9001"
```

minio控制台

http://192.168.0.11:9001

http://192.168.0.12:9001

http://192.168.0.13:9001

http://192.168.0.14:9001

账号：miniouser

密码：miniopassword

## 其他

Minio数据迁移
```
# 创建一个临时的minio-client容器
docker run -it --rm --entrypoint /bin/sh bitnami/minio-client
 
# 源minio服务器
mc alias set source http://192.168.0.11:9000 miniouser miniopassword
 
# 目标minio服务器
mc alias set target http://xxx.xxx.x.xx:9000 miniouser miniopassword
 
# copy文件
mc cp --recursive source/bucket1/ target/bucket1/
```