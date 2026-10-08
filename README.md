# mcp-local-search-integration

## 这是什么

这是一个在 Windows 10 LTSC + WSL2 (Ubuntu) 环境下，将本地免费搜索服务（ai-search-mcp）通过 supergateway 桥接为 HTTP/StreamableHTTP 端点，同时供本地支持MCP协议的应用例如： Dify 、 DeepSeek Harness 等调用的完整操作记录。内容涵盖架构说明、WSL2 安装 Node.js、ai-search-mcp 说明、supergateway 桥接、Dify 与 DSH 两端接入、自动启动脚本、故障排查与安全规范。

## 解决了什么问题

- **搜索服务按次计费问题**：用免费的 ai-search-mcp 替代 Tavily 等按次计费的搜索 API，实现零成本联网搜索。
- **MCP 协议不兼容问题**：ai-search-mcp 只支持 stdio 模式，通过 supergateway 桥接为 StreamableHTTP/SSE，Dify 和 DeepSeek Harness 都能调用。
- **一次配置、多端复用**：同一套搜索服务同时供 Dify 和 DSH 等支持MCP的工具使用，无需重复部署。
- **国内网络直连**：通过 `SEARCH_REGION=cn-zh` 强制走 Bing、百度、360 等国内可直连引擎，无需代理。
- **长连接会话保活**：supergateway 使用 `--stateful` 保持子进程常驻，避免 DSH 报 Connection error。
- **自动化启动**：进入 WSL 终端时自动拉起服务，无需手动执行。

## 怎么用

下面按步骤操作即可。环境准备 → 安装 Node.js → 启动 supergateway → 接入 Dify → 接入 DeepSeek Harness → 配置自动启动。每一步都附有命令说明。如遇问题，可查阅「踩坑记录」章节。

## 环境准备

- 操作系统：Windows 10 IoT 企业版 LTSC 21H2
- WSL2：已启用，发行版 Ubuntu
- Node.js：WSL2 内安装 Node.js >= 18（推荐 20）
- 容器运行时：WSL2 内安装 Docker 或 Podman（已有 Dify 运行）
- 客户端：Dify 或 DeepSeek Harness 至少一个

## 架构说明

```
┌─────────────────┐     ┌──────────────┐     ┌─────────────────┐
│ Dify (容器)     │────▶│ supergateway │────▶│ ai-search-mcp   │
│ DSH (Windows)   │────▶│   :11000     │     │ (stdio 模式)    │
└─────────────────┘     └──────────────┘     └─────────────────┘
```

- **ai-search-mcp**：stdio 模式的 MCP 服务器，提供 search / fetch_page / research / status 四个工具。使用 Bing、百度、360 等免费引擎，无需 API Key。
- **supergateway**：将 stdio 桥接为 StreamableHTTP/SSE，监听 11000 端口。
- **客户端**：Dify（容器内，用 host.containers.internal 访问）和 DeepSeek Harness（Windows 侧，用 WSL2 实际 IP 访问）。

## 部署步骤

### 1. WSL2 安装 Node.js

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node -v   # 应显示 v20.x
```

### 2. 测试 ai-search-mcp 是否能正常运行

```bash
npx -y ai-search-mcp --help
```

若正常，会输出 usage 信息。ai-search-mcp 是 stdio 模式，不能直接监听 HTTP，必须通过 supergateway 桥接。

### 3. 启动 supergateway 桥接

```bash
SEARCH_REGION=cn-zh nohup npx -y supergateway \
  --stdio "npx -y ai-search-mcp" \
  --port 11000 \
  --outputTransport streamableHttp \
  --stateful \
  --baseUrl "http://$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -n1):11000" \
  > ~/supergateway.log 2>&1 &
```

确认端口监听：

```bash
ss -tln | grep 11000
```

### 4. 接入 Dify

- 进 Dify 工作空间 → 工具 → MCP → 添加 MCP 服务器 (HTTP)
- **服务端点 URL**：`http://host.containers.internal:11000/sse`
- **名称**：`free-web-search`
- **服务器标识符**：`free-web-search`
- 认证：留空
- 使用动态客户端注册：关闭
- 点「添加并授权」

成功后，Dify 会列出 4 个工具：`search`、`fetch_page`、`research`、`status`。

### 5. 接入 DeepSeek Harness

**方式一：使用 dsh-mcp-manager 插件（图形界面，推荐）**

安装插件（需系统有 Git）：

```
https://github.com/JokerAn/dsh-mcp-manager
```

安装后完全退出 DSH 再重新打开。进入 设置 → 内置插件 → MCP → 添加：

- **服务器名称**：`free-search`
- **显示名称**：`免费搜索`
- **端点 URL**：`http://<WSL2_IP>:11000/mcp`
- **传输方式**：流式 HTTP 服务器

**方式二：手动编辑 cordis.patch.yml**

文件位置：`C:\Users\<用户名>\.dsh\profiles\desktop\cordis.patch.yml`

```yaml
- insert:
    - id: mcp-free-search
      name: '@deepseek-ai/dsh-mcp-client'
      config:
        serverName: free-search
        transport: streamable-http
        url: http://<WSL2_IP>:11000/mcp
        toolCallTimeoutMs: 60000
        failOnStartupError: false
```

保存后完全退出 DSH 再重新打开。

### 6. 配置自动启动

创建 `~/start-supergateway.sh`：

```bash
#!/bin/bash
# MCP 本地免费搜索 - 启动脚本

set -e

PORT=11000
LOG_FILE=~/supergateway.log
SEARCH_REGION=cn-zh

WSL_IP=$(ip -4 addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}' | head -n1)
if [ -z "$WSL_IP" ]; then
  echo "[ERROR] 无法获取 WSL2 IP，跳过 supergateway 启动"
  exit 1
fi

if ss -tln | grep -q ":${PORT} "; then
  echo "supergateway 已在运行，跳过启动"
  echo "  监听端口：${PORT}"
  echo "  WSL2 IP：${WSL_IP}"
  exit 0
fi

echo "正在启动 supergateway..."
SEARCH_REGION=${SEARCH_REGION} nohup npx -y supergateway \
  --stdio "npx -y ai-search-mcp" \
  --port ${PORT} \
  --outputTransport streamableHttp \
  --stateful \
  --baseUrl "http://${WSL_IP}:${PORT}" \
  > ${LOG_FILE} 2>&1 &

sleep 2

if ss -tln | grep -q ":${PORT} "; then
  echo "supergateway 已启动"
  echo "  WSL2 IP：${WSL_IP}"
  echo "  监听端口：${PORT}"
  echo "  日志文件：${LOG_FILE}"
  echo ""
  echo "客户端配置："
  echo "  Dify MCP URL： http://host.containers.internal:${PORT}/sse"
  echo "  DSH  MCP URL： http://${WSL_IP}:${PORT}/mcp"
else
  echo "[ERROR] supergateway 启动失败，请查看日志：${LOG_FILE}"
  exit 1
fi
```

赋予执行权限：

```bash
chmod +x ~/start-supergateway.sh
```

在 `~/.bashrc` 末尾添加：

```bash
if [[ $- == *i* ]]; then
  ~/start-supergateway.sh
fi
```

下次打开 WSL 终端时自动启动服务。

## 可用工具

| 工具 | 功能 | 适用场景 |
|---|---|---|
| **search** | 单次搜索，返回结构化结果 | 简单查询，需要快速结果 |
| **fetch_page** | 抓取指定 URL 内容转 Markdown | 已知网页，需要提取正文 |
| **research** | 一次调用完成"搜索 + 抓取 Top 页 + 生成证据简报" | 复杂查询，需要综合分析 |
| **status** | 诊断引擎健康状态、缓存、运行配置 | 搜索失败时排查 |

## 踩坑记录

| 问题 | 现象 | 原因 | 解决方法 |
|---|---|---|---|
| ai-search-mcp 无法直接 HTTP 监听 | 进程在跑，但 ss 查不到端口，curl 拒绝 | ai-search-mcp 是 stdio 模式，不支持 HTTP | 用 supergateway 桥接为 HTTP |
| DSH 报 Connection error | DSH 发请求后连接断开 | supergateway 默认非 stateful，子进程被 SIGTERM 杀死 | 加 --stateful 参数 |
| DSH 启动崩溃 | patch.insert?.forEach is not a function | cordis.patch.yml 的 insert 不是数组 | insert 后面必须是数组，每个元素以 - 开头 |
| DSH 安装插件失败 | 'git' 不是内部或外部命令 | Windows 未安装 Git | 安装 Git for Windows，PATH 选 "Git from the command line and also from 3rd-party software" |
| Dify 连不上 MCP | Internal Server Error | Dify 容器网络与宿主机隔离 | Dify 用 host.containers.internal，DSH 用 WSL2 实际 IP |
| 中文搜索返回空 | 搜索无结果 | 默认引擎选择可能走海外 | 设置 SEARCH_REGION=cn-zh |
| WSL2 重启后 DSH 断连 | 搜索报错 | WSL2 IP 变化，DSH 的 URL 失效 | 查新 IP，更新 DSH 的 MCP URL |
| supergateway 被终端关闭杀死 | 服务突然停止 | WSL 实例关闭或终端退出 | 至少保留一个 WSL 终端窗口，或改用 setsid |

## 长期使用建议

- **WSL2 终端别全关**：至少保留一个终端窗口，避免 WSL 实例自动关闭。
- **WSL2 重启后同步 IP**：查 `ip addr show eth0 | grep inet`，更新 DSH 的 MCP URL。
- **Dify 端不用改**：Dify 用 `host.containers.internal`，不受 IP 变化影响。
- **看日志**：搜索异常时先 `tail -f ~/supergateway.log`，看请求是否到达、引擎是否切换。
- **防火墙规则**：11000 端口仅对 WSL2 网段（172.29.0.0/16）开放，禁止对公网暴露。
- **服务自动启动**：通过 `~/.bashrc` 调用 `~/start-supergateway.sh`，进 WSL 自动拉起。

## 安全提醒

- 11000 端口仅对 WSL2 网段开放，Windows 防火墙规则限定 `172.29.0.0/16`。
- Dify 用 `host.containers.internal` 访问，DSH 用 WSL2 实际 IP。
- 不要将 11000 端口对公网开放。
- 不要将 Dify 或 Ollama 的端口对公网开放。
- 云端模型处理敏感信息时，注意不要上传敏感数据。

## 许可

MIT License

Copyright (c) 2026 Author

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
