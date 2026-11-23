# Panel Provider 开发 Backlog

状态：Research First / Not Yet Released  
维护方式：GitHub Issues 已启用并创建以下四个正式任务。本文件作为与 Issues 对应的简要路线图；实际实施与验收证据以各 Issue/PR 为准。

## P0 — Fork 更新安全（[#2](https://github.com/Skyline-Gazer/miaomiaowu/issues/2)）

- [ ] 防止内置更新直接安装 upstream 原版，覆盖包含 Plugin Host 的自定义二进制。
- [ ] 区分上游版本通知和本 Fork Release 的应用通道。
- [ ] 引入构建指纹：fork commit、upstream base、Host ABI version。
- [ ] 建立 upstream 同步、Build/Tests、回归与 Release 验收工作流。
- [ ] 审计生产实际运行二进制及更新路径；未经授权不得更改生产。
- [ ] 备份、回滚与异常恢复演练。

参考：../operations/fork-update-safety.md

## P1 — 订阅获取插件宿主（[#3](https://github.com/Skyline-Gazer/miaomiaowu/issues/3)）

- [ ] 把 Fetch 从外部订阅同步内部解耦为小而稳定的 SourceFetcher 接口。
- [ ] 保留普通订阅行为；新增 Provider → Account → Subscription 关系。
- [ ] 调研使用既有 goja 的可信 Provider 脚本机制与权限边界，先做隔离 PoC。
- [ ] 定义 Manifest、Host ABI 版本、合法 HTTP 能力和 Error State。
- [ ] 实现安全的快照、串行同步、无效订阅防删除、失败后的 Last Good 恢复。
- [ ] UI 原生管理与普通订阅回归验收。

参考：../architecture/provider-plugin-host.md

## P1 — MESL PoC 迁入正确基线（[#4](https://github.com/Skyline-Gazer/miaomiaowu/issues/4)）

- [ ] 本地审查 mesl-native / a8fa738 的真实 Diff 和测试证据（尚未上传本 Fork）。
- [ ] 核实生产实际运行版本；不得按 Docker 镜像 v0.4.9 假定目标版本。
- [ ] 按 Plugin Host 最小变更原则移植 MESL，使用 Mock 验证认证、开窗、下载、错误隔离。
- [ ] 将 Cloudflare 1010 视为需要服务商允许的正式接入障碍，不绕过风控。
- [ ] 实测 VPS 可正常授权 GET、必要时开窗及下载前，不宣布生产可用。
- [ ] 解决正常会话轮换；遇人机验证进入 AUTH_REQUIRED 并保留旧节点。

参考：../research/mesl-panel-gate.md

## P2 — 多机场扩展（[#5](https://github.com/Skyline-Gazer/miaomiaowu/issues/5)）

- [ ] 研究 V2Board、XBoard、Custom 的真实 API 契约（不事先假设）。
- [ ] 支持各 Provider 独立账户、凭据、状态、调度及故障隔离。
- [ ] 添加新 Adapter 不得要求重写解析器、节点同步和 YAML 发布。
- [ ] 仅在必要时考虑 WASM 或更强隔离，不引入独立 Sidecar。

## 全局交付门

- [ ] Go / 前端构建、全部相关单测、普通订阅回归和 Mock Panel PASS。
- [ ] 插件协议及 Fork 版本在更新后仍兼容，失败时 fail closed。
- [ ] 不泄露 JWT、订阅 Token、Node 凭据和敏感日志。
- [ ] 完成 Owner 授权前，生产数据库、容器、配置、代理节点均保持不变。

> 本文件是待办清单，不代表任何一项已经实施或通过 CI。