# Fork 更新安全：避免上游二进制覆盖自定义功能

状态：已核实的上游机制 + 待实施的 Fork 策略（未部署）  
日期：2026-11-23

## 源码已证实的机制

1. docker-entrypoint.sh 首选可执行的 /app/data/server；若不存在才使用镜像内的 /app/server。
2. /app/data 通常是持久化数据卷。因此 Docker 镜像原始标签可与当前运行程序版本不同。
3. internal/handler/update.go：githubRepo 常量当前是 iluobei/miaomiaowu；从上游 Releases 下载二进制，Docker 模式把新程序安装到 /app/data/server。
4. 前端与后端在单个 Go 二进制分发，更新后可能同时更换前端静态资源和后端逻辑。
5. 本次 MMW Fork 当前未提供自动保留自定义插件的官方原生机制。光把 adapter.js 放在持久化目录，不能阻止 Host 被官方二进制替换。

因此 Docker 镜像显示旧版本而 MMW UI 显示新版本，不足以推断实际运行的源码版本。之前隔离 MESL PoC 所使用的 v0.4.9 基线不得未经复核直接当作生产运行版本。

## 拟定的 Fork 规则（必须在发布前实现）

- 禁止 Fork 定制版内置更新器静默下载或应用 upstream 原版二进制。
- 更新检查可以继续读取 upstream 版本信息，但“应用更新”只能经过 Fork CI 构建、完整回归、人工审批及本项目发布产物校验。
- 生产版本应包含明确的 Fork 构建标识、upstream base SHA、plugin host API version 和构建 SHA，便于 API 与 UI 对照。
- Provider 插件目录和配置应独立存放在持久化卷，但启动前必须校验 Host ABI 与插件版本匹配。
- CI 应支持同步 upstream 改动 → 构建 → Go/前端测试 → 普通订阅回归 → Provider Mock → 发布 Fork Release。
- 无成功备份、无精确二进制版本和无回滚验证时不得更新生产服务。
- 插件文件与数据卷被保留，不等于生产功能一定被保留；必须验证当前运行进程实际支持相应 Hook。

## 开发前的只读版本取证

在 Owner 授权的环境中，对实际生产容器进行只读核验：

- /proc/1/exe 指向的实际可执行路径；
- /app/server 与 /app/data/server 的存在状态和 SHA-256；
- 当前进程编译/版本信息与真实 UI build ID；
- 当前运行二进制与已发布 Tag/Commit 的对应关系；
- 当前持久化目录、插件目录（若未来新增）的挂载与权限。

不要因 /app/data/server 存在就直接覆盖它；不要把旧镜像的版本号自动当成实际运行版本。

## 发布与回滚要求

1. 在隔离环境完成发布候选构建和测试，确保自定义 Hook/Adapter 全部存在。
2. 备份 SQLite、配置、节点/订阅 YAML、凭据密钥及当前成功运行的二进制；备份访问权限需限制。
3. 经 Owner 授权后采用可回滚的原子部署方式；校验目标二进制 SHA 和启动后的构建标识。
4. 出现 Host ABI 不匹配、插件崩溃或鉴权故障时，先禁用 Provider，再回滚 Fork 二进制，不自动触发上游更新。
5. 如果节点同步已经修改生产 DB/YAML，单纯回滚可执行文件未必恢复旧状态；需要一致的部署前数据备份。

## 关联任务

- docs/architecture/provider-plugin-host.md：以稳定 SourceFetcher 扩展取代大范围分叉。
- docs/research/mesl-panel-gate.md：MESL 实验结论与尚未解决的合法认证阻塞。

生产环境在此阶段不应发生任何改变。
