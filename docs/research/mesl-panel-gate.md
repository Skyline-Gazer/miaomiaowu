# MESL Panel Gate：已验证事实与隔离 PoC 交接

状态：Research / P0 部分通过，真实无人值守流程未验收  
记录日期：2026-11-23

> 本文记录 Owner 提供的 ego(lite) + VPS 实测报告及本次仓库代码核查结果。并非由 GitHub CI 重跑的生产验收。公开仓库中严禁提交 JWT、订阅 Token、完整节点正文、账号凭据、真实配置、私有日志或凭据文件。

## 目标

为 MMW 外部订阅增加可扩展的 Panel Gate 获取方式：验证授权 → 查询窗口 → 必要时开启窗口 → 下载订阅 → 严格校验 → 交给现有节点同步器。不能把 MESL 的特殊逻辑写死到解析和 YAML 发布层。

## 已观察到的 Panel 行为

- GET /api/v1/access/getSubscribe：已登录浏览器请求可获得订阅门控元数据（包含窗口相关字段）。
- POST /api/v1/access/openSubscribe，空 JSON 对象：浏览器正常调用后返回成功及 data.expired_at，窗口约 10 分钟。
- POST /api/v1/portal/auth/login：正常登录路径；前端另有设备验证和 2FA 分支。不得绕过任何验证。
- 当前前端使用浏览器 localStorage 的 pu:auth 保存 JWT，请求 Authorization 直接采用 JWT 值，不加 Bearer；旧 auth_data 存在兼容读取。
- 本次观察 JWT 签发到过期约 7 天；未发现可证明存在正式自动续期的 Refresh Token 流程。不能据此宣称长期自动登录能力已具备。

以上为观察到的当前接口行为，不构成第三方 Panel API 的稳定契约承诺。

## 网络与订阅下载 P0 证据（来自已完成实测）

- ego(lite) 授权会话成功开启订阅访问窗口。
- MMW VPS 从不同的公网出口在同一个有效窗口内下载成功；原始响应 HTTP 200，内容可以按 Clash YAML 解析，节点列表非空（结构检查为 172 条）。
- 这次服务端声明 Content-Type: text/html，但实际正文为有效订阅，因此不能仅凭 Content-Type 判失败；必须以严格解析和结构校验为准。
- 同一 JWT 用于 VPS 的 Panel 授权 GET 时，Cloudflare 返回 HTTP 403 / 错误码 1010 / browser_signature_banned；未拿到 Panel 业务数据。
- 这证明本次 VPS HTTP 客户端被边缘访问控制拒绝；没有证据足以断言 VPS IP/ASN 被封锁，也不能证明 Panel 后端拒绝会话。
- VPS 独立登录、VPS 开窗、会话过期后重新授权均未完成验收。

判定：

~~~text
CROSS_IP_SUBSCRIPTION_FETCH = PASS
VPS_PANEL_AUTHENTICATED_GET = BLOCKED_BY_CF_1010
VPS_PANEL_OPEN_GATE = NOT_TESTED
INDEPENDENT_LOGIN = UNKNOWN
AUTOMATIC_REAUTH = UNKNOWN
LIVE_END_TO_END = BLOCKED
PRODUCTION_READY = NO
~~~

## 已有隔离 PoC（尚未上传此 Fork）

此前本地代理开发报告称存在分支 mesl-native、提交 a8fa738，基于当时判定的 v0.4.9 后端基线；包含 Provider → Account → Subscription 绑定、MESL 开窗与获取、AUTH_REQUIRED、1010 错误分类、最后成功快照及 Mock 测试。以上是交接线索，不是本仓库可验证的 Commit，也未经过本仓库 CI。

移植前必须拿到并审查完整本地 Diff，且确认 VPS 真正运行的二进制版本，不能仅凭 Docker 镜像标签判定生产代码版本。不要直接合并旧 PoC。

## 功能边界和验收

1. 任何 Provider 的认证失效不得清除历史节点和 last-good snapshot。
2. 开窗幂等检查；失败不得无限重试；边缘明确拒绝时不进行指纹伪装或代理轮换。
3. 订阅必须一次获取、严格校验后才进入解析/同步；无效、空或异常大幅减少应阻止破坏性更新。
4. 独立管理账号、凭据、订阅、锁、调度和状态；普通外部订阅保持原样。
5. JWT 不得出现在 HTTP 响应、日志、前端普通 API 或仓库中；不要把原始订阅 URL 写入日志。
6. 真实端到端 PASS 需要服务商正式允许的 VPS API 接入方式、合法的授权/轮换策略与一轮实际开窗 + 下载验收。
7. 本文不授权修改线上容器、数据库、节点或代理服务。

## 后续任务

- RFC：docs/architecture/provider-plugin-host.md
- 更新策略：docs/operations/fork-update-safety.md
- 正式接入：优先协商服务商认可的 API / 精确授权机制，禁止绕过 Cloudflare 1010。
- P2：Multi-Provider Panel Adapter（其他 Panel 的 API 未实测，不预先实现）。
