---
title: MongoDB-ServerSelectionTimeoutError的问题
tags: mongoDB
categories: 日常踩坑
abbrlink: 65529
date: 2020-11-24 23:00:42
---
如何解决MongoDB-ServerSelectionTimeoutError的问题
<!-- more -->

### 问题现象

拉取mongoDB镜像：

```bash
docker pull mongo
```

按照官方给的命令启动MongoDB的docker实例：

```bash
$ docker run --name some-mongo -d mongo:latest
```

使用python客户端pymongo执行简单的测试：

```python
if __name__ == "__main__":
    sheet = pymongo.MongoClient('127.0.0.1', 27017)['poetry']['poetry']
    sheet.insert_one({'name': 'test'})
```

会出现ServerSelectionTimeoutError: 127.0.0.1:27017: [Errno 61] Connection refused的错误：

> pymongo.errors.ServerSelectionTimeoutError: 127.0.0.1:27017: [Errno 61] Connection refused, Timeout: 30s, Topology Description: <TopologyDescription id: 5fbcd8655e2cdacc38da6dd6, topology_type: Single, servers: [<ServerDescription ('127.0.0.1', 27017) server_type: Unknown, rtt: None, error=AutoReconnect('127.0.0.1:27017: [Errno 61] Connection refused')>]>



### 问题分析

从`Connection refused, Timeout`可以推断出是由于客户端与服务器无法建立连接，通过 `docker ps` 查看

>CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                  NAMES
>ee1d4ac763ff        mongo:latest        "docker-entrypoint.s…"   21 hours ago        Up 7 seconds        27017/tcp     some-mongo

可以看到容器实例端口27017，协议tcp，但没有本地映射端口， 所以无法与容器内mongo服务通信。

### 正确的使用方式

第一种方式：使用docker run命令并增加端口映射：

```bash
docker run -p 127.0.0.1:27017:27017 --name some-mongo -d mongo:latest
```

第一种方式：使用docker-compose方式增加端口映射，官方文档中有示例如下：

```yaml
# Use root/example as user/password credentials
version: '3.1'

services:

  mongo:
    image: mongo
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: example

  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: example
```

在mongo节点下增加:

```yaml
    ports:
       - 27017:27017
```

mongo-express提供了monggoDB的管理功能。

### 总结

如果希望主机与容器实例通信，需要增加端口映射，容器间的通信则不需要。