## 单机

```
docker run \
--name mongodb1 \
-d -p 27018:27017 \
--restart=always \
-v /data/mongodb/data:/data/db \
mongo:4.2.18 \
--auth
 
# 进入mongodb 
mongo
 
# 创建账号
db.createUser({user:"root",pwd:"123456",roles:[{role:"root",db:"admin"}]})
```

## 一主一从一仲裁

```
# 创建秘钥
openssl rand -base64 756 > /data/mongodb/conf/mongoKeyfile.key
# 给秘钥创建赋予权限
chmod 600 keyfile.key
# 分发到其他服务器
 
 
# 主
docker run -d --name mongodb \
--restart=always \
-p 27017:27017 \
-v /data/mongodb/data:/data/db \
-v /data/mongodb/conf:/data/configdb \
-v /data/mongodb/conf/mongoKeyfile.key:/data/configdb/mongoKeyfile.key \
mongo:4.2.18 \
--replSet=mongo_rs --bind_ip_all --logappend --keyFile /data/configdb/mongoKeyfile.key 
 
# 从节点
docker run -d --name mongodb \
-p 27017:27017 \
-v /data/mongodb/data:/data/db \
-v /data/mongodb/conf:/data/configdb \
-v /data/mongodb/conf/mongoKeyfile.key:/data/configdb/mongoKeyfile.key \
mongo:4.2.18 \
--replSet=mongo_rs --bind_ip_all --logappend --keyFile /data/configdb/mongoKeyfile.key 
 
# 仲裁节点
docker run -d --name mongodb \
-p 27017:27017 \
-v /data/mongodb/data:/data/db \
-v /data/mongodb/conf:/data/configdb \
-v /data/mongodb/conf/mongoKeyfile.key:/data/configdb/mongoKeyfile.key \
mongo:4.2.18 \
--replSet=mongo_rs --bind_ip_all --logappend --keyFile /data/configdb/mongoKeyfile.key 
 
# 配置副本集
# 进入主节点容器
docker exec -it mongodb mongo
 
# 切换数据库并登录
use admin;
 
# 配置副本集
rs.initiate({
  _id: "mongo_rs",
  members: [
    { _id: 0, host: "192.168.0.11:27017" },  // 主节点
    { _id: 1, host: "192.168.0.12:27017" },  // 从节点
    { _id: 2, host: "192.168.0.13:27017", arbiterOnly: true }  // 仲裁节点
  ]
})
 
# 查看集群状态
rs.status()
 
# 切换至admin
use admin
 
# 添加用户
db.createUser({user:"root",pwd:"123456",roles:[{role:"root",db:"admin"}]})
 
# 登录
db.auth('root','123456')
```