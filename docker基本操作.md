# docker基本操作

### 镜像下载

使用pull命令下载镜像

```shell
admin@SW1:~$ docker pull asternos/bfnstratum
```

这个例子就是下载一个名为`asternos/bfnstratum`的镜像。

要想列出已经下载下来的镜像，可以使用 `docker images` 命令。

```shell
admin@SW1:~$ docker images
REPOSITORY                    TAG                           IMAGE ID            CREATED             SIZE
asternos/bfnstratum           latest                        b8fb5cff3d87        2 days ago          5.25GB
```

### 容器运行

使用`docker run`来新建一个容器并运行，注意这个会基于给出的镜像新建一个容器，所以每运行一个都会新建出来一个容器。

```shell
admin@SW1:~$ docker run -i -t asternos/bfnstratum:latest bash
root@96e7476490a8:/# 
```

在这个命令中，使用到的是`asternos/bfnstratum`这个镜像，用来创建的容器，其中的`-i -t`是为了让容器产生终端，并一直处于打开状态。后面的`/bin/bash`是为了紧接着出来一个bash终端，来进行交互操作。

此时在交换机中使用`docker ps `命令就可以查看当前运行着哪些容器。

```shell
admin@SW1:~$ docker ps
CONTAINER ID        IMAGE                        COMMAND             CREATED             STATUS              PORTS               NAMES
96e7476490a8        asternos/bfnstratum:latest   "bash"         4 minutes ago       Up 4 minutes                            relaxed_elbakyan
```

此时如果在容器内输入`exit`命令进行退出，则会终止容器，但不会进行删除，此时可以使用`docker ps -a`来进行查看所有现在存在的容器

```shell
admin@SW1:~$ docker ps -a
CONTAINER ID        IMAGE                                COMMAND                  CREATED             STATUS                       PORTS               NAMES
96e7476490a8        asternos/bfnstratum:latest           "bash"              5 minutes ago       Exited (130) 4 seconds ago                       relaxed_elbakyan
```

或者直接使用`Ctrl + p + q`来从容器中出来，退回交换机之中

```shell
admin@SW1:~$ docker run -it asternos/bfnstratum:latest bash
root@96e7476490a8:/# admin@SW1:~$ 
admin@SW1:~$ 
admin@SW1:~$ docker ps
CONTAINER ID        IMAGE                        COMMAND             CREATED             STATUS              PORTS               NAMES
96e7476490a8        asternos/bfnstratum:latest   "bash"              15 seconds ago      Up 13 seconds                            relaxed_elbakyan
```

这个样子的退出不会终止容器的运行，会让容器在后台运行。

### 启动终止的容器

这时可以在交换机中，也就是容器外面使用`docker start`来重新启动容器

```shell
admin@SW1:~$ docker start -i relaxed_elbakyan 
root@96e7476490a8:/# 
```

其中，`-i`是让它以交互模式启动，后面的`relaxed_elbakyan`是容器的名称。

### 进入在后台运行的容器之中

有两种方法进入容器使用 `docker attach` 命令或 `docker exec` 命令，，推荐大家使用 `docker exec` 命令。

使用`docker attach` 命令时

```shell
admin@SW1:~$ docker attach  relaxed_elbakyan 
root@96e7476490a8:/# 
```

在`docker attach` 命令进入容器的情况下，如果使用`exit`命令则会使容器终止。

使用`docker exec` 命令时

```
admin@SW1:~$ docker exec -it relaxed_elbakyan bash
root@96e7476490a8:/# 
```

在`docker exec` 命令进入容器的情况下，使用`exit`命令不会导致容器的停止。这就是为什么推荐使用 `docker exec` 的原因。
