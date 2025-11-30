# Ubuntu 24.04 环境下的 IAM 安装迁移评估与可执行方案

本文档提供将原生 CentOS 8.x 自动化安装流程迁移到 Ubuntu 24.04 的评估结果，并给出在当前容器环境可执行的部署步骤（已验证官方源可用，`apt-get update` 正常）。

## 现状评估

- **操作系统差异**：现有安装脚本大量使用 `yum`、CentOS repo 和 RPM 包，不适用于 Ubuntu 的 `apt`/`deb` 生态。
- **网络情况**：当前容器可直接访问官方 Ubuntu 源，无需额外更换镜像即可拉取软件包。
- **已具备工具**：容器内已预装 Go 1.25.1，可满足 `go 1.21+` 的编译需求，避免单独安装 Go。
- **缺失组件**：默认未安装数据库/缓存服务（MariaDB、Redis、MongoDB）；MongoDB 仍需视网络情况选择官方源或容器化方式。

## 依赖映射（CentOS → Ubuntu）

| 功能 | CentOS 8.x 脚本中的组件 | Ubuntu 24.04 对应组件/包 | 说明 |
| ---- | ---------------------- | ------------------------- | ---- |
| 基础构建工具 | `make`, `gcc`, `gcc-c++`, `cmake`, `autoconf`, `automake`, `libtool`, `glibc-headers`, `zlib-devel` | `build-essential`, `cmake`, `autoconf`, `automake`, `libtool`, `zlib1g-dev` | `build-essential` 已包含 gcc/g++/glibc headers |
| 常用工具 | `git-lfs`, `telnet`, `lrzsz`, `jq`, `expat-devel`, `openssl-devel`, `libcurl-devel` | `git-lfs`, `telnet`, `lrzsz`, `jq`, `libexpat1-dev`, `libssl-dev`, `libcurl4-openssl-dev` | 如无需 `lrzsz` 可省略 |
| Go 环境 | 手动下载 `go1.18.3` | Ubuntu 可直接使用现有 Go（1.25.1）；如需固定版本，可用官方 tar 包 | GOPROXY 建议设为国内镜像 |
| Protobuf | 源码编译 `protoc 3.21.1` + `protoc-gen-go` | `protobuf-compiler` + `protoc-gen-go` (go install) | apt 包版本可能略新，可接受 |
| 数据库 | `mariadb-server` | `mariadb-server` | 注意默认 root 用户无密码，需手动设置 |
| 缓存 | `redis` | `redis-server` | 默认监听 127.0.0.1，生产需调优 |
| MongoDB | `mongodb` | `mongodb-org`（官方源）或 `mongodb`（Ubuntu 源） | 若官方源不可用，可使用 `podman/docker` 拉取社区镜像 |

## 可执行的 Ubuntu 部署步骤
以下步骤假设你拥有 `sudo` 权限，且可直接访问 Ubuntu 官方软件源（当前容器已验证可用）。

1. **更新 apt 索引**：
   ```bash
   sudo apt-get update
   ```

2. **安装系统依赖**：
   ```bash
   sudo apt-get install -y build-essential cmake autoconf automake libtool pkg-config \
       zlib1g-dev libssl-dev libcurl4-openssl-dev libexpat1-dev jq git-lfs telnet lrzsz \
       protobuf-compiler redis-server mariadb-server
   # MongoDB：Ubuntu 24.04 默认仓库无 mongodb 元包，可考虑配置官方 mongodb-org 源，
   # 或使用 docker/podman 直接拉取镜像（例如 docker run -p27017:27017 mongo:6）。
   ```

3. **可选：安装/固定 Go 版本（若不使用系统 Go）**：
   ```bash
   GO_VERSION=1.21.6
   wget https://go.dev/dl/go${GO_VERSION}.linux-amd64.tar.gz
   sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go${GO_VERSION}.linux-amd64.tar.gz
   echo 'export PATH=/usr/local/go/bin:$PATH' >> ~/.bashrc
   source ~/.bashrc
   ```

4. **配置 Go 模块代理（提升下载成功率）**：
   ```bash
   echo 'export GOPROXY=https://goproxy.cn,direct' >> ~/.bashrc
   source ~/.bashrc
   ```

5. **获取代码并编译**（当前仓库已存在，可直接在 `/workspace/iam` 使用）：
   ```bash
   cd /workspace/iam
   export GOPROXY=https://goproxy.cn,direct  # 提升 go mod 下载成功率
   make build  # 或者 make apiserver/authz-server 等目标
   ```

6. **初始化数据库与缓存**：
   - MariaDB：创建数据库、用户并导入 `configs/db/init.sql`（如有）。
   - Redis：默认即可启动，生产可调整 `/etc/redis/redis.conf`。
   - MongoDB：若通过包或容器安装，确保监听端口与 `configs` 中的连接串一致。

7. **运行服务示例**：
   ```bash
   # 使用 configs 中的示例配置启动 apiserver
   ./_output/platforms/linux/amd64/iam-apiserver -c configs/iam-apiserver.yaml
   ```

8. **启动与验证数据库/缓存服务（当前容器已验证可启动）**：
   ```bash
   sudo service redis-server start
   # policy-rc.d 会阻止 MariaDB 自动启动，可用 init 脚本拉起
   sudo /etc/init.d/mariadb start
   ```

## 实测：在 Ubuntu 24.04 容器内完成 apiserver 启动
下面的命令均已在当前容器实测通过，可直接复制执行：

1. **初始化 MariaDB**（注意 `!` 需要转义或使用单引号）：
   ```bash
   mysql -uroot -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'iam59\!z$';"
   mysql -uroot -p'iam59!z$' -e 'GRANT ALL ON iam.* TO iam@127.0.0.1 IDENTIFIED BY "iam59!z$"; FLUSH PRIVILEGES;'
   mysql -h127.0.0.1 -uiam -p'iam59!z$' < configs/iam.sql
   ```

2. **编译并准备配置**：
   ```bash
   export GOPROXY=https://goproxy.cn,direct
   make build

   # 渲染配置文件（使用 envsubst 展开 YAML 里的环境变量占位符）
   mkdir -p _output/configs _output/logs
   export MARIADB_HOST=127.0.0.1:3306 MARIADB_USERNAME=iam MARIADB_PASSWORD='iam59!z$' MARIADB_DATABASE=iam \
          REDIS_HOST=127.0.0.1 REDIS_PORT=6379 REDIS_PASSWORD= \
          IAM_LOG_DIR=$(pwd)/_output/logs \
          IAM_APISERVER_GRPC_BIND_ADDRESS=0.0.0.0 IAM_APISERVER_GRPC_BIND_PORT=8081 \
          IAM_APISERVER_INSECURE_BIND_ADDRESS=0.0.0.0 IAM_APISERVER_INSECURE_BIND_PORT=8080 \
          IAM_APISERVER_SECURE_BIND_ADDRESS=0.0.0.0 IAM_APISERVER_SECURE_BIND_PORT=8443 \
          IAM_APISERVER_SECURE_TLS_CERT_KEY_CERT_FILE=$(pwd)/configs/cert/iam.pem \
          IAM_APISERVER_SECURE_TLS_CERT_KEY_PRIVATE_KEY_FILE=$(pwd)/configs/cert/iam-key.pem
   envsubst < configs/iam-apiserver.yaml > _output/configs/iam-apiserver.yaml
   ```

3. **启动 apiserver 并查看日志**：
   ```bash
   ./_output/platforms/linux/amd64/iam-apiserver -c _output/configs/iam-apiserver.yaml > /tmp/iam-apiserver.log 2>&1 &
   tail -n 20 /tmp/iam-apiserver.log  # 日志中能看到 /healthz 返回 ok 即启动成功
   # 需要停止时执行：pkill -f iam-apiserver
   ```

## 运行状态与访问地址

- **当前容器默认并未运行 IAM 进程**：可用 `pgrep -af iam-apiserver` 或 `ps -ef | grep iam-` 验证；若无输出即未启动。
- **如何检查服务是否成功启动**：
  - 观察日志：`tail -f /tmp/iam-apiserver.log`，应出现 `"GET /healthz" 200` 或 "healthz ok" 日志行。
  - HTTP 健康检查：`curl -i http://127.0.0.1:8080/healthz`，返回 200/`ok` 表示就绪。
  - HTTPS（如启用）：`curl -k https://127.0.0.1:8443/healthz`。
- **默认监听地址**（见 `configs/iam-apiserver.yaml`）：
  - GRPC：`0.0.0.0:${IAM_APISERVER_GRPC_BIND_PORT:-8081}`
  - HTTP：`0.0.0.0:${IAM_APISERVER_INSECURE_BIND_PORT:-8080}`
  - HTTPS：`0.0.0.0:${IAM_APISERVER_SECURE_BIND_PORT:-8443}`（如端口设为 0 则未启用）
- **容器外访问提示**：本容器未暴露端口给宿主，如需从外部访问，请在容器启动时通过 `-p 8080:8080 -p 8081:8081 -p 8443:8443` 进行端口映射，或将绑定地址改为宿主可达的接口。

## 备注
- 如受网络限制无法安装 MongoDB，可优先启动与 MongoDB 无关的组件进行部分功能验证，或使用容器化 MongoDB。
