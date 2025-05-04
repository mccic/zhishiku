# MySQL MCP 节点 (mysql_mcp_node) 配置指南

## 概述

本文档旨在指导如何在 Cursor IDE 中正确配置和使用 `mcp-mysql-server` 作为 MySQL 数据库的 MCP（Model Control Protocol）工具节点。该工具允许你通过自然语言与 Cursor 交互来执行 SQL 查询、管理表结构等操作。

此配置基于将 `mcp-mysql-server` 运行在 Docker 容器中，并依赖 Docker Compose 来管理 MySQL 服务和 MCP 工具容器。

## 先决条件

-   **Git:** 用于克隆 `mcp-mysql-server` 仓库。
-   **Docker:** 用于运行容器。
-   **Docker Compose:** 用于编排 Docker 服务 (MySQL 和 MCP 工具容器)。
-   **Cursor IDE:** 用于配置和使用 MCP 服务。

## 核心概念 (重要!)

理解以下概念对于避免配置错误至关重要：

1.  **MCP 服务类型:** `mcp-mysql-server` 是一个**基于 `stdio` (标准输入/输出)** 的 MCP 工具，它**不是**一个 HTTP 服务。它被设计为由客户端（Cursor）按需启动，并通过 `stdio` 进行通信，完成后即退出。
2.  **Docker Compose 角色:**
    *   主要负责**运行依赖服务**，即 `mysql` 数据库容器。
    *   负责**构建** `mcp-mysql-server` 的 Docker 镜像。
    *   负责**提供一个持续运行的容器环境**，让 Cursor 的 `exec` 命令可以在其中执行 Node.js 脚本。
3.  **Cursor MCP 配置 (`~/.cursor/mcp.json`):**
    *   定义了 Cursor 如何**按需启动**并与 MCP 工具**通信**。
    *   我们使用 `docker-compose exec` 命令，因为它可以在指定的 Docker Compose 服务对应的容器内部执行命令。
4.  **`docker-compose exec` 的要求:** 此命令需要目标服务的容器处于**正在运行**的状态。
5.  **保持容器运行:** 由于 `mcp-mysql-server` 的 Node.js 脚本作为 `stdio` 工具执行完会自然退出，而 `docker-compose exec` 需要一个运行中的容器，我们采用一种技巧：在 `docker-compose.yml` 中**覆盖容器的入口点 (`entrypoint`) 和命令 (`command`)**，让它运行 `tail -f /dev/null`。这会使容器启动后保持空闲运行状态，为 `docker-compose exec` 提供一个稳定的执行目标。

## 设置步骤

### 1. 克隆源码

在你的项目工作区（例如 `/Users/kaifa/AIBUBB`）或合适的位置克隆 `mcp-mysql-server` 仓库：

```bash
git clone https://github.com/enemyrr/mcp-mysql-server.git
```

### 2. 配置 `docker-compose.yml`

确保你的 `docker-compose.yml` 文件（通常位于项目根目录）包含以下服务定义。**关键在于 `mcp-mysql-server` 服务的 `entrypoint` 和 `command` 配置**。

```yaml
version: "3.8" # 版本号可选，新版 Compose 会忽略

services:
  # --- 其他服务 (例如 backend, redis) ---
  # ... 保留你现有的 backend 和 redis 配置 ...

  # --- MySQL 数据库服务 ---
  mysql:
    image: mysql:8.0
    container_name: aibubb-mysql # 确保容器名一致 (可选但推荐)
    restart: always
    ports:
      - "3306:3306"
    # 使用环境变量定义数据库凭据
    environment:
      MYSQL_ROOT_PASSWORD: secret # 请使用强密码
      MYSQL_DATABASE: demo_db
      MYSQL_USER: demo
      MYSQL_PASSWORD: demo_pass # 请使用强密码
    volumes:
      - mysql_data:/var/lib/mysql
    networks:
      - aibubb-network
    # 健康检查确保 MySQL 完全启动 (可选但推荐)
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "demo", "-pdemo_pass"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # --- MCP MySQL 工具服务 ---
  mcp-mysql-server:
    build:
      # 指向克隆下来的 mcp-mysql-server 目录
      context: ./mcp-mysql-server
      dockerfile: Dockerfile
    # 传递数据库连接 URL 给 Node 脚本 (当被 exec 启动时)
    environment:
      DATABASE_URL: mysql://demo:demo_pass@mysql:3306/demo_db
    # 不再需要端口映射
    # depends_on 确保先启动 mysql
    depends_on:
      - mysql
    networks:
      - aibubb-network
    # !!! 关键：覆盖入口点和命令，让容器保持运行 !!!
    entrypoint: tail
    command: ["-f", "/dev/null"]

# --- 数据卷和网络定义 ---
volumes:
  mysql_data:
    name: aibubb-mysql-data # 统一命名 (可选)
  # ... 其他卷 (例如 redis-data) ...

networks:
  aibubb-network:
    name: aibubb-network # 统一命名 (可选)
    driver: bridge
```

**注意:**
*   将 `MYSQL_ROOT_PASSWORD` 和 `MYSQL_PASSWORD` 替换为安全的密码。
*   确保 `mcp-mysql-server` 的 `context` 指向你克隆仓库的正确相对路径。
*   确保 `DATABASE_URL` 中的主机名是 `mysql`（或其他你在 Docker Compose 中为 MySQL 服务定义的名称），并且用户名、密码、数据库名正确。
*   所有需要互相通信的服务（`mysql`, `mcp-mysql-server`, `backend` 等）都应连接到同一个网络 (`aibubb-network`)。

### 3. 启动 Docker 服务

在包含 `docker-compose.yml` 的目录下，运行以下命令启动所有服务。由于 `mcp-mysql-server` 配置了 `tail` 命令，它会启动并保持运行。

```bash
docker-compose up -d
```

你可以通过 `docker-compose ps` 或 Docker Desktop/扩展 确认 `mysql` 和 `mcp-mysql-server`（以及你的 `backend`, `redis` 等）都处于运行状态。`mcp-mysql-server` 的命令应该显示为 `tail -f /dev/null`。

### 4. 配置 Cursor (`~/.cursor/mcp.json`)

编辑你的**全局** Cursor MCP 配置文件 `~/.cursor/mcp.json` (或者如果只想用于当前项目，编辑项目下的 `.cursor/mcp.json`)。确保它包含以下 `mysql_mcp_node` 的定义：

```json
{
  "mcpServers": {
    // ... 其他 MCP 服务配置 (如 figma_composio, context7) ...

    "mysql_mcp_node": {
      "command": "docker-compose", // 主命令
      "args": [                  // 参数列表
        "-f",                    // 指定 compose 文件
        "/Users/kaifa/AIBUBB/docker-compose.yml", // !! 替换为你的 docker-compose.yml 绝对路径 !!
        "exec",                  // 执行命令
        "-T",                    // 禁用 TTY (推荐)
        "mcp-mysql-server",      // Docker Compose 服务名
        "node",                  // 在容器内要执行的命令
        "/app/build/index.js"    // Node 脚本在容器内的路径
      ]
      // 不需要 "env" 或 "url"
    }
  }
}
```

**重要:**
*   确保 `"command"` 是 `"docker-compose"`。
*   确保 `"args"` 是一个包含所有参数的**数组**。
*   将 `"/Users/kaifa/AIBUBB/docker-compose.yml"` **替换为你环境中 `docker-compose.yml` 文件的实际绝对路径**。使用绝对路径可以确保 Cursor 在任何工作目录下都能正确找到配置文件。
*   确保服务名 `mcp-mysql-server` 与 `docker-compose.yml` 中定义的服务名一致。
*   确保 Node 脚本路径 `/app/build/index.js` 与 `mcp-mysql-server` 的 Dockerfile 中定义的工作目录和构建输出路径一致。

### 5. 重启 Cursor

保存 `~/.cursor/mcp.json` 文件后，**完全退出并重新启动 Cursor** 以加载新的配置。

### 6. 验证连接

打开 Cursor Settings → Features → MCP，检查 `mysql_mcp_node`：
*   应该显示**绿色指示灯**。
*   展开后应该列出可用的工具 (`connect_db`, `query`, `execute` 等)。
*   显示的 Command 应该类似 `docker-compose -f /path/to/your/docker-compose.yml exec -T mcp-mysql-server node /app/b...`。

## 故障排除

-   **错误: "Client closed" 或 "Failed to create client"**
    1.  **检查容器状态:** 运行 `docker-compose ps` 确认 `mcp-mysql-server` 容器**正在运行**，并且其命令是 `tail -f /dev/null`。如果未运行，检查 `docker-compose.yml` 中的 `entrypoint` 和 `command` 配置是否正确，然后尝试 `docker-compose up -d --force-recreate mcp-mysql-server`。
    2.  **检查 `mcp.json` 命令:** 确认 `~/.cursor/mcp.json` 中的 `command` 和 `args` 严格按照上述格式配置，特别是 `docker-compose.yml` 的**绝对路径**是否正确。
    3.  **检查 Docker Daemon:** 确保 Docker 服务本身正在运行。
    4.  **手动测试 `exec`:** 在终端的项目目录下尝试手动运行 `mcp.json` 中的 `docker-compose exec ... node ...` 命令（不包括 `-f` 参数），看是否能连接并打印 `MySQL MCP server running on stdio`（然后按 Ctrl+C 退出）。如果手动执行失败，可能是 Docker 或容器内部问题。
-   **JSON 语法错误:** 仔细检查 `~/.cursor/mcp.json` 文件，确保它是有效的 JSON 格式（括号匹配、逗号使用正确等）。确保没有项目级的 `.cursor/mcp.json` 文件包含错误或冲突的配置。

遵循此指南应该可以稳定地配置好 `mysql_mcp_node` 服务。
