
> **解决什么问题**：每台机器环境不一样 → "在我电脑上是好的"。

> **一句话**：
> 	把程序和它需要的环境打包成一个标准单位（容器）
> 	在任何机器上运行，结果都一致。

## 1. 容器是什么

| 对比    | **虚拟机 VM**        | **容器 Container**    |
| ----- | ----------------- | ------------------- |
| 虚拟化层  | Hypervisor 虚拟整块硬件 | 直接复用宿主机 **OS 内核**   |
| 是否带系统 | 每个带完整 Guest OS    | 只带应用+依赖，**共享宿主内核**  |
| 内核数量  | N 个 VM = N 个内核    | N 个容器 = 1 个宿主内核     |
| 启动速度  | 秒级~分钟             | **毫秒~秒级**           |
| 资源占用  | 大（每台几千 MB）        | 小（每份几十 MB）          |
| 隔离    | 强（硬件级）            | 中（OS 级，靠 namespace） |

> 关键：**容器不是虚拟机**。
> 它不多装一个系统，而是用宿主内核加轻量隔离，
> 所以启动快、占用小。

## 2. 隔离的核心：namespace + cgroup（Linux 机制）

容器看起来像独立系统，靠内核的两个机制。

| 机制            | 管什么                          | 类比             |
| ------------- | ---------------------------- | -------------- |
| **namespace** | **隔离"看得见的东西"**：进程、网络、文件系统、用户 | 给每个容器一个独立的"视野" |
| **cgroup**    | **限制"用得着的资源"**：CPU、内存、磁盘配额   | 控制它能吃多少资源      |

```
容器A          容器B
━━━━━━━       ━━━━━━━
自己的进程表     自己的进程表    ← namespace（隔离）
自己的网络      自己的网络
自己的/root    自己的/root
──────────────────────────

  共享宿主 Linux 内核
  由 cgroup 限制 A/B 各能占多少 CPU / 内存
```

> 容器怎么实现的？
> 答案：因为 Linux 内核有 namespace 和 cgroup。

> Docker 是一个程序，它把这两个机制包装成易用的工具。

**三者的确切类型**：

| 名称            | 类型                       | 本质                                                                   |
| ------------- | ------------------------ | -------------------------------------------------------------------- |
| **namespace** | Linux 内核系统调用（syscall）API | 通过 `clone()`、`unshare()`、`setns()` 触发，传 `CLONE_NEWPID` 等 flag 选择隔离维度 |
| **cgroup**    | 内核 API，接口是文件系统           | 通过 `/sys/fs/cgroup/` 读写文件操作，写 pid 到 tasks 文件 = 加入控制组                 |
| **runC**      | 可执行文件（Go 编写，二进制）         | 用户空间程序，内部调用 namespace/cgroup 的 syscall 来创建容器进程                       |

**三者关系**：
- namespace/cgroup 是内核提供的能力（API）。
- runC 是使用这个能力的工具（调用者）。

## 3. Docker 架构：Client / Server（守护进程）

```
你敲 docker 命令
    ↓
Docker Client（docker CLI）
	    ↓ 走 API（socket 或 http）
Docker Daemon（dockerd，常驻后台）
    ↓ 调用
containerd（容器运行时，真正管理容器生命周期）
    ↓
runc（用 namespace/cgroup 实际启动容器进程）
```

| 组件                          | 角色                          |
| --------------------------- | --------------------------- |
| **Docker CLI**              | 你的命令行工具，只负责发指令              |
| **dockerd / Docker Engine** | 后台守护进程，真正执行操作               |
| **containerd**              | 高级容器运行时，管理容器（Docker 现在植根于此） |
| **runc**                    | 底层，用内核机制真正创建容器进程            |

> 关键：**CLI 和 Engine 分离**。

> CLI 轻，只负责发指令。
> 引擎重，常驻后台运行。

## 4. 核心概念四件套

```
Dockerfile（蓝图，一段文本）
   ↓ docker build
Image 镜像（只读模板，一个 Dockerfile → 一个镜像）
   ↓ docker run（可多次，每次建一个）
Container 容器（运行中的实例）
   ↓ 推到
Registry 仓库（镜像托管，如 Docker Hub）
```

| 概念               | 是什么                        | 类比        |
| ---------------- | -------------------------- | --------- |
| **Dockerfile**   | 定义容器内部环境的文本（系统/依赖/代码/启动命令） | 菜谱        |
| **Image 镜像**     | 只读模板，可被反复实例化               | 类 / 光盘    |
| **Container 容器** | 镜像运行后的实例，一个镜像可多个容器         | 对象 / 多台电脑 |
| **Registry**     | 存镜像的地方，可拉取/推送              | 软件商店      |

**物理形态**：

| 概念            | 类型                  | 细节                                                                     |
| ------------- | ------------------- | ---------------------------------------------------------------------- |
| **Image**     | **文件**（只读 Layer 集合） | 1 个 manifest.json + 多个 layer tarball 存于磁盘，`docker save` 可导出为 `.tar`    |
| **Container** | **不是单一文件**（进程 + 文件） | 文件系统（writable layer）+ 元数据（config.json）是文件，存在磁盘；容器进程本身是进程，不是文件；网络栈是内核对象 |

## 5. Dockerfile 实战

一个简单的 Python 后端 Dockerfile：

```dockerfile
# 基础镜像（带 Python 和系统）
FROM python:3.12-slim

# 工作目录
WORKDIR /app

# 先复制依赖文件 → 单独缓存依赖层（加速构建）
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 再复制代码（改代码不用重装依赖）
COPY . .

# 声明容器监听端口（仅声明，不映射）
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

**分层构建逻辑**：每一行产生一个只读层。
后面改动只需重建变动的层。
`COPY requirements.txt` 放 `COPY .` 前面，是为了依赖层能缓存。

### build 时内部发生了什么

**本质**：
	CLI 只发指令，不执行构建。
	dockerd 后台进程才真正执行构建。

```
docker build -t myapp .
  ↓
Docker CLI → 把 Dockerfile + 构建上下文（`.` 目录）打包成 tar，通过 HTTP API 发给 dockerd
  ↓
dockerd → 解析 Dockerfile，逐条指令执行
  ↓
BuildKit → 现代 Docker 的构建引擎（containerd 内置），负责解析 DAG、缓存、并行构建
  ↓
containerd → 管理临时容器
  ↓
runc → 创建临时容器执行每条指令
```

| 用户写                       | dockerd 实际做的事                                                  |
| ------------------------- | -------------------------------------------------------------- |
| `FROM python:3.12-slim`   | 拉取该 image 的 manifest 和所有 layer 到 `/var/lib/docker/overlay2/`   |
| `COPY requirements.txt .` | 用 runc 创建临时容器，把文件写入容器文件系统，捕获文件系统差异，生成新 layer（tar 包存 overlay2/） |
| `RUN pip install ...`     | 用 runc 创建临时容器（挂载已有 layer 为根文件系统），执行命令，等退出，捕获差异，生成新 layer       |
| `COPY . .`                | 同上                                                             |
| `CMD [...]`               | 不生成 layer，只写入 image 的 config.json 的 Cmd 字段                     |

**最终产物**：
- N 个只读 layer 目录 + 1 个 manifest.json。
- layer 目录在 `/var/lib/docker/overlay2/` 下。
- manifest.json 记录 layer 的叠加顺序。

### run 时内部发生了什么

**本质**：dockerd 从磁盘读取已构建好的 image 文件。
它组装出容器文件系统，然后启动进程。

```
docker run -p 8000:8000 myapp
  ↓
Docker CLI → 把运行参数（镜像名、端口映射等）通过 HTTP API 发给 dockerd
  ↓
dockerd → 从 /var/lib/docker/overlay2/ 读取 image 的 manifest.json 和 layer 文件
  ↓
containerd → 构造 OCI 配置（JSON），传给 runc
  ↓
runc → 用 namespace/cgroup 创建隔离环境，启动进程
```

runc 做的事（按顺序）：

| 顺序  | 动作           | 说明                                                                               |
| --- | ------------ | -------------------------------------------------------------------------------- |
| ①   | 创建容器根文件系统    | 用 overlayfs 挂载：lowerdir=所有读取的 image layer，upperdir=新建可写层                         |
| ②   | 设置 cgroup    | 在 `/sys/fs/cgroup/` 下创建容器子目录，写入 CPU/内存限制                                         |
| ③   | 创建 namespace | `clone(CLONE_NEWPID \| CLONE_NEWNS \| CLONE_NEWNET \| ...)`，新进程有独立 PID/网络/文件系统空间 |
| ④   | pivot_root   | 切换进程根文件系统到容器根目录（overlayfs 挂载点）                                                   |
| ⑤   | exec CMD     | 在隔离环境中执行 CMD（如 `uvicorn main:app`）                                               |

**容器文件系统构成**：
- overlayfs 合并。
- lowerdir = 所有 image 只读层。
- upperdir = 新建可写目录。

容器内写文件 → 写入 upperdir。
容器删除 → upperdir 被删，数据消失（除非用 volume）。

## 6. docker-compose.yml（多容器编排，本地/小规模）

Dockerfile 负责容器内部，compose 负责容器外部配合。

```yaml
version: "3.8"
services:
  web:
    build: .                # 用目录里的 Dockerfile 构建
    ports:
      - "8000:8000"        # 端口映射：宿主:容器
    depends_on:
      - db                 # 先启动 db
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - data:/var/lib/postgresql/data   # 数据卷，容器删了数据还在
volumes:
  data:
```

| 意义                    | 负责什么              |
| --------------------- | ----------------- |
| 不用记一堆 `docker run` 参数 | 一键启停整套服务          |
| `ports` 端口映射          | 宿主 8000 → 容器 8000 |
| `volumes` 数据卷         | 容器删了，数据不丢         |
| `depends_on` 依赖顺序     | 先起数据库再起服务         |

## 7. 常用命令速查

| 命令                              | 作用                |
| ------------------------------- | ----------------- |
| `docker build -t myapp .`       | 从 Dockerfile 构建镜像 |
| `docker run -p 8000:8000 myapp` | 跑一个容器并映射端口        |
| `docker ps` / `docker ps -a`    | 看运行中 / 所有容器       |
| `docker images`                 | 列出本地镜像            |
| `docker exec -it <id> bash`     | 进入运行中容器的 shell    |
| `docker logs <id>`              | 看容器日志             |
| `docker compose up -d`          | 后台启动整套 compose    |
| `docker push <user>/<img>`      | 推到仓库              |

## 8. 高频一句话

- **容器 = 复用宿主内核 + namespace 隔离 + cgroup 限资源**，不是虚拟机。
- **Dockerfile → image → container**，一个 Dockerfile 对应一个 image。
- **CLI 发指令、Engine 执行**，架构是 client/server 分离。
- **compose 负责外部配合**（端口/卷/多服务），Dockerfile 负责内部环境。
- 卷（volume）→ 容器删除后数据不丢，生产必备。

## 9. macOS 上 Docker 的特殊架构

### 为什么 macOS 没有 `/var/lib/docker`

| 项目                     | 事实                                                                      |
| ---------------------- | ----------------------------------------------------------------------- |
| 操作系统                   | macOS，不是 Linux                                                          |
| Docker Desktop 实现方式    | 跑一个轻量 Linux VM（HyperKit），Docker daemon 和所有存储都在 VM 内部                    |
| `/var/lib/docker` 实际位置 | 在 Linux VM 内部，不在 macOS 宿主文件系统上                                          |
| 验证方法                   | `docker info \| grep "Docker Root Dir"` → 输出 `/var/lib/docker`（VM 内部路径） |

### macOS 上 Docker 相关文件在宿主的实际位置

| 内容                | 宿主路径（macOS）                                        |
| ----------------- | -------------------------------------------------- |
| Docker Desktop 程序 | `/Applications/Docker.app`                         |
| 虚拟机磁盘镜像           | `~/Library/Containers/com.docker.docker/Data/vms/` |
| 用户数据（volume 等）    | `~/Library/Containers/com.docker.docker/Data/`     |

### Docker Daemon 和 HyperKit VM 是不同进程

| 进程                | 类型       | 运行环境                   |
| ----------------- | -------- | ---------------------- |
| HyperKit 进程       | macOS 进程 | macOS 宿主上              |
| dockerd           | Linux 进程 | **在 VM 内部**，是 VM 内部的程序 |
| containerd / runc | Linux 进程 | 在 VM 内部，dockerd 的子调用链  |

**关键**：
	dockerd 是 VM 内部的程序，不是 macOS 进程。
	HyperKit 是 macOS 进程，负责启动 VM。
二者不是同一个进程。

## 10. 我们自己开发、生产怎么用 Docker

> 一句话分工：
> 	- 自己的代码自己构建镜像。
> 	- 数据库和中间件用官方镜像，拉来直接用。
> 	- compose / k8s 把两者串起来。

### 开发：本机怎么跑

```yaml
# docker-compose.yml（项目根目录）
services:
  db:
    image: postgres:16          # 官方镜像，拉来直接用
    environment:
      POSTGRES_PASSWORD: secret
    ports:
      - "5432:5432"             # 本机 5432 → 容器 5432
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

| 做法            | 说明                                             |
| ------------- | ---------------------------------------------- |
| ✅ 数据库、中间件用容器跑 | `docker compose up -d` 一键起整套，本机不用装 MySQL/Redis |
| ✅ 自己的代码未必容器化  | 本地开发要热重载，代码直接在本机跑、连容器里的数据库                     |
| ⚠️ 端口冲突       | 本机已有服务占用端口时，改 `ports` 左侧的映射                    |

### 构建：CI 出镜像

```
代码 push → CI（持续集成流水线，如 GitHub Actions）
  → docker build（按 Dockerfile）
  → docker push 到镜像仓库
  → 产物：myapp:1.2.3（版本号 或 commit sha）
```

| 要点              | 说明                                  |
| --------------- | ----------------------------------- |
| ✅ tag 不用 latest | latest 会漂移，环境不可复现；用版本号 / commit sha |
| 镜像仓库            | 公司私有仓库（如 Harbor）或云厂商镜像服务            |
| 同一镜像一路部署        | 测试用的镜像 = 生产用的镜像                     |

### 部署：生产环境

| 规模       | 工具              |
| -------- | --------------- |
| 单机 / 小规模 | docker compose  |
| 多机 / 大规模 | k8s（Kubernetes） |

### 生产四个铁律

| 原则                 | 含义                         |
| ------------------ | -------------------------- |
| ✅ 镜像不可变（immutable） | 镜像构建完不再改；改代码 = 重新构建新镜像再部署  |
| ✅ 容器无状态（stateless） | 容器随时可删可重建；数据放 volume 或云数据库 |
| ✅ 配置走环境变量          | 密码、连接串不写进镜像，运行时注入          |
| ✅ 日志输出 stdout      | 不落盘写文件，由平台统一采集             |

### 极简总结

- 开发：依赖容器化，代码在本机跑。
- 构建：CI 里 build 并 push 到镜像仓库。
- 生产：k8s / compose 部署，镜像不可变、容器无状态。