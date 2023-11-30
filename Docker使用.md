个人使用来看，Docker的使用主要可以分为三个部分：`docker image`、`docker container`和`docker-compose`。其中，Image（镜像）是一个完整的OS环境，其中安装了运行服务必要的依赖。而每次启动Image时，会生成一个相应的Container（容器），不同的container之间相互独立。

# Image

## Image管理

- 从仓库下载image：docker pull repo:tag
- 显示本地所有image：docker image list/ls
- 删除某个image：docker rmi ID/repo:tag 或者 docker image rm ID/repo:tag

## Image构建/Dockerfile

在存放Dockerfile的目录下，运行`docker build -t repo:tag .`生成image。

Dockfile的结构大致如下：

```dockerfile
FROM base
RUN xxx && yyy
ENTRYPOINT ["xxx","yyy"]
```

其中的一些指令：

- FROM：指定基础镜像，可以是OS环境（alpine、ubuntu），也可以是某个应用环境（nginx）
- RUN：构建过程中在镜像中执行的命令。每运行一次RUN时，就会在容器上构建一层，因此要避免使用过多RUN命令（可以在不同命令之间使用`&&`连接）
- ENV：设置环境变量，`ENV key value`或是`ENV key1=value1 key2=value2`
- COPY：将本机文件或目录复制到镜像
- WORKDIR：设置后续指令和容器的工作目录
- ENTRYPOINT：从镜像创建容器时的启动命令

# Container

## container启动

将image启动为container的最基本命令：`docker run IMAGE_NAME`。其中在`IMAGE_NAME`前常用的一些参数：

- -d：后台运行容器
- -it：交互模式运行容器，另外分配一个terminal
- -p：端口映射，格式为`host_port:container_port`
- --name="NAME"：为容器指定名称，否则随机分配
- -e ENV="VALUE"：为容器设置环境变量，多个环境变量用多个-e设置
- --network，--net="NET"：指定容器的网络连接类型
- -v：绑定一个docekr卷，或是进行文件映射，格式为`host_path:container_path`

## container管理

下列命令中的CONTAINER可以为ID和name形式

- 停止容器：docker stop CONTAINER

    ​	需要注意的是，容器停止后并没有消失，还可以重新启动，如下

- 启动容器：docker start CONTAINER

- 重启容器：docker restart CONTAINER

    ​	此种方式不包括文件系统的卸载和挂载，一般意义的重启不会使用此命令

- 进入容器：docker exec -it CONTAINER /bin/bash

    ​	此种方式会进入容器，以bash方式交互

- 删除容器：docker rm CONTAINER

- 删除所有停止容器：docker container prune

- 实时输出容器日志：docker logs -f CONTAINER

- 显示正在运行的所有容器：docker container list/ls 或 docker ps

# Docker compose

docker compose可以同时管理多个容器。
