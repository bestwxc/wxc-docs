# 深入了解docker build

https://docs.docker.com/build/

### 1. 安装buildx
当前版本包中包含buildx，docker build 是默认docker buildx 的别名。

### 2. 构建程序
使用Dockerfile构建

1. context
指的是构建的上下文，即构建路径。除传统的路径外，还支持url, git, tar包，文件等。
2. 多阶段构建
    - 阶段可以命名，也可以不命名，不命名的情况下，阶段可以通过序号引用
    - 可以引用外部阶段，将镜像名作为阶段名
    - 可以命名阶段名
    - 前面的阶段名可以作为后续的基础镜像，即可以 FROM 阶段名
    - 使用--target可以指定构建阶段
3. 构建多架构镜像
    默认的build不支持多架构，需要安装模拟器。新版本应该是官方提供了
    https://docs.docker.com/build/building/multi-platform/
   主要疑问，为什么自带的也可以构建
### 3. builder driver
1. 类型

|类型|解释|自动加载镜像|导出缓存|导出tarball|多架构镜像|buildkit配置|
|----|----|----|----|----|----|----|
|docker|docker二进制包中的|✓|inline| | | |
|docker-container|创建一个容器运行工具| |✓|✓|✓|✓|
|kubernetes|在k8s中运行| |✓|✓|✓|✓|
|remote|连接远程的工具| |✓|✓|✓|Managed externally|

2. docker类型驱动
默认，无需配置
   
3. docker-container

```shell
docker buildx create \
  --name container \
  --driver=docker-container \
  --driver-opt=[key=value,...]
```

- 参数 
  
|参数|类型|默认|说明|
|----|----|----|----|
|image|String| |设置镜像|
|network|String| |设置网路|
|cgroup-parent|String|/docker/buildx|设置cgroup|
|env.<key>|String| |设置环境变量|

- 将镜像加载到本地
```shell
 docker buildx build --load -t <image> --builder=container .
```

- 持久化缓存
```shell
docker buildx create --name=container --driver=docker-container --use --bootstrap
docker buildx ls
# 查看卷
docker volume ls
docker buildx rm --keep-state container
# 与之前的卷相同
docker volume ls
```

- QEME
支持构建非当前系统架构的镜像
  
- 指定网路

4. k8s驱动
   https://docs.docker.com/build/drivers/kubernetes/
   
5. 远程驱动
略
   
### 4. 构建缓存
对于镜像的操作将作为最终的镜像的一个层， 可以把镜像看作一个栈，每次构建就是在上面增加内容。

所以当构建某个步骤发生改变时，之前的步骤可以复用缓存，后面的步骤都需要重新执行。 

#### 优化缓存的方式
1. 优化层的顺序，如果没有先后依赖性， 让有变化的层在后面。
2. 尽可能让镜像的每一层都变得小，镜像中不要包含不必要的文件，对于包管理工具安装的内容， 如非必要，尽可能不安装他们。 
3. 使用RUN指令专用的缓存，如

  ``` text
    RUN \
      --mount=type=cache,target=/var/cache/apt \
      apt-get update && apt-get install -y git
  ```

#### 尽可能减少镜像的层数
1. 使用适当的基础镜像
2. 合理利用多阶段构建
3. 尽可能将多个命令组合在一起

#### 垃圾回收
配置方式
对于docker,在docker服务配置中；对于其他类型的driver，在buildkit配置中配置

``` json
{
  "builder": {
    "gc": {
      "enabled": true,
      "defaultKeepStorage": "10GB",
      "policy": [
          {"keepStorage": "10GB", "filter": ["unused-for=2200h"]},
          {"keepStorage": "50GB", "filter": ["unused-for=3300h"]},
          {"keepStorage": "100GB", "all": true}
      ]
    }
  }
}
```

``` toml
[worker.oci]
  gc = true
  gckeepstorage = 10000
  [[worker.oci.gcpolicy]]
    keepBytes = 512000000
    keepDuration = 172800
    filters = [ "type==source.local", "type==exec.cachemount", "type==source.git.checkout"]
  [[worker.oci.gcpolicy]]
    all = true
    keepBytes = 1024000000
```

默认回收策略；
```
GC Policy rule#0:
        All:            false
        Filters:        type==source.local,type==exec.cachemount,type==source.git.checkout
        Keep Duration:  48h0m0s
        Keep Bytes:     512MB
GC Policy rule#1:
        All:            false
        Keep Duration:  1440h0m0s
        Keep Bytes:     26GB
GC Policy rule#2:
        All:            false
        Keep Bytes:     26GB
GC Policy rule#3:
        All:            true
        Keep Bytes:     26GB
```

#### 缓存实现（backend）
inline registry local gha s3 azblob

##### 格式
``` bash
docker buildx build --push -t <registry>/<image> \
  --cache-to type=registry,ref=<registry>/<cache-image>[,parameters...] \
  --cache-from type=registry,ref=<registry>/<cache-image>[,parameters...] .
```
##### 使用多个缓存
目前只支持到处一个缓存，但是可以从多个来源导入缓存。
``` bash
 docker buildx build --push -t <registry>/<image> \
  --cache-to type=registry,ref=<registry>/<cache-image>:<branch> \
  --cache-from type=registry,ref=<registry>/<cache-image>:<branch> \
  --cache-from type=registry,ref=<registry>/<cache-image>:main .
```

##### 配置选项
1. 缓存模式
导出缓存时时候可以选择mode选项， 可以选择min和max，支持除inline以外的实现。
2. 缓存压缩
支持local和registry实现。示例：
``` shell
 docker buildx build --push -t <registry>/<image> \
  --cache-to type=registry,ref=<registry>/<cache-image>,compression=zstd \
  --cache-from type=registry,ref=<registry>/<cache-image> .
```
3. 支持oci媒体类型，支持Local和registry实现。
``` shell
docker buildx build --push -t <registry>/<image> \
  --cache-to type=registry,ref=<registry>/<cache-image>,oci-mediatypes=true \
  --cache-from type=registry,ref=<registry>/<cache-image> .
```

具体配置方式见连接

https://docs.docker.com/build/cache/backends/


### 5. 导出器
#### 总览
1. 包含
- image
- registry
- local
- tar
- oci
- docker 
- cacheonly

2. 使用方式
``` shell
 docker buildx build --tag <registry>/<image> \
  --output type=<TYPE> .
```
3. 使用示例
``` shell
# 导出到本地存储
docker buildx build \
  --output type=docker,name=<registry>/<image> .
# 导出
docker buildx build --tag <registry>/<image> --load .
# 推送到仓库
 docker buildx build \
  --output type=image,name=<registry>/<image>,push=true .
# 推送到仓库
docker buildx build \
  --output type=registry,name=<registry>/<image> .
# 变成文件
docker buildx build --output type=oci,dest=./image.tar .
# 到文件系统
docker buildx build --output type=tar,dest=<path/to/output> .
# cacheonly
docker buildx build --output type=cacheonly
``` 

4. 类型配置见文档
https://docs.docker.com/build/exporters/

### 5. 持续集成
两种方式
1. 位于当前环境中
2. 运行在容器内，即dind(docker in docker) 

### 6. bake高级构建

### 7. 留痕（Attestations）
用于在镜像中写入构建信息，以便了解镜像的来源。
该方式并不能防伪，所以不能简单的认为是签名或者说证书。 

### 8. buildkit
改进版的builder
1. 配置仓库

``` toml
debug = true
[registry."docker.io"]
  mirrors = ["mirror.gcr.io"]
```
``` shell
docker buildx create --use --bootstrap \
  --name mybuilder \
  --driver docker-container \
  --config /etc/buildkitd.toml
```

2. 设置仓库证书
``` toml
# /etc/buildkitd.toml
debug = true
[registry."myregistry.com"]
  ca=["/etc/certs/myregistry.pem"]
  [[registry."myregistry.com".keypair]]
    key="/etc/certs/myregistry_key.pem"
    cert="/etc/certs/myregistry_cert.pem"
```

``` shell
# 创建builder
docker buildx create --use --bootstrap \
  --name mybuilder \
  --driver docker-container \
  --config /etc/buildkitd.toml
# 查看配置
docker exec -it buildx_buildkit_mybuilder0 cat /etc/buildkit/buildkitd.toml
# 查看证书
docker exec -it buildx_buildkit_mybuilder0 ls /etc/buildkit/certs/myregistry.com/
# 推送镜像
docker buildx build --push --tag myregistry.com/myimage:latest .
```

3. 配置cni 网路
构建cni网络支持
``` dockerfile
# syntax=docker/dockerfile:1

ARG BUILDKIT_VERSION=v{{ site.buildkit_version }}
ARG CNI_VERSION=v1.0.1

FROM --platform=$BUILDPLATFORM alpine AS cni-plugins
RUN apk add --no-cache curl
ARG CNI_VERSION
ARG TARGETOS
ARG TARGETARCH
WORKDIR /opt/cni/bin
RUN curl -Ls https://github.com/containernetworking/plugins/releases/download/$CNI_VERSION/cni-plugins-$TARGETOS-$TARGETARCH-$CNI_VERSION.tgz | tar xzv

FROM moby/buildkit:${BUILDKIT_VERSION}
ARG BUILDKIT_VERSION
RUN apk add --no-cache iptables
COPY --from=cni-plugins /opt/cni/bin /opt/cni/bin
ADD https://raw.githubusercontent.com/moby/buildkit/${BUILDKIT_VERSION}/hack/fixtures/cni.json /etc/buildkit/cni.json
```

``` shell
docker buildx build --tag buildkit-cni:local --load .
 docker buildx create --use --bootstrap \
  --name mybuilder \
  --driver docker-container \
  --driver-opt "image=buildkit-cni:local" \
  --buildkitd-flags "--oci-worker-net=cni"
```

4. 资源限制
``` toml
# /etc/buildkitd.toml
[worker.oci]
  max-parallelism = 4
```

6. TCP连接数限制

6. 完整的配置
``` toml
debug = true
# root is where all buildkit state is stored.
root = "/var/lib/buildkit"
# insecure-entitlements allows insecure entitlements, disabled by default.
insecure-entitlements = [ "network.host", "security.insecure" ]

[grpc]
  address = [ "tcp://0.0.0.0:1234" ]
  # debugAddress is address for attaching go profiles and debuggers.
  debugAddress = "0.0.0.0:6060"
  uid = 0
  gid = 0
  [grpc.tls]
    cert = "/etc/buildkit/tls.crt"
    key = "/etc/buildkit/tls.key"
    ca = "/etc/buildkit/tlsca.crt"

# config for build history API that stores information about completed build commands
[history]
  # maxAge is the maximum age of history entries to keep, in seconds.
  maxAge = 172800
  # maxEntries is the maximum number of history entries to keep.
  maxEntries = 50

[worker.oci]
  enabled = true
  # platforms is manually configure platforms, detected automatically if unset.
  platforms = [ "linux/amd64", "linux/arm64" ]
  snapshotter = "auto" # overlayfs or native, default value is "auto".
  rootless = false # see docs/rootless.md for the details on rootless mode.
  # Whether run subprocesses in main pid namespace or not, this is useful for
  # running rootless buildkit inside a container.
  noProcessSandbox = false
  gc = true
  gckeepstorage = 9000
  # alternate OCI worker binary name(example 'crun'), by default either 
  # buildkit-runc or runc binary is used
  binary = ""
  # name of the apparmor profile that should be used to constrain build containers.
  # the profile should already be loaded (by a higher level system) before creating a worker.
  apparmor-profile = ""
  # limit the number of parallel build steps that can run at the same time
  max-parallelism = 4
  # maintain a pool of reusable CNI network namespaces to amortize the overhead
  # of allocating and releasing the namespaces
  cniPoolSize = 16

  [worker.oci.labels]
    "foo" = "bar"

  [[worker.oci.gcpolicy]]
    keepBytes = 512000000
    keepDuration = 172800
    filters = [ "type==source.local", "type==exec.cachemount", "type==source.git.checkout"]
  [[worker.oci.gcpolicy]]
    all = true
    keepBytes = 1024000000

[worker.containerd]
  address = "/run/containerd/containerd.sock"
  enabled = true
  platforms = [ "linux/amd64", "linux/arm64" ]
  namespace = "buildkit"
  gc = true
  # gckeepstorage sets storage limit for default gc profile, in MB.
  gckeepstorage = 9000
  # maintain a pool of reusable CNI network namespaces to amortize the overhead
  # of allocating and releasing the namespaces
  cniPoolSize = 16

  [worker.containerd.labels]
    "foo" = "bar"

  [[worker.containerd.gcpolicy]]
    keepBytes = 512000000
    keepDuration = 172800 # in seconds
    filters = [ "type==source.local", "type==exec.cachemount", "type==source.git.checkout"]
  [[worker.containerd.gcpolicy]]
    all = true
    keepBytes = 1024000000

# registry configures a new Docker register used for cache import or output.
[registry."docker.io"]
  # mirror configuration to handle path in case a mirror registry requires a /project path rather than just a host:port
  mirrors = ["yourmirror.local:5000", "core.harbor.domain/proxy.docker.io"]
  http = true
  insecure = true
  ca=["/etc/config/myca.pem"]
  [[registry."docker.io".keypair]]
    key="/etc/config/key.pem"
    cert="/etc/config/cert.pem"

# optionally mirror configuration can be done by defining it as a registry.
[registry."yourmirror.local:5000"]
  http = true
```


