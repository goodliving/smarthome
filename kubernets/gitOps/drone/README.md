### 概述

这是关于`Drone`持续集成工具的安装与使用教程。

### 安装

`drone`的安装可以参考官方的教程，以下是个人的`docker-compose.yml`文件

```shell
version: '2'

services:
  drone-server:
    image: drone/drone:0.8.5

    ports:
      - 8080:8000
      - 9000
    volumes:
      - /var/lib/drone:/var/lib/drone/
    restart: always
    depends_on:
      - db
    environment:
      - DRONE_OPEN=true
      - DRONE_HOST=http://192.168.100.65:8080
      - DRONE_GOGS=true
      - DRONE_GOGS_URL=http://192.168.100.65:3000
      - DRONE_SECRET=x25WM5TaTXGK5vQ0kgv
      - DRONE_DATABASE_DRIVER=mysql
      - DRONE_DATABASE_DATASOURCE=root:86LzseBTczbp@tcp(db:3306)/drone?parseTime=true

  drone-agent:
    image: drone/agent:0.8.5

    restart: always
    depends_on:
      - drone-server
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - DRONE_SERVER=drone-server:9000
      - DRONE_SECRET=x25WM5TaTXGK5vQ0kgv

  db:
    image: bitnami/mariadb
    ports:
      - 3306
    restart: always
    volumes:
      - /data/drone/mariadb:/bitnami
    environment:
      - MARIADB_ROOT_PASSWORD=86LzseBTczbp
      - MARIADB_DATABASE=drone
```

> **以上需要注意的是bitnami/mariadb镜像有个bug，需要对挂载出来的目录修改`owner`，`chown -R 1001:1001 /to/path`。当然，我们可以更换其他官方镜像，具体参考官方说明**

