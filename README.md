## Kong Docker环境部署

+ 0.14迁移至1.x

  1. 将1.x版本Kong集群的数据库配置为和0.14集群相同的数据库，执行如下命令进行数据迁移：

     ```bash
     docker-compose up --no-deps kong-migration
     ```

     其中服务`kong-migration`的启动命令为`kong migrations up`；

  2. 此时0.14和1.0版本的Kong集群都将基于同一个数据库提供服务，但是不可使用1.0版本的Admin API，如果必须要使用，请使用0.14的API；

  3. 将指向0.14的流量逐步指向1.0版本，即在`nginx-lb`中修改0.14版本的访问比重，在此之前先启动1.0版本的集群：

     ```bash
     docker-compose up -d --no-deps --scale kong=3
     ```

     使用如下命令获得0.14版本的Kong的IP：

     ```bash
     docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' container_name_or_id
     ```

  4. 监控一段时间后停止0.14的Kong集群；

  5. 最后执行如下命令完成数据迁移：

     ```bash
     docker-compose up --no-deps kong-migration
     ```

     其中服务`kong-migration`的启动命令为`kong migrations finish`。

+ 连接Kong集群所在网络，并启动kong-dashboard

  ```bash
  docker run --rm --network compose-0xx_default \
    -p 9080:8080 \
    -d pgbi/kong-dashboard start \
    --kong-url http://kong:8001 \
    --basic-auth zjesaas=123456
  ```

  


