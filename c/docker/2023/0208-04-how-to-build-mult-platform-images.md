# 使用docker构建多平台的镜像
一、使用buildx来构建不同架构的示例

1. 创建dockerfile

``` Dockerfile
FROM alpine
RUN apk --no-cache add curl
```

2. 创建builder并使用builder构建

``` shell
sudo docker buildx create --name mybuilder
sudo docker buildx use mybuilder
sudo docker buildx build --platform linux/amd64 -t alpine-amd64 --load .
sudo docker buildx build --platform linux/arm64 -t alpine-arm64 --load .
sudo docker buildx build --platform linux/arm/v7 -t alpine-arm32 --load .
```

3. 运行镜像测试
``` shell
# 在不同的机器上运行
sudo docker run alpine-amd64 uname -a
sudo docker run alpine-arm64 uname -a
sudo docker run alpine-arm32 uname -a
```

4. 如果基础镜像支持多平台， 则可以用单个buildx来构建
``` shell
# 构建并推送至仓库，默认至中心仓库
sudo docker buildx build --platform=linux/amd64,linux/arm64,linux/arm/v7 -t wxc252/alpine-test --push .
# 查看镜像信息
sudo docker buildx imagetools inspect wxc252/alpine-test
```

