# RFC：Subscription Provider Plugin Host（提案，未实现）

状态：Draft RFC / 暂不视为已实现的插件能力  
适用仓库：Skyline-Gazer/miaomiaowu  
目标：降低向上游持续同步 MMW 时 MESL 和未来机场适配器的合并成本。

## 背景：当前源码的真实边界

- internal/handler/external_sync.go 的单订阅同步函数内直接执行 HTTP GET，再进行 YAML/Base64 解析和数据库/YAML 更新。理想的扩展边界是“获取订阅字节”的前置阶段，不是另写一套节点同步器。
- internal/handler/ssrf_safe_fetch.go 已有 SSRF 防护；插件请求不能绕过该策略。
- internal/scriptengine/engine.go 已使用 goja 作为 JS 引擎，只提供有限的脚本能力与 5 秒运行超时。
- 现有 override scripts 支持 post_fetch 和 pre_save_nodes，但 MESL 必须先做认证与开窗，再 fetch；现成脚本钩子不足以实现这个功能。
- internal/handler/update.go 将 GitHub Release 来源写为 iluobei/miaomiaowu，且在 Docker 中把更新二进制放入 /app/data/server。插件目录持久化不意味着官方更新后仍能加载插件，必须保留插件宿主。

## 方案比较

| 方案 | 上游兼容成本 | 隔离/安全 | 建议 |
| --- | --- | --- | --- |
| 直接在 external_sync.go 写 MESL 代码 | 高 | 核心代码膨胀 | 不采用 |
| Go 内部接口 + 静态 Adapter 注册 | 低至中 | Go 类型安全、简单 | 可作为第一阶段安全基线 |
| goja JS 插件 + 稳定 Host API | 宿主仍需少量 Fork patch；Provider 可独立更新 | 需要严格能力控制，JS 与宿主同进程 | 推荐研究并以隔离 PoC 验证 |
| Go plugin .so 热加载 | Go 版本/编译依赖高度耦合，易被更新破坏 | 同进程完整权限 | 不采用 |
| WASM Provider 插件 | 宿主运行时成本较大 | 可定义较清晰的权限边界 | Future Research，不在第一阶段 |
| HTTP Sidecar / 独立服务 | 独立部署和运维 | 需额外认证/网络面 | Owner 已明确拒绝 |

## 推荐的目标结构

~~~text
MMW External Subscription Scheduler / Manual Sync
                 |
                 v
    Source Fetcher Selection (唯一核心挂钩点)
       |                        |
       +-- standard http -------+
       |                        |
       +-- plugin host -> MESL / Future Providers
                                |
                 FetchResult（一次获取）
                                |
                Strict Validation / Snapshot
                                |
             Existing Parser + Node Sync + YAML
~~~

关键约束：插件不能自行修改 SQLite 节点表或 YAML 文件。插件只负责合法取得订阅响应（及安全的非敏感元数据），现有 MMW 负责校验、同步与发布。

## 建议协议 v1

宿主负责 Provider、Account、Subscription 三层关系，凭据由宿主管理。Adapter 不能在配置、Manifest 或日志中保存长期明文秘钥。

~~~text
FetchRequest:
  provider_id
  account_id
  subscription_id
  correlation_id
  deadline

FetchResult:
  body
  response_headers (仅受允许的字段)
  resolved_origin
  fetched_at
  non_sensitive_metadata

Error:
  AUTH_REQUIRED
  GATE_REJECTED
  EDGE_ACCESS_DENIED
  NETWORK_ERROR
  INVALID_CONTENT
  RATE_LIMITED
  RETRY_LATER
~~~

建议 Manifest（字段仍需设计评审，不是现有格式）：

~~~json
{
  "api_version": 1,
  "id": "mesl",
  "kind": "subscription_source",
  "entry": "adapter.js",
  "capabilities": ["auth-status", "open-gate", "download"]
}
~~~

插件加载目录建议位于持久化的数据卷：

~~~text
/app/data/provider-plugins/
  mesl/
    manifest.json
    adapter.js
~~~

**注意：这只是目标设计，当前程序并不会自动发现或执行该目录。**

## 安全边界

1. 插件必须经管理员显式启用，提供版本、签名/哈希及来源核查；避免加载来源不明的脚本。
2. goja 环境不开放文件系统、Shell、任意出站 HTTP 或任意 Go 对象；网络由 Host 提供经过授权的有限操作，按 Provider 允许域名、端点、方法和 SSRF 规则校验。
3. Goja 是同进程解释器，不能被视为完整的安全沙箱；首次只运行自己审查的可信插件。需测试终止、超时、资源消耗和崩溃风险。
4. 账号/订阅需严格鉴权隔离；请求重定向、DNS 重新解析及私有地址访问仍受 SSRF 防护约束。
5. 保护机密：不输出 Authorization、JWT、订阅 Token、节点密码；尤其应修复上游同步路径日志可能记录完整 sub.URL 的风险。
6. CF 1010 被归类为访问控制拒绝，不做伪装客户端指纹、代理轮换或绕过风控。
7. 插件需要能力版本协商：不兼容时拒绝运行，不应默默回退成破坏性普通订阅同步。

## 分阶段落地

**P0：代码级稳定边界。** 从单订阅同步逻辑中抽离 SourceFetcher 接口，将普通 HTTP 与 MESL 的抓取结果送入同一解析/同步管线。无需动态加载。Mock 认证、窗口和失败保护。  

**P1：小型插件宿主。** 验证 Goja 现有引擎是否适合有状态的 pre-fetch 工作流；实现版本化 Manifest、受限 Host API、显式启停、持久化插件目录与最小 UI。支持 MESL 脚本独立发布，但不承诺能直接运行未经修改的官方二进制。  

**P2：多机场。** 再研究 V2Board、XBoard、Custom 等；仅在取得真实 API 证据后实现 Adapter。不做无需求的大型插件市场或热重载框架。

## Upstream 同步原则

- 保持对 upstream/main 的可追溯同步；插件宿主 Patch 尽量小，避免改动解析与 YAML 发布逻辑。
- 不允许上游官方 Release 覆盖已经启用 Host 的自定义二进制。Fork 构建采用自己的 CI、验证和更新渠道，详见 docs/operations/fork-update-safety.md。
- 每次同步 upstream 必须完成：Host ABI/Manifest 兼容检查、普通订阅回归、MESL Mock、节点保护、状态机、UI 构建，以及实际二进制版本验证。
- 无论插件机制如何，未经实测的真实 MESL API 仍保持 BLOCKED_BY_CF_1010，禁止宣称生产可用。

## 未决问题

- 是否先采用静态 Provider Adapter、再演进到 JS 插件（推荐先行验证）。
- Goja 同步 Host 入口、阻塞 IO 与超时资源隔离的具体契约。
- 插件是否需要独立仓库、完整能力清单和签名策略。
- Provider Account 凭据持久化与数据库迁移边界。
- 如何最小限度接入现有 Web UI，而不导致日后与 upstream 产生大量冲突。

本 RFC 仅代表待评审的工程方向；不包含真实认证资料，不是生产部署授权。
