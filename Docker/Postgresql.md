## 单机
```
# 创建映射目录
mkdir /data/postgresql/data
 
# 启动容器
docker run -d -p 5432:5432 --restart=always -v /data/postgresql/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=123456 --name postgres postgres:14
 
# 修改配置文件/data/postgresql/data/postgresql.conf
# 设置时区
timezone = 'Asia/Shanghai'
# 连接数
max_connections = 1000
```

## 集群
### 主机
```
# 创建映射目录
mkdir /data/postgresql/data
 
# 启动容器
docker run -d -p 5432:5432 -v /data/postgresql/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=123456 --name postgres_master postgres:14
 
# 修改vi /data/postgresql/data/pg_hba.conf，允许从机复制
host replication all 192.168.0.12/32 trust
host replication all 193.168.0.13/32 trust
 
# 修改vi /data/postgresql/data/postgresql.conf
# 启用归档模式，允许数据库将 WAL（Write-Ahead Logging）日志文件存档。
archive_mode = on
# 连接数
max_connections = 500
# 设置时区
timezone = 'Asia/Shanghai'
 
# 重启容器
docker restart postgres_master 
```

### 从机
```
# 启动容器，注意这里没有进行数据卷挂载，因为后面要删除数据将主节点数据同步过来
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=123456 -v /data/postgresql/data:/var/lib/postgresql/data --name postgres_slave1 postgres:14
 
# 进入容器
docker exec -it postgres_slave1 bash
 
# 删除数据，并将主节点数据同步过来（注意-h后面跟着的是主节点ip，-U后面是主节点创建的用于复制的用户名，需要输入密码时就是用于复制的用户名的密码）
rm -rf /var/lib/postgresql/data/* && pg_basebackup -h 192.168.0.11 -p 5432 -U postgres -Fp -Xs -Pv -R -D /var/lib/postgresql/data
 
# 数据同步后容器会关闭。再启动即可
docker restart postgres_slave1
 
# 修改文件vi /data/postgresql/data/postgresql.conf
# 配置主库ip地址以及端口号，以及用于复制的用户名和密码        
primary_conninfo = 'host=192.168.0.11 port=5432 user=postgres password=123456'   
# 在恢复期间允许查询。这是在流复制过程中，从库在进行 WAL 日志恢复的同时允许查询读取。
# 从机只读
hot_standby = on
# 设置恢复的目标时间线。在这里，设置为 latest 表示从库将一直尝试连接到主库的最新时间线上。
recovery_target_timeline = latest
# 必须大于主节点的连接数。这确保从库可以处理主库发送的所有连接请求。
max_connections = 1000
# 设置时区
timezone = 'Asia/Shanghai'
                         
# 重启从库
docker restart postgres_slave1 
 
# 校验，在主机执行sql
select * from pg_stat_replication;
```

### 从机2
重复从机1的操作

## Pgpool
负载均衡，读写分离
```
docker run -d -p 9999:5432 --name pgpool \
--env PGPOOL_BACKEND_NODES=0:192.168.0.11:5432,1:192.168.0.12:5432,2:192.168.0.13:5432 \
--env PGPOOL_SR_CHECK_USER=postgres \
--env PGPOOL_SR_CHECK_PASSWORD=123456 \
--env PGPOOL_ENABLE_LDAP=no \
--env PGPOOL_POSTGRES_USERNAME=postgres \
--env PGPOOL_POSTGRES_PASSWORD=123456 \
--env PGPOOL_ADMIN_USERNAME=postgres \
--env PGPOOL_ADMIN_PASSWORD=123456 \
--env PGPOOL_USERNAME=postgres \
--env PGPOOL_PASSWORD=123456 \
--env PGPOOL_AUTHENTICATION_METHOD=scram-sha-256 \
pgpool
```

## postgresql角色权限设置
```
-- 创建账号
CREATE ROLE admin WITH LOGIN PASSWORD '123456' NOSUPERUSER NOCREATEDB NOCREATEROLE;
 
-- 给角色/账号访问数据库的权限
GRANT ALL PRIVILEGES ON DATABASE testDb TO admin;
 
-- 设置所有schema的访问权限
DO
$$
    DECLARE
        schema_name TEXT;
    BEGIN
        FOR schema_name IN
            SELECT nspname
            FROM pg_namespace
            WHERE nspname NOT IN ('pg_catalog', 'information_schema')
            LOOP
                EXECUTE format('GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA %I TO admin;', schema_name);
                EXECUTE format('GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA %I TO admin;', schema_name);
                EXECUTE format('GRANT ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA %I TO admin;', schema_name);
            END LOOP;
    END
$$;
 
-- 解绑所有schema的访问权限
DO
$$
    DECLARE
        schema_name TEXT;
    BEGIN
        FOR schema_name IN
            SELECT nspname
            FROM pg_namespace
            WHERE nspname NOT IN ('pg_catalog', 'information_schema')
            LOOP
                EXECUTE format('REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA %I FROM admin;', schema_name);
                EXECUTE format('REVOKE ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA %I FROM admin;', schema_name);
                EXECUTE format('REVOKE ALL PRIVILEGES ON ALL FUNCTIONS IN SCHEMA %I FROM admin;', schema_name);
            END LOOP;
    END
$$;
 
 
-- 设置默认权限，新创建的表都属于这个角色/账号
DO
$$
    DECLARE
        schema_name TEXT;
    BEGIN
        FOR schema_name IN
            SELECT nspname
            FROM pg_namespace
            WHERE nspname NOT IN ('pg_catalog', 'information_schema')
            LOOP
                EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT ALL ON TABLES TO admin;',
                               schema_name);
                EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT ALL ON SEQUENCES TO admin',
                               schema_name);
                EXECUTE format('ALTER DEFAULT PRIVILEGES IN SCHEMA %I GRANT ALL ON FUNCTIONS TO admin',
                               schema_name);
            END LOOP;
    END
$$;
 
-- 查询角色/账号可以连接的数据库
SELECT datname
FROM pg_database
WHERE has_database_privilege('admin', datname, 'CONNECT');
 
-- 解绑连接其他数据库
REVOKE CONNECT ON DATABASE postgres FROM PUBLIC;
 
-- 查询数据库所绑定的角色/用户
SELECT datacl
FROM pg_database
WHERE datname = 'testDb';
 
-- 删除角色/账号
DROP OWNED BY admin;
```