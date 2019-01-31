## Kong Docker环境迁移部署

+ 准备

  ```bash
  #将旧版本的compose置为无效
  mv docker-compose.yml docker-compose.yml.bak
  ```

  ```bash
  # 将迁移使用的compose文件置为当前有效文件
  mv docker-compose-migration.yml docker-compose.yml 
  ```

+ 0.14迁移至1.x

  1. 将1.x版本Kong集群的数据库配置为和0.14集群相同的数据库，执行如下命令进行数据迁移：

     ```bash
     docker-compose up --no-deps kong-migration
     ```

     **注意**：其中服务`kong-migration`的启动命令为`kong migrations up`；

  2. 此时0.14和1.0版本的Kong集群都将基于同一个数据库提供服务，但是不可使用1.0版本的Admin API，如果必须要使用，请使用0.14的API；

  3. 部署启动1.0版本的集群：

     ```bash
     docker-compose up -d --no-deps --no-recreate --scale kong-v1=3
     ```

  4. 最后执行如下命令完成数据迁移：

     ```bash
     docker-compose up --no-deps --remove-orphans kong-migration
     ```

     **注意**：其中服务`kong-migration`的启动命令为`kong migrations finish`。

  5. 重启Nginx服务：

     ```bash
     docker-compose up -d --force-recreate --no-deps nginx-lb
     ```

+ 连接Kong集群所在网络，并启动[kong-dashboard](https://github.com/PGBI/kong-dashboard)

  ```bash
  docker run --rm --network kong-gw-cluster_default \
    -p 9080:8080 \
    -d pgbi/kong-dashboard start \
    --kong-url http://nginx-lb:8001 \
    --basic-auth abcde=123456
  ```


### Docker环境下Kong高可用部署

> 使用Docker swarm mode实现Kong的高可用

```bash
# 创建kong数据库
docker service create \
    --name kong-db \
    --network kong-network \
    -e POSTGRES_USER=kong \
    -e POSTGRES_PASSWORD=kong \
    -e POSTGRES_DB=kong \
    --mount type=bind,source=/home/docker/api-data,destination=/var/lib/postgresql/data \
    -p "5432:5432" \
    --constraint 'node.role == manager' \
    --hostname kong-database \
    postgres:9.5

# 初始化或迁移kong数据
docker service create --name kong-migration \
    --network kong-network \
    -e KONG_DATABASE=postgres \
    -e KONG_PG_HOST=kong-db \
    -e KONG_PG_DATABASE=kong \
    -e KONG_PG_PASSWORD=kong \
    -e KONG_PG_USER=kong \
    --restart-max-attempts 3 \
    kong:0.14 kong migrations up

# 创建kong服务
docker service create \
    --name kong \
    --replicas 3 \
    --network kong-network \
    -e KONG_DATABASE=postgres \
    -e KONG_PG_HOST=kong-db \
    -e KONG_PG_DATABASE=kong \
    -e KONG_PG_PASSWORD=kong \
    -e KONG_PG_USER=kong \
    -e KONG_ADMIN_LISTEN=0.0.0.0:8001 \
    -p 8000:8000 \
    -p 8001:8001 \
    -p 8443:8443 \
    -p 8444:8444 \
    kong:0.14

# 创建kong-dashboard
docker service create \
    --name kong-dashboard \
    --network kong-network \
    --constraint 'node.role == manager' \
    -p 9080:8080 \
    pgbi/kong-dashboard start \
    --kong-url http://kong:8001 \
    --basic-auth abcde=123456
```




