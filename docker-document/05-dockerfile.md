# Dockerfile


[Dockerfile 介绍](https://docs.docker.com/engine/reference/builder/#usage)

Dockerfile是由一系列命令和参数构成的脚本，这些命令应用于基础镜像并最终创建一个新的镜像。它们简化了从头到尾的流程并极大的简化了部署工作。Dockerfile从FROM命令开始，紧接着跟随者各种方法，命令和参数。其产出为一个新的可以用于创建容器的镜像。


Docker能自动从Dockerfile中读取指令来创建镜像
Dockerfile是一个文本文档,包含了创建镜像需要的所有命令行命令,使用`docker build` 能创建一个自动构建,并且连续执行`Dockerfile`中的命令


Dockerfile 格式


1. FROM
```
FROM <image> [AS <name>]
FROM <image>[:<tag>] [AS <name>]
FROM <image>[@<digest>] [AS <name>]
```

初始化一个新的构建镜像, 这个镜像将作为后来指令的基础镜像. 一个合法的`Dockerfile`必须以`FROM`指令开始. 这里的镜像可以是任何合法的镜像,一种比较简单的情况是从[公共仓库]()中拉取镜像( pulling an image)
    * `ARG` 是唯一一个可以在`FROM`之前出现的指令
    * `FROM`在同一个Dockerfile中可以出现多次来创建多个镜像,每个FROM指令会清空之前的FROM指令的状态

`FROM`支持在他之前的`ARG`指令所声明的变量
```
ARG  CODE_VERSION=latest
FROM base:${CODE_VERSION}
CMD  /code/run-app

FROM extras:${CODE_VERSION}
CMD  /code/run-extras
```
在`FROM`之前声明的`ARG`是在构建阶段之外的,所以不能在`FROM`以后的指令当中使用.如果需要在构建阶段使用`FROM`之前的`ARG`所声明的变量,需要使用`ARG`指令重新书写,并且此时的`ARG`是没有赋值的

```
ARG VERSION=latest
FROM busybox:$VERSION
ARG VERSION
RUN echo $VERSION > image_version
```
2. RUN

* RUN <command> (shell form, the command is run in a shell, which by default is /bin/sh -c on Linux or cmd /S /C on Windows)
* RUN ["executable", "param1", "param2"] (exec form)

RUN指令会在当前镜像的基础上,建立新的layer并且执行传入的命令,最后将结果commit. 这个commit结果会被用于Dockerfile中的下一步
RUN指令是默认使用cache的,例如`RUN apt-get dist-upgrade -y`会在build阶段重复利用的, 如需关闭cache功能,需要加上`--no-cache`参数,如: `docker build --no-cache`

3. CMD
CMD指令有3种格式
* CMD ["executable","param1","param2"] (exec form, this is the preferred form)
* CMD ["param1","param2"] (as default parameters to ENTRYPOINT)
* CMD command param1 param2 (shell form)

CMD指令只能存在一个,如果有多个,后出现的会覆盖前面的
he main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.

CMD的主要目的是为运行容器是提供默认的启动参数

> 如果CMD是用来为`ENTRYPOINT`提供默认参数的时候,`CMD`和`ENTRYPOINT`都必须用JSON的数组形式进行表示
> 执行的格式是用JSON的数组格式传递的,意味着必须使用`""`而不是`''`

When used in the shell or exec formats, the CMD instruction sets the command to be executed when running the image.

If you use the shell form of the CMD, then the <command> will execute in /bin/sh -c:

当使用shell或者可执行的形式时,CMD指令所设置的命令会在镜像运行的时候被执行
当使用shell的形式的时候,命令会默认用`/bin/sh -c`的方式来执行

```
FROM ubuntu
CMD echo "This is a test." | wc -
```

如果不需要shell去执行CMD传入的命令,这时候需要提供`executable`的完整路径.这是CMD命令推荐的格式,除了`executable`以外的其他参数,都必须在数组中单独传递.

```
FROM ubuntu
CMD ["/usr/bin/wc","--help"]
```

如果容器每次启动的时候都执行相同的参数,可以考虑将`ENTRYPOINT`和`CMD`结合使用

如果在启动的时候` docker run `带入了参数 , 这时候会覆盖CMD中所指定的参数

4. LABEL
5. MAINTAINER
6. EXPOSE
7. ENV
7. ADD
7. COPY
7. ENTRYPOINT
7. VOLUME
7. USER
7. WORKDIR
7. ARG
7. ONBUILD
7. STOPSIGNAL
7. HEALTHCHECK
7. SHELL