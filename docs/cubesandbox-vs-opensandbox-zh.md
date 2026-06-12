# CubeSandbox vs OpenSandbox 对比文档

> **说明**：CubeSandbox 于 2026 年 4 月由腾讯云开源，版本迭代较快；本文基于公开资料（GitHub、官方文档、技术文章）整理，部分特性可能随版本变化，请以官方最新文档为准。

---

## 1. 一句话总结

- **CubeSandbox**：专为 AI Agent 代码执行优化的**轻量 KVM MicroVM 沙箱服务**，强调“60 ms 冷启动、5 MB 内存开销、硬件级隔离”，并原生兼容 E2B SDK。
- **OpenSandbox**：面向通用 AI 场景的**全栈沙箱平台**，以 OpenAPI 协议为中心，支持 Docker / Kubernetes 双运行时、多种安全容器运行时（runc / gVisor / Kata / Firecracker）及多语言 SDK/CLI/MCP。

---

## 2. 速查：什么场景选谁？

| 你的需求 | 推荐方案 | 理由 |
|---|---|---|
| 需要把 E2B 商业方案替换为自托管，追求极低延迟 | **CubeSandbox** | E2B 兼容、MicroVM 冷启动 <60 ms |
| 已基于 E2B SDK 构建，想零成本迁移 | **CubeSandbox** | 改 URL 即可 |
| 需要容器化快速落地，不想改造 KVM 环境 | **OpenSandbox** | Docker 一键本地运行，运维成熟 |
| 需要 K8s 大规模调度、池化、RL 训练 | **OpenSandbox** | BatchSandbox + Pool CRD，生产级编排 |
| 需要在同一平台内按需切换隔离级别 | **OpenSandbox** | runc / gVisor / Kata / Firecracker 可配 |
| 需要 GPU 沙箱 | **OpenSandbox** | CubeSandbox GPU 直通仍在开发 |
| 对“凭证不落地沙箱”有强需求 | **OpenSandbox** | Credential Vault 是原生能力 |
| 需要事件级快照回滚 / Fork | **CubeSandbox** | CubeCoW 毫秒级快照、克隆、回滚 |

---

## 3. 详细对比表

| 维度 | CubeSandbox | OpenSandbox |
|---|---|---|
| **开源方 / 时间** | 腾讯云 / 2026-04 | 阿里巴巴 / 更早进入 CNCF Landscape |
| **核心定位** | AI Agent 代码执行沙箱 | 通用 AI 沙箱平台（Agent、代码、浏览器、桌面、RL） |
| **架构哲学** | 自研 Rust 全栈 MicroVM 服务 | 协议优先、控制平面 + 数据平面分离、运行时无关 API |
| **隔离模型** | 统一 KVM MicroVM（独立 Guest OS 内核） | 可插拔：runc / gVisor / Kata(QEMU) / Kata(Firecracker) / Kata(CLH) |
| **虚拟化层** | RustVMM + KVM | 容器运行时 + 可选安全容器运行时 |
| **启动延迟** | 冷启动 <60 ms（单并发），50 并发 P95 ≈ 90 ms | 取决于运行时：runc ~500 ms，gVisor ~550 ms，Kata/FC ~625 ms，Kata/QEMU ~1000 ms；Pool 热启动可降至 50–200 ms |
| **内存开销** | 基础开销 <5 MB | runc / Kata-FC ~5 MB；gVisor ~50 MB；Kata-QEMU 20–50 MB |
| **部署密度** | 单 96 vCPU 节点 2000+ 并发 | 高密度依赖运行时选择，Kata-FC 可接近 CubeSandbox |
| **运行时后端** | 自研 Cubelet + CubeHypervisor + CubeShim | Docker / Kubernetes（BatchSandbox / agent-sandbox） |
| **集群调度** | CubeMaster 分布式调度，号称分钟级 10 万+ | Kubernetes 原生调度 + OpenSandbox Controller + Pool |
| **API 兼容性** | 原生兼容 E2B SDK，支持 OpenAI Python SDK | 自有 OpenAPI 协议（`specs/`），多语言 SDK |
| **多语言 SDK** | Python SDK、Go SDK（v0.3.0+），借 E2B 覆盖更多语言 | Python、JS/TS、Java/Kotlin、C#/.NET、Go |
| **CLI / MCP** | 自带 CLI | `osb` CLI + MCP Server |
| **网络策略** | CubeVS（eBPF 虚拟交换机），内核级隔离与出站过滤 | Egress Sidecar（DNS / DNS+nft）+ Ingress 网关 + Secure Access |
| **凭证安全** | 未公开类似 Credential Vault 的原生方案 | Credential Vault：真实凭证不落地沙箱，出站 MITM 注入 |
| **快照能力** | CubeCoW：百毫秒级快照、克隆、回滚；soft-dirty 增量内存快照 | Docker 快照（OCI 镜像）；K8s pause/resume（rootfs 快照）；公共 Snapshot API |
| **GPU 支持** | 当前以 CPU 场景为主，GPU 直通在开发 | 支持 GPU resource limit（Docker / K8s） |
| **存储 / 卷** | 主机挂载、模板镜像 | host / pvc（Docker named volume / K8s PVC）/ OSSFS |
| **部署复杂度** | 需要 x86_64 Linux + KVM，一键安装脚本较完善 | Docker 本地极简；K8s 需要安装 Controller、RuntimeClass 等 |
| **适用环境** | 裸金属或支持嵌套虚拟化的云主机 | 任意 Docker/Kubernetes 环境 |
| **浏览器/桌面/RL 示例** | 有浏览器、RL 示例 | Chrome、Playwright、Desktop、VS Code、RL 训练等完整示例 |
| **协议/规范** | 兼容 E2B 接口 | 自有 OpenAPI：Lifecycle / Execd / Egress / Diagnostics |
| **许可证** | Apache 2.0 | Apache 2.0 |

---

## 4. 架构对比

### CubeSandbox 架构

```
Client (E2B / OpenAI SDK / Python / Go)
    ↓
CubeAPI   —— REST 网关（Rust，E2B 兼容）
    ↓
CubeMaster —— 集群调度器
    ↓
Cubelet / CubeProxy / CubeVS —— 节点代理、反向代理、eBPF 虚拟交换
    ↓
CubeHypervisor + CubeShim —— KVM MicroVM + containerd Shim v2
    ↓
Guest OS Kernel + Agent Workload
```

### OpenSandbox 架构

```
Client (SDK / osb CLI / MCP)
    ↓
Lifecycle Server (FastAPI) —— 鉴权、校验、编排
    ↓
Runtime Provider
    ├── Docker Sandbox Service
    └── Kubernetes Sandbox Service
            ├── BatchSandbox Provider
            └── agent-sandbox Provider
    ↓
Sandbox Pod / Container
    ├── execd（命令/文件/代码解释器/PTY/指标）
    ├── 可选 Egress Sidecar（出站网络策略）
    └── 用户镜像 + 卷
    ↓
Ingress Gateway / Server Proxy —— 入站访问
```

---

## 5. 隔离模型对比

| 特性 | CubeSandbox | OpenSandbox |
|---|---|---|
| 默认隔离 | 硬件虚拟化（独立内核） | 进程级容器（runc） |
| 是否共享内核 | 否 | 默认共享；可选 gVisor/Kata 不共享 |
| 逃逸风险 | 极低（MicroVM） | runc 较高，gVisor/Kata 较低 |
| 兼容性 | 完整 Linux 兼容 | runc 最佳；gVisor 部分 syscall 受限；Kata 最好 |
| 启动速度 | 极快（<60 ms） | runc 较快，安全运行时较慢 |
| 密度 | 极高（<5 MB/实例） | 依赖运行时，Kata-FC 可接近 |

**结论**：

- CubeSandbox 把“硬件级隔离”作为唯一且不可降低的基线，适合对隔离有强需求、可接受 KVM 环境的用户。
- OpenSandbox 把“隔离级别”作为配置项，允许同一平台从“快速容器”平滑升级到“安全 VM”。

---

## 6. 性能对比

| 指标 | CubeSandbox（官方数据） | OpenSandbox（OSEP 参考数据） |
|---|---|---|
| 冷启动 | <60 ms（单并发）；50 并发 avg 67 ms / P95 90 ms | runc ~500 ms；gVisor ~550 ms；Kata-FC ~625 ms；Kata-QEMU ~1000 ms |
| 热启动（Pool） | 资源池预分配 + 快照克隆 | runc ~50 ms；gVisor ~100 ms；Kata-FC ~125 ms；Kata-QEMU ~200 ms |
| 内存开销 | <5 MB | runc ~5 MB；gVisor ~50 MB；Kata-QEMU 20–50 MB |
| 单节点并发 | 2000+（96 vCPU） | 取决于运行时与资源，Kata-FC 可达相近量级 |

> 注：CubeSandbox 数据来自官方公开 benchmark；OpenSandbox 数据来自 OSEP-0004 设计文档，实际数值与工作负载、节点配置相关。

---

## 7. 网络与安全对比

| 能力 | CubeSandbox | OpenSandbox |
|---|---|---|
| 出站策略 | CubeVS eBPF 过滤 | Egress Sidecar：DNS / DNS+nft |
| 入站访问 | CubeProxy 反向代理 | Ingress 网关 / Server Proxy |
| 沙箱间隔离 | eBPF 内核级隔离 | 默认依赖集群 CNI；推荐 Egress `deny.always` 强制隔离 |
| 凭证保险库 | 未公开原生支持 | Credential Vault（真实凭证不落地） |
| 端点安全 | 依赖 E2B 协议 | Secure Access：端点凭证 + 签名路由令牌 |

**结论**：OpenSandbox 在“凭证不落地”和“细粒度出站策略”上提供了更完整的企业级方案；CubeSandbox 在网络隔离上更偏向内核级 eBPF 实现。

---

## 8. 快照与状态管理对比

| 能力 | CubeSandbox | OpenSandbox |
|---|---|---|
| 快照引擎 | CubeCoW（Copy-on-Write） | Docker：commit 为 OCI 镜像；K8s：rootfs 快照 |
| 快照粒度 | 事件级 / 百毫秒级 | 容器级 / Pod 级 |
| 内存快照 | soft-dirty 增量内存快照 | 当前不保留进程/内存状态 |
| 回滚 / Fork | 支持从任意快照回滚、Fork 并行探索 | Docker 快照可创建新沙箱；K8s pause/resume 恢复 rootfs |
| 公开 API | 有 | 有（Snapshot API），K8s 内部使用 SandboxSnapshot CRD |

**结论**：CubeSandbox 的快照/回滚/Fork 能力更先进，适合需要高频 checkpoint 的场景；OpenSandbox 的快照更偏向持久化与恢复。

---

## 9. SDK 与生态对比

| 能力 | CubeSandbox | OpenSandbox |
|---|---|---|
| 自有 SDK | Python、Go | Python、JS/TS、Java/Kotlin、C#/.NET、Go |
| 兼容 SDK | E2B SDK、OpenAI Python SDK | 自有协议 SDK |
| CLI | 有 | `osb` CLI |
| MCP | 未公开 | 官方 MCP Server |
| 示例覆盖 | 代码执行、浏览器、RL、网络策略 | Coding Agent、浏览器、桌面、RL、LangGraph、ADK 等 |
| 上游生态 | 兼容 E2B 生态 | CNCF Landscape、kubernetes-sigs/agent-sandbox |

---

## 10. 部署与运维对比

| 维度 | CubeSandbox | OpenSandbox |
|---|---|---|
| 环境要求 | x86_64 Linux + KVM | Docker 即可本地运行；K8s 需要集群 |
| 安装方式 | 一键安装脚本 | `uvx opensandbox-server` / Helm / Kustomize |
| 单机部署 | 支持 | Docker 模式极简 |
| 集群部署 | CubeMaster + Cubelet | Kubernetes + OpenSandbox Controller |
| 云厂商依赖 | 腾讯云出品，但可私有部署 | 阿里巴巴出品，可私有部署 |
| 运维对象 | CubeMaster、Cubelet、CubeVS、Guest Kernel | Lifecycle Server、Controller、execd/egress/ingress、RuntimeClass |

---

## 11. 选型建议

### 选 CubeSandbox，如果：

1. 你当前使用 E2B 商业服务，希望**零成本迁移**到自托管。
2. 你主要运行**AI Agent 代码执行**，对冷启动极度敏感（<100 ms）。
3. 你愿意接受 **x86_64 Linux + KVM** 环境，且需要硬件级隔离作为默认基线。
4. 你需要**高频快照、回滚、Fork**能力。
5. 你的工作负载以 CPU 为主，暂不需要 GPU。

### 选 OpenSandbox，如果：

1. 你希望**先跑起来再逐步增强隔离**：从 Docker/runc 开始，按需切到 gVisor / Kata / Firecracker。
2. 你需要**Kubernetes 原生的大规模调度、池化、RL 训练**。
3. 你需要**多语言 SDK** 和企业级工具链（CLI、MCP）。
4. 你需要**Credential Vault** 让真实凭证不落地沙箱。
5. 你需要**GPU 沙箱**或更灵活的存储（PVC / OSSFS）方案。
6. 你希望协议层清晰，方便自定义客户端或集成到现有平台。

---

## 12. 综合结论

| 维度 | 胜者 |
|---|---|
| 冷启动速度 | **CubeSandbox** |
| 硬件级隔离默认基线 | **CubeSandbox** |
| 平台通用性与生态 | **OpenSandbox** |
| Kubernetes 大规模编排 | **OpenSandbox** |
| 隔离级别灵活度 | **OpenSandbox** |
| 凭证与出站安全 | **OpenSandbox** |
| 快照/回滚/Fork | **CubeSandbox** |
| 多语言 SDK/CLI/MCP | **OpenSandbox** |
| 落地门槛 | **OpenSandbox**（Docker 即可） |
| E2B 迁移成本 | **CubeSandbox** |

两者并非完全互斥：CubeSandbox 更适合替代 E2B 的“极速代码执行沙箱”场景；OpenSandbox 更适合作为企业 AI 平台的“通用沙箱基础设施”。如果团队已经在 Kubernetes 上运行多种 AI 工作负载，OpenSandbox 的协议统一性和运维成熟度更具优势；如果追求极致启动速度和 E2B 兼容，CubeSandbox 是值得重点评估的方案。

---

## 13. 参考来源

- OpenSandbox GitHub：<https://github.com/alibaba/OpenSandbox>
- OpenSandbox 官方文档：<https://open-sandbox.ai/zh/>
- CubeSandbox GitHub：<https://github.com/TencentCloud/CubeSandbox>
- CubeSandbox 官方文档：<https://cubesandbox.com/>
- 腾讯云技术文章：《60ms 启动一个安全沙箱：深入解析腾讯云 CubeSandbox 的架构设计》
