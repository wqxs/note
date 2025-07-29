参考文档：https://www.cnblogs.com/morang/p/devops-prometheus-grafana-install-use-db-site-monitor.html

- Prometheus：监控中心，监控exporter的实时数据
- exporter：采集数据工具，如下：![image](image.png)
- Grafana：可视化监控

## 搭建Prometheus

```
# 创建目录并赋权
mkdir -p /data/prometheus/data && chown 65534 /data/prometheus/data
# 创建配置目录
mkdir /data/prometheus/conf
# 配置prometheus.yml文件
vi /data/grafana/conf/prometheus.yml
```

配置prometheus.yml
```
# prometheus.yml
global:
  scrape_interval: 15s
  external_labels:
    monitor: 'monitor'
 
scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 5s
    static_configs:
      - targets: 
        - localhost:9090
  - job_name: 'linux服务器'
    # 每隔5秒从该作业中抓取目标
    scrape_interval: 5s
    static_configs:
        # 需要监控的服务器列表
      - targets:
        - 127.0.0.1:9100
```

docker安装Prometheus
```
# docker安装Prometheus
docker run -d \
--restart always \
--name prometheus \
-p 9090:9090 \
-v /data/prometheus/conf/prometheus.yml:/etc/prometheus/prometheus.yml \
-v /data/prometheus/data:/prometheus \
prom/prometheus:v2.47.2 \
--config.file=/etc/prometheus/prometheus.yml \
--storage.tsdb.retention.time=45d
```

Prometheus监控页面：http://127.0.0.1:9090

## 搭建Exporter

exporter下载地址: https://github.com/orgs/prometheus/repositories?type=all&q=exporter

```
node_exporter端口：9100
mysqld_exporter端口：9104
postgres_exporter端口：9187
mongodb_exporter端口：9216
redis_exporter端口：9121
 
# centos开放9100端口
firewall-cmd --zone=public --add-port=9100/tcp --permanent && firewall-cmd --reload
 
# ubuntu开放9100端口
ufw allow 9100
 
# 检查是否成功
http://ip:9100/metrics
```

### Linux数据采集：node_exporter

下载地址：https://github.com/prometheus/node_exporter

安装数据采集工具
```
pkill -9 node_expo
 
# systemd配置自启动
cat > /usr/lib/systemd/system/node_exporter.service <<EOF
[Unit]
Description=node_exporter
After=network.target
 
[Service]
Type=simple
# 如果开启了SELinux，则需要临时关闭它
ExecStartPre=/usr/sbin/setenforce 0
ExecStart=/data/node_exporter/node_exporter
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
EOF
 
systemctl daemon-reload && systemctl start node_exporter && systemctl enable node_exporter
 
# 重新加载配置
systemctl daemon-reload
 
# 启动开机自启
systemctl enable node_exporter
# 关闭开机自启
systemctl disable node_exporter
 
# 启动
systemctl start node_exporter
# 关闭
systemctl stop node_exporter
# 查看状态
systemctl status node_exporter
# 重启
systemctl restart node_exporter
 
# 查看日志
journalctl -u node_exporter -f -n 100
 
# 如果上述失败，需要临时关闭SElinux
setenforce 0
 # 开启
setenforce 1
 
# 解压、重命名、后台启动
tar -zxvf /data/node_exporter-1.7.0.linux-amd64.tar.gz -C /data  && mv /data/node_exporter-1.7.0.linux-amd64 /data/node_exporter && 
nohup /data/node_exporter/node_exporter >/data/node_exporter/node_exporter.log 2>&1 &
```

### Mysql数据采集：mysqld_exporter

下载地址：https://github.com/prometheus/mysqld_exporter

添加Mysql监控账号，并设置权限
```
# 进入mysql命令行
mysql -uxxx -pxxx
# 创建监控账号
create user 'prometheus'@'%' identified by 'prometheus';
# 设置权限
grant all privileges on *.* to 'prometheus'@'%' with grant option;
# 刷新
flush privileges;
```

安装数据采集工具
```
# 解压、重命名
tar -zxvf /data/mysqld_exporter-0.15.1.linux-amd64.tar.gz -C /data  && mv /data/mysqld_exporter-0.15.1.linux-amd64 /data/mysqld_exporter
 
# 创建配置文件
# vi /data/mysqld_exporter/.my.cnf
[client]
host=xxx
port=xxx
user=prometheus
# 密码中不能带有 # 符号
password=prometheus
 
# systemd配置自启动
cat > /usr/lib/systemd/system/mysqld_exporter.service <<EOF
[Unit]
Description=mysqld_exporter
After=network.target
 
[Service]
Type=simple
ExecStart=/data/mysqld_exporter/mysqld_exporter --config.my-cnf=/data/mysqld_exporter/.my.cnf
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
EOF
 
# 重新加载配置
systemctl daemon-reload
 
# 启动开机自启
systemctl enable mysqld_exporter
# 关闭开机自启
systemctl disable mysqld_exporter
 
# 启动
systemctl start mysqld_exporter
# 关闭
systemctl stop mysqld_exporter
# 查看状态
systemctl status mysqld_exporter
# 重启
systemctl restart mysqld_exporter
 
# 查看日志
journalctl -u mysqld_exporter -f -n 100
 
# 后台启动
nohup /data/mysqld_exporter/mysqld_exporter --config.my-cnf=/data/mysqld_exporter/.my.cnf > /data/mysqld_exporter/mysqld_exporter.log 2>&1 &
```

### PostgreSQL数据采集：postgres_exporter

postgres版本 < 10，需要执行以下命令
```
-- To use IF statements, hence to be able to check if the user exists before
-- attempting creation, we need to switch to procedural SQL (PL/pgSQL)
-- instead of standard SQL.
-- More: https://www.postgresql.org/docs/9.3/plpgsql-overview.html
-- To preserve compatibility with <9.0, DO blocks are not used; instead,
-- a function is created and dropped.
CREATE OR REPLACE FUNCTION __tmp_create_user() returns void as $$
BEGIN
  IF NOT EXISTS (
          SELECT                       -- SELECT list can stay empty for this
          FROM   pg_catalog.pg_user
          WHERE  usename = 'prometheus') THEN
    CREATE USER prometheus;
  END IF;
END;
$$ language plpgsql;
 
SELECT __tmp_create_user();
DROP FUNCTION __tmp_create_user();
 
ALTER USER prometheus WITH PASSWORD 'prometheus';
ALTER USER prometheus SET SEARCH_PATH TO prometheus,pg_catalog;
 
-- If deploying as non-superuser (for example in AWS RDS), uncomment the GRANT
-- line below and replace <MASTER_USER> with your root user.
-- GRANT postgres_exporter TO <MASTER_USER>;
 
GRANT CONNECT ON DATABASE postgres TO prometheus;
```

postgres版本 >= 10，需要执行以下命令
```
# 创建监控账号
CREATE USER prometheus WITH PASSWORD 'prometheus';
# 赋权
GRANT pg_monitor to prometheus;
# 创建函数
CREATE SCHEMA IF NOT EXISTS postgres_exporter;
GRANT USAGE ON SCHEMA postgres_exporter TO prometheus;
 
CREATE OR REPLACE FUNCTION get_pg_stat_activity() RETURNS SETOF pg_stat_activity AS
$$ SELECT * FROM pg_catalog.pg_stat_activity; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;
 
CREATE OR REPLACE VIEW postgres_exporter.pg_stat_activity
AS
  SELECT * from get_pg_stat_activity();
 
GRANT SELECT ON postgres_exporter.pg_stat_activity TO prometheus;
 
CREATE OR REPLACE FUNCTION get_pg_stat_replication() RETURNS SETOF pg_stat_replication AS
$$ SELECT * FROM pg_catalog.pg_stat_replication; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;
 
CREATE OR REPLACE VIEW postgres_exporter.pg_stat_replication
AS
  SELECT * FROM get_pg_stat_replication();
 
GRANT SELECT ON postgres_exporter.pg_stat_replication TO prometheus;
 
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE OR REPLACE FUNCTION get_pg_stat_statements() RETURNS SETOF pg_stat_statements AS
$$ SELECT * FROM public.pg_stat_statements; $$
LANGUAGE sql
VOLATILE
SECURITY DEFINER;
 
CREATE OR REPLACE VIEW postgres_exporter.pg_stat_statements
AS
  SELECT * FROM get_pg_stat_statements();
 
GRANT SELECT ON postgres_exporter.pg_stat_statements TO prometheus;
 
docker run --name postgres -e POSTGRES_PASSWORD=123456 -p 5431:5432 -d postgres:12.3
```

安装数据采集数据
```
# 解压、重命名
tar -zxvf /data/postgres_exporter-0.15.0.linux-amd64.tar.gz -C /data  && mv /data/postgres_exporter-0.15.0.linux-amd64 /data/postgres_exporter
 
# 创建配置文件/data/postgres_exporter/postgres.conf
DATA_SOURCE_URI=10.31.83.113:5431/postgres?sslmode=disable
DATA_SOURCE_USER=prometheus
DATA_SOURCE_PASS=prometheus
 
# systemd配置自启动
cat > /usr/lib/systemd/system/postgres_exporter.service <<EOF
[Unit]
Description=postgres_exporter
After=network.target
 
[Service]
Type=simple
EnvironmentFile=/data/postgres_exporter/postgres.conf
ExecStart=/data/postgres_exporter/postgres_exporter
Restart=on-failure
 
[Install]
WantedBy=multi-user.target
EOF
 
# 重新加载配置
systemctl daemon-reload
 
# 启动开机自启
systemctl enable postgres_exporter
# 关闭开机自启
systemctl disable postgres_exporter
 
# 启动
systemctl start postgres_exporter
# 关闭
systemctl stop postgres_exporter
# 查看状态
systemctl status postgres_exporter
# 重启
systemctl restart postgres_exporter
 
# 查看日志
journalctl -u postgres_exporter -f -n 100
 
# 设置环境变量
export DATA_SOURCE_NAME=postgresql://prometheus:prometheus@10.31.83.113:5431/postgres?sslmode=disable
# 后台启动
nohup /data/postgres_exporter/postgres_exporter  > /data/postgres_exporter/postgres_exporter.log 2>&1 &
```

### MongoDB数据采集：mongodb_exporter

```
# 本地安装
nohup /data/mongodb_exporter/mongodb_exporter --mongodb.uri=mongodb://admin:123456@127.0.0.1:27017 > /data/mongodb_exporter/mongodb_exporter.log 2>&1 &
 
# docker启动
docker run -d -p 9216:9216 \
--name mongodb_exporter  \
--restart always \
percona/mongodb_exporter:0.40 \
--mongodb.uri=mongodb://admin:123456@127.0.0.1:27017/admin
 
 
# centos开放9216端口
firewall-cmd --zone=public --add-port=9216/tcp --permanent && firewall-cmd --reload
 
# 赋权
use admin
db.createUser({ 
    user: "prometheus",
    pwd: "prometheus",
    roles: [
      {
         "role":"clusterMonitor",
         "db":"admin"
      },
      {
         "role":"read",
         "db":"local"
      }
    ]
});
```

### Redis数据采集：redis_exporter

```
# 解压并重命名
tar -zxvf /data/redis_exporter-v1.58.0.linux-amd64.tar.gz -C /data  && mv /data/redis_exporter-v1.58.0.linux-amd64 /data/redis_exporter
 
 
直接再页面上配置
# 后台启动
nohup /data/redis_exporter/redis_exporter --redis.password gusuredis_!@# > /data/redis_exporter/redis_exporter.log 2>&1 &
```

## 搭建Grafana

```
# 创建数据保存目录
mkdir -p /data/grafana/data && chown -R 472:472 /data/grafana/data
# 创建配置文件目录
mkdir -p /data/grafana/conf
# 配置defaults.ini文件
vi /data/grafana/conf/defaults.ini
```
配置defaults.ini文件：[defaults.ini](defaults.ini)

docker安装grafana
```
# docker安装grafana
docker run -d \
--restart always \
--name grafana \
--user=root \
-p 3000:80 \
-v /data/grafana/data:/var/lib/grafana \
-v /data/grafana/conf/defaults.ini:/usr/share/grafana/conf/defaults.ini \
-e GF_SECURITY_ADMIN_USER=admin \
-e GF_SECURITY_ADMIN_PASSWORD=123456789 \
-e GF_USERS_ALLOW_SIGN_UP=false \
grafana/grafana:10.2.0
 
#挂载目录并设置匿名登录（GF_AUTH_ANONYMOUS_ORG_ROLE值 Viewer：只读模式，Editor：可编辑模式，Admin：管理员模式）
# -e GF_AUTH_ANONYMOUS_ORG_ROLE=Admin \
# -e GF_AUTH_DISABLE_LOGIN_FROM=true=false \
#docker run -d --name grafana -p 3000:3000 -v /opt/fdop/grafana/conf:/etc/grafana -v /opt/fdop/grafana/data:/var/lib/grafana -v /opt/fdop/grafana/logs:/var/log/grafana  -e "GF_AUTH_ANONYMOUS_ENABLED=true" -e "GF_AUTH_ANONYMOUS_ORG_ROLE=Admin" -e "GF_AUTH_DISABLE_LOGIN_FROM=true" -e "GF_AUTH_PROXY_ENABLED=true" -e "GF_SECURITY_ALLOW_EMBEDDING=true" grafana/grafana:7.3.6
```
Grafana监控页面：http://ip:3000

账号：admin

密码：123456789

## 配置监控页面
1. 设置数据源Prometheus
2. 导入模板
    1. Linux服务器模板ID：
        - 16098
        - 10180
    2. Mysql数据库模板ID：
        - 7362，模板json：mysql-overview.json
        - 14057
    3. PostgreSQL数据库模板ID：
        - 9628/13115
        - 455
        - 6742
