# Overview

This project implements a custom plugin of Kong to query an auth server to validate a JWT.
这个（原）项目是一个自定义Kong插件的例子，例子中调用了auth server验证JWT。

## 修改点

### Kong的版本升级

升级到0.11.1

升级后不能再使用kong.tools.database_cache，详见[kong upgrade](https://github.com/Kong/kong/blob/master/UPGRADE.md)

### Kong的数据库

改成了postgres

### 增加中文注释

### 提供了一个token供测试（演示）使用


## Start Kong

### Build Kong image

The logs cannot be displayed with [official Docker image](https://hub.docker.com/_/kong/) 
when executing 'docker logs' command. So I wrap a run.sh to 'tail' the error.log. Build the 
image as following:

```
% docker build -t my-kong .
```

### Start containers

Start a Postgres container:

```
$ sudo docker run -d --name kong-database -p 5432:5432 -e "POSTGRES_USER=kong" -e "POSTGRES_DB=kong" postgres:9.4
```

Start a Kong container:

```
$ sudo docker run -d --name kong \
      --link kong-database:kong-database \
      -e "KONG_DATABASE=postgres" \
      -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
      -e "KONG_PG_HOST=kong-database" \
      -e "KONG_LOG_LEVEL=debug" \
      -p 8000:8000 \
      -p 8443:8443 \
      -p 8001:8001 \
      -p 7946:7946 \
      -p 7946:7946/udp \
      my-kong
```

检查启动日志：
cd /var/lib/docker/containers/{容器ID}

tail {容器ID}.log


## Run with plugin

### Start a resource server

After the Kong container started, the custom plugin has been already loaded. Run a resource
server that should be accessed with a token e.g. the main.go in this project:

```
$ go run main.go 
2017/04/28 15:42:24 listening 19000
```

### Register API and Enable plugin

Register the resource API:

这里注册URL的几个参数意思是，这个API注册的名字叫demo。它被挂载在网关的/demo路径下，上游转发到http://<your server ip>:19000去处理

```
$ curl -i -XPOST localhost:8001/apis/ \
    --data 'name=demo' --data 'upstream_url=http://<your server ip>:19000' --data 'uris=/demo'
```

Then the API could be queried with /demo path prefix:

```
$ curl localhost:8000/demo/foo
hello world
```

Enable the custom authorization plugin for this API with the auth server url specified by 
'auth_server_url' configuration:

KONG的插件独立作用于每一个API，不同的API可以使用完全不同的插件。提供了相当灵活的配置策略。

这个URL的意思是，为名字叫demo的API，注册插件whispir-token-auth（这个项目自定义的），并且配置config.auth_server_url=<URL of verification API>

```
$ curl -i -XPOST localhost:8001/apis/demo/plugins \
    --data 'name=whispir-token-auth' \
    --data 'config.auth_server_url=<URL of verification API>'
```

My another project [auth-server](https://github.com/FlyingShit-XinHuang/auth-server) could be used as an auth server. The 'auth_server_url' could
be 'http://&lt;your server ip&gt;:18080/info'.

The resource API is protected now:

```
$ curl localhost:8000/demo/foo
{"message":"Missing token"}
```

Query the auth server to generate a token and query the resource API with the 'Authorization' 
header:

```
$ curl localhost:8000/demo/foo \
    -H "Authorization: bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE0OTMzNzAwNjUsIm5iZiI6MTQ5MzM2NjQ2NSwiaWF0IjoxNDkzMzY2NDY1LCJjbGllbnRfaWQiOiI1Si1iMHRNclRGUzNBeExuckNmSDVBIn0.XUMUYMrrtKRKS11fVOvy4Vr4whS9ffRIxOQ_psSubwo"
hello world
```
