### docker

1. 背景

   不同机器，不同版本，不同的环境配置下，软件的部署和运行难的问题，提出了一个环境配置打包的理念，应用从本地开发联调->部署测试->应用上线，如何保证各个环境下的机器，软件的配置和版本以及所依赖的环境都是一致的，就是docker需要解决的软件开发部署的难题，往往不同的环境需要将本地配置的环境和依赖重新配置一遍，而由于机器环境的不同，导致各种各样的问题。

   1. 虚拟机

      > 将应用安装在虚拟机中，如果需要迁移环境，可以将虚拟机打包重新在新机器上安装，可以解决这个问题，但是由于现代虚拟机 1、资源占用多（虚拟机是一个虚拟在机器上的一个完整的操作系统，除了应用所需的资源外，还有操作系统本身庞大的结构资源占用） 2、冗余步骤多 3、启动慢

   2. Linux容器

      > Linux针对虚拟机的缺点，提出了Linux虚拟化技术，Linux容器，它不是模拟的一个完整的操作系统，而是对进程进行隔离，在进程外面套了一层保护壳，对于容器里面的进程来说，它接触的各种资源都是虚拟的，从而实现与底层系统的隔离。启动快，占用资源少（只需要自己本身的资源和依赖环境和软件的资源），体积小（不是一个完整的操作系统，而是一个包装的进程）

   3. Docker

      > docker就是linux容器的封装，提供简单的容器使用接口来对容器内部进行编排操作
      >
      > 提供一次性服务，提供云服务（随时动态扩容和缩容），组建微服务架构，一台机器可以跑多个容器，对外暴露不同的应用端口，模拟微服务架

2. docker安装

   mac: `brew install docker`

   执行brew的包安装命令 下载的是docker的服务端和客户端的整合包，需要手动启动app

   docker version/info 查看安装的docker信息

3. docker相关概念

   Image：<b>镜像</b>，容器模板，是系统运行的容器的创建模板，类似于类和对象的关系，image是类模版，可以编写makefile文件来自定义或者改造本地拉取的镜像生成自己的

   Container：<b>容器</b>，系统运行的实例，是一个独立可运行的进程，

   Docker Hub：<b>镜像仓库</b>，官方镜像仓库，可以上传和拉取镜像（类比github 仓库）

4. hello-world

   1. 安装好docker后，运行第一个hello-world
   2. 从docker hub 拉取hello-world镜像，`docker pull hello-world`
   3. docker image ls 可以查看本地镜像，发现有了一个hello-world镜像

   ![截屏2021-05-30 上午11.16.01](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530111623.png)

   4. 生成一个容器，并运行

   `docker container run hello-world`

   ![截屏2021-05-30 上午11.18.43](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530111852.png)

   注意，<span style="background:pink;border-radius: 0px;">docker container run</span> 会自动拉取image文件，如果本地没有，则会从远程仓库拉取

   hello-world输出后，就会自动停止，容器自动终止，有些容器提供服务，则不会自动终止，比如ubuntu，ng等应用型容器

   > docker run 参数
   >
   > - -d 后台运行容器，并返回容器id
   > - -i 交互模式运行容器，-t 为容器分配一个伪终端，通常搭配使用 -it
   > - --name 指定一个容器名
   > - -p 指定端口映射  主机端口:容器端口

5. 容器

   > image文件生成的容器实例，本质上也是一个文件，容器文件，此时会有两个文件，一个是镜像文件，一个是容器文件，关闭容器并不会删除容器文件，只是容器的进程停止了而已

   ```shell
   # 本机正在运行的容器
   $ docker container ls
   # 信息包括 容器id-containId  依赖镜像-image 创建时间 status状态 prots端口 names名称
   # 列出本机所有容器，包括停止的
   $ docker container ls --all
   # 终止容器运行
   $ docker container kill [containid]
   # 删除容器文件
   $ docker container rm [containID]
   
   # 重复使用容器文件-run 会重新生成一个容器文件运行，而start会复用一个容器文件
   $ docker container start [containerId]
   
   # 终止容器运行，非强制
   $ docker container stop [containerId]
   # 和kill的区别在于，kill是向容器的主进程发出SIGKILL信号，stop是先发出SIGTERM信号，进行收尾清理工作，过会儿再发出SIGKILL信号，强行终止，正在进行中的操作会丢失-推荐
   
   # 重启容器
   $ docker restart [容器名]
   ```

   ![截屏2021-05-30 上午11.33.04](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530113323.png)

   > 端口映射
   >
   > 服务容器启动会会绑定容器内的端口，如果需要外界（比如本机）需要将本地端口映射到容器的端口，这样就可以在本机访问到docker容器中的服务了

   ```shell
   $ docker run -d --name es -p 9200:9200 -p 9300:9300 -e image:version
   ```

   > 在docker容器中安装好之后，我们需要进入容器修改配置

   ```shell
   $ docker container exec -it [containerId] /bin/bash
   ```

   ```shell
   # 将本地文件拷贝到docker容器，（安装ik）
   $ docker cp [from] [to]
   ```

   

6. DockerFile文件编写

   > 如7所提及的 可以通过commit push的方式 来将容器打包，但是这种手工打包的方式效率低而且容易出错且可重复性弱， 如果需要在上面打包过的镜像修改后再做增加，还得重复前面所有步骤，并且这个过程是不透明的，无法知道这里面的构建分层步骤，所以推荐是使用Dockerfile的方式来构建镜像，但是dockerfile的方式里面还是通过docker commit一层一层构建镜像的，有一个记录可重复的过程

   是什么：

   ​	Dockerfile是一个文本文件，注意：文件名必须是Dockerfile，且没有后缀，通过docker bulid命令来解析Dockerfile文件里的构建步骤，打包生成镜像

   ```shell
   # build参数解析
   $ docker bulid [OPTION] build-context(会将这个目录下的所有文件发送给docker daemon，这个path为镜像构建提供所需要的文件和目录，默认Dockerfile会从这个目录下找，或者也可以通过-f 来指定)
   option:
   	-t imagename:tag 指定生成镜像的名称和tag
   	-f 指定Dockerfile的查找路径
   es:
   $ docker build -t ubuntu-vim:v1 .
   # 表示使用.当前目录下的Dockerfile构建镜像，并命名为ubuntu-vim:v1
   ```

   ```dockerfile
   # Dockerfile内容
   FROM ubuntu
   RUN apt-get update && apt-get install -y vim
   # FROM 表示base镜像
   # RUN 在基础镜像顶部增加一个镜像 装载了vim install
   ```

   可以使用docker history imagename来查看构建log

   ![截屏2021-05-31 下午10.17.55](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210531221833.png)

   可以看到ubuntu是base镜像，如果在本地已经有了，前面的image就会显示本地的镜像id，如果是从远程拉取的，则是missing，在base的基础上，增加了一层vim镜像，对已有的镜像，docker会进行缓存，在使用到的时候，会直接使用该镜像，而因为镜像的构建是向上叠加镜像，所以如果某一层发生了变化，那么上层的所有都会缓存失效。

   比如在上面的dockerfile中 加一行 ，新的Dockerfile就用不到原先的镜像了

   > 因为Dockerfile和 build log，如果构建失败，我们可以通过log来查看原因，方便追溯哪一步的构建出了问题。

   > Dockerfile常用指令：
   >
   > - FROM
   >
   >   指定base镜像
   >
   > - MAINTAINER
   >
   >   镜像作者-字符串
   >
   > - COPY
   >
   >   COPY str dest / ["src","dest"]  将文件从build context 复制到镜像
   >
   > - ADD
   >
   >   类似COPY 只是会对压缩文件自动解压
   >
   > - RUN
   >
   >   在容器中运行指定命令
   >
   > - CMD
   >
   >   容器启动时运行指定的命令
   >
   > - ENTRYPOINT
   >
   >   容器启动时运行的命令
   >
   > -- 注释#开头

7. 容器的提交和拉取

   > 容器可以打包到docker hub，实现一次发布，到处拉取运行，保证不同的环境的部署是相同的
   >
   > 1、登陆docker hub
   >
   > 2、创建镜像仓库
   >
   > 3、本地提交
   >
   > ```shell
   > # 提交容器到仓库
   > $ docker commit -a "desc" -p [containerId] [repository:tag]
   > ```
   >
   > ![截屏2021-05-30 下午7.18.26](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530191839.png)
   >
   > 可以看到已经生成了一个镜像，下一步上传这个镜像
   >
   > ```shell
   > $ docker push image:tag
   > ```
   >
   > denied: requested access to the resource is denied 
   >
   > 处理原因和方法：上传自己的镜像必须前面是自己的dockerid(用户名)，在使用容器生成镜像的时候处理或者通过docker tag来修改.
   >
   > ![截屏2021-05-30 下午7.25.03](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530192516.png)
   >
   > ![截屏2021-05-30 下午7.38.02](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530193820.png)
   >
   > ![截屏2021-05-30 下午7.38.32](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530193844.png)
   >
   > 上传成功
   >
   > 4、清空本地容器和镜像，从远程拉取
   >
   > ```shell
   > $ docker pull image:tag
   > ```
   >
   > ![截屏2021-05-30 下午7.47.30](https://cdn.jsdelivr.net/gh/MeqiangX/cloud-img@master/20210530194805.png)

8. 查看docker容器启动日志

   ```shell
   $ docker logs containerId/name
   ```

9. 查看docker容器ip和状态

   ```shell 
   #查看全部状态
   $ docker inspect container-name
   # 查看ip
   $ docker inspect --format='{{.NetworkSettings.IPAddress}}' container-name
   # --format里的{{}}是从 全部状态返回的json里面查找出来的，类似的还有{{.Name}}容器名
   #{{.State.Running}} 运行状态等等
   ```

10. docker-compose

    用于定义和运行多容器docker应用程序的工具，可以使用yml文件来配置应用程序需要的所有服务，然后使用一个命令就可以运行所有服务

    > 使用docker-compose搭建 单机的es-集群

    MacOS/windows下docker app自带了compose，docker-compose version 查看版本

    > 常用命令

    ```shell
    $ docker-compose up 执行构建
    $ docker-compose down 停止容器服务并rm
    ```

    