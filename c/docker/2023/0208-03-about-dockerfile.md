# 关于Dockerfile

## 格式

Dockerfile语法格式如下
```text
# 注释（Comment）
指令（INSTRUCTION） 参数（arguments）
```
其中指令（INSTRUCTION）大小写不敏感，但是通常大写来与后面的参数区分

## 全局参数
Dockerfile中第一条参数之前ARG指定的参，特点：
1. 能被FROM使用
2. 可以指定多个
3. 不能被FROM后的指令解析

## 解析指示（directive）
翻译成指示，是因为他是用作干涉解析器解析Dockerfile的，并不是命令。 

格式
```text
# directive=value1
```

特点：
1. 必须在文件最开始的位置
2. 不可以分行，但是可以有多余的空格
3. 相同指示不能使用多次
4. 对于无效的指示，当作普通的注释
5. 如果一条有效指示前面有一条无效指示，无效指示会当作注释，第二条指示由于前面有注释,也会被当作注释

#### 支持的指示
1. syntax

    在BuildKit中使用，传统的builder中将被忽略
2. escape

    用于设置换行字符，默认"\"

    ```text
    # escape=\ (backslash)
    # or
    # escape=` (backtick)
    ```

## 环境变量替换
1. 使用ENV指令指定
2. 可以使用下列形式引用，$variable_name${variable_name}${foo}_bar
3. 支持下列特殊写法
  - ${variable:-word} : 如果未指定，则结果为word，反之为指定的值
  - ${variable:+word} : 如果指定，结果为world,反之为空
4. 可以在$前加反斜杠进行转义
5. 支持的指令
    - ADD
    - COPY
    - ENV
    - EXPOSE
    - FROM
    - LABEL
    - STOPSIGNAL
    - USER
    - VOLUME
    - WORKDIR
    - ONBUILD 与上述结合使用

## .dockerignore file

## FROM 指令
```
FROM [--platform=<platform>] <image> [AS <name>]
FROM [--platform=<platform>] <image>[:<tag>] [AS <name>]
FROM [--platform=<platform>] <image>[@<digest>] [AS <name>]
```

每个Dockerfile必须包含至少一个构建阶段（stage），每个构建阶段从FROM指令开始。

注意： 大多数说法中，Dockerfile必须由FROM指令开始是不严谨的（可能早期docker版本是严谨的），FROM指令前面可以有注释、解析指令和ARG指令。

FROM 指令只能解析全局参数，不能解析其他参数。（全局参数: 第一个FROM之前定义的ARG都是全局ARG参数）

同一Dockerfile中，可以有多个FROM指令

## RUN 指令
两种形式
```
RUN <command>
RUN ["executable", "param1", "param2"]
```

特殊：
1. RUN --mount=[type=<TYPE>,][opt1=<value>,]...[optn=<valuen>]
type：
    - bind: default
    - cache
    - secret
    - ssh

2. RUN --network=<TYPE>

3. RUN --security
    - RUN --security=insecure  相当于 docker run --privileged
    - RUN --security=sandbox (默认)

## CMD 指令
三种形式
```
CMD ["executable","param1","param2"]
CMD ["param1","param2"] 
CMD command param1 param2
```

其中第二种情况比较特殊，CMD是作为ENTRYPOINT的参数

## LABEL 指令
LABEL <key>=<value> <key>=<value> <key>=<value> ...

## MAINTAINER 指令
MAINTAINER <name> 
被遗弃，替代方式
LABEL org.opencontainers.image.authors="SvenDowideit@home.org.au"

## EXPOSE 指令
指定暴露的端口
EXPOSE <port> [<port>/<protocol>...]

## ENV 指令
ENV <key>=<value> ...
允许一次设置多个。 

ENV会保留到镜像中，如果不需要保留镜像，可以考虑：
1. 为单个RUN设置
RUN DEBIAN_FRONTEND=noninteractive apt-get update && apt-get install -y ...
2. 使用ARG
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y ...

## ADD 指令
ADD [--chown=<user>:<group>] [--checksum=<checksum>] <src>... <dest>
ADD [--chown=<user>:<group>] ["<src>",... "<dest>"]

src 支持http地址和git地址

## ADD --link 见COPY --link

## COPY 指令
COPY [--chown=<user>:<group>] <src>... <dest>
COPY [--chown=<user>:<group>] ["<src>",... "<dest>"]

## COPY --link

## ENTRYPOINT 指令
ENTRYPOINT ["executable", "param1", "param2"]
ENTRYPOINT command param1 param2

## VOLUME 指令
指定卷

## USER 指令
指定用户

## WORKDIR
指定工作目录

## ARG 指令
ARG <name>[=<default value>]

### Predefined ARGs
  - HTTP_PROXY
  - http_proxy
  - HTTPS_PROXY
  - https_proxy
  - FTP_PROXY
  - ftp_proxy
  - NO_PROXY
  - no_proxy
  - ALL_PROXY
  - all_proxy

### Automatic platform ARGs in the global scope
  - TARGETPLATFORM - platform of the build result. Eg , , .linuxamd64linux/arm/v7windows/amd64
  - TARGETOS - OS component of TARGETPLATFORM
  - TARGETARCH - architecture component of TARGETPLATFORM
  - TARGETVARIANT - variant component of TARGETPLATFORM
  - BUILDPLATFORM - platform of the node performing the build.
  - BUILDOS - OS component of BUILDPLATFORM
  - BUILDARCH - architecture component of BUILDPLATFORM
  - BUILDVARIANT - variant component of BUILDPLATFORM

### BuildKit built-in build args
ARG                                   TYPE      DESC
BUILDKIT_CACHE_MOUNT_NS	              String	Set optional cache ID namespace.
BUILDKIT_CONTEXT_KEEP_GIT_DIR	      Bool	    Trigger git context to keep the .git directory.
BUILDKIT_INLINE_BUILDINFO_ATTRS2	  Bool	    Inline build info attributes in image config or not.
BUILDKIT_INLINE_CACHE2	              Bool	    Inline cache metadata to image config or not.
BUILDKIT_MULTI_PLATFORM	              Bool	    Opt into determnistic output regardless of multi-platform output or not.
BUILDKIT_SANDBOX_HOSTNAME	          String	Set the hostname (default buildkitsandbox)
BUILDKIT_SYNTAX	                      String	Set frontend image
SOURCE_DATE_EPOCH	                  Int	    Set the UNIX timestamp for created image and layers. More info from reproducible builds. Supported since Dockerfile 1.5, BuildKit 0.11

### Impact on build caching 对缓存的影响
值改变时，不能再命中缓存

## ONBUILD 指令
ONBUILD <INSTRUCTION>

当前构建不会执行，子镜像构建时会执行。
可以精简子项目的Dockerfile, 实现子项目只需要一个FROM指令

## STOPSIGNAL 指令
STOPSIGNAL signal

## HEALTHCHECK 指令
健康检查指令
HEALTHCHECK [OPTIONS] CMD command
HEALTHCHECK NONE

### 参数
--interval=DURATION (default: 30s)
--timeout=DURATION (default: 30s)
--start-period=DURATION (default: 0s)
--retries=N (default: 3)

## SHELL 指令
SHELL ["executable", "parameters"]

指定shell

## Here-Documents
```
# syntax=docker/dockerfile:1
FROM debian
RUN <<EOT bash
  apt-get update
  apt-get install -y vim
EOT
```

```
# syntax=docker/dockerfile:1
FROM python:3.6
RUN <<EOT
#!/usr/bin/env python
print("hello world")
EOT
```

```
FROM alpine
RUN <<FILE1 cat > file1 && <<FILE2 cat > file2
I am
first
FILE1
I am
second
FILE2
```

```
FROM alpine
COPY <<-"EOT" /app/script.sh
	echo hello ${FOO}
EOT
RUN FOO=abc ash /app/script.sh
```


