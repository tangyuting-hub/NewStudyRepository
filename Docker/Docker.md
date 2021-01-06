

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

本地镜像都保存在Docker宿主机的/var/lib/docker

在/var/lib/docker/containers目录下面可以看到所有的容器

```java
列出镜像 docker images
    拉取镜像 docker pull centos
    查找镜像 docker search centos
    //推荐使用Dockerfile文件
    构建镜像 dcoker commit 或 docker build 配合Dockerfile文件
    提交定制容器 docker commit dockerID myDocker
    //提交定制容器详细信息,-m指定镜像的提交信息，--author指定镜像作者，:webserver为标签
    docker commit -m="A new image" --author="mine" dockerID myDocker:webserver
   
```

Docker按照如下流程执行Dockerfile中的指令

- Docker从基础镜像运行一个容器
- 执行一条指令，对容器作出修改
- 执行类似commit的操作，提交一个新的镜像层
- Docker再基于刚提交的镜像运行一个新容器
- 执行Dockerfile的下一条指令，直至所有指令执行完毕

每个Dockerfile的第一条指令都是FROM。FROM指定一个已经存在的镜像，，这个镜像成为基础镜像。

接着指定MAINTAINER镜像，这条指令的作用是告诉Docker，该镜像作者是谁。

接下来就是RUN指令。

EXPOSE指令，指定Docker该容器内的应用程序将会使用的端口。

```java
 //Dockerfile创建镜像
    创建示例仓库
    mkdir static_web
    cd static_web
    touch Dockerfile
        
 
 DockerFile文件内容
 	FROM ubuntu:14.04
    MAINTAINER tangyuting "18636303146@163.com"
    RUN apt-get update
    RUN apt-get install -y nginx
    RUN echo 'Hi,I am ......'
        >/usr/share/nginx/html/index.html
    EXPOSE 80
 
 运行Dockerfile 
        cd static_web
        docker build -t="mydocker/static_web:v1" . //-t为新镜像设置了仓库和名称.:v1为该镜像设置的标签，如未设置标签，Docker默认设置为latest。命令最后的.告诉Docker到本地目录去找Dockerfile文件
        如果再Git仓库根目录下存在Dockerfile文件
        docker build -t="mydocker/static_web:v1" git@github.com:tangyuting-hub/NewStudyRepository
        忽略Dockerfile的构建缓存
        docker build --no-cache -t="mydocker/static_web:v1" .
        查看镜像的构建历史  docker history mydockerID
        
        查看Docker端口映射情况  docker ps -l  或 docker port mydockerID 80
```

#### Dockerfile指令

##### CMD

CMD指令用于指定一个容器启动时要运行的命令。docker run 的命令会覆盖CMD指令。

指定要运行的特殊命令 docker run -i -t ubuntu /bin/true

使用CMD指令  CMD ["/bin/true"]

用CMD命令指定参数 CMD ["/bin/bash", "-1"]

##### ENTRYPOINT

ENTRYPOINT和CMD类似，但是ENTRYPOINT指令提供的命令不容易在启动容器时被覆盖。

##### WORKDIR

WORKDIR指令用来从镜像创建一个新容器，在容器内部设置一个工作目录。ENTRYPOINT或CMD指定的程序会在这个目录下执行。

==使用WORKDIR指令==

WORKDIR /opt/webapp/db

RUN bundle install

WORKDIR /optwebapp

ENTRYPOINT ["rackup"]

可以使用-w在运行时覆盖工作目录

docker run -ti -w /var/log ubuntu pwd /var/log

##### ENV

ENV指令用来在镜像构建过程中设置环境变量

##### USER

USER指令用来指定该镜像以什么样的用户去运行

##### VOLUME

VOLUME指令用来向基于镜像创建的容器添加卷。一个卷可以存在于一个或者多个容器内特定的目录，这个目录可以绕过联合文件系统，并提供如下共享数据或者对数据进行持久化的功能。

- 卷可以在容器间共享和重用
- 一个容器可以不是必须和其他容器共享卷
- 对卷的修改是立即生效的
- 对卷的修改不会对更新镜像产生影响
- 卷会一直存在直到没有任何容器再使用它

卷功能让我们可以将数据、数据库、或其他内容添加到镜像中，而不是将这些内容提交到镜像中，并允许我们再多个容器间共享这些内容。

使用VOLUME指令： VOLUME ["opt/project"]

##### ADD

ADD指令用来将构建环境下的文件和目录复制到镜像中。

##### ONBUILD

ONBUILD指令能为镜像添加触发器，当一个镜像被用做其他镜像的基础镜像时，该镜像的触发器将会被执行。

```java
推送镜像到dockerHub docker push mydocker
    删除镜像 docker rmi mydocker
```

