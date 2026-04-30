# JumpServer KoKo 镜像构建实操记录

## 一、实操目标

本次实操围绕 JumpServer KoKo 组件进行 Docker 镜像构建练习，主要目标如下：

1. 理解 JumpServer KoKo 组件的 Dockerfile 构建流程；
2. 掌握 Docker Buildx 的安装和基本使用；
3. 理解 amd64 / arm64 镜像构建的区别；
4. 在内网服务器环境下尝试完成 KoKo amd64 镜像构建；
5. 分析镜像构建过程中遇到的问题，并总结后续解决方案。

本次实操最终没有完整生成 KoKo 镜像，但已经完成了 Docker 环境修复、Buildx 离线安装、BuildKit 镜像离线导入、基础镜像离线导入、amd64 构建流程验证以及 Dockerfile 构建逻辑分析，具备较完整的排错过程和实操价值。

---

## 二、实验环境

| 项目          | 内容                                  |
| ----------- | ----------------------------------- |
| 操作系统        | Ubuntu 24.04 LTS                    |
| 服务器架构       | x86_64                              |
| Docker 版本   | docker.io 29.1.3                    |
| Buildx 版本   | v0.33.0                             |
| BuildKit 镜像 | moby/buildkit:buildx-stable-1       |
| 构建目标        | linux/amd64                         |
| 构建组件        | JumpServer KoKo                     |
| 源码路径        | /opt/koko                           |
| 网络环境        | 内网环境，无法稳定访问 Docker Hub 和 Debian 官方源 |

服务器架构检查命令：

```bash
uname -m
```

输出结果：

```text
x86_64
```

因此，本次构建目标平台为：

```text
linux/amd64
```

---

## 三、KoKo 组件说明

KoKo 是 JumpServer 的字符协议连接组件，主要负责：

* SSH 连接；
* SFTP 文件传输；
* Telnet 连接；
* Web Terminal；
* 命令记录；
* 字符会话审计。

在 JumpServer 架构中，KoKo 不是 Core 主服务，而是连接组件。用户通过 JumpServer 访问 Linux 资产时，Core 负责权限判断和连接 Token 生成，KoKo 负责真正连接目标服务器，并记录用户的会话和命令操作。

基本流程可以理解为：

```text
用户点击 Web SSH
    ↓
Core 校验用户权限
    ↓
Core 生成连接 Token
    ↓
KoKo 获取连接信息
    ↓
KoKo 连接目标 Linux 资产
    ↓
记录命令和会话审计
```

---

## 四、Docker Buildx 安装过程

### 1. 问题背景

服务器最初安装的是 Ubuntu 源中的 `docker.io`，执行以下命令安装 Buildx 插件时失败：

```bash
apt install docker-buildx-plugin
```

报错信息：

```text
E: Unable to locate package docker-buildx-plugin
```

原因是：

```text
docker-buildx-plugin 通常来自 Docker 官方 apt 源，而不是 Ubuntu 默认 docker.io 源。
```

由于当前服务器处于内网环境，无法正常访问 Docker 官方源，因此无法直接通过官方 apt 源安装 Buildx 插件。

---

### 2. 访问 Docker 官方源失败

尝试下载 Docker 官方 GPG Key：

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

报错：

```text
curl: (35) Recv failure: Connection reset by peer
```

进一步测试发现：

```bash
ping -c 4 download.docker.com
```

可以 ping 通，但执行：

```bash
curl -I https://download.docker.com
```

失败。

说明服务器可以解析域名，也可以进行 ICMP 通信，但 HTTPS 访问被重置或拦截。

结论：

```text
当前内网环境无法直接访问 Docker 官方源。
```

---

### 3. 离线安装 Buildx

由于服务器架构为 `x86_64`，因此需要下载 Linux amd64 版本的 Buildx 二进制文件：

```text
buildx-v0.33.0.linux-amd64
```

将文件上传至服务器 `/tmp` 后，执行：

```bash
mkdir -p /usr/local/lib/docker/cli-plugins

cp /tmp/buildx-v0.33.0.linux-amd64 /usr/local/lib/docker/cli-plugins/docker-buildx

chmod +x /usr/local/lib/docker/cli-plugins/docker-buildx
```

验证 Buildx 是否安装成功：

```bash
docker buildx version
```

输出：

```text
github.com/docker/buildx v0.33.0
```

说明 Buildx 插件安装成功。

---

## 五、Docker daemon 问题处理

### 1. 问题现象

重新安装 `docker.io` 后，Docker 服务启动失败。

执行：

```bash
docker ps
```

报错：

```text
Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
```

查看 Docker 服务日志后，发现关键错误：

```text
failed to load listeners: no sockets found via socket activation: make sure the service was started by systemd
```

原因是 Docker 的 systemd socket activation 出现问题。`dockerd` 使用了：

```text
-H fd://
```

但 systemd 没有正确传递 socket，导致 Docker daemon 启动失败。该错误来自 Docker 服务日志。

---

### 2. 修复方式

通过启用或修复 `docker.socket`，使 Docker daemon 正常启动。

修复后执行：

```bash
docker ps
```

可以正常输出：

```text
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
```

说明 Docker daemon 已经恢复正常。

---

## 六、BuildKit 镜像离线导入

### 1. 问题现象

创建 buildx builder：

```bash
docker buildx create --name multi-builder --use
docker buildx inspect --bootstrap
```

报错：

```text
failed to resolve reference "docker.io/moby/buildkit:buildx-stable-1"
```

原因是 Buildx 创建 `docker-container` 类型 builder 时，需要拉取：

```text
moby/buildkit:buildx-stable-1
```

但当前服务器无法访问 Docker Hub。

---

### 2. 离线导入 BuildKit 镜像

在外网环境中提前拉取并导出：

```bash
docker pull moby/buildkit:buildx-stable-1
docker save moby/buildkit:buildx-stable-1 -o buildkit-buildx-stable-1.tar
```

上传至内网服务器后执行：

```bash
docker load -i /tmp/buildkit-buildx-stable-1.tar
```

验证：

```bash
docker images | grep buildkit
```

可以看到：

```text
moby/buildkit:buildx-stable-1
```

说明 BuildKit 镜像已经导入成功。

---

### 3. 创建 buildx builder

重新创建 builder：

```bash
docker buildx rm multi-builder

docker buildx create --name multi-builder --use

docker buildx inspect --bootstrap
```

最终状态：

```text
Name:          multi-builder
Driver:        docker-container
Status:        running
BuildKit version: v0.29.0
Platforms:     linux/amd64, linux/amd64/v2, linux/amd64/v3, linux/amd64/v4, linux/386
```

说明 BuildKit 容器已经正常运行。

需要注意的是，当前 builder 只支持 amd64 和 386，不支持 arm64。如果后续需要构建 arm64 镜像，还需要额外安装 QEMU/binfmt 支持。

---

## 七、amd64 构建目标说明

本次服务器架构为：

```text
x86_64
```

对应 Docker 平台为：

```text
linux/amd64
```

因此本次实操只进行 amd64 镜像构建。

常见平台对应关系如下：

| 服务器架构   | Docker 平台   |
| ------- | ----------- |
| x86_64  | linux/amd64 |
| aarch64 | linux/arm64 |

如果镜像架构和宿主机架构不一致，容器运行时可能出现：

```text
exec format error
```

---

## 八、KoKo 基础镜像离线导入

### 1. 问题现象

执行 KoKo 镜像构建：

```bash
docker buildx build \
  --platform linux/amd64 \
  -t koko:amd64-test \
  --load .
```

报错显示无法访问 Docker Hub 拉取基础镜像：

```text
failed to resolve source metadata for docker.io/library/debian:trixie
```

KoKo Dockerfile 中需要使用以下基础镜像：

```text
debian:trixie
jumpserver/koko-base:20260422_103200
```

---

### 2. 离线导入基础镜像

在外网环境中执行：

```bash
docker pull debian:trixie
docker pull jumpserver/koko-base:20260422_103200

docker save debian:trixie -o debian-trixie.tar
docker save jumpserver/koko-base:20260422_103200 -o koko-base-20260422_103200.tar
```

上传到内网服务器后执行：

```bash
docker load -i /tmp/debian-trixie.tar
docker load -i /tmp/koko-base-20260422_103200.tar
```

验证：

```bash
docker images
```

可以看到：

```text
debian:trixie
jumpserver/koko-base:20260422_103200
moby/buildkit:buildx-stable-1
```

说明基础镜像已经导入成功。

---

## 九、使用默认 Docker builder 构建

由于 `docker-container` 类型的 buildx builder 不一定直接使用宿主机已有镜像，因此在只构建 amd64 的场景下，切换为普通 Docker 构建更适合当前内网环境。

执行：

```bash
cd /opt/koko

docker build \
  --pull=false \
  -t koko:amd64-test .
```

参数说明：

| 参数                   | 说明             |
| -------------------- | -------------- |
| `--pull=false`       | 不主动从远程仓库拉取基础镜像 |
| `-t koko:amd64-test` | 设置镜像名称和标签      |
| `.`                  | 使用当前目录作为构建上下文  |

---

## 十、最终构建失败点分析

### 1. 构建进度

执行普通 Docker 构建后，基础镜像已经成功识别：

```text
FROM docker.io/jumpserver/koko-base:20260422_103200
FROM docker.io/library/debian:trixie
```

构建过程已经进入 Dockerfile 的运行阶段。

---

### 2. 失败位置

失败发生在 Dockerfile 中类似以下步骤：

```dockerfile
RUN set -ex \
    && sed -i "s@http://.*.debian.org@${APT_MIRROR}@g" /etc/apt/sources.list.d/debian.sources \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && apt-get update \
    && apt-get install -y --no-install-recommends ${DEPENDENCIES} \
    && apt-get clean all \
    && rm -rf /var/lib/apt/lists/*
```

失败原因：

```text
容器内部无法访问 deb.debian.org
```

具体报错包括：

```text
Unable to connect to deb.debian.org:80
Could not connect to deb.debian.org:80
Package 'bash-completion' has no installation candidate
Unable to locate package jq
Unable to locate package redis-tools
```

构建日志显示，`apt-get update` 无法访问 Debian 官方源，导致软件包索引没有下载成功，后续 `apt-get install` 无法找到 `bash-completion`、`jq`、`redis-tools`、`ca-certificates` 等软件包。

本质原因是：

```text
apt-get update 无法访问 Debian 官方源，导致软件包索引没有下载成功，后续 apt-get install 无法找到软件包。
```

---

## 十一、KoKo Dockerfile 内容理解

### 1. 多阶段构建

KoKo Dockerfile 使用了多阶段构建。

第一阶段类似：

```dockerfile
FROM jumpserver/koko-base:20260422_103200 AS stage-build
```

作用：

```text
作为构建环境，负责编译 KoKo 源码和前端资源。
```

这个镜像中通常已经包含 Go、Node、Yarn 等构建依赖。

---

### 2. 构建阶段主要流程

从构建日志可以看到，构建阶段包括：

```text
WORKDIR /opt/koko
COPY . .
WORKDIR /opt/koko/ui
RUN yarn build
```

作用分别是：

| 步骤                     | 作用             |
| ---------------------- | -------------- |
| `WORKDIR /opt/koko`    | 设置源码工作目录       |
| `COPY . .`             | 将 KoKo 源码复制进镜像 |
| `WORKDIR /opt/koko/ui` | 进入前端目录         |
| `RUN yarn build`       | 构建前端资源         |

这一阶段可以理解为：

```text
源码 → 编译 → 生成可运行产物
```

---

### 3. 运行阶段基础镜像

第二阶段使用：

```dockerfile
FROM debian:trixie
```

作用：

```text
作为最终运行环境，减少镜像体积，避免把完整构建工具链带入最终镜像。
```

多阶段构建的核心价值是：

```text
构建阶段依赖多，负责生成产物；
运行阶段依赖少，只负责运行程序。
```

这样可以让最终镜像更小、更干净，也更安全。

---

### 4. TARGETARCH 参数

Dockerfile 中存在：

```dockerfile
ARG TARGETARCH
```

该参数用于接收目标架构。

当执行：

```bash
docker buildx build --platform linux/amd64 ...
```

对应：

```text
TARGETARCH=amd64
```

当执行：

```bash
docker buildx build --platform linux/arm64 ...
```

对应：

```text
TARGETARCH=arm64
```

这说明 KoKo Dockerfile 具备一定的多架构构建能力，可以根据不同 CPU 架构处理不同的构建产物。

---

### 5. APT_MIRROR 参数

Dockerfile 中有类似逻辑：

```dockerfile
sed -i "s@http://.*.debian.org@${APT_MIRROR}@g" /etc/apt/sources.list.d/debian.sources
```

作用是：

```text
将 Debian 默认 apt 源替换为 APT_MIRROR 指定的镜像源。
```

这说明 Dockerfile 已经预留了换源能力。

如果在内网环境中有 Debian trixie 代理源，可以通过构建参数传入：

```bash
docker build \
  --pull=false \
  --build-arg APT_MIRROR=http://内网Debian源地址 \
  -t koko:amd64-test .
```

---

### 6. 时区设置

Dockerfile 中设置：

```dockerfile
ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
```

作用：

```text
将容器时区设置为 Asia/Shanghai。
```

KoKo 作为堡垒机连接组件，会产生会话、命令、审计相关数据，因此时间准确性比较重要。

---

### 7. 运行依赖安装

Dockerfile 中安装的依赖包括：

```text
bash-completion
jq
less
redis-tools
ca-certificates
```

这些包的作用大致如下：

| 软件包               | 作用                      |
| ----------------- | ----------------------- |
| `bash-completion` | Shell 命令补全              |
| `jq`              | 处理 JSON 数据              |
| `less`            | 查看文本或日志                 |
| `redis-tools`     | 提供 redis-cli 等 Redis 工具 |
| `ca-certificates` | HTTPS 证书校验              |

这些并不是 KoKo 的核心业务代码，而是容器运行、脚本执行、排错和通信时需要的基础工具。

---

### 8. 清理缓存

Dockerfile 中有：

```dockerfile
apt-get clean all
rm -rf /var/lib/apt/lists/*
```

作用是：

```text
清理 apt 缓存，减少最终镜像体积。
```

这是 Dockerfile 中常见的镜像优化方式。

---

## 十二、失败原因总结

本次 KoKo 镜像最终未完整构建成功，主要原因不是 Dockerfile 错误，而是内网环境依赖缺失。

当前构建链路中的问题可以分为四类：

| 阶段            | 问题                             | 处理方式                        |
| ------------- | ------------------------------ | --------------------------- |
| 安装 Buildx     | apt 源没有 `docker-buildx-plugin` | 离线安装 Buildx 二进制             |
| 创建 builder    | 无法拉取 `moby/buildkit` 镜像        | 离线导入 BuildKit 镜像            |
| 构建 KoKo 镜像    | 无法访问 Docker Hub 拉基础镜像          | 离线导入基础镜像                    |
| Dockerfile 执行 | 容器无法访问 `deb.debian.org`        | 需要配置 Debian trixie 内网 apt 源 |

最终卡点：

```text
容器构建阶段 apt-get update 无法访问 Debian 官方源。
```

---

## 十三、后续解决方案

如果后续继续完成 KoKo 镜像构建，可以考虑以下方案。

### 方案一：配置 Debian trixie 内网 apt 源

向内网仓库管理员确认是否存在 Debian trixie 代理源，例如：

```text
http://10.1.13.115:8081/repository/debian-proxy
```

然后执行：

```bash
docker build \
  --pull=false \
  --build-arg APT_MIRROR=http://10.1.13.115:8081/repository/debian-proxy \
  -t koko:amd64-test .
```

注意：宿主机当前使用的是 Ubuntu noble 源，不能直接作为 Debian trixie 容器的 apt 源。

---

### 方案二：提前制作带依赖的 Debian 基础镜像

在外网环境中基于 `debian:trixie` 安装好所需依赖：

```text
bash-completion
jq
less
redis-tools
ca-certificates
```

然后打包成内部基础镜像，在内网直接使用。

---

### 方案三：使用内网制品库统一管理基础镜像

将以下镜像推送到内网 Harbor / Nexus / Registry：

```text
debian:trixie
jumpserver/koko-base:20260422_103200
moby/buildkit:buildx-stable-1
```

之后构建时统一从内网镜像仓库拉取，避免每次手动 `docker save` / `docker load`。

---

## 十四、本次实操收获

通过本次实操，掌握了以下内容：

1. Ubuntu 源中的 `docker.io` 和 Docker 官方源中的 `docker-ce` 在插件支持上存在差异；
2. `docker-buildx-plugin` 无法安装时，可以通过二进制方式离线安装；
3. Buildx 的 `docker-container` driver 依赖 `moby/buildkit` 镜像；
4. 内网环境下构建镜像时，需要提前准备基础镜像；
5. 多阶段 Dockerfile 通常分为构建阶段和运行阶段；
6. KoKo Dockerfile 使用 `jumpserver/koko-base` 作为构建环境，使用 `debian:trixie` 作为最终运行环境；
7. 本机 amd64 构建可以优先使用普通 `docker build`，不一定必须使用 buildx；
8. 当前构建失败的根因是容器内部无法访问 Debian apt 源；
9. 运维排错不能只看最后一行报错，需要判断失败发生在镜像拉取、构建器初始化，还是 Dockerfile 执行阶段。

---

## 十五、总结

本次 JumpServer KoKo 镜像构建实操虽然没有最终生成完整镜像，但完整经历了内网环境下 Docker 镜像构建的典型问题，包括 Docker 插件缺失、Docker daemon 启动异常、BuildKit 镜像拉取失败、基础镜像离线导入以及容器内部 apt 源不可达等问题。

最终定位到的核心阻塞点是：

```text
KoKo Dockerfile 在运行阶段需要执行 apt-get update 和 apt-get install，但当前内网服务器无法访问 deb.debian.org，且暂未配置 Debian trixie 内网 apt 源。
```