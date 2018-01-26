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
在`FROM`之前声明的`ARG`是在构建阶段之外的,所以不能在`FROM`以后的指令当中使用.如果需要在构建阶段使用`FROM`之前的`ARG`所声明的变量,需要使用`ARG`指令重新书写,并且此时的`ARG`不需要赋值

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
the main purpose of a CMD is to provide defaults for an executing container. These defaults can include an executable, or they can omit the executable, in which case you must specify an ENTRYPOINT instruction as well.

CMD的主要目的是为运行容器是提供默认的启动参数,在启动参数中可以提供可执行路径,如果未提供则必须指定`ENTRYPOINT`指令

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
```
LABEL <key>=<value> <key>=<value> <key>=<value> ...
```
LABEL指令以`key-value`的形式为镜像添加metadata,用`""`将value包裹起来,这时候value中可以使用空格或者`\`

```
LABEL "com.example.vendor"="ACME Incorporated"
LABEL com.example.label-with-value="foo"
LABEL version="1.0"
LABEL description="This text illustrates \
that label-values can span multiple lines."
```
一个镜像可以有多个LABEL,也可以在单条指令中指定多个label,写法如下:
```
LABEL multi.label1="value1" multi.label2="value2" other="value3"
```
```
LABEL multi.label1="value1" \
      multi.label2="value2" \
      other="value3"
```
在基础镜像或者父镜像中指定的LABEL默认是会被子镜像所继承的,如果子镜像中为同一个key匹配了不懂的value,则后面声明的LABEL会覆盖之前的

使用`docker inspect`可以查看镜像的LABEL

5. MAINTAINER

```
MAINTAINER <name>
```
MAINTAINER指令是设置当前镜像的作者,在新版本中已经不推荐使用. 取而代之的是更具有灵活性的LABEL,设置方式如下:
```
LABEL maintainer="SvenDowideit@home.org.au"
```
使用`docker inspect`的时候可以查看到和其他label一起出现的`maintainer`

6. EXPOSE

```
EXPOSE <port> [<port>/<protocol>...]
```
EXPOSE指令会通知Docker当前容器在运行时监听的端口,可以指定TCP或者UDP协议,默认是TCP.

EXPOSE指令在实际应用的时候并不一定是发布在改端口上,它只是为使用该镜像的人提供一个说明,指明该镜像内部所发布的端口.
在正真运行该容器的时候应该使用`docker run -p`来发布实际端口,或者使用`-P`将容器内EXPOSE指定的端口直接映射到宿主环境

7. ENV
```
ENV <key> <value>
ENV <key>=<value> ...
```
ENV指令用于设置环境变量,设置出来的环境变量在此Dockerfile中后续出现的指令当中有效.
* 第一种格式`ENV <key> <value>`一次只能定义一个环境变量,在第一个空格之后的所有字符串会被视为该key对应的value,包括空格和引号
* 第二种格式`ENV <key>=<value>`可以一次性定义多个环境变量,这时候可以用`"`去包裹value,可以用`\`作为换行的连接符.

例如:
```
ENV myName="John Doe" myDog=Rex\ The\ Dog \
    myCat=fluffy
```
```
ENV myName John Doe
ENV myDog Rex The Dog
ENV myCat fluffy
```
上述两种定义方式的结果是一样的,推荐使用第一种方式,因为只会产生一个`cache layer`.
当容器启动后,ENV会持续存在,可以使用`docker inspect`来查看.如果需要改变这些环境变量,可以在启动容器的时候使用`docker run --env <key>=<value>`

环境变量的持续存在会引起一些无法预料的问题,比如:设置环境变量`ENV DEBIAN_FRONTEND noninteractive`可能会在用户使用`apt-get`的时候引起混乱.只在单一命令中设置参数可以这样设置:
```
RUN <key>=<value> <command>.
```

8. ADD
ADD有两种格式:
```
ADD [--chown=<user>:<group>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"] (this form is required for paths containing whitespace)
```

ADD指令会复制`src`参数中的文件或者目的到`dest`所指定的目录当中去,`src`可以是一个远程的URL

src可以指定多个文件,当时本地文件的时候,src只能被解释为相对路径
src可以使用通配符,规则符合Go语言的[filepath.Match](https://golang.org/pkg/path/filepath/#Match)

```
ADD hom* /mydir/        # adds all files starting with "hom"
ADD hom?.txt /mydir/    # ? is replaced with any single character, e.g., "home.txt"
```

<dest>是一个绝对路径,或者相对于`WORKDIR`的相对路径
```
ADD test relativeDir/          # adds "test" to `WORKDIR`/relativeDir/
ADD test /absoluteDir/         # adds "test" to /absoluteDir/
```
在添加带有特殊符号(如[])的文件或者目录时,需要对其进行转译,防止GO语言将特殊符号视为了匹配模式.
例如添加一个文件名为`arr[0].txt`
```
ADD arr[[]0].txt /mydir/    # copy a file named "arr[0].txt" to /mydir/
```
如果没有使用`--chown`显示指明被添加文件的`username`和`groupname`或者`UID/GID`,那么所有被添加进来的文件的UID和GID都是0.如果只提供`username`而不提供`groupname`这时候GID和UID会使用同一个数字.
`--chown`可以使用数字或者字符串的任意组合,当`username`或者`groupname`被提供的时候,容器的文件系统中`/etc/passwd` and `/etc/group`会分别执行从name到 `UID or GID`的翻译

以下表示方法都是合法的:
```
ADD --chown=55:mygroup files* /somedir/
ADD --chown=bin files* /somedir/
ADD --chown=1 files* /somedir/
ADD --chown=10:11 files* /somedir/
```
当容器的文件系统中不存在`/etc/passwd`和`/etc/group`,而又在`--chown`后面添加了`username:groupname`这时候构建会失败.
当`src`是一个远程的URL,需要保证URL的文件权限至少是`600`

ADD指令还服从以下规则:

* `src`路径必须是在上下文以内的,不能使用上下文以外的相对路径,如`ADD ../something /something`,因为`docker build`的第一步就是将上下文中的文件全部发送给`docker daemon`
* 如果`src`是一个URL并且`dest`不以`/`结尾,这时`src`下载完成之后copy到`dest`
* 如果`src`是一个URL并且`dest`以`/`结尾,这时`src`会被下载到<dest>/<filename>. 例如: `ADD http://example.com/foobar /`会创建文件`/foobar`
* 如果`src`是一个目录,那整个目录包含的内容以及文件系统的`metadata`都会被`copy`.(目录本身不会被copy,只是它的内容)
* 如果`src`是本地的一个压缩文档(gzip, bzip2 or xz),他会被copy到`dest`并且被解压
* 如果`src`是其他类型的文件,并且dest是以'/'结尾,这时候`src`会被copy到<dest>/base(<src>
* 如果`src`提供了多个资源(通配符等),`dest`必须得以`/`结尾
* 如果`dest`不以`/`结尾,src会被写入`dest`
* 如果`dest`的路径并不存在,这个路径将会被创建.

9. COPY
有两种格式:
```
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"] (this form is required for paths containing whitespace)
```
COPY和ADD基本相同,COPY的`src`只能是本地文件不能是远程URL


10. ENTRYPOINT
ENTRYPOINT has two forms:
```
ENTRYPOINT ["executable", "param1", "param2"] (exec form, preferred)
ENTRYPOINT command param1 param2 (shell form)
```
ENTRYPOINT提供配置容器启动后执行的命令.
如: 使用默认配置启动nginx,并且监听在80端口
```
docker run -i -t --rm -p 80:80 nginx
```
`docker run <image>`中的命令行参数会被添加到exec form形式的参数之后.并且会覆盖CMD中指定的所有参数<br>
`docker run <image> -d`会将`-d`传递到`ENTRYPOINT`指定的参数之后.如需覆盖`ENTRYPOINT`配置,可以使用`docker run --entrypoint`

以shell的形式声明ENTRYPOINT时,CMD和启动命令行参数都会被忽略.这样会有一个劣势,ENTRYPOINT会以`/bin/sh -c`作为开始命令.这时候可执行的容不能接收到docker的stop命令

当Dockerfile中存在多个ENTRYPOINT指令时,只有最后一个ENTRYPOINT会生效.

例子1 exec form:
Dockerfile
使用ENTRYPOINT设置默认的命令和参数,用CMD来设置可变参数
```
FROM ubuntu
ENTRYPOINT ["top", "-b"]
CMD ["-c"]
```
运行容器,可以看到`top`是当前唯一的进程

```
root@yyserver:~ # docker run tangl2014/ubuntu:v1                                                    
top - 03:04:16 up 167 days, 15:15,  0 users,  load average: 0.45, 0.25, 0.15
Tasks:   1 total,   1 running,   0 sleeping,   0 stopped,   0 zombie
%Cpu(s):  0.3 us,  0.4 sy,  0.0 ni, 99.2 id,  0.0 wa,  0.0 hi,  0.0 si,  0.0 st
KiB Mem :   995012 total,   201044 free,   206240 used,   587728 buff/cache
KiB Swap:  4193276 total,  3985012 free,   208264 used.   598622 avail Mem 

   PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
     1 root      20   0   36528   1580   1220 R  0.0  0.2   0:00.00 top -b -c
```
例子2 shell form:

```
FROM ubuntu
ENTRYPOINT exec top -b
```
CMD和ENTRYPOINT的相互作用

* Dockerfile需要显示声明ENTRYPOINT和CMD,二者至少要有一个
* 当作为可执行容器时,ENTRYPOINT必须被指定.
* CMD可以用作定义ENTRYPOINT命令的默认参数
* 当运行容器时有在命令行显示指定参数时,CMD中预定义的参数会被覆盖

||No ENTRYPOINT|ENTRYPOINT exec_entry p1_entry|	ENTRYPOINT [“exec_entry”, “p1_entry”]|
|----|----|----|----|
|No CMD|error, not allowed|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry|
|CMD [“exec_cmd”, “p1_cmd”]|exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry exec_cmd p1_cmd|
|CMD [“p1_cmd”, “p2_cmd”]|p1_cmd p2_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry p1_cmd p2_cmd|
|CMD exec_cmd p1_cmd|/bin/sh -c exec_cmd p1_cmd|/bin/sh -c exec_entry p1_entry|exec_entry p1_entry /bin/sh -c exec_cmd p1_cmd|

11. VOLUME
```
VOLUME ["/data"]
```
VOLUME命令用于让你的容器访问宿主机上的目录。

12. USER
```
USER <user>[:<group>] or
USER <UID>[:<GID>]
```
用于设置运行容器的username或者UID,:后面可以跟上可选参数group和GID. 在USER命令之后的RUN, CMD 和 ENTRYPOINT都会以这里设置的USER为准

> 如果user没有用户群组,运行的时候会视为root群组

> On Windows, the user must be created first if it’s not a built-in account. This can be done with the net user command called as part of a Dockerfile.

```
    FROM microsoft/windowsservercore
    # Create Windows user in the container
    RUN net user /add patrick
    # Set it for subsequent commands
    USER patrick
```



13. WORKDIR
```
WORKDIR /path/to/workdir
```
WORKDIR是为它之后的RUN, CMD, ENTRYPOINT, COPY and ADD指令设置运行目录.
如果WORKDIR不存在,它将会被默认创建.
WORKDIR可以重复定义,如果是个相对路径,它将相对于它的前一个WORKDIR.
```
WORKDIR /a
WORKDIR b
WORKDIR c
RUN pwd
```
最终会输出: `/a/b/c`

WORKDIR能处理ENV指令设置的环境变量
```
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```
最终会输出: `/path/$DIRNAME`
14. ARG
```
ARG <name>[=<default value>]
```
ARG所声明的变量可以通过`docker build --build-arg <varname>=<value>`进行传入,如果传入的参数在Dockerfile中没有预先定义,则会收到一个警告`[Warning] One or more build-args [foo] were not consumed.`

ARG可以多次使用
```
FROM busybox
ARG user1
ARG buildno
...
```

注意: 传入build阶段的变量是可以通过`docker history`查看的,所以不推荐将一些敏感参数(secret key等)传入其中

* ARG的默认值
ARG可以设置变量的默认值,当build镜像的时候并没有任何` --build-arg <varname>=<value>`传入,这时候将使用ARG的默认值
```
FROM busybox
ARG user1=someuser
ARG buildno=1
...
```

* ARG的作用域


ARG变量生效是在定义该变量的ARG之后,而不是在命令行传入就能生效

如下Dockerfile
```
1 FROM busybox
2 USER ${user:-some_user}
3 ARG user
4 USER $user
...
```
build
```
$ docker build --build-arg user=what_user .
```
第2行 user 的值是some_user
第3行 通过ARG定义了user变量
第4行 中的user的值是命令行中传入的what_user


ARG的作用域会在不同的build阶段清除,如果在不同的build阶段需要使用之前的ARG,则需要重新引入ARG指令
```
FROM busybox
ARG SETTINGS
RUN ./run/setup $SETTINGS

FROM busybox
ARG SETTINGS
RUN ./run/other $SETTINGS
```

* 使用ARG变量

可以使用ARG和ENV指令来指明变量提供给RUN指令使用.当定义的变量名称相同时,环境变量会覆盖掉ARG变量

```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 ENV CONT_IMG_VER v1.0.0
4 RUN echo $CONT_IMG_VER
```
构建
```
 docker build --build-arg CONT_IMG_VER=v2.0.1 .
```
在此例子中,RUN指令会使用`v1.0.0`而不是`v2.0.1`,这个特性好比局部变量的优先级要高于外部传入的变量

```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 ENV CONT_IMG_VER ${CONT_IMG_VER:-v1.0.0}
4 RUN echo $CONT_IMG_VER
```
build
```
 docker build .
```
`CONT_IMG_VER`会在镜像中持续存在,其值是`v1.0.0`在第3行由ENV指令设置.

* 预定义的ARGs
```
HTTP_PROXY
http_proxy
HTTPS_PROXY
https_proxy
FTP_PROXY
ftp_proxy
NO_PROXY
no_proxy
```
如果需要使用这些ARG,可以从命令行中传入`--build-arg <varname>=<value>`


例子:
```
docker build --build-arg HTTP_PROXY=http://user:pass@proxy.lon.example.com .
```
Dockerfile
```
FROM ubuntu
RUN echo "Hello World"
```

此例中,`HTTP_PROXY`并没有在docker history中使用,所以也没有缓存.如果这时将proxy改变为`http://user:pass@proxy.sfo.example.com`, 在随后的build中不会命中之前的缓存.
如果需要覆盖这个特性,可以再次声明ARG
```
FROM ubuntu
ARG HTTP_PROXY
RUN echo "Hello World"
```
当build这个配置文件的时候,HTTP_PROXY就会存在docker history中, 改变ARG的值并不会音响缓存.


* ARG在构建缓存中的影响
ARG变量是不会在镜像中留存的(ENV变量会),但是ARG变量会对构建缓存造成影响.如果在Dockerfile中定义的ARG变量与之前的构建环境不同,这时候会丢失缓存,详细来说,所有在ARG之后的RUN指令使用会将ARG变量当做ENV变量来使用,这样会造成缓存的丢失.所有预定义的ARG变量都在缓存之外,除非在Dockerfile中再次声明这个ARG

例子:
Dockerfile1
```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 RUN echo $CONT_IMG_VER
```

Dockerfile2
```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 RUN echo hello
```

build
```
docker build --build-arg CONT_IMG_VER=<value>
```

两种情况中,第2行都指定了ARG,所以都不会造成缓存丢失;Dockerfile1中第3行,当CONT_IMG_VER=<value>改变时,会造成缓存丢失.


```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 ENV CONT_IMG_VER $CONT_IMG_VER
4 RUN echo $CONT_IMG_VER
```

In this example, the cache miss occurs on line 3. The miss happens because the variable’s value in the ENV references the ARG variable and that variable is changed through the command line. In this example, the ENV command causes the image to include the value.



If an ENV instruction overrides an ARG instruction of the same name, like this Dockerfile:
```
1 FROM ubuntu
2 ARG CONT_IMG_VER
3 ENV CONT_IMG_VER hello
4 RUN echo $CONT_IMG_VER
```
Line 3 does not cause a cache miss because the value of CONT_IMG_VER is a constant (hello). As a result, the environment variables and values used on the RUN (line 4) doesn’t change between builds.



15. ONBUILD

```
ONBUILD [INSTRUCTION]
```

ONBUILD指令会给镜像添加一个能够被触发的指令.当此镜像作为其他构建的基础镜像时,触发将会被执行(此时命令的上下文指向触发该指令的build)
当你需要构建一个镜像来作为其他构件的基础镜像时,这个特性将变得非常有用
例如:你的镜像是重复利用Python的应用,它需要应用源码添加到指定的目录,在这之后也许还需要调用一个构建脚本.
这时候就不能仅仅靠ADD,RUN来实现了,因为你无法访问到应用的源码,而且对于每次构建,源码可能是不同的.
这时候就应该使用ONBUILD来提前注册指令,提供给下游的builder进行调用

Here’s how it works:

When it encounters an ONBUILD instruction, the builder adds a trigger to the metadata of the image being built. The instruction does not otherwise affect the current build.
At the end of the build, a list of all triggers is stored in the image manifest, under the key OnBuild. They can be inspected with the docker inspect command.
Later the image may be used as a base for a new build, using the FROM instruction. As part of processing the FROM instruction, the downstream builder looks for ONBUILD triggers, and executes them in the same order they were registered. If any of the triggers fail, the FROM instruction is aborted which in turn causes the build to fail. If all triggers succeed, the FROM instruction completes and the build continues as usual.
Triggers are cleared from the final image after being executed. In other words they are not inherited by “grand-children” builds.
For example you might add something like this:

```
[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]
```


16. STOPSIGNAL

```
STOPSIGNAL signal
```

STOPSIGNAL 设置系统退出信号.

17. HEALTHCHECK

The HEALTHCHECK instruction has two forms:
```
HEALTHCHECK [OPTIONS] CMD command (check container health by running a command inside the container)
HEALTHCHECK NONE (disable any healthcheck inherited from the base image)
```
HEALTHCHECK 指令告诉Docker如何检测容器是否还在工作.这能检测到诸如一个web服务被死循环阻塞不能处理新的连接,虽然这个web服务仍然处于running状态.
当一个容器指定了healthcheck,它将会在正常状态之外还有一个健康状态.这个状态是默认开启的,当一个healthcheck通过后,他将变成healthy.当多次healthcheck失败之后他将变成unhealthy

CMD之前的OPTION取值如下:
```
--interval=DURATION (default: 30s)
--timeout=DURATION (default: 30s)
--start-period=DURATION (default: 0s)
--retries=N (default: 3)
```

The health check will first run interval seconds after the container is started, and then again interval seconds after each previous check completes.

If a single run of the check takes longer than timeout seconds then the check is considered to have failed.

It takes retries consecutive failures of the health check for the container to be considered unhealthy.

start period provides initialization time for containers that need time to bootstrap. Probe failure during that period will not be counted towards the maximum number of retries. However, if a health check succeeds during the start period, the container is considered started and all consecutive failures will be counted towards the maximum number of retries.

There can only be one HEALTHCHECK instruction in a Dockerfile. If you list more than one then only the last HEALTHCHECK will take effect.

The command after the CMD keyword can be either a shell command (e.g. HEALTHCHECK CMD /bin/check-running) or an exec array (as with other Dockerfile commands; see e.g. ENTRYPOINT for details).

The command’s exit status indicates the health status of the container. The possible values are:

0: success - the container is healthy and ready for use
1: unhealthy - the container is not working correctly
2: reserved - do not use this exit code
For example, to check every five minutes or so that a web-server is able to serve the site’s main page within three seconds:

HEALTHCHECK --interval=5m --timeout=3s \
  CMD curl -f http://localhost/ || exit 1
To help debug failing probes, any output text (UTF-8 encoded) that the command writes on stdout or stderr will be stored in the health status and can be queried with docker inspect. Such output should be kept short (only the first 4096 bytes are stored currently).

When the health status of a container changes, a health_status event is generated with the new status.



18. SHELL

```
SHELL ["executable", "parameters"]
```

SHELL指令允许覆盖默认的shell,在linux中默认的shell是["/bin/sh", "-c"],在windows中是["cmd", "/S", "/C"].在Dockerfile中SHELL指令必须写成JSON格式.
当系统中存在多个shell的时候,SHELL指令变得非常有用
SHELL指令可以出现多次,每个SHELL指令覆盖在它之前的shell,并且作用于它之后的所有指令

```
FROM microsoft/windowsservercore

# Executed as cmd /S /C echo default
RUN echo default

# Executed as cmd /S /C powershell -command Write-Host default
RUN powershell -command Write-Host default

# Executed as powershell -command Write-Host hello
SHELL ["powershell", "-command"]
RUN Write-Host hello

# Executed as cmd /S /C echo hello
SHELL ["cmd", "/S"", "/C"]
RUN echo hello
```
RUN, CMD and ENTRYPOINT 会被SHELL instruction所影响

The following example is a common pattern found on Windows which can be streamlined by using the SHELL instruction:

```
...
RUN powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"
...

```
The command invoked by docker will be:

```
cmd /S /C powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"

```
This is inefficient for two reasons. First, there is an un-necessary cmd.exe command processor (aka shell) being invoked. Second, each RUN instruction in the shell form requires an extra powershell -command prefixing the command.

To make this more efficient, one of two mechanisms can be employed. One is to use the JSON form of the RUN command such as:

```
RUN ["powershell", "-command", "Execute-MyCmdlet", "-param1 \"c:\\foo.txt\""]
```

While the JSON form is unambiguous and does not use the un-necessary cmd.exe, it does require more verbosity through double-quoting and escaping. The alternate mechanism is to use the SHELL instruction and the shell form, making a more natural syntax for Windows users, especially when combined with the escape parser directive:

```
# escape=`

FROM microsoft/nanoserver
SHELL ["powershell","-command"]
RUN New-Item -ItemType Directory C:\Example
ADD Execute-MyCmdlet.ps1 c:\example\
RUN c:\example\Execute-MyCmdlet -sample 'hello world'
```

Resulting in:

```
PS E:\docker\build\shell> docker build -t shell .
Sending build context to Docker daemon 4.096 kB
Step 1/5 : FROM microsoft/nanoserver
 ---> 22738ff49c6d
Step 2/5 : SHELL powershell -command
 ---> Running in 6fcdb6855ae2
 ---> 6331462d4300
Removing intermediate container 6fcdb6855ae2
Step 3/5 : RUN New-Item -ItemType Directory C:\Example
 ---> Running in d0eef8386e97


    Directory: C:\


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
d-----       10/28/2016  11:26 AM                Example


 ---> 3f2fbf1395d9
Removing intermediate container d0eef8386e97
Step 4/5 : ADD Execute-MyCmdlet.ps1 c:\example\
 ---> a955b2621c31
Removing intermediate container b825593d39fc
Step 5/5 : RUN c:\example\Execute-MyCmdlet 'hello world'
 ---> Running in be6d8e63fe75
hello world
 ---> 8e559e9bf424
Removing intermediate container be6d8e63fe75
Successfully built 8e559e9bf424
PS E:\docker\build\shell>
```

The SHELL instruction could also be used to modify the way in which a shell operates. For example, using SHELL cmd /S /C /V:ON|OFF on Windows, delayed environment variable expansion semantics could be modified.

The SHELL instruction can also be used on Linux should an alternate shell be required such as zsh, csh, tcsh and others.

The SHELL feature was added in Docker 1.12.