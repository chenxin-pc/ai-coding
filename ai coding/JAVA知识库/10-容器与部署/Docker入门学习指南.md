# Docker 入门学习指南：从 Java 开发环境到容器通信

> 适合已经会运行 Spring Boot 项目、刚接触 Docker 的 Java 开发者。建议边读边执行命令：先弄清“为什么”，再记命令，最后用实际输出验证理解。示例以 macOS 上的 Docker Desktop 为主；Linux Docker Engine 的差异会单独说明。

## 学习路线

| 顺序 | 问题 | 完成标准 |
| --- | --- | --- |
| 1 | Docker 解决什么问题？ | 能区分镜像、容器、仓库 |
| 2 | 如何安装与拉取镜像？ | `docker version` 和 `docker run hello-world` 成功 |
| 3 | 如何管理镜像和容器？ | 能启动 Nginx、查状态、看日志、停止和删除 |
| 4 | 删除容器后数据怎么办？ | 能用数据卷证明“容器删了，数据还在” |
| 5 | Java、MySQL、Redis 如何互通？ | 能根据访问方向选对地址和端口 |

学习时不要先背全部命令。每个命令都问四件事：**操作的对象是什么、动作是什么、参数改变了什么、怎样验证结果**。

## 1. Docker 为什么会出现？

一个 Spring Boot 服务可能依赖特定 JDK、MySQL、Redis 和系统库。每个人手工安装一遍，环境容易不同；交付到测试或服务器时，还要重新配置。Docker 将运行所需文件和配置打包为**镜像**，再从镜像创建相互隔离的**容器**，让应用环境更容易复现和交付。容器是隔离的进程，并不等于一台完整虚拟机。[Docker 概览](https://docs.docker.com/get-started/docker-overview/)

```text
Dockerfile（制作说明）→ 镜像（可分发的模板）→ 容器（正在运行的实例）
                                               ├─ 网络：与外部通信
                                               └─ 数据卷：保存需要留存的数据
```

以 Java 项目为例：团队可以先用 Docker 启动同版本的 MySQL、Redis；再把 Spring Boot 的 JAR 构建为镜像。**仓库**（如 Docker Hub）用于保存和分发镜像。一个镜像可以创建多个容器；删除容器不会自动删除镜像。[镜像概念](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-an-image/)、[容器概念](https://docs.docker.com/get-started/docker-concepts/the-basics/what-is-a-container/)

## 2. 安装 Docker 与镜像加速

### 2.1 安装和验证

- **macOS**：从 [Docker Desktop for Mac 官方页面](https://docs.docker.com/desktop/setup/install/mac-install/)按 Apple 芯片或 Intel 芯片选择安装包，安装后启动 Docker Desktop。
- **Windows**：按 [Docker Desktop for Windows 官方指南](https://docs.docker.com/desktop/setup/install/windows-install/)安装。
- **Linux**：按发行版参考 [Docker Engine 安装指南](https://docs.docker.com/engine/install/)，不要照搬旧视频中的一键脚本或过时软件源。

安装后运行：

```bash
docker version
docker run hello-world
```

`docker version` 应能看到客户端与服务端信息；只有客户端信息时，先确认 Docker Desktop 或 Docker Engine 已启动。`hello-world` 会拉取镜像、创建容器并输出成功消息，因此一次验证了“客户端 → 服务端 → 拉取镜像 → 运行容器”这条基本链路。

### 2.2 阿里云镜像加速：知道如何配，也要知道限制

镜像加速器用于改善从 Docker Hub 拉取镜像的速度。阿里云 ACR 为账号提供专属地址：登录 **容器镜像服务 ACR → 镜像工具 → 镜像加速器**，复制自己账号显示的地址，不能直接使用教程作者的地址。

在 macOS Docker Desktop 中打开 **Settings → Docker Engine**，将以下配置项加入现有 JSON；Linux Docker Engine 通常编辑 `/etc/docker/daemon.json`，随后重启 Docker 服务：

```json
{
  "registry-mirrors": ["https://你的专属地址.mirror.aliyuncs.com"]
}
```

如果原配置还有其他字段，应保留并正确放置逗号。配置后用 `docker info` 查看 `Registry Mirrors`。**配置被读取不代表镜像一定能拉取**：截至 2026-09-26，阿里云官方说明 ACR 镜像加速已停止同步最新镜像，某些镜像可能拉取失败，`latest` 也可能并非最新版本；该服务面向个人开发场景。学习和项目中尽量使用明确版本标签，并按 [阿里云官方说明](https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images)选择可用来源。[Docker Desktop 设置](https://docs.docker.com/desktop/settings-and-maintenance/settings/)

## 3. 先学会读命令：`-`、`--` 和缩写

Docker 命令通常可按 **`docker 对象 动作 [选项] 目标`** 理解，例如 `docker image pull nginx:alpine` 是对镜像执行下载。常见简写 `docker pull nginx:alpine` 与它等价。

| 写法 | 叫法 | 例子 | 记忆 |
| --- | --- | --- | --- |
| `-` 加一个字母 | 短选项 | `-d`、`-p` | 输入快 |
| `--` 加完整名称 | 长选项 | `--detach`、`--publish` | 含义清楚 |

下面两种写法等价，练习时任选一条执行即可：

```bash
docker run -d -p 8080:80 nginx:alpine
docker run --detach --publish 8080:80 nginx:alpine
```

`-i -t` 常合写为 `-it`。不是所有长选项都有短写法，例如 `--name`。**同一短字母在不同命令里可能含义不同**：`docker build -t` 的 `t` 是 *tag*，`docker logs -t` 的 `t` 是 *timestamps*；`docker build -f` 的 `f` 是 *file*，`docker logs -f` 的 `f` 是 *follow*。`-p` 和 `-P` 也不同，大小写不能互换。忘记时直接运行 `docker run --help` 或相应命令的 `--help`。[Docker CLI 参考](https://docs.docker.com/reference/cli/docker/)

### 常见缩写速记

| 缩写 | 英文或长选项 | 常见场景 |
| --- | --- | --- |
| `ls` | list | `docker image ls`、`docker volume ls` |
| `rm` / `rmi` | remove / remove image | 删除容器 / 删除镜像 |
| `cp` | copy | 在宿主机与容器之间复制文件 |
| `-a` | all | `docker ps -a`，列出所有容器 |
| `-q` | quiet | 只输出 ID，常用于命令组合 |
| `-d` | detach | 后台运行容器 |
| `-i` | interactive | 保持交互输入 |
| `-t` | tty（teletypewriter） | 给交互程序分配终端；在 `build`/`logs` 中另有含义 |
| `-p` | publish | 指定宿主机与容器的端口映射 |
| `-P` | publish-all | 将声明的端口发布到宿主机随机端口 |
| `-e` | env / environment | 注入环境变量 |
| `-v` | volume | 挂载数据卷或宿主机目录 |
| `-m` | memory | 限制容器内存 |
| `-f` | 随命令变化 | `logs` 为 follow；`build` 为 file；`ps` 为 filter |

## 4. 镜像命令：管理运行模板

镜像名称通常写成 `名称:标签`，例如 `nginx:alpine`、`mysql:8.4`。标签可以用来指定版本或变体；不写标签时往往会使用 `latest`，但不应把 `latest` 理解为可复现的固定版本。[镜像命令参考](https://docs.docker.com/reference/cli/docker/image/)

| 目的 | 命令 | 关键参数 |
| --- | --- | --- |
| 下载镜像 | `docker pull nginx:alpine` | `名称:标签` 是下载目标 |
| 查看本地镜像 | `docker images` 或 `docker image ls` | `-q` 只输出 ID；`-f` 按条件筛选 |
| 从 Dockerfile 构建 | `docker build -t my-api:1.0 .` | `-t` = tag；末尾 `.` 是构建上下文目录 |
| 指定 Dockerfile | `docker build -f Dockerfile.dev -t my-api:dev .` | `-f` = file |
| 添加另一个标签 | `docker tag my-api:1.0 my-api:stable` | 原镜像、新标签 |
| 查看镜像详情 | `docker image inspect my-api:1.0` | 查架构、配置等 |
| 删除镜像或标签 | `docker rmi my-api:stable` | `rmi` = remove image |

记忆顺序：**pull 拉取 → images 查看 → build 制作 → tag 标记 → rmi 删除**。构建自己的 Java 镜像是下一阶段任务；入门先会使用现成镜像。[构建命令参考](https://docs.docker.com/reference/cli/docker/buildx/build/)

## 5. 运行命令：从镜像创建容器

```text
docker run [选项] 镜像名[:标签] [容器内执行的命令]
```

`run` 是 **创建新容器并启动**；已停止的容器再启动用 `start`。重复执行 `run` 不会“回到原容器”，而是再创建一个容器。[`docker run` 参考](https://docs.docker.com/reference/cli/docker/container/run/)

| 选项 | 含义 | 示例或记忆 |
| --- | --- | --- |
| `--name` | 给容器取名 | `--name my-web` |
| `-d` / `--detach` | 后台运行 | Web、数据库服务常用 |
| `-i` / `--interactive` | 保持输入 | 通常与 `-t` 合用 |
| `-t` / `--tty` | 分配终端 | `-it` 用来进入交互式 shell |
| `--rm` | 容器退出后自动删除 | 适合临时练习；具名卷仍保留 |
| `-p` / `--publish` | 发布端口 | `127.0.0.1:8080:80` = 本机 8080 → 容器 80 |
| `-e` / `--env` | 设置环境变量 | `-e SPRING_PROFILES_ACTIVE=dev` |
| `-v` / `--volume` | 挂载数据 | `-v mysql-data:/var/lib/mysql` |
| `--mount` | 显式写出挂载类型、来源和目标 | 便于阅读复杂挂载 |
| `--network` | 加入网络 | `--network app-net` |
| `--restart` | 自动重启策略 | `--restart unless-stopped` |
| `-m` / `--memory` | 内存上限 | `-m 512m` |

先运行一个网站：

```bash
docker run -d --name my-web -p 127.0.0.1:8080:80 nginx:alpine
docker ps
```

浏览器打开 `http://localhost:8080`。这里的 `127.0.0.1` 表示只允许本机访问；若只写 `-p 8080:80`，Docker 默认会在宿主机所有网络接口发布端口。`EXPOSE` 只是镜像中的端口说明，**不会自动对宿主机开放端口**。[发布端口说明](https://docs.docker.com/get-started/docker-concepts/running-containers/publishing-ports/)

记住运行服务的四个问题：**叫什么**（`--name`）、**怎么访问**（`-p`）、**配置是什么**（`-e`）、**数据放哪里**（`-v`/`--mount`）。

## 6. 日志与容器生命周期

### 6.1 日志命令

`docker logs` 查看容器写到标准输出和标准错误的内容；如果 Java 应用只写容器内文件，`docker logs` 不会自动读取那个文件。[日志命令参考](https://docs.docker.com/reference/cli/docker/container/logs/)

| 命令 | 作用 | 缩写记忆 |
| --- | --- | --- |
| `docker logs my-web` | 查看已有日志 | logs = 日志 |
| `docker logs -f my-web` | 持续看新日志 | follow = 跟随 |
| `docker logs --tail 100 my-web` | 只看最后 100 行 | tail = 尾部 |
| `docker logs --since 30m my-web` | 看最近 30 分钟 | since = 自某时起 |
| `docker logs -t my-web` | 显示时间戳 | timestamps = 时间戳 |

最常用组合是 `docker logs -f --tail 100 my-web`。启动失败时，先 `docker ps -a` 找到已退出容器，再查看它的日志。

### 6.2 容器命令

```text
镜像 --run--> 新容器 --stop--> 已停止容器 --start--> 同一个容器
                                    └--rm--> 删除容器
```

| 目的 | 命令 | 说明 |
| --- | --- | --- |
| 看运行中的容器 | `docker ps` | `ps` 可记作 process status |
| 看所有容器 | `docker ps -a` | 包括已停止、启动失败的容器 |
| 只看容器 ID | `docker ps -aq` | `a` = all，`q` = quiet |
| 停止、再次启动 | `docker stop my-web`、`docker start my-web` | `start` 不创建新容器 |
| 重启 | `docker restart my-web` | 对已有容器执行停止和启动 |
| 在运行中的容器执行命令 | `docker exec -it my-web sh` | `exec` = execute；`sh` 是容器内 shell |
| 查看详情 | `docker inspect my-web` | 重点找 `Mounts`、`NetworkSettings`、`State` |
| 复制文件 | `docker cp my-web:/app/file.txt ./file.txt` | `cp` = copy，源路径在前 |
| 看资源占用 | `docker stats my-web` | `--no-stream` 只取一次 |
| 删除已停止的容器 | `docker rm my-web` | `rm` = remove；`-f` 可强制删除运行中的容器 |

`exec` 只能对运行中的容器执行；精简镜像未必有 `bash`，可以先试 `sh`。区分三词：**run 新建、start 再启动、exec 在里面另执行一个命令**。完成 Nginx 练习后，可运行 `docker stop my-web` 和 `docker rm my-web` 清理示例容器。[容器命令参考](https://docs.docker.com/reference/cli/docker/container/)

## 7. 数据卷：为什么删除容器后数据还能在？

容器的可写层随容器删除而删除。数据库文件等需要保留的数据不应只放在这层。**数据卷**是 Docker 单独管理的持久化存储，可以挂到容器中的目录；删除容器后，具名卷仍可供新容器使用。[数据卷说明](https://docs.docker.com/engine/storage/volumes/)

| 类型 | 写法示意 | 适合什么 |
| --- | --- | --- |
| 具名卷 | `-v mysql-data:/var/lib/mysql` | MySQL 等容器产生的数据；容易复用 |
| 匿名卷 | `-v /data` | 不指定名字，Docker 分配随机卷名；不便于手工复用 |
| 目录绑定挂载 | `--mount type=bind,source=/本机绝对路径,target=/app/config` | 本机编辑的代码或配置要直接被容器读取 |

`-v 卷名:容器内路径` 和 `--mount type=volume,source=卷名,target=容器内路径` 是具名卷的两种写法。`--mount` 更明确；要注意左侧卷名不是 Mac 上的目录路径。在 macOS Docker Desktop 中，卷由 Docker 的运行环境管理，不应靠手工修改某个猜测的宿主机路径来管理内容。[卷与绑定挂载](https://docs.docker.com/engine/storage/volumes/)

### 动手实验：容器删了，文件还在

```bash
docker volume create java-demo-data
docker run --rm -v java-demo-data:/data alpine sh -c 'echo "hello volume" > /data/hello.txt'
docker run --rm -v java-demo-data:/data alpine cat /data/hello.txt
docker volume ls
docker volume inspect java-demo-data
```

两个 `docker run --rm` 创建的临时容器结束后都被删除；第二个仍读到 `hello volume`，说明文件在**具名卷**中。确认不再需要练习数据后，才运行 `docker volume rm java-demo-data`。删除卷会删除卷中的数据。

Java 项目的典型用法是把 MySQL 官方镜像的数据目录 `/var/lib/mysql` 挂到具名卷，例如 `-v mysql-data:/var/lib/mysql`。`docker compose down` 通常会保留具名卷；加上 `--volumes` 则会删除 Compose 管理的卷，不适合在需要保留数据库数据时随手执行。[MySQL 官方镜像](https://hub.docker.com/_/mysql)、[Compose 数据卷](https://docs.docker.com/engine/storage/volumes/)

## 8. Docker 网络：先明确“谁访问谁”

**`localhost` 永远指发起连接的一方自己。**Mac 上的 Spring Boot 连接 `localhost` 是 Mac；Spring Boot 在容器中连接 `localhost` 是 Spring Boot 容器自己。容器里写 `jdbc:mysql://localhost:3306/...`，并不会自动找到另一个 MySQL 容器。

### 8.1 三个通信方向

| 谁访问谁 | 地址写法 | 原因或前提 |
| --- | --- | --- |
| 宿主机 → 容器 | `localhost:宿主机端口` | 容器启动时要用 `-p` 发布端口 |
| 容器 → 宿主机 | `host.docker.internal:宿主机端口` | Docker Desktop 提供这个宿主机名称；宿主机服务必须可访问 |
| 容器 A → 容器 B | `容器名:容器内端口` | 两个容器加入同一个自定义网络；不需要 `-p` |

例子：

```bash
# 宿主机访问 Nginx 容器：Mac 浏览器打开 http://localhost:8081
docker run -d --name network-web -p 127.0.0.1:8081:80 nginx:alpine
```

如果 MySQL 直接运行在 Mac 上，而 Spring Boot 在容器里，Spring Boot 的 JDBC 地址可写为 `jdbc:mysql://host.docker.internal:3306/appdb`。在**原生 Linux Docker Engine** 中，如需同名访问，可给容器加 `--add-host=host.docker.internal:host-gateway`。若连接失败，还要核对宿主机服务监听地址与防火墙。[Docker Desktop 网络说明](https://docs.docker.com/desktop/features/networking/networking-how-tos/)、[`host-gateway` 说明](https://docs.docker.com/reference/cli/docker/container/run/)

### 8.2 容器互通：为什么推荐自定义 bridge 网络？

Docker 默认 `bridge` 网络与自己创建的 bridge 网络不同：**自定义网络**提供容器名解析，适合把同一项目的应用、数据库、缓存放在一起。容器 IP 可能变化，连接地址优先使用容器名或 Compose 服务名。旧教程中的默认桥接网络 `--link` 属于旧方式，学习时以自定义网络为主。[bridge 网络说明](https://docs.docker.com/engine/network/drivers/bridge/)

```bash
docker network create app-net
docker run -d --name redis-demo --network app-net redis:7-alpine
docker run --rm --network app-net redis:7-alpine redis-cli -h redis-demo ping
docker network inspect app-net
```

第三条预期返回 `PONG`：客户端容器通过 `redis-demo` 找到 Redis，使用 Redis **容器内端口** 6379，整个过程不需要发布宿主机端口。实验后清理：

```bash
docker rm -f redis-demo
docker network rm app-net
docker rm -f network-web
```

`docker network ls` 查看网络；`docker network inspect 网络名` 查看成员；`docker network connect 网络名 容器名` 可把已有容器接入网络。[网络命令参考](https://docs.docker.com/reference/cli/docker/network/)

### 8.3 Java 项目的地址速查

| Spring Boot 在哪里 | MySQL / Redis 在哪里 | 应使用的地址 |
| --- | --- | --- |
| Mac 本机 | 容器，已发布宿主机端口 | `localhost:宿主机端口` |
| 容器 | Mac 本机 | `host.docker.internal:宿主机端口`（Docker Desktop） |
| 容器 | 同一自定义网络中的容器 | `容器名:容器内端口` |
| Compose 服务 | 同一 Compose 项目的另一服务 | `服务名:容器内端口` |

Compose 默认会为一个项目创建网络，服务之间可用服务名互访。例如 Spring Boot 服务连接 `redis:6379`；`6379` 是 Redis 的容器内端口，不是发布到 Mac 的端口。此时通常不需要手动运行 `docker network create`。[Compose 网络说明](https://docs.docker.com/compose/how-tos/networking/)

## 9. 把知识串成一条排查路线

以“Java 服务连不上 Redis”为例，按顺序判断：

1. **Redis 容器是否运行？** `docker ps -a`。
2. **Redis 是否启动成功？** `docker logs --tail 100 redis-demo`。
3. **Java 在哪儿运行？** 在 Mac 本机、在独立容器中，还是在 Compose 中？
4. **地址是否与通信方向匹配？** 本机用发布的宿主机端口；同一 Docker 网络中的容器用服务名和容器内端口。
5. **两容器是否同网？** `docker network inspect app-net`。
6. **能否直接测试？** 在同一网络运行 `redis-cli -h redis-demo ping`，看到 `PONG` 后再查 Java 配置。

完成本指南后，应能口头解释：**为什么需要镜像和容器；为什么删除容器后数据卷仍在；为什么容器中的 `localhost` 找不到另一个容器；什么时候用 `-p`，什么时候用容器名；`run`、`start`、`exec` 有什么区别。**

## 资料与后续学习

- [Docker 官方入门](https://docs.docker.com/get-started/)：概念和第一批实操。
- [Docker Java 指南](https://docs.docker.com/guides/java/)：下一步将 Spring Boot 项目做成镜像，再用 Compose 连接数据库。
- [Docker CLI 参考](https://docs.docker.com/reference/cli/docker/)：查当前命令和参数，优先于记忆旧视频中的写法。
- [Docker Compose 入门](https://docs.docker.com/compose/gettingstarted/)：学会用一个 `compose.yaml` 管理 Spring Boot、MySQL、Redis。

> 文中命令用于本地学习。镜像标签、平台安装步骤和镜像加速服务可能变化；执行前以对应官方页面和 `docker <命令> --help` 为准。
