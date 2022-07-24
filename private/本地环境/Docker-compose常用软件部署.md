---
title: Docker-compose常用软件部署
tags: []
categories:
  - docker
top_img: >-
  https://raw.githubusercontents.com/wangyyovo/CDN/master/cover/docker/4d51cecf7d26dfa720682ee0425872d5.jpg
cover: >-
  https://raw.githubusercontents.com/wangyyovo/CDN/master/cover/docker/4d51cecf7d26dfa720682ee0425872d5.jpg
copyright: true
date: 2022-07-17 18:00:39
keywords:
description:
comments:
toc:
toc_number:
---

# Docker-compose常用软件部署

## docker-compose模板
### 多个服务

文件存放的位置

```shell
# Compose 版本 Version 2支持更多的指令。Version 1将来会被弃用。
version: "3"

# 定义服务
services:
  # 为project定义服务
  redis:
    # 服务的镜像名称或镜像ID。如果镜像在本地不存在，Compose将会尝试拉取镜像
    image: redis:4.0
    # 配置端口 - "宿主机端口:容器暴露端口"
    ports:
      - 6379:6379
    # 指定一个自定义容器名称，而不是生成的默认名称。
    container_name: redis
    # 容器总是重新启动
    restart: always
    # 使用该参数，container内的root拥有真正的root权限。
    privileged: true
    # 相当于执行一些命令
    command:
      redis-server /etc/redis/redis.conf --appendonly yes
    # 配置容器连接的网络，引用顶级 networks 下的条目(就是最下面配置的networks(一级目录))
    networks:
      network_name:
       # 为单redis创建别名,REDIS_URL标记为redis服务的地址. (不配置aliases也可以, 这样就通过定义的服务名: redis链接)
       aliases:
         - REDIS_URL
    # 挂载
    volumes:
      - "/docker/redis/conf/redis.conf:/etc/redis/redis.conf"
      - "/docker/redis/data:/data"
  db:
    image: mysql:5.7
    ports:
   	  - 3306:3306
    restart: always
    command: --init-file /docker-entrypoint-initdb.d/init.sql
    container_name: mysql
    privileged: true
    # 添加环境变量
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
      volumes:
        - "/docker/mysql/conf/my.cnf:/etc/mysql/conf.d/my.cnf"
        - "/docker/mysql/logs:/var/log/mysql"
        - "/docker/mysql/data:/var/lib/mysql"
        - "/docker/mysql/sql/init.sql:/docker-entrypoint-initdb.d/init.sql"
        - "/etc/localtime:/etc/localtime"
    networks:
      network_name:
      aliases:
        - MYSQL_URL
    project-name:
      # 服务的镜像名称或镜像ID。如果镜像在本地不存在，Compose将会尝试拉取镜像
      image: project-name:1.0.0
      # 构建镜像
      build:
        # 指定项目的地址
        context: /root/docker_mysql_redis
        # 指定Dockerfile
        dockerfile: Dockerfile
      privileged: true
      restart: always
      container_name: test-name
      ports:
		- 8080:8080
		# 从文件添加环境变量
	  env_file:
		- /root/environment.env
      networks:
        network_name:
          aliases:
            - PROJECT_URL
            
# ........可以继续添加

networks:
  # bridge：默认，需要单独配置ports映射主机port和服务的port，并且开启了容器间通信
  network_name:
    driver: bridge

```

### 单个服务
```shell
version: "3"

services:
  nacos:
    image: nacos/nacos-server:1.2.1
    ports:
      - 8848:8848
    # 加入已存在的网络docker_network_mysql(docker-mysql.yaml), 并创建一个新的网络network_nacos
    networks:
      network_nacos:
        aliases:
          - NACOS_URL
      # 通过docker network ls 进行获取
      docker_network_mysql:
    restart: always
    environment:
      MODE: standalone
    container_name: nacos
    privileged: true

networks:
  docker_network_mysql:
    external: true
  network_nacos:
    driver: bridge
```


## network以路径为前缀

```shell
# -p Specify an alternate project name (default: directory name) 比如配置的network, 会以路径为前缀, 配置该值, 可以代替
# -f Specify an alternate compose file (default: docker-compose.yml) 指定yaml文件
# -d 后台运行
docker-compose -p docker -f /root/docker-init.yaml up -d
```

## 加入已存在的网络
加入一个存在的网络, 同时创建一个新的网络

```shell
#加入已存在的网络 docker_network_mysql(docker-mysql.yaml), 并创建一个新的网络 network_nacos
```

```shell
services:
  nacos:
    # 加入已存在的网络 docker_network_mysql(docker-mysql.yaml), 并创建一个新的网络 network_nacos
  networks:
    network_nacos:
      aliases:
        - NACOS_URL
      # 通过docker network ls 进行获取
      docker_network_mysql:
networks:
  docker_network_mysql:
    external: true
  network_nacos:
    driver: bridge
```

将已经运行的容器, 新增到一个新的网络中
```shell
创建一个网络docker_mysql: docker network create docker_mysql
将已运行的容器mysql加入该网络: docker network connect --alias MYSQL_URL docker_mysql mysql
--alias MYSQL_URL可以通过别名通信. 也可以不要, 但是这样就只能通过容器的ID或者通信
docker network connect 一个已存在网络 容器名/容器ID
```


## 将当前镜像备份
```shell
docker tag docker.io/chaim2436/sentinel-dashboard:1.7.1 chaim2436/sentinel-dashboard:1.7.1
docker rmi docker.io/chaim2436/sentinel-dashboard:1.7.1后chaim2436/sentinel-dashboard:1.7.1会依旧存在z这里不能rmi id, 因为tag之后两个镜像的ID将会是一样的
```


## 启动docker-compose.yaml中的某一个服务
```shell
docker-compose -f /root/docker-init.yaml up -d db
```


## 查看network
docker network inspect docker_mysql, 查看网络里面有哪些容器
这些容器之间都是可以相互通信的, 服务名或者别名都是可行的



## 常用软件docker-compose

### mysql

#### 8.0

> https://hub.docker.com/_/mysql

```yaml
version: "3"		# 切记版本号不要高于4.0，这里指的是compose的版本号

# docker network create mysql_bridge
networks:
  mysql_bridge:
    driver: bridge

services:
  mysql8:
    image: mysql:8.0.26
    container_name: mysql8  # container name
    ports:
      - "3306:3306"
    volumes:
      - /root/docker/mysql8/my.cnf:/etc/mysql/my.cnf
      - /root/docker/mysql8/data:/var/lib/mysql        # 数据存放在/root/docker/mysql8/data
      - /root/docker/mysql8/conf.d:/etc/mysql/conf.d   # custom config
      - /root/docker/mysql8/log:/var/log/mysql         # log
    environment:
      - MYSQL_USER=wyy # 添加新用户
      - MYSQL_PASSWORD=123456
      - MYSQL_ROOT_PASSWORD=123456             # init pwd
      - MYSQL_DATABASE=mydb
      - TZ=Asia/Shanghai
    command:
      #  这个参数尽量加上，防止因为使用的mysql版本过高而导致密码加密的方式改变，进而会造成在认证时不通过
      #  表名统一不区分大小写
      --default-authentication-plugin=mysql_native_password
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1		
    restart: always		# docker重启时，容器自动重启
    privileged: true	# 为此容器开启root权限，防止出现一些权限问题
    networks:
      - mysql_bridge

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    ports:
      - "80:80"
    environment:
      - MAX_EXECUTION_TIME=600 
      - UPLOAD_LIMIT=800M
      - PMA_HOST=mysql8
      - PMA_PORT=3306
      - PMA_ARBITRARY=1
      - MYSQL_USER="wyy"
      - MYSQL_PASSWORD="123456"
      - MYSQL_ROOT_PASSWORD="123456"
    depends_on:
      - mysql8
    networks:
      - mysql_bridge
```

cnf添加如下内容：

```ini
[mysqld]
user=mysql
default-storage-engine=INNODB
#character-set-server=utf8
character-set-server=utf8mb4
default_authentication_plugin=mysql_native_password
secure_file_priv=/var/lib/mysql
expire_logs_days=7
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
max_connections=1024

[client]
#utf8mb4字符集可以存储emoji表情字符
#default-character-set=utf8
default-character-set=utf8mb4

[mysql]
#default-character-set=utf8
default-character-set=utf8mb4
```



#### 5.7

> https://hub.docker.com/_/mysql

```shell
version: "3.5"

# docker network create mysql_bridge
networks:
  mysql_bridge:
    driver: bridge

services:
  mysql5:
    image: mysql:5.7.38
    container_name: mysql5
    restart: always
    environment:
      # 指定root密码。不指定则会启动失败
      - MYSQL_ROOT_PASSWORD=123456
      - MYSQL_DATABASE=mydb
      - MYSQL_USER=wyy # 添加新用户
      - MYSQL_PASSWORD=123456
      - TZ=Asia/Shanghai
    volumes:
      # 初始化执行的sql
      # - /root/docker/mysql5/init:/docker-entrypoint-initdb.d/
      # db配置
      - /root/docker/mysql5/my.cnf:/etc/my.cnf
      # db文件存放地址
      - /root/docker/mysql5/data:/var/lib/mysql
      - /root/docker/mysql5/conf.d:/etc/mysql/conf.d   # custom config
      - /root/docker/mysql5/log:/var/log/mysql         # log
    command:
      # --lower_case_table_names 表名统一不区分大小写
      --character-set-server=utf8mb4
      --collation-server=utf8mb4_general_ci
      --explicit_defaults_for_timestamp=true
      --lower_case_table_names=1
    ports:
      - 3306:3306
    networks:
      - mysql_bridge

  phpmyadmin:
    image: phpmyadmin/phpmyadmin:latest
    ports:
      - "80:80"
    environment:
      - MAX_EXECUTION_TIME=600 
      - UPLOAD_LIMIT=800M
      - PMA_HOST=mysql5
      - PMA_PORT=3306
      - PMA_ARBITRARY=1
      - MYSQL_USER="wyy"
      - MYSQL_PASSWORD="123456"
      - MYSQL_ROOT_PASSWORD="123456"
    depends_on:
      - mysql5
    networks:
      - mysql_bridge
```

cnf添加如下内容：

```ini
[mysqld]
user=mysql
default-storage-engine=INNODB
#character-set-server=utf8
character-set-client-handshake=FALSE
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
init_connect='SET NAMES utf8mb4'
[client]
#utf8mb4字符集可以存储emoji表情字符
#default-character-set=utf8
default-character-set=utf8mb4
[mysql]
#default-character-set=utf8
default-character-set=utf8mb4
```



备份

```shell
docker exec some-mysql sh -c 'exec mysqldump --all-databases -uroot -p"$MYSQL_ROOT_PASSWORD"' > /some/path/on/your/host/all-databases.sql
```



恢复

```shell
docker exec -i some-mysql sh -c 'exec mysql -uroot -p"$MYSQL_ROOT_PASSWORD"' < /some/path/on/your/host/all-databases.sql
```



### redis

> https://hub.docker.com/_/redis

```yaml
version: '3'

# docker network create redis_bridge
networks:
  redis_bridge:
    driver: bridge
    
services:
  redis7:
    image: redis:7.0.3
    container_name: redis7
    restart: always
    command: redis-server /usr/local/etc/redis/redis-6379.conf
    environment:
      - TZ=Asia/Shanghai
    volumes:
      # http://download.redis.io/redis-stable/redis.conf
      - /root/docker/redis/redis.conf:/usr/local/etc/redis/redis.conf
      - /root/docker/redis/redis-6379.conf:/usr/local/etc/redis/redis-6379.conf
      - /root/docker/redis/data:/var/lib/redis
    ports:
      - 6379:6379
    deploy:
      resources:
        limits:
          cpus: '0.50'
          memory: 500M
    networks:
      - redis_bridge

  phpredisadmin:
    image: erikdubbelboer/phpredisadmin
    container_name: phpredisadmin
    restart: always
    ports:
      - 80:80
    environment:
      - TZ=Asia/Shanghai
      - REDIS_1_HOST=redis7
      # - REDIS_1_NAME=redis-localhost
      - REDIS_1_PORT=6379
      - REDIS_1_AUTH=123456
      - ADMIN_USER=root # phpredisadmin登录账号
      - ADMIN_PASS=123456
    depends_on:
      - redis7
    networks:
      - redis_bridge
```



redis-6379.conf

```shell
# 引入原配置文件（未修改内容）
# http://download.redis.io/redis-stable/redis.conf
# 下载https://download.redis.io/releases/redis-6.0.16.tar.gz获取redis.conf
include /usr/local/etc/redis/redis.conf
# 以下是要修改的内容
# 任意ip可访问
bind 0.0.0.0
# 自定义启动端口
port 6379
# 自定义密码
requirepass 123456
# rdb或aof文件存储位置
dir /var/lib/redis
```



### rabbitmq

#### 单节点

```yaml
version: '3'

# docker network create rabbitmq_bridge
networks:
  rabbitmq_bridge:
    driver: bridge

services:
  rabbit:
    image: rabbitmq:3.8-management
    hostname: rabbitmq
    container_name: rabbitmq
    restart: always
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - TZ=Asia/Shanghai
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin
      - RABBITMQ_ERLANG_COOKIE=rabbitcookie
    volumes:
      - /root/docker/rabbitmq/data:/var/lib/rabbitmq
    logging:
      driver: "json-file"
      options:
        max-size: "200k"
        max-file: "10"
    networks:
      - rabbitmq_bridge
```



#### 集群

```yaml
version: '3'
services:
    rabbitmq01:
        image: rabbitmq:3.8-management
        container_name: rabbitmq01
        ports:
          - "15673:15672"
          - "5673:5672"
        hostname: rabbitmq01
        environment:
          - RABBITMQ_ERLANG_COOKIE=rabbitcookie
        volumes:
          - /data/rabbitmq/rabbitmq01:/var/lib/rabbitmq
          - /etc/localtime:/etc/localtime

    rabbitmq02:
        image: rabbitmq:3.8-management
        container_name: rabbitmq02
        ports:
          - "15674:15672"
          - "5674:5672"
        hostname: rabbitmq02
        environment:
          - RABBITMQ_ERLANG_COOKIE=rabbitcookie
        volumes:
          - /data/rabbitmq/rabbitmq02:/var/lib/rabbitmq
          - /etc/localtime:/etc/localtime

    rabbitmq03:
        image: rabbitmq:3.8-management
        container_name: rabbitmq03
        ports:
          - "15675:15672"
          - "5675:5672"
        hostname: rabbitmq03
        environment:
          - RABBITMQ_ERLANG_COOKIE=rabbitcookie
        volumes:
          - /data/rabbitmq/rabbitmq03:/var/lib/rabbitmq
          - /etc/localtime:/etc/localtime
```









docker pull microsoft/mssql-server-linux
docker run --name mssql -e ACCEPT_EULA=Y -e SA_PASSWORD=Aa123456 --privileged=true -v D:\database\data:/var/opt/mssql/data -p 11433:1433 -d microsoft/mssql-server-linux



### kafka

```yaml
version: '3'
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
  kafka:
    image: wurstmeister/kafka
    depends_on: [ zookeeper ]
    ports:
      - "9092:9092"
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 192.168.243.134
      KAFKA_CREATE_TOPICS: "test"
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
    #volumes:
    #  - /var/run/docker.sock:/var/run/docker.sock
```



### nginx

```yaml
version: "3.0"
services:
  nginx:
    restart: always
    image: nginx:stable-alpine-perl
    container_name: nginx
    ports:
      - 81:80
    networks:
      default:
        ipv4_address: 181.18.0.3
    volumes:
      - /home/docker/nginx/nginx.conf:/etc/nginx/nginx.conf
      - /home/docker/nginx/html:/usr/share/nginx/html
      - /home/docker/nginx/logs:/var/log/nginx
networks:
  default:
    external:
      name: zsnet
```





https://blog.csdn.net/qq_38637558/article/details/121391324

https://blog.csdn.net/cold___play/article/details/125306132
