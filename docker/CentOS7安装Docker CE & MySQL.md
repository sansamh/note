## 安装前检查

Docker 目前支持 CentOS 7 及以后的版本，内核要求至少为 3.10。

Docker 官网有安装步骤，本文只是记录一下，您也可以参考 [Get Docker CE for CentOS](https://docs.docker.com/install/linux/docker-ce/centos/)

```shell
$ cat /etc/redhat-release 
CentOS Linux release 7.7.1908 (Core)
$ uname -a
Linux sansam 3.10.0-862.el7.x86_64 #1 SMP Fri Apr 20 16:44:24 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux

```

## 卸载旧版本

旧版本的 Docker 被叫做 `docker` 或 `docker-engine`，如果您安装了旧版本的 Docker ，您需要卸载掉它。

```shell
$ sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

旧版本的内容在 `/var/lib/docker` 下，目录中的镜像(images), 容器(containers), 存储卷(volumes), 和 网络配置（networks）都可以保留。

Docker CE 包，目前的包名为 `docker-ce`。

## 安装准备

为了方便添加软件源，支持 devicemapper 存储类型，安装如下软件包

```shell
$ sudo yum update
$ sudo yum install -y yum-utils \
  device-mapper-persistent-data \
  lvm2
```

## 添加 yum 软件源

添加 Docker 稳定版本的 yum 软件源

```shell
$ sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
```

## 安装Docker

```shell
$ sudo yum install -y docker-ce
```

如果弹出 GPG key 的接收提示，请确认是否为 `060a 61c5 1b55 8a7f 742b 77aa c52f eb6b 621e 9f35`，如果是，可以接受并继续安装。

至此，Docker 已经安装完成了，Docker 服务是没有启动的，操作系统里的 docker 组被创建，但是没有用户在这个组里。

至此，Docker 已经安装完成了，Docker 服务是没有启动的，操作系统里的 docker 组被创建，但是没有用户在这个组里。

> **`注意`**
>
> 默认的 docker 组是没有用户的（也就是说需要使用 sudo 才能使用 docker 命令）。
> 您可以将用户添加到 docker 组中（此用户就可以直接使用 docker 命令了）。

加入 docker 用户组命令

```shell
$ sudo usermod -aG docker USER_NAME
```

用户更新组信息后，重新登录系统即可生效。

## 安装指定版本

如果想安装指定版本的 Docker，可以查看一下版本并安装。

```shell
$ yum list docker-ce --showduplicates | sort -r

docker-ce.x86_64  3:18.09.1-3.el7                     docker-ce-stable
docker-ce.x86_64  3:18.09.0-3.el7                     docker-ce-stable
docker-ce.x86_64  18.06.1.ce-3.el7                    docker-ce-stable
docker-ce.x86_64  18.06.0.ce-3.el7                    docker-ce-stable
```

可以指定版本安装,版本号可以忽略 `:` 和 `el7`，如 `docker-ce-18.09.1`

```shell
$ sudo yum install docker-ce-<VERSION STRING>
```

至此，指定版本的 Docker 也安装完成，同样，操作系统内 docker 服务没有启动，只创建了 docker 组，而且组里没有用户。

## 启动 Docker

如果想添加到开机启动

```shell
$ sudo systemctl enable docker
```

启动 docker 服务

```shell
$ sudo systemctl start docker
```

## 验证安装

验证 Docker CE 安装是否正确，可以运行 `hello-world` 镜像

```shell
$ sudo docker run hello-world
```

## 添加国内镜像加速器，下载更快速

在 **`/etc/docker/daemon.json`**里添加以下内容，如果文件不存在则创建文件。

```shell
{
  "registry-mirrors": [
    "https://dockerhub.azk8s.cn",
    "https://reg-mirror.qiniu.com"
  ]
}
```

之后重新启动服务。

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart docker
```

## 检查加速器是否生效

查看docker info中的镜像信息

```shell
$ docker info
Registry Mirrors:
 https://dockerhub.azk8s.cn/
 https://reg-mirror.qiniu.com/

```



## 更新 Docker CE

```shell
$ sudo yum update docker-ce
```

## 卸载 Docker CE

```shell
$ sudo yum remove docker-ce
```

## 删除本地文件

注意，docker 的本地文件，包括镜像(images), 容器(containers), 存储卷(volumes)等，都需要手工删除。默认目录存储在 `/var/lib/docker`。

```shell
$ sudo rm -rf /var/lib/docker
```

---

## 安装MySQL

## 搜索镜像

```shell
$ sudo docker search mysql
```

## 拉取MySQL镜像

```shell
# 指定版本使用 sudo docker pull mysql:<version>，拉取最新使用pull mysql:latest 或者直接pull mysql
$ sudo docker pull mysql
```

## 查看本地镜像

```shell
$ sudo docker images | grep mysql
```

## 复制镜像并重命名

```shell
$ sudo docker tag mysql:latest mysql:8.0
```

## 创建存储目录

创建MySQL持久化文件目录 文件目录、配置文件目录、日志文件目录，同时创建配置文件my.cnf

```shell
$ sudo mkdir -p /home/data/mysql/data /home/data/mysql/conf /home/data/mysql/logs
$ sudo touch /home/data/mysql/conf/my.cnf

```

## 启动容器

options说明:
–restart=always: 重启策略
-d: 后台运行容器，并返回容器ID
-p: 端口映射，格式为：**主机(宿主)端口:容器端口**
–name: 为容器指定一个名称
-v: 给容器挂载存储卷，挂载到容器的某个目录
–mount: 绑定数据目录和服务器配置文件
-e MYSQL_ROOT_PASSWORD: 设置数据库密码
–character-set-server: 设置编码
–collation-server: 设置编码

```shell
$ sudo docker run --restart=always --name=mysql8.0 -p 3306:3306 \
--mount type=bind,src=/home/data/mysql/conf/my.cnf,dst=/etc/my.cnf \
--mount type=bind,src=/home/data/mysql/data,dst=/var/lib/mysql \
-e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0 --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
```

## 检查配置信息

```shell
$ sudo docker inspect mysql8.0
...
"Mounts": [
	{
		"Type": "bind",
		"Source": "/home/data/mysql/conf/my.cnf",
		"Target": "/etc/my.cnf"
	},
	{
		"Type": "bind",
		"Source": "/home/data/mysql/data",
		"Target": "/var/lib/mysql"
	}
],
...
```

## 查看容器是否启动成功

```shell
$ sudo docker ps -a
```

## 远程工具连接

使用远程工具连接会报错: 2059 - Authentication plugin ‘caching_sha2_password’ cannot be loaded: …
原因: MySql 8.0 换了新的身份验证插件caching_sha2_password, 原来的身份验证插件为mysql_native_password。而客户端工具Navicat Premium中找不到新的身份验证插件caching_sha2_password;
解决方法如下

```shell
# 进入mysql容器
$ sudo docker exec -it mysql8.0 mysql –user=root –password=123456
# 切换数据库
mysql> use mysql;
# 修改校验插件规则
mysql> ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
# 刷新数据库
mysql> flush privileges;
```

