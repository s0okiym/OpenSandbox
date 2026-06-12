# OpenSandbox 简明介绍

## 一句话定位

OpenSandbox 是阿里巴巴开源的**通用 AI 沙箱平台**，提供多语言 SDK、统一生命周期 API、Docker / Kubernetes 双运行时，以及命令 / 文件 / 代码解释器 / 浏览器 / 桌面等沙箱环境，面向 Coding Agent、GUI Agent、AI 代码执行、RL 训练等场景。

---

## 1. 核心架构（六层）

| 层级 | 说明 | 对应目录/组件 |
|---|---|---|
| 客户端 | Python / JS·TS / Java·Kotlin / C# / Go SDK、`osb` CLI、MCP Server | `sdks/`、`cli/`、`sdks/mcp/` |
| 协议层 | OpenAPI 规范，所有 SDK 与实现的契约 | `specs/` |
| 控制平面 | FastAPI 生命周期服务器，负责鉴权、校验、编排 | `server/` |
| 运行时后端 | Docker（本地/单节点）、Kubernetes（BatchSandbox / agent-sandbox） | `server/services/`、`kubernetes/` |
| 沙箱数据平面 | 用户容器 + `execd` 执行守护进程 + 可选 Jupyter/Code Interpreter + 卷 | `components/execd/`、`sandboxes/` |
| 网络与安全平面 | Ingress 网关、Egress Sidecar、凭证保险库、安全容器运行时 | `components/ingress/`、`components/egress/` |

设计原则：**协议优先、控制平面与数据平面分离、运行时无关 API + 运行时相关实现**。

---

## 2. 关键机制与功能

### 2.1 生命周期 API

- 创建 / 列表 / 查询 / 删除沙箱
- 暂停（Pause）、恢复（Resume）、续期（Renew Expiration）
- 快照（Snapshot）：从运行沙箱生成持久镜像，再从快照创建新沙箱
- 端口端点解析：把沙箱内服务端口暴露为外部可访问地址

### 2.2 沙箱内执行（execd）

`execd` 是注入沙箱内部的 Go/Gin 守护进程，提供：

- 命令执行（前台/后台、SSE 流式输出）
- 文件/目录 CRUD
- 持久 Bash Session、交互式 PTY（WebSocket）
- Jupyter 代码解释器（Python / Java / Node / Go / Bash）
- CPU / 内存指标

### 2.3 网络策略

- **Egress Sidecar**：按 FQDN、通配符、CIDR、IP 控制出站流量；支持 `dns` 与 `dns+nft` 两种模式
- **Ingress 网关**：Kubernetes 场景下 HTTP/WebSocket 反向代理，支持 Header / URI / 通配子域名路由
- **Secure Access**：为 Ingress 端点签发凭证与签名路由令牌
- **自动续期**：访问流量触发 TTL 续期（可选）

### 2.4 Credential Vault（凭证保险库）

- 真实凭证写在宿主机侧，通过 SDK 写入 Egress Sidecar
- 沙箱内只持有假凭证或空值
- 出站 HTTPS 请求经过 Sidecar 透明 MITM，按 scheme/host/port/method/path 匹配后自动注入鉴权头
- 降低 Prompt 注入或恶意代码导致的凭据泄露风险

### 2.5 安全容器运行时

管理员在 `~/.sandbox.toml` 中统一配置，**SDK/用户无感知**：

| 运行时 | 隔离机制 | 启动开销 | 内存开销 | 适用场景 |
|---|---|---|---|---|
| runc（默认） | 进程级 cgroup | ~0 ms | 极小 | 本地开发、可信负载 |
| gVisor | 用户态内核 / 系统调用拦截 | ~10–50 ms | ~50 MB | 通用 AI 代码执行 |
| Kata (QEMU) | 完整 VM | ~500 ms | 20–50 MB | 最高兼容性、强隔离 |
| Kata (Firecracker) | MicroVM | ~125 ms | ~5 MB | 高密度多租户 |
| Kata (Cloud Hypervisor) | MicroVM | ~200 ms | 10–20 MB | 性能与隔离平衡 |

### 2.6 暂停 / 恢复 / 快照

- **Docker**：容器级 pause/resume；快照提交为本地 OCI 镜像
- **Kubernetes**：`BatchSandbox` 通过 rootfs 快照实现 pause/resume（提交 OCI → 释放 Pod → 恢复时从快照重建）
- 公共 Snapshot API 用于持久化与克隆

### 2.7 客户端工具

- **多语言 SDK**：5 种语言，统一模型
- **CLI `osb`**：`sandbox`、`command`、`file`、`egress`、`devops`、`skills`
- **MCP Server**：Claude Code / Cursor 等 MCP 客户端可直接调用

---

## 3. 部署形态

| 形态 | 说明 | 典型用途 |
|---|---|---|
| Docker 单机 | `opensandbox-server` + Docker 守护进程 | 本地开发、单机服务 |
| Kubernetes（BatchSandbox） | OpenSandbox CRD + Controller + Helm | 大规模、高吞吐、池化 |
| Kubernetes（agent-sandbox） | 兼容 `kubernetes-sigs/agent-sandbox` | 与社区方案共存 |

---

## 4. 典型场景

- **Coding Agent**：Claude Code、Gemini CLI、Codex CLI、Qwen Code、Kimi CLI 等运行在隔离沙箱
- **AI 代码执行**：模型生成代码 → 沙箱内执行 → 流式结果返回
- **浏览器自动化**：Chrome / Playwright 沙箱，带 DevTools / VNC
- **远程开发**：VS Code Web、VNC 桌面
- **RL 训练与评测**：BatchSandbox + Pool 高并发交付
- **企业多租户**：安全运行时 + Egress + Credential Vault + Ingress 安全访问

---

## 5. 与纯容器/纯 VM 方案的差异

- 不是单纯容器编排，而是**面向 AI Agent 的完整沙箱协议 + 执行平面**
- 不是单一隔离技术，而是**按需选择 runc / gVisor / Kata / Firecracker**
- 不是只提供 SDK，而是**协议 → 服务端 → 运行时 → 网络/安全 → 客户端工具全栈**

---

## 6. 快速体验

```bash
# 安装并启动服务器
uvx opensandbox-server init-config ~/.sandbox.toml --example docker
uvx opensandbox-server

# Python SDK 示例
pip install opensandbox-code-interpreter
```

完整示例见仓库 `examples/` 目录与官方文档：<https://open-sandbox.ai/zh/>
