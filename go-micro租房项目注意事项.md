# 1、mac端go直接编译的可执行文件无法在docker环境（amd架构）执行，需要交叉编译。

![](/Users/liuting/Library/Application%20Support/marktext/images/2023-04-05-19-20-08-image.png)

# 2、改了代码记得重新make build可执行文件。

# 3、改了dockerfile || docker-compose.yml文件记得删除images重新生成。

# 4、进入容器调试环境

```shell
docker run -it images_name bash // 容器未启动
```

```shell
docker exec -it container_name bash //容器已启动
```

# 5、镜像内执行命令

不需要在执行docker命令时加入内部命令参数使用CMD

```shell
FROM ubuntu:18.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
CMD [ "curl", "-s", "http://myip.ipip.net" ]
```

```shell
$ docker run myip
当前 IP：61.148.226.66 来自：北京市 联通
```

跟在镜像名后面的是 `command`，运行时会替换 `CMD` 的默认值。因此这里的 `-i` 替换了原来的 `CMD`，而不是添加在原来的 `curl -s http://myip.ipip.net` 后面。而 `-i` 根本不是命令，所以自然找不到。

需要在外部附加参数，使用ENTRYPOINT

```shell
FROM ubuntu:18.04
RUN apt-get update \
    && apt-get install -y curl \
    && rm -rf /var/lib/apt/lists/*
ENTRYPOINT [ "curl", "-s", "http://myip.ipip.net" ]
```

```shell
$ docker run myip
当前 IP：61.148.226.66 来自：北京市 联通

$ docker run myip -i
HTTP/1.1 200 OK
Server: nginx/1.8.0
Date: Tue, 22 Nov 2016 05:12:40 GMT
Content-Type: text/html; charset=UTF-8
Vary: Accept-Encoding
X-Powered-By: PHP/5.6.24-1~dotdeb+7.1
X-Cache: MISS from cache-2
X-Cache-Lookup: MISS from cache-2:80
X-Cache: MISS from proxy-2_6
Transfer-Encoding: chunked
Via: 1.1 cache-2:80, 1.1 proxy-2_6:8006
Connection: keep-alive

当前 IP：61.148.226.66 来自：北京市 联通
```

# Mysql指定端口登陆

```shell
mysql -uroot -P port -h IP -ppasswor
```

# Mysql 账户管理相关

出了一个bug，localhost能连上root账号，docker容器内的172.17.0.2 ip却连不上，最后靠生成一个test@% 的账号解决了。

mysql用户信息在mysql.user这张表上。

# 容器互相访问相关

同一个宿主机的容器互相访问要靠记录IP地址(一般是172.xx打头)


