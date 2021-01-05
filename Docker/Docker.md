# Docker

因为docker需要操作文件系统，所以默认需要管理员权限

可以为当前用户分配用户组，但是这是一个安全隐患。因为用户组对docker具有与root用户相同的权限，所以docker用户组中应该只能添加那些确实需要使用docker的用户和程序。



一下命令行代码省略前面的 sudo（获取管理员权限）

```java
拉取镜像 docker pull nginx
启动docker systemctl start docker
配置以守护进程开机启动 systemctl enable docker
停止docker systemctl stop docker
重启docker systemctl restart docker
重新加载配置文件 systemctl daemon-reload
//这条命令会将Docker守护进程绑定到宿主机上的所有网络接口。Docker不会自动监测到网络变化，我们需要通过-H选项指定服务器的地址。docker -H：4200
修改Docker的守护进程 /usr/bin/docker -d -H tcp://0.0.0.0:2375
//如果不想每次运行客户端时都加上-H标志，可以通过设置DOCKER_HOST环境变量来省略此步骤
使用DOCKER_HOST环境变量 export DOCKER_HOST="tcp://0.0.0.0:2375"
//将Docker守护进程绑定到非默认套接字 
/usr/bin/docker -d -H unix=//home/docker/docker.dock
//将Docker守护进程绑定到多个地址
/usr/bin/docker -d -H tcp://0.0.0.0:2375 -H unix=//home/docker/docker.dock
```

### 容器



```java
//-i保证容器中STDIN是开启的。-t告诉docker为要创建的容器分配一个伪tty终端
创建容器 docker run -i -t mysql /bin/bash
//---连接上服务后，再交互式shell中
容器的主机名 hostname
检查容器的进程 ps -aux
退出 exit

查看系统中的容器列表 docker ps -a
通过命名方式启动容器 docker run --name mydocker -i -t mysql /bin/bash
启动已停止的容器，可以通过name或id启动 docker start mydocker
    重新附着到容器，显示交互式shell docker attach mydocker
//创建守护式容器 ,-d会将容器放到后台运行。命令中还有一个while循环，该循环会一直打印hello world，直到容器或进程停止运行
docker run --name mydocker -d mysql /bin/bash -c "while true; do echo hello world; sleep 1; done"
获取容器的日志 docker logs mydocker
    跟踪容器日志 docker logs -f mydocker
    获取日志最新几行 docker logs --tail 10 mydocker
    跟踪容器最新日志 docker logs --tail 0 -f mydocker
    每条日志加上时间戳 docker logs -ft mydocker
    查看容器内进程 docker top mydocker
    //-d表明要运行一个后台进程，-d标志之后，指定的是要在内部执行这个命令的容器名字。
    在容器中运行后台任务 docker exec -d mydocker touch /etc/new_config_file
    在容器中运行交互式命令 docker exec -t -i mydocker /bin/bash
    停止正在运行的docker容器 docker stop mydocker
    //启动重启容器，由于某种错误导致容器停止运行，可以通过--restart让Docker自动重启该容器。--restart会检查容器的退出代码，并决定是否重启容器。--restart=always无论推出代码是什么，都会重启。--restart=on-failure:5，表示当容器退出代码为非0时，Docker会尝试自动重启该容器，最多重启5次
    docker run --restart=always --name mydocker -d ubuntu /bin/bash -c "while true; do echo hello world; sleep 1; done"
    查看容器详细信息 docker inspect mydocker
    有选择地获取容器信息 docker inspect --format={{".State.Running"}} mydocker
    或docker inspect -f {{".ID"}} mydocker
    删除容器 docker rm mydocker
```

### 镜像