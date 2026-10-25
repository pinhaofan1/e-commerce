# 新机器部署指南

从一台干净的新机器，到浏览器里能对话，全程约 40 分钟（其中 20 分钟是下载）。

本文以 **Windows 10/11 + WSL2 + Docker Desktop** 为主路径，Linux 服务器见文末附录。

---

## 0. 先看清楚要装什么

| 层 | 组件 | 装在哪 | 大概多大 |
|---|---|---|---|
| 容器运行时 | Docker Desktop | Windows | 1.5 GB |
| 容器 | MySQL / Milvus / etcd / MinIO | Docker 的虚拟磁盘 | 2.7 GB 镜像 |
| Linux 环境 | WSL2 + Ubuntu | Windows | 1 GB |
| Python 运行时 | uv + 依赖 | Ubuntu 内 | 1.5 GB |

**中间件全部在 Docker 里，只有 Python 应用跑在 Ubuntu 上。**

磁盘预留 **15 GB**，内存建议 **16 GB**（Milvus 比较吃内存，8 GB 机器能跑但会卡）。

---

## 1. 前置检查

### 1.1 确认 Windows 版本

按 `Win + R`，输入 `winver`：

- Windows 11：任意版本都行
- Windows 10：需要 2004 以上（内部版本 19041+）

低于这个版本，WSL2 装不了，Docker Desktop 也装不了。

### 1.2 确认虚拟化已开启

任务管理器 → 性能 → CPU，看右下角有没有「虚拟化: 已启用」。

显示「已禁用」的话，重启进 BIOS，把 `Intel VT-x` / `AMD-V`（或叫 SVM Mode）打开。

### 1.3 确认 3306 没被占用

这是本机部署最容易翻车的地方。有装过 MySQL 的机器，它经常开机会自启并抢走 3306。

```powershell
Get-NetTCPConnection -LocalPort 3306 -State Listen -ErrorAction SilentlyContinue
```

有输出就说明被占了，处理见第 10 节。

---

## 2. 装 WSL2 与 Ubuntu

**以管理员身份**打开 PowerShell：

```powershell
wsl --install -d Ubuntu
```

这条命令会自动开启「虚拟机平台」和「适用于 Linux 的 Windows 子系统」两个功能，然后下载 Ubuntu。

装完**必须重启电脑**。

重启后 Ubuntu 会自己弹出来让你设用户名密码。记住这个用户名，后面要用。

验证：

```powershell
wsl -l -v
```

期望看到：

```
  NAME      STATE           VERSION
* Ubuntu    Running         2
```

**`VERSION` 必须是 2**。如果是 1，执行 `wsl --set-version Ubuntu 2` 转换。

### 可选：限制 WSL 内存

8 GB 或 16 GB 内存的机器建议做，否则 WSL 可能吃掉一半内存。

在 `C:\Users\<你的用户名>\.wslconfig` 新建文件：

```ini
[wsl2]
memory=6GB
processors=4
```

改完 `wsl --shutdown` 重启生效。

---

## 3. 装 Docker Desktop

### 3.1 下载安装

到 <https://www.docker.com/products/docker-desktop/> 下载 Windows 版，双击安装。

安装时勾选 **Use WSL 2 instead of Hyper-V**（默认就是勾上的）。

装完启动 Docker Desktop，同意协议，跳过登录（可以不注册账号）。

### 3.2 关键一步：开启 WSL Integration

**这一步不做，Ubuntu 里敲 `docker` 会报 command not found。**

Docker Desktop → 右上角齿轮 **Settings** → **Resources** → **WSL Integration**：

- 打开「Enable integration with my default WSL distro」
- 在下方列表里把 **Ubuntu** 的开关拨到 ON

点 **Apply & Restart**。

### 3.3 验证

关掉所有终端，重新开一个：

```powershell
wsl
```

进入 Ubuntu 后：

```bash
docker version
docker compose version
```

两个都有输出就对了。`docker version` 的 Server 段如果报错，说明 Docker Desktop 没启动，去开始菜单启动它。

### 3.4 把数据目录挪到空间大的盘（强烈建议）

Docker 的镜像默认存在 C 盘，几个项目下来就是几十 GB。现在就挪走：

Settings → **Resources** → **Advanced** → **Disk image location**，改成 `D:\Docker\data` 之类，点 Apply。

**只能在新装、还没有镜像的时候改**，有镜像后改会丢数据。

---

## 4. 配置国内拉取（必做，否则大概率卡住）

Docker Hub 在国内直连基本拉不动，会出现 `i/o timeout` 或卡在 `Pulling fs layer`。

### 方案 A：走代理（推荐，最稳）

你的代理软件在 `127.0.0.1:7897`（Clash 系）的话：

Settings → **Resources** → **Proxies** → 打开 **Manual proxy configuration**，填：

```
http://127.0.0.1:7897
```

下面两行 Web Server (HTTPS) 也填同一个。点 Apply & Restart，然后重启 Docker Desktop。

### 方案 B：配镜像加速器（没有代理时）

Settings → **Docker Engine**，在 JSON 里加 `registry-mirrors`：

```json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerproxy.com"
  ]
}
```

> 说明：国内公共镜像站这两年大量关停，上面的地址不一定长期可用。配完先跑一次 `docker pull hello-world` 试，拉不下来就换成方案 A。

### 验证

```bash
docker pull hello-world
```

20 秒内下完说明通了。

---

## 5. 取代码

```bash
cd ~
git clone https://github.com/a3510420770-star/e-commerce.git pxx
cd pxx
```

> **路径很重要**：一定放在 `~/` 下（也就是 WSL 自己的 ext4 文件系统）。
> 放 `/mnt/c/...` 会让文件读写慢 100 倍，`uv sync` 装包能卡到怀疑人生。

没有 git 的话先装：

```bash
sudo apt update && sudo apt install -y git
```

如果目标机器连不上 GitHub（国内常见），用代理：

```bash
git -c http.proxy=http://<Windows主机IP>:7897 clone https://github.com/a3510420770-star/e-commerce.git pxx
```

---

## 6. 配置环境变量

```bash
cp .env.example .env
```

编辑 `.env`，至少填这四项：

```dotenv
CHAT_BASE_URL=https://api.deepseek.com/v1
CHAT_MODEL=deepseek-v4-flash
CHAT_API_KEY=sk-你的DeepSeek密钥

EMBED_API_KEY=sk-你的硅基流动密钥
RERANK_API_KEY=sk-同一个硅基流动密钥
```

其他保持默认即可，其中两个别动：

| 变量 | 值 | 为什么不能改 |
|---|---|---|
| `COMPOSE_PROJECT_NAME` | `pxx` | 决定容器/网络前缀 |
| `CHAT_THINKING` | `disabled` | DeepSeek V4 思考模式下多轮工具调用会 400 |

`.env` 已经在 `.gitignore` 里，不会被提交。

---

## 7. 启动中间件（就是你要的「拉取」）

```bash
docker compose up -d
```

**第一次会下载 4 个镜像，共 2.7 GB**，视网速 5～20 分钟。

期望输出：

```
 ✔ Container pxx-milvus-etcd   Started
 ✔ Container pxx-milvus-minio  Started
 ✔ Container pxx-mysql         Started
 ✔ Container pxx-milvus        Started
```

下载的是这四个：

| 镜像 | 大小 | 作用 |
|---|---|---|
| `mysql:8` | 1.12 GB | 业务库 |
| `milvusdb/milvus:v2.6.22` | 1.3 GB | 向量库 |
| `quay.io/coreos/etcd:v3.5.16` | 85 MB | Milvus 的元数据 |
| `milvusdb/minio:...` | 220 MB | Milvus 的对象存储 |

再执行一次会发现是秒级返回——镜像已在本地，Docker 只做状态检查。

---

## 8. 等待就绪并验证

Milvus 启动要 60～90 秒，MySQL 要建表。

```bash
watch -n 3 docker compose ps
```

等到全部 `healthy`（Ctrl+C 退出 watch）：

```
pxx-mysql          Up 40 seconds (healthy)
pxx-milvus         Up 90 seconds (healthy)
pxx-milvus-etcd    Up 90 seconds (healthy)
pxx-milvus-minio   Up 90 seconds (healthy)
```

### 验证 MySQL 表建好了

首次启动会自动执行 `sql/` 下 12 个脚本：

```bash
docker exec pxx-mysql mysql -uroot -proot \
  -N -e 'select count(*) from information_schema.tables where table_schema="pxx";'
```

**期望输出 `13`**。如果是 0，看第 10 节「表没建出来」。

### 验证 Milvus

```bash
curl -s http://localhost:9091/healthz
```

期望返回 `ok`。

---

## 9. 跑起应用

### 9.1 装 Python 依赖

```bash
# 没装 uv 的话
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc

# 装依赖，必须指定 3.12
uv sync -p 3.12
```

> 为什么钉 3.12：`asyncmy` 在 Python 3.14 上没有预编译 wheel，会现场编译并失败。

国内网络慢的话加镜像：

```bash
UV_DEFAULT_INDEX=https://pypi.tuna.tsinghua.edu.cn/simple uv sync -p 3.12
```

### 9.2 灌数据

```bash
make seed        # 6 个订单
make seed-conv   # 2 个历史会话
make kb-build    # 知识库切块，52 块
make kb-vectorize # 向量化写 Milvus，要调嵌入接口
```

`kb-vectorize` 这一步会调硅基流动接口，需要外网。跑完应该看到：

```
✅ 本次向量化 52 块;Milvus 现有 52 条
```

### 9.3 启动

```bash
make dev
```

这会拉起两个 MCP Server（:8101 / :8102）和 FastAPI（:8000）。**前台常驻**，Ctrl+C 停。

### 9.4 验收

另开一个终端：

```bash
curl -s http://localhost:8000/ -o /dev/null -w '%{http_code}\n'
```

期望 `200`。然后浏览器打开 <http://localhost:8000>，问一句：

> 订单 1001 的物流到哪了

回答里应该有物流单号 `SF213502378238`、状态「已揽件」、位置「成都」。

---

## 10. 故障排查

### 3306 被占用

```
Error response from daemon: ports are not available: exposing port TCP 0.0.0.0:3306
```

本机有 MySQL 在跑。查占用者并停掉：

```powershell
Get-NetTCPConnection -LocalPort 3306 -State Listen | Select-Object OwningProcess
Get-Process -Id <上面查到的PID>
```

确认是 `mysqld` 后：

```powershell
Stop-Service MySQL        # 需要管理员权限
# 或
taskkill /PID <PID> /F
```

**每次开机都会复发**。根治办法：`services.msc` 找到 `MySQL`，把启动类型改成「手动」。

### 表没建出来（getTables 返回 0）

`sql/` 的初始化**只在数据卷为空时执行一次**。如果卷已经存在但表是空的，脚本不会重跑。

```bash
docker compose down
docker volume rm pxx-mysql-data
docker compose up -d
```

### Milvus 一直 unhealthy

首次启动慢，先等满 2 分钟。还不好的话：

```bash
docker logs pxx-milvus --tail 50
```

常见原因：内存不够（Milvus 至少要 4 GB 可用）、磁盘满。

### Ubuntu 里 docker: command not found

Docker Desktop 的 **WSL Integration** 没开。回到第 3.2 节，勾上 Ubuntu 并 Apply & Restart。改完必须重开终端。

### 镜像拉不下来 / manifest unknown

三种可能：

1. 网络问题 → 配代理（第 4 节）
2. 镜像被上游删了 → 换源（本项目的 `milvusdb/minio` 就是因为官方删了 `minio/minio` 才换的）
3. tag 写错 → `docker manifest inspect <镜像名>` 检查是否存在

### 应用起来了但问答报 400 `reasoning_content`

`.env` 里的 `CHAT_THINKING` 没设成 `disabled`。DeepSeek V4 思考模式下，多轮工具调用要求回传推理内容，langchain 不回传就会 400。

---

## 附录：Linux 服务器（无 Windows）

```bash
# 1. 装 Docker Engine（官方脚本）
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER   # 免 sudo，重新登录生效

# 2. 装 compose 插件
sudo apt install -y docker-compose-plugin

# 3. 后面的步骤完全一样
git clone https://github.com/a3510420770-star/e-commerce.git pxx
cd pxx && cp .env.example .env && vim .env
docker compose up -d
```

Linux 上没有 WSL 那层，`~/pxx` 本来就是原生文件系统，不用考虑路径性能问题。

如果是云服务器，记得在安全组放行 **8000**（应用）和 **19530**（Milvus，如果要从外部连）。3306 建议不要对外暴露。

---

## 一句话速查

```bash
# 最短路径
wsl --install -d Ubuntu                          # Windows 侧，管理员 PowerShell
# 装 Docker Desktop，开 WSL Integration，配代理
git clone <repo> ~/pxx && cd ~/pxx
cp .env.example .env && vim .env                 # 填 3 个 key
docker compose up -d                             # 拉 2.7 GB 镜像
uv sync -p 3.12
make seed && make seed-conv && make kb-build && make kb-vectorize
make dev                                         # 打开 http://localhost:8000
```
