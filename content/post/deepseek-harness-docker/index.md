---
title: "把 DeepSeek Harness 装进个人服务器的 Docker 里"
slug: "deepseek-harness-docker"
description: "从装 Docker、拉源码，到写 Dockerfile 和 compose，把 DSH 跑在自己的小主机上，再用 nginx 加一层密码。"
image: "cover.png"
date: 2026-11-25
tags:
    - Docker
    - 自建服务
    - DeepSeek
categories:
    - 技术
---

前阵子把 DeepSeek Harness（下面简称 DSH）搬到了家里那台小主机上。它本来可以直接 `pnpm` 跑在宿主机上，但我还是选择塞进 Docker。

这篇把整个过程完整记一遍：为什么用 Docker、怎么装、源码怎么拉、Dockerfile 和 compose 怎么写、启动时要注意什么，以及以后怎么更新。

<!--more-->

## 一、装进 Docker 的好处

**1. 兼顾轻便与良好的隔离性。** 容器里可以放心开“完全模式”，搞坏了也不慌：一条命令删掉旧实例，再一条命令创建新的，配置和数据都留在挂载目录里，宿主机不受影响。

**2. 服务器可以 24 小时不关机。** 需要长时间运行的任务，放在小主机上比一直开着主力电脑更省电，也更省心。

**3. 不污染宿主机。** 主程序、各种插件和对应的系统依赖全部打包进镜像，宿主机始终保持干净。

**4. 迁移方便。** 镜像和 compose 文件拷到新机器上马上就能跑，不会出现“A 机能跑、B 机不行”的情况。

**5. 回滚成本低。** 每次重新构建都用不同的镜像 tag；新版本用出问题，compose 里改一行 tag、重新 `up -d`，就回到旧版本。直接装在宿主机上可没这么干脆。

**6. 和别的服务好相处。** 我这台机器上还跑着几个容器，也可以跑小龙虾之类的其他 agent 程序，不用担心彼此冲突。

## 二、准备环境

一台能当个人服务器用的小主机就行：N100 之类的四核小机器，4GB 内存、32GB 以上的 SSD，装 Debian 13，能 SSH 登录、有 sudo 权限，就够跑了。

如果你打算让它处理音视频，最好选带核显的机器——后面可以把 `/dev/dri/renderD128` 直通给容器，ffmpeg 转码会快很多。没有核显也能转码，只是会慢不少。

Debian 13 的安装过程这里就不展开了，装好系统、能联网、能 sudo 就行。

## 三、用官方 apt 仓库安装 Docker

Debian 自己的源里有个 `docker.io`，但版本偏旧，而且不带 compose 插件。更推荐加 Docker 官方的 apt 仓库：

```bash
# 1. 基础工具
sudo apt update
sudo apt install -y ca-certificates curl

# 2. 添加官方 GPG 公钥
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# 3. 添加软件源（Debian 13 的代号是 trixie）
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
https://download.docker.com/linux/debian trixie stable" | \
sudo tee /etc/apt/sources.list.d/docker.list

# 4. 安装
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# 5. 开机自启
sudo systemctl enable --now docker
```

装完跑一句验证：

```bash
sudo docker run --rm hello-world
```

看到那句 “Hello from Docker!” 就成了。

**6. 可选但推荐：把当前用户加入 docker 组**，这样以后运行 docker 命令就不用每次都加 sudo：

```bash
# 创建 docker 组（一般已存在，提示已存在可忽略）
sudo groupadd docker

# 把当前用户加入 docker 组
sudo usermod -aG docker $USER

# 让当前 shell 立即生效；或者注销重新登录
newgrp docker

# 验证：不再需要 sudo 也能跑
docker run --rm hello-world
```

需要提醒的是，**docker 组的权限基本等同于 root**（能挂载宿主机目录、能操作所有容器），只把自己信任的账号加进去。

这种方式的好处：版本跟官方同步、升级走 `apt upgrade` 就行；仓库有签名校验；`docker compose` 和 `docker buildx` 作为插件一起装好，不用再单独折腾 compose 的二进制文件；哪天不想用了，`apt purge` 删得也干净。

## 四、拉取源码

先创建一个目录，专门存放容器相关的文件：

```bash
mkdir deepseek-harness-docker
cd deepseek-harness-docker

git clone https://github.com/deepseek-ai/deepseek-harness.git
```

这会把 `deepseek-harness` 源码项目克隆到当前目录下。执行完，目录结构应该是：

```text
deepseek-harness-docker/
└── deepseek-harness/     # 源码
```

目录名可以随便起，比如 `mkdir aaa`，自己记得住就行。

## 五、写 Dockerfile，把源码变成镜像

在 `deepseek-harness-docker` 目录下创建名为 `Dockerfile` 的文件：

```bash
vi Dockerfile
```

文件名必须是 `Dockerfile`，否则 docker 引擎不认识。

写入以下内容：

```dockerfile
# 基础镜像：Debian 12 + Node.js 22，自带 corepack
FROM node:22-bookworm

# 复制源码
COPY ./deepseek-harness /deepseek-harness

# 把镜像里的工作目录切换到 /deepseek-harness
WORKDIR /deepseek-harness

# 安装所需的系统依赖
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
    bubblewrap \
    ffmpeg \
    i965-va-driver \
    vainfo \
    build-essential \
    python3 \
    python3-dev \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# 启用 Corepack，使用仓库锁定的 pnpm 版本
RUN corepack enable

# 安装项目依赖
RUN pnpm install --frozen-lockfile

# 编译
RUN pnpm run build

# 声明 Web UI 监听的端口（只是声明，映射在 compose 里做）
EXPOSE 3080

# 启动 dsh web；--no-open 避免容器里尝试打开浏览器
CMD ["pnpm", "dsh", "web", "--no-open"]
```

逐行解释一下：

- `FROM node:22-bookworm`：指定基础镜像。**Dockerfile 必须以 `FROM` 开头**，否则构建直接报错。这里选 Debian 12 的 Node 22 镜像，是因为后面要用 `apt-get` 装系统依赖；Node 镜像自带 corepack，正好用来锁 pnpm 版本。
- `COPY ./deepseek-harness /deepseek-harness`：把构建上下文里的 `deepseek-harness` 文件夹复制到镜像根目录下的 `/deepseek-harness`。
- `WORKDIR /deepseek-harness`：把镜像里的工作目录切过去，后面的 `RUN`、`CMD` 都相当于在这个目录下执行。
- `RUN apt-get update && apt-get install ...`：装系统依赖。
  - `bubblewrap`：给工具执行套一层轻量沙箱；
  - `ffmpeg`：音视频的剪辑、转码、抽帧都靠它；
  - `i965-va-driver`、`vainfo`：Intel 核显的 VA-API 驱动和检测工具，配合后面的设备直通做硬解；
  - `build-essential`：编译原生模块要用的 gcc、make；
  - `python3`、`python3-dev`：部分工具链和原生扩展需要；
  - `git`、`ca-certificates`：拉依赖、校验 HTTPS 证书；
  - `--no-install-recommends` 不装推荐包，`rm -rf /var/lib/apt/lists/*` 清理 apt 缓存，都是为了让镜像小一点。
- `RUN corepack enable`：打开 Node 自带的包管理器代理，让它按 `package.json` 里声明的 `packageManager` 使用对应版本的 pnpm。
- `RUN pnpm install --frozen-lockfile`：严格按 `pnpm-lock.yaml` 安装依赖。lock 文件和 `package.json` 对不上会直接报错，换来的是每次都一致的依赖。
- `RUN pnpm run build`：执行项目定义的构建脚本，把前端和服务端编译成可运行的产物。
- `EXPOSE 3080`：声明容器里的 Web 服务监听 3080 端口，只是说明性信息，去掉也不影响运行。
- `CMD ["pnpm", "dsh", "web", "--no-open"]`：容器启动时执行的命令。写成 JSON 数组（exec 形式），`dsh` 会成为容器里的 1 号进程，能正确收到停止信号；`--no-open` 是因为容器里没有浏览器。

注意：**Dockerfile 的注释必须单独占一行**。如果写成 `COPY ./deepseek-harness /deepseek-harness # 复制源码`，`#` 后面的内容会被当成参数，构建直接失败。

构建镜像：

```bash
docker build -t deepseek-harness:1.1.1 .
```

- `-t` 是打标签：`deepseek-harness` 是镜像名，`1.1.1` 是版本号。建议固定版本号，别用 `latest`，回滚的时候会感谢自己。
- 最后那个 `.` 表示以当前目录作为构建上下文，因为源码文件夹和 Dockerfile 都在这里。

## 六、写 compose 文件

先用一条命令创建一个 bridge 网络：

```bash
docker network create --subnet=172.24.0.0/16 aigent-a
```

`--subnet=172.24.0.0/16` 指定网段，不写的话 docker 会自动分配。网络名也可以换，自己记得就行。

然后在 `deepseek-harness-docker` 目录下创建 `compose.yml`：

```bash
vi compose.yml
```

写入：

```yaml
services:

  # 服务名：搭配 nginx 使用
  nginx-dsh:
    # 镜像名
    image: 'nginx:alpine'
    # 实例的名字
    container_name: nginx-dsh
    # 是否自动重启
    restart: unless-stopped
    # 环境变量
    environment:
      TZ: "Asia/Shanghai"   # 容器内使用上海时区
    # 加入 aigent-a 网络
    networks:
      - aigent-a
    # 把容器里的 30800 端口映射到宿主机 30800（DNAT）
    ports:
      - "30800:30800"
    # 挂载 nginx 配置与密码文件（:ro 表示只读）
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./htpasswd:/etc/nginx/.htpasswd:ro
    # 大意是：deepseek-harness 这个容器启动了，再启动本容器
    depends_on:
      - deepseek-harness

  deepseek-harness:
    container_name: deepseek-harness
    image: 'deepseek-harness:1.1.1'
    restart: unless-stopped
    # 直通核显，供 ffmpeg 硬解
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    environment:
      TZ: "Asia/Shanghai"
    networks:
      aigent-a:
        ipv4_address: 172.24.0.10
    volumes:
      - ./dsh:/root/.dsh

# 声明 aigent-a 是已经存在的网络，创建容器时不会重复创建
networks:
  aigent-a:
    external: true
```

几个关键点：

- `networks` 里给 `deepseek-harness` 指定了固定 IP `172.24.0.10`，nginx 反代时直接写这个地址即可。如果创建网络时没加 `--subnet=172.24.0.0/16`，网段不一定是 `172.24.0.0/16`，这个地址就要跟着改。
- `ports` 只映射 nginx 的 30800，**DSH 的 3080 不对外**，外部只能通过加了密码的入口访问。
- `depends_on` 只管启动顺序，不代表 DSH 真的已经就绪。

### 关于 volumes：为什么数据挂载出来就丢不了

容器销毁后，它运行时产生的文件会一起消失。所以需要用 `volumes` 把要保留的数据映射到宿主机目录上，容器重建后还能接着用。

DSH 启动后会在当前用户的主目录下创建 `.dsh` 文件夹，用来放会话缓存、插件文件等。我们的镜像没有切换用户，基础镜像默认以 root 运行，所以它对应的是容器里的 `/root/.dsh`。

因此 compose 里写了：

```yaml
    volumes:
      - ./dsh:/root/.dsh
```

意思是把宿主机上的 `./dsh` 挂到容器的 `/root/.dsh`。这样只要备份宿主机的 `./dsh` 文件夹，就相当于备份了 DSH 的会话和插件数据。

有的插件会把配置放在 `/root/.config` 下，需要的话也按同样方式挂载。也可以图省事把整个 `/root` 挂出来，比如 `/aaaa/root:/root`：

- 注意**别把宿主机自己的 `/root` 挂进去**；
- 挂载默认以宿主机为准，第一次启动时宿主机目录是空的，容器里的 `/root` 也会跟着变空。如果基础镜像里的系统依赖 `/root` 下的某些默认文件，就可能出问题。目前的实际使用中没有遇到，但这属于需要知道的风险。

其他 `volumes` 也都可以按需增减。

### nginx 配置

`compose.yml` 里引用的 `./nginx.conf` 内容如下：

```nginx
# 全局配置：仅保留必要的事件和 HTTP 块
events {
    worker_connections 1024;
}

http {
    # 隐藏 Nginx 版本号，减少信息暴露
    server_tokens off;

    # WebSocket 升级头的映射
    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }

    server {
        listen 30800;
        server_name _;   # 如果使用域名，替换为实际域名

        # ===== Basic Auth 认证 =====
        auth_basic           "Restricted";
        auth_basic_user_file /etc/nginx/.htpasswd;

        location / {
            # 转发到 dsh 容器
            proxy_pass http://172.24.0.10:3080;

            # WebSocket 必须用 HTTP/1.1 并透传 Upgrade/Connection
            proxy_http_version 1.1;
            proxy_set_header Upgrade    $http_upgrade;
            proxy_set_header Connection $connection_upgrade;

            # 保留原始 Host 与 Origin，dsh 的 trustedHosts 需要精确匹配
            proxy_set_header Host              172.24.0.10:3080;
            proxy_set_header Origin            http://172.24.0.10:3080;
            proxy_set_header X-Real-IP         $remote_addr;
            proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # ===== SSE / 流式输出优化 =====
            # 必须关闭缓冲和缓存，否则对话会“卡住”或延迟数秒才显示
            proxy_buffering off;
            proxy_cache off;
            proxy_request_buffering off;

            chunked_transfer_encoding on;

            # dsh 可能执行长时间任务，超时设长一点
            proxy_read_timeout  3600s;
            proxy_send_timeout  3600s;
            proxy_connect_timeout 30s;
        }
    }
}
```

注意核对里面的 `172.24.0.10:3080` 和你实际的容器 IP、端口是否一致。

### htpasswd：账号密码文件长什么样

`htpasswd` 是 nginx 做 Basic Auth 用的账号文件，格式很简单，每行一个用户：

```text
用户名:密码哈希
```

例如：

```text
luren:$apr1$m7Q2b9xT$8s0mX1kPq7yLz4rV6nB2c0
```

冒号前的 `luren` 是用户名，冒号后那一长串是**加盐后的密码哈希**，不是明文。`$apr1$` 表示使用的是 Apache 的 MD5 变体（apr1）；常见的还有：

- `$2y$...`：bcrypt，安全性更好，推荐；
- `{SHA}...`：SHA-1 摘要（Base64 编码），比较旧；
- 明文（不推荐，几乎没人用）。

nginx 的 `auth_basic_user_file` 支持 crypt、apr1、SHA-1 和 bcrypt 几种格式。

生成文件最省事的办法是用 `htpasswd` 命令（Debian 上装在 `apache2-utils` 里）：

```bash
# 安装工具
sudo apt install -y apache2-utils

# 交互式输入密码，创建（或覆盖）文件
htpasswd -c ./htpasswd luren

# 之后再加用户就不要带 -c 了，否则会清空原文件
htpasswd ./htpasswd zhangsan

# 也可以在命令行里直接给密码（注意会留在 shell 历史里）
htpasswd -b ./htpasswd luren 'MyStrongPass'
```

几点注意：

- 用 `-c` 会**新建并覆盖**文件，第二个用户开始就别再加 `-c`。
- 哈希每次生成都不一样，这是正常的（盐不同）。
- 文件里是哈希不是明文，但仍然建议 `chmod 600 htpasswd`，不要提交到 git。
- Basic Auth 的密码在 HTTP 下是明文传输的，只适合内网；要暴露到公网，请套一层 HTTPS。
- 改完密码文件后让 nginx 重新加载：`docker compose restart nginx-dsh`。

## 七、启动

在存放 `compose.yml` 的目录执行：

```bash
sudo docker compose up -d
```

看状态和日志：

```bash
sudo docker compose ps
sudo docker compose logs deepseek-harness
```

**这时候还不算成功。** DSH 默认只监听 `127.0.0.1`，所以通过 `http://服务器IP:30800` 是打不开的。需要多做一步：

```bash
# 先停止容器
docker compose stop
```

然后修改容器里的 `/root/.dsh/profiles/web/cordis.patch.yml`，也就是宿主机上的 `./dsh/profiles/web/cordis.patch.yml`，把 `host` 字段从 `"127.0.0.1"` 改成容器的 IP `172.24.0.10`：

```yaml
host: "172.24.0.10"
```

**不要改成 `"0.0.0.0"`。** DSH 会对请求的 Host / Origin 做白名单校验，`0.0.0.0` 不是合法的 Host 值，匹配不上；写成容器 IP 才能和 nginx 转发时带的 `Host`/`Origin` 对上。

改完再启动：

```bash
docker compose start
```

这里用 `start` 而不是 `up -d`，是因为容器已经创建好了，`start` 只是把它重新跑起来。如果你执行过 `docker compose down`（删除了容器），那就要重新 `docker compose up -d`。

浏览器打开 `http://服务器IP:30800`，输入刚才设置的账号密码，就能看到 DSH 的 Web 界面了。

## 八、以后更新版本

流程不复杂：

```bash
# 1. 删除现有容器
docker compose down

# 2. 更新源码
cd deepseek-harness
git pull
cd ..

# 3. 用新版本号重新构建
sudo docker build -t deepseek-harness:1.1.2 .

# 4. 改 compose 里的镜像 tag
#    image: 'deepseek-harness:1.1.1'  ->  image: 'deepseek-harness:1.1.2'

# 5. 重新创建容器
docker compose up -d
```

`down` 只删容器和网络，挂载目录 `./dsh` 里的数据不受影响。

## 九、额外建议：给宿主机做只读审计挂载

可以顺手把宿主机的一些配置目录以**只读**方式挂进容器，以后让 agent 帮忙审计或者排障会方便很多。注意一定要只读：

```yaml
    volumes:
      # 系统基本信息
      - /etc/os-release:/aa-mount/host/etc/os-release:ro
      - /etc/debian_version:/aa-mount/host/etc/debian_version:ro
      - /etc/passwd:/aa-mount/host/etc/passwd:ro
      - /etc/group:/aa-mount/host/etc/group:ro
      - /etc/hostname:/aa-mount/host/etc/hostname:ro
      - /etc/hosts:/aa-mount/host/etc/hosts:ro
      - /etc/fstab:/aa-mount/host/etc/fstab:ro

      # 内核参数配置
      - /etc/sysctl.d:/aa-mount/host/etc/sysctl.d:ro
      - /usr/lib/sysctl.d:/aa-mount/host/usr/lib/sysctl.d:ro

      # systemd 服务配置
      - /etc/systemd/system:/aa-mount/host/etc/systemd/system:ro
      - /usr/lib/systemd/system:/aa-mount/host/usr/lib/systemd/system:ro

      # 定时任务
      - /etc/crontab:/aa-mount/host/etc/crontab:ro
      - /etc/cron.d:/aa-mount/host/etc/cron.d:ro
      - /etc/cron.daily:/aa-mount/host/etc/cron.daily:ro
      - /etc/cron.hourly:/aa-mount/host/etc/cron.hourly:ro
      - /etc/cron.weekly:/aa-mount/host/etc/cron.weekly:ro
      - /etc/cron.monthly:/aa-mount/host/etc/cron.monthly:ro

      # SSH 配置（不挂载私钥）
      - /etc/ssh/sshd_config:/aa-mount/host/etc/ssh/sshd_config:ro
      - /etc/ssh/sshd_config.d:/aa-mount/host/etc/ssh/sshd_config.d:ro

      # 软件包与 APT 配置
      - /var/lib/dpkg/status:/aa-mount/host/var/lib/dpkg/status:ro
      - /etc/apt:/aa-mount/host/etc/apt:ro
      - /var/log/dpkg.log:/aa-mount/host/var/log/dpkg.log:ro
      - /var/log/apt:/aa-mount/host/var/log/apt:ro

      # 系统日志
      - /var/log/journal:/aa-mount/host/var/log/journal:ro
```

几点必须说清楚：

- 宿主机侧的目录必须存在，容器里不存在的目录会自动创建。
- **千万不要图省事写 `/:/aa-mount/host:ro`**。这等于把宿主机整个根目录挂进去，哪怕是只读的也不行——宿主机里有 `/var/run/docker.sock` 这类非普通文件，它们能接收指令，只读挂载挡不住“通过它下命令”。
- 同理，`docker.sock` 不要挂进容器，那等于把 docker 的控制权交出去。

## 十、几个注意事项

- **备份 `./dsh`。** 会话、插件、配置都在里面，丢了就得重来。
- **别把 3080 直接开到公网。** DSH 本身没有登录鉴权，暴露出去等于把服务器交给别人。要么只在内网用，要么像上面这样套 nginx + Basic Auth，条件允许再加 HTTPS。
- **`restart: "no"` 不是长久之计**，机器重启或容器崩溃都不会自愈，建议用 `unless-stopped`。
- **核显权限。** 直通 `/dev/dri` 后，容器里的用户要能访问这个设备，必要时在 compose 里加 `group_add`，把 `render`、`video` 组的 GID 加进去。
- **定期清理磁盘。** 镜像和构建缓存会越攒越多，隔段时间 `docker system df` 看看，`docker image prune`、`docker builder prune` 清一清。
- **不要图省事加 `--privileged`。** 需要什么设备就直通什么设备，最小权限比“全开”省心得多。

