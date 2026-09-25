# 自用维护记录

## 分支与更新方式

- 维护分支：`infty/main`，远端 `fork` 指向 `JinchengGao-Infty/vm2api`。
- 上游来源：`origin` 指向 `dofastted/vm2api`。
- 2026-09-20 从已保留缓存部署修复的 `a283a61` 建立维护分支。该提交已合入上游 `30c06a9`，版本字段为 1.2.9。
- 上游 PR #18 已按用户要求关闭；修复继续保留在本分支。

接收上游更新时，先查看更新涉及哪些代码，再合并到 `infty/main`。上游已解决的问题采用上游实现，只保留现场复现后仍必要的修复，推送到 `fork`。维护分支和生产部署分别记录，不把合并完成当作服务器已升级。

上游面板更新和一键安装脚本会选择上游 release tag，不能用它们覆盖这个维护分支。部署应使用我们选定的维护分支提交，并保留现场修改和可恢复的旧版本。

## 历史修复（1.3.9 起采用上游实现）

`src/lib/vm/wrap-cli-runtime.mjs` 中的 `kernelPayloadPath()` 统一内核来源：

1. `KIN_KERNEL_BIN` 指向的文件存在时，优先采用它。
2. 否则保留模板或槽位的 `kin-kernel.bin`，再回退到已有 wrapper。
3. 安装/同步、制作模板、从槽位晋升模板都使用这一规则；CLI 和 glibc 文件仍来自所选模板。

原因：旧模板和旧槽位会跨版本保留，原来的来源优先级能把已经升级的主内核覆盖回旧内核。单纯重启控制面不能让正在运行的槽内进程加载新内核；相应部署需要同步并重启目标槽。

这项修复此前已在现有部署回放三条连续真实 OMP 请求：缓存读取为 0、40,645、42,333 tokens，新增历史能进入缓存。原始请求和响应留在仓库之外。

2026-09-22 用户明确要求上游实现可用时以上游为准。v1.3.9 已实现安装、同步和模板制作时优先主内核，自用补丁已移除；晋升模板沿用上游复制所选槽位快照的语义。1.3.9 阶段另外保留了下文两处现场复现所需的小修复；上游 v1.3.21 已解决，现已一并采用上游实现。

## 独立问题：缓存 TTL

此前实际响应将缓存写入计入 `ephemeral_5m_input_tokens`，虽然面板选择了 1h。已查到的内核实现重建消息断点时只输出 `type: ephemeral`，没有传递 1h TTL。这与上面的内核来源修复是两个问题。

截至建立维护分支时，这项 TTL 问题没有在本分支修复。后续需要核对可维护的内核源码及实际响应；不能靠把面板显示改成 1h 或仅改控制面字段宣称已修复。

## 自用账号额度设置（2026-09-20）

用户要求取消账号尚有额度时的提前保护。isif 生产配置的 default/pro/max 三档 `limit_5h`、`limit_7d` 及对应 `safety_ratio`、`weekly_safety_ratio` 已全部设为 1，保留官方实际额度限制。升级或部署时保留这项设置，不恢复上游的 85%/80% 或 95% 提前停用阈值。并发设置未调整。

当时 vm-01 被 `quota_5h_safety` 停止调度，五小时用量回到 0% 后仍未恢复；请求日志为 `account_pool_exhausted`，客户端被统一显示成“号池负载过高”。通过路由 API 热更新阈值，再通过 schedulable API 恢复 vm-01。实际 `/v1/messages` 请求 claude-opus-4-6 返回 HTTP 200、`OK.`、`end_turn`，没有重启控制面或槽位。

修改前的 routing.json 和 vm-01.json 备份在 isif `/opt/vm2api/backups/quota-reserve-20260920T094056Z/`。这些部署文件含敏感配置，不提交到 Git。

本次另发现路由保存接口在完成写盘和热更新后，因 `publicRoutingNotify is not defined` 返回 500；已通过 GET 和磁盘内容确认配置生效。该响应错误本次未修改，应与保存是否成功分别判断。

## 单账号 VPS 本地出口刷新修复（2026-09-20）

isif 上的 vm-01 绑定 `px-local`，聊天和凭证刷新都应使用 VPS 本地出口。实际控制面仍运行 1.2.4，其 `host-token-refresh.mjs` 要求非空 SOCKS URL，导致刷新时报 `slot SOCKS5 is required for token refresh`。维护分支已有本地出口支持，本次将 `host-token-refresh.mjs` 和 `proxy-resolve.mjs` 定向部署到生产，未合入其他上游变化。

生产镜像为 `vm2api:1.2.4-infty-local-refresh-20260920`，宿主机对应两个源码文件和 `docker-compose.yml` 的镜像标签已同步更新。镜像基于此前实际运行的镜像，只叠加这两个源码文件和当前已部署的内核二进制，避免旧入口脚本在重启时将内核覆盖回旧版本；cookie-auth 使用现有 `.elf` 原件，保留启动时生成兼容包装器的行为。

构建目录、原文件、旧 compose 和脱敏验证结果位于 isif `/opt/vm2api/backups/local-refresh-20260920T122313Z/`。复建本次镜像使用该目录的 `image/Dockerfile`；后续完整升级仍从维护分支进行，不用历史 1.2.4 工作目录的普通全量构建替代本次镜像。

部署前主代理空闲、服务器在途请求为 0。重建控制面期间，同一个 `kin-01` 容器也发生了重启；OMP 进程和会话未重启。部署后的实际凭证刷新返回 HTTP 200、`refreshed: true`，并写入新的过期时间；随后 `claude-opus-4-6` 的 `/v1/messages` 请求返回 HTTP 200、正文 `OK`、`end_turn`。


## 升级到 1.3.6（2026-09-21）

正式发布版本通过 GitHub Releases API 确认为 v1.3.6。维护分支将此 tag 合入为 `d01c291`，没有合并冲突，并已推送到 fork。保留 `kernelPayloadPath()` 的主内核优先规则；上游本地出口刷新支持继续保留。

生产从自用 1.2.4 修复镜像升级为 `vm2api:1.3.6-infty`，镜像直接由维护分支代码构建。宿主机 `/opt/vm2api` 同步到维护分支，实际控制面 VERSION 为 1.3.6，并通过带 `ids: ["vm-01"]` 的 wrap-cli/sync 接口同步和重启活动槽位。内核配置的 native session 位为 20；原并发设置未修改，vm-02 保持停止。原额度阈值 100%、本地出口、Key、账号凭证和禁用 distill 检测的设置保留。新版 cookie-auth 已能在控制面镜像直接启动，旧 PyInstaller 兼容包装器不再使用。

备份位于 isif `/opt/vm2api/backups/upgrade-1.3.6-20260921T052726Z/`，包括运行文件归档、在线 SQLite 备份、切换前 SQLite 备份、原部署差异与镜像信息。旧镜像保留。镜像构建目录 `/opt/vm2api-build-1.3.6` 保留用于复建。

### 1h 缓存仍不兼容，部署默认修正为 5m

升级后使用本机 OMP 的 provider URL、Key 和 User-Agent 进行真实连续工具调用。默认 1h 设置时，前两轮成功，第三轮稳定返回 502 incomplete_response；同一第三轮仅加 `x-kin-cache-ttl: 5m` 即恢复 HTTP 200、tool_use，读取 8,582 token 并新增 2,380 token 缓存。增加 max_tokens 或改为 SSE 均不能解决 1h 条件下的问题。

实际响应始终将缓存写入记入 ephemeral_5m_input_tokens。生产已通过路由接口将 compatibility.cache_ttl 热更新为 5m，保留其他额度和并发设置。之前宣称 1h 的配置并未提供实际一小时缓存，不能在后续升级时直接恢复为 1h；需先以真实工具循环和响应中的 TTL 用量确认新内核支持。

改为 5m 后，以默认配置重新执行三轮连续工具调用，全部返回 HTTP 200、tool_use，工具参数与对应轮次一致；缓存读取依次为 6,202、8,582、10,962 tokens。结果保存在本机 `/Users/gaojincheng/.omp/reports/vm2api-upgrade-1.3.6-verification.json`，只包含合成验证内容和用量，不含凭证。OMP 无须重启或新开会话。

这属于当前 VM2 调用链的兼容问题。Anthropic [API 官方文档](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#1-hour-cache-duration)仍支持显式 `ttl: "1h"`；[Claude Code 官方文档](https://code.claude.com/docs/en/prompt-caching#which-ttl-each-request-gets)也说明订阅套餐内的主对话默认请求 1h。官方要求混用 TTL 时，1h 断点必须在所有 5m 断点之前；当前尚未取得上游原始校验错误，不能把本次失败的具体原因写成已确定。

## 执行槽卡死恢复与 1.3.9 升级（2026-09-22）

北京时间 01:05 起请求连续报 `503 slot_busy: rust kernel has no free slot`，每次等待约 45 秒。现场 `inflight=0`、`ready_slots=0`，凭证 fresh、账号可用；最后成功请求记录五小时额度使用 7%、周额度使用 29%。确认无在途请求后重启 `kin-01`，真实公网请求返回 HTTP 200、OK、end_turn，确认本次故障是执行槽卡住。没有观察到封号；尚未定位 native slot 长时间运行后耗尽的内部原因。

已将最新正式版 v1.3.9 合入维护分支并部署到 isif。用户要求优先采用可用的上游实现，因此移除了此前的内核来源自用补丁。当前镜像 `vm2api:1.3.9-infty`，代码提交 `3214c56`，构建目录 `/opt/vm2api-build-1.3.9`；备份 `/opt/vm2api/backups/upgrade-1.3.9-20260921T171919Z/`，旧 1.3.6 镜像保留。同步 vm-01 新内核，并通过上游路由接口补写旧 kernel.json 缺失的 `default_cache_ttl` 等新版配置。账号、Key、本地出口、额度阈值、禁用 distill 设置和 OMP 会话保留。

### 1.3.9 阶段的两处修复（1.3.21 已由上游取代）

1. `writeKernelJsonAtomically()` 保留原配置文件 UID/GID。上游原子替换生成 root:root 0600 文件，而 kin-01 使用 10001:987，保存缓存设置并重启后实际报 `read worker config: Permission denied`，进入重启循环。修复后再次调用真实 PUT routing 接口返回 200，文件仍为 10001:987 0600，槽内运行用户可读取。提交 `4ce6fc7`。
2. `prepareCliHopBody()` 最后用已有 `applyCacheTtlToBody()` 将标记统一为已解析的请求 TTL。上游模板中的 1h 标记会被误当成用户选择，覆盖已解析的 5m；现场输出 `resolved=5m` 但消息标记仍为 1h。新版完整错误明确返回 Anthropic 400：1h 标记不能出现在 5m 标记之后。官方客户端仍走原有提前返回分支，其自有标记不改。提交 `3214c56`。

最终使用实际 OMP provider URL、Key、User-Agent 发三轮连续工具调用，全部 HTTP 200、tool_use，参数分别为 1、2、3。缓存读取 6,519 → 6,519 → 12,465 tokens；第二、三轮分别新增 5,946 / 2,380 tokens 的 5m 缓存。结果 `/Users/gaojincheng/.omp/reports/vm2api-upgrade-1.3.9-default-verification.json`。保持默认 5m：1h 请求在本次升级验证中仍触发混合 TTL 400，不根据发布说明直接启用。OMP 无须重启。

## 升级到上游原版 1.3.21（2026-09-22 中午）

GitHub Releases API 确认最新正式版 v1.3.21，发布时间 2026-09-22T04:00:22Z。上游已修复槽配置原子替换的 UID/GID 问题，并将 cli-hop 线上缓存标记统一为 5m；两处自用源码补丁均已移除。维护分支合并提交 `2d5960e`，应用代码与 v1.3.21 一致，只额外保留 AGENTS.md 和本维护记录。

生产改为直接运行固定版本官方镜像 `ghcr.io/dofastted/vm2api:v1.3.21`，不再为相同应用源码自行构建。服务器源码快照 `/opt/vm2api-release-1.3.21`，部署目录仍为 `/opt/vm2api`。实际控制面 VERSION 已确认 1.3.21；新 `share/wrap-cli/cli-node` 和 kernel 同步到 vm-01，并明确重启 dataplane，未删除槽容器。槽启动健康返回 ready_slots=20，凭证 fresh。

切换前确认 OMP 主会话空闲、服务在途请求为 0。备份 `/opt/vm2api/backups/upgrade-1.3.21-20260922T042345Z/`，包含完整运行文件、在线 DB 与切换前 DB、原部署差异及镜像信息；旧 `vm2api:1.3.9-infty` 镜像保留。账号凭证、Key、本地出口 px-local、default/pro/max 100% 额度阈值、distill disabled、5m 缓存及 OMP 原会话保留。

使用上游真实 PUT routing 接口保存 5m 后返回 HTTP 200；kernel.json、worker.json、internal.token 均保持 10001:987、0600。用当前 OMP 的实际 URL、Key、User-Agent 发三轮工具调用，全部 HTTP 200、tool_use，参数 1/2/3 正确；缓存读取 0 → 6,519 → 12,468 tokens，写入均记录为 5m。结果 `/Users/gaojincheng/.omp/reports/vm2api-upgrade-1.3.21-default-verification.json`。无须重启 OMP 或新建对话。

## 升级到上游原版 1.3.27（2026-09-23）

用户要求先更新上游，等待上游适配 Opus 5.5。本次通过 GitHub Releases API 确认最新正式版为 v1.3.27，将 tag 无冲突合入维护分支，合并提交 `f800e48`，已推送 fork。应用代码与该 tag 一致，额外文件仍只有 AGENTS.md 和本记录；没有增加模型或修改 OMP。

生产使用固定版本官方镜像 `ghcr.io/dofastted/vm2api:v1.3.27`，运行中容器 VERSION 已核实。源码快照 `/opt/vm2api-release-1.3.27`。宿主源码、share/wrap-cli 和内核更新后，通过上游 `wrap-cli/sync` 明确同步并重启 vm-01；新执行进程已启动，ready_slots=20。vm-02 继续停止。

切换前确认服务在途请求为 0。回退备份 `/opt/vm2api/backups/upgrade-1.3.27-20260922T171149Z/` 包含运行文件归档、在线 DB、切换前 DB、原部署差异和镜像信息；旧 1.3.21 镜像保留。部署后核对 .env、routing.json、distill-rules.json 与备份内容一致，保留账号、Key、本地出口、100% 额度阈值和 5m 缓存。kernel.json、worker.json、internal.token 仍为 10001:987、0600。

### 实际请求与缓存

- 使用当前 OMP provider 的 URL、Key、User-Agent，Opus 5 和 Opus 4.6 普通请求均返回 HTTP 200、end_turn。Opus 5 请求读取 README.md 时正常返回 `read_file` 工具调用，参数为 README.md。
- Opus 4.6 三轮工具调用全部返回 HTTP 200、tool_use，参数 1/2/3 正确。新版保留客户端消息缓存标记；不带消息标记时，本组合成请求只复用了工具/系统前缀，缓存读取为 0 → 6,519 → 6,519。不能把旧验证脚本未打消息标记当作缓存增长验证。
- 已核对 OMP `packages/ai/src/providers/anthropic.ts` 的 `applyHeadCaching` 和 `applyPromptCaching`，客户端会标记稳定头部和近期消息。增加近期消息标记后，三轮缓存读取为 6,519 → 12,465 → 14,845 tokens，后两轮各新增 2,380 tokens 的 5m 缓存，未缓存输入各为 6 tokens。第三轮首次在本机 TLS 握手时 EOF，随后仅重发该轮成功，没有重跑前两轮。未变更默认 TTL，也未验证 1h。
- 同一组合成 `deployment_ack` 长填充请求在 Opus 5 上触发了上游 Usage Policy 拒绝，强制工具和 auto 两种方式均出现；普通对话和自然的读文件工具请求成功。该现象单独记录，未判断为强制工具专属问题、封号或整个工具通道不可用，也未修改上游拒绝处理。

本机报告位于 `/Users/gaojincheng/.omp/reports/`：`vm2api-upgrade-1.3.27-default-verification.json`、`vm2api-upgrade-1.3.27-client-cache-verification.json`、`vm2api-upgrade-1.3.27-basic-requests.json`、`vm2api-upgrade-1.3.27-opus5-read-file.json`；失败的合成请求另存 forced/auto-tool-probe 报告。报告不含凭证。OMP 无须重启或新开对话，当前主模型继续使用原配置。

## 升级到上游原版 1.3.29，接入 Opus 5.5（2026-09-23）

上游 v1.3.28 加入 `claude-opus-5-5`，将出站 Claude Code 版本改为 2.1.280；v1.3.29 修复 native Claude 多轮缓存只写不读。维护分支无冲突合并 v1.3.29，提交 `f2754ff`，已推送 fork；应用代码仍采用上游原版。生产运行官方镜像 `ghcr.io/dofastted/vm2api:v1.3.29`，源码快照 `/opt/vm2api-release-1.3.29`，原工作目录 `/opt/vm2api`。

切换前主工作台无运行中代理，vm-01 无在途请求。备份 `/opt/vm2api/backups/upgrade-1.3.29-20260923T032848Z/` 包含运行文件、DB、差异和原镜像信息；旧 1.3.27 镜像保留。仅同步并重启 vm-01，vm-02 继续停止。旧槽的 `kernel.json` 在第一次同步后仍保留 `cli_version=2.1.278`；用上游 `PUT /api/panel/routing` 重存已有 5m 配置，投影出 2.1.280，再同步重启 vm-01，内核健康信息确认实际运行版本 2.1.280、ready_slots=20。`.env` 和蒸馏配置与备份一致；routing 相比备份仅移除了上游已弃用的 `codex.plugin` 字段，缓存仍为 5m。

实际公网 `/v1/models` 已列出 `claude-opus-5-5`；用 OMP 原有 VM2 URL、Key 和 User-Agent 发请求，Opus 5.5 返回 HTTP 200、`read_file` 工具调用及正确文件参数。OMP `~/.omp/agent/models.yml` 新增此模型，沿用 300k 上下文，设 adaptive thinking 的 medium 默认、官方价格元数据；其余模型配置未变。备份在 `~/.omp/backups/models-before-opus-5-5-20260923T033241Z.yml`。临时 OMP CLI 请求返回 OK，科研工作台 `/api/models` 已显示 Opus 5.5。当前主代理模型仍为 Opus 5，可在工作台选择新模型。

科研工作台原 `next start` 进程重启时发现 `.next` 生产构建文件缺失，已按该项目 AGENTS.md 的开发启动方式改为 `next dev --webpack`，仍监听 127.0.0.1:30141。服务日志为 `/tmp/omp-research-web-30141-dev-20260923.log`。原会话保留在磁盘上。

## 启用 1h 缓存并升级上游 1.3.32（2026-09-23 晚）

用户要求查清 1h 未生效的原因，并在可用时启用。1.3.29 上仅热改 `compatibility.cache_ttl=1h`、将设置投影到两个槽的 `kernel.json`，真实 Claude Code 两轮请求仍返回 8,090 个 5m 写入 token、1h 为零。确认 vm-01 的 20 个执行位全部空闲后，使用上游 `restartRustKernel()` 重载，保持相同二进制和请求设置，返回变为 8,093 个 1h 写入 token、5m 为零，第二轮读取 7,989 tokens。因此本次直接故障是运行中进程没有采用热更新后的 TTL；实际账号可以接受 1h。

排查期间上游新发布 1.3.32。1.3.30 让官方 Claude Code 请求也参与 TTL 解析，并按会话固定 TTL；1.3.31 更新会话到执行位的绑定；1.3.32 重编 CLI 并稳定缓存前缀中的账单字段。维护分支无冲突合并 tag，提交 `d28f6aa`。应用源码仍采用上游原版，没有增加自用协议补丁。

生产运行固定官方镜像 `ghcr.io/dofastted/vm2api:v1.3.32`，源码快照 `/opt/vm2api-release-1.3.32`。升级前再次确认执行位全部空闲，备份 `/opt/vm2api/backups/upgrade-1.3.32-20260923T123523Z/` 包含运行文件、在线 SQLite 和旧镜像记录。更新控制面、模板后，调用上游 `syncWrapSample()` 和 `restartRustKernel()` 同步并重载 vm-01；vm-02 保持停止。实际 CLI 与主 kernel 对应新版部署文件，健康返回 20 个空闲执行位。账号、Key、出口、额度和蒸馏配置保留。

VM2 默认缓存保持 `1h`，两槽 `kernel.json` 的 `default_cache_ttl` 均为 `1h`，文件权限与所属用户正确。本机 `~/.claude/settings.json` 和 CC Switch 的 VM2 provider 同步保存 `promptCacheTtl: "1h"`；保留用户的 `model: "opus"` 和状态栏，any provider 未修改。客户端备份位于 `~/.claude/backups/cache-1h-20260923-203359/`。

新版本使用真实 Claude Code 2.1.280、Opus 5.5，按两次顺序 Bash 调用执行三轮模型请求，全部成功；原始客户端响应的缓存读取为 3,026 → 8,544 → 8,642 tokens，新增 1h 缓存为 5,518 / 98 / 96 tokens，5m 写入均为零。证据来自客户端原始 usage，不依赖控制面按配置重标 TTL 的日志统计。验证会话为 `a7ea1e6a-5e7a-4131-bd82-4ce8d898d32d`，本机结果目录 `~/.omp/reports/vm2api-cache-1h-20260923/`。

该会话完成后空闲 360 秒，没有中间保活请求，再通过 `claude --resume` 续接原会话。请求正常完成，读取 8,738 个缓存 token，新增 632 个 1h 缓存 token、5m 写入为零，未缓存输入为 4 tokens。由此实测确认这段上下文在超过五分钟后仍可复用；结果为同目录的 `delayed-result.json` 与 `verification-state.json`。现有用户对话可以继续使用，无须新开对话或自行重载 VM2。

## 升级到上游 1.3.35（2026-09-23 晚）

用户确认升级后，将 v1.3.35 无冲突合入维护分支，合并提交 `e5c3dd6`。1.3.33–35 修复 cli-hop 搬移对话内 system 提醒而改变缓存前缀的问题，扩展中继官方请求识别，并增加缓存前缀诊断和更新控制台槽更新页。应用代码保持上游原版。

生产运行官方镜像 `ghcr.io/dofastted/vm2api:v1.3.35`，容器内 VERSION 已确认。源码与前端快照 `/opt/vm2api-release-1.3.35`。切换前核实 vm-01 的 20 个执行位全部空闲；备份 `/opt/vm2api/backups/upgrade-1.3.35-20260923T144743Z/` 包含原控制面文件、配置、槽登记、在线 SQLite 和旧镜像记录。只更新并重启控制面；二进制没有变化，没有执行槽内内核同步或重载。kin-01 的容器启动时间保持不变，CLI PID 11 与内核运行时长连续，凭证 fresh。`.env`、routing 与 distill 配置保持原内容，一小时缓存保留。

真实 Claude Code 2.1.280、Opus 5.5 连续两次 Bash 调用（三轮模型请求）成功，再用 `--resume` 续接同一验证会话成功。客户端原始 usage 的缓存读取依次为 0 → 9,063 → 9,155 → 9,247 tokens，写入依次为 9,063 / 92 / 92 / 771 tokens，全部为 1h，5m 写入为零；未缓存输入为 2 / 2 / 2 / 4 tokens。后台对应四次请求均 HTTP 200，`cache_prefix` 的 turn 连续为 1–4，break 均为 null。验证会话 `76678409-ce40-45cc-91e0-00f1d924ae98`，证据目录 `~/.omp/reports/vm2api-upgrade-1.3.35/`。现有用户对话可以继续使用，无须客户端重启或新建对话。

## 升级到上游 1.3.42（2026-09-24）

用户确认升级后，无冲突合入 v1.3.42，合并提交 `2c69069`。本次采用上游原版，包含静默任务取消与卡死执行位处理、额度/过载错误识别、槽内 OAuth 操作及可选 crag 数据面。现有数据面继续使用 wrap，没有切换到 crag。

生产运行固定官方镜像 `ghcr.io/dofastted/vm2api:v1.3.42`，源码快照 `/opt/vm2api-release-1.3.42`。备份 `/opt/vm2api/backups/upgrade-1.3.42-20260924T053942Z/` 包含配置、运行代码及二进制、在线 SQLite 和容器信息；旧镜像保留。切换控制面前、同步并重启账号槽前分别确认 vm-01 的 20 个执行位全部空闲。控制面更新后通过上游 `syncWrapSample()` 和 `writeKernelConfig()` 铺设新版，再重启原 kin-01 容器，使文件挂载的 kin-worker 及执行内核都实际更新；没有删除账号容器，vm-02 保持停止。

重启后内核健康为 `ready_slots=20`、`wedged_slots=0`、凭证 fresh，CLI 版本 2.1.280。实际运行 CLI 为新版 34,645,320 字节文件，容器挂载的 kin-worker 为 16,070,541 字节。`.env`、`routing.json`、`distill-rules.json` 内容与备份一致，一小时缓存、账号出口和原额度阈值保留。官方镜像未包含可选 crag 文件，另从相同 v1.3.42 tag 的 `share/crag/kin-kernel` 补入部署目录；仅提供切换所需文件，未启用该模式。

本机 Claude Code 沿用保存的 `claude-opus-5-5[1m]`，实际依次执行两次 Bash 后返回 `VM2_1342_OK`。对应三次请求全部 HTTP 200；客户端原始缓存读取 0 → 7,978 → 8,084，写入 7,978 / 106 / 106，全部为 1h，5m 为零，未缓存输入每次 2 tokens。客户端报告 `contextWindow=1000000`。验证会话 `0c12b0a4-0ae3-454c-b9be-427e8faa9100`，证据目录 `~/.omp/reports/vm2api-upgrade-1.3.42/`。没有进行百万 token 的大请求。

本次额外核查发现：当前 `credentials.json` 的 scopes 只有 `user:inference`，槽内官方 `/api/oauth/usage` 返回 HTTP 403 `oauth_scope_insufficient`，要求 `user:profile`。调用使用的就是该凭证路径，属于当前授权范围限制；聊天和工具调用已实际成功，不把此结果解释为账号不可用。未更换登录凭证、强制刷新或调整权限。后续若要恢复主动额度查询，需要具有相应 profile 权限的 OAuth 登录。

## 升级到上游 1.3.49（2026-09-25）

用户确认升级后，无冲突合入 v1.3.49，合并提交 `e41af6c`。生产采用固定官方镜像 `ghcr.io/dofastted/vm2api:v1.3.49`，继续使用 wrap。此次上游更新包括旧 CPU 的 CLI 指令集兼容、空输出重试与过载判定、Codex 用量报告及路由设置保存修复；没有增加自用协议补丁。

升级前确认在途请求为零、20 个执行位全部空闲。备份 `/opt/vm2api/backups/upgrade-1.3.49-20260924T162801Z/` 保存运行文件、在线数据库、切换前数据库、容器信息及升级前后内核健康记录。新版文件从官方镜像提取到 `/opt/vm2api-release-1.3.49`，同步至 `/opt/vm2api` 后重建控制面；上游自动同步日志显示 `2/2, failed=0`。vm-01 实际 CLI 进程已加载新版 34,287,864 字节文件，健康返回 `ready_slots=20`、`wedged_slots=0`、凭证 fresh、CLI 2.1.280；vm-02 保持停止。

`.env`、`routing.json`、`distill-rules.json` 保留原内容；账号、Key、出口、额度阈值和 1h 缓存保留。Claude Code 沿用已配置的 `claude-opus-5-5[1m]`，实际完成两次 Bash 调用后返回 `VM2_1349_OK`。三轮客户端原始 usage 中缓存读取为 0 → 42,359 → 42,465 tokens，写入为 42,359 / 106 / 106 tokens，全部计入 1h，5m 写入均为零，未缓存输入每轮 2 tokens。客户端报告上下文容量 1,000,000。证据目录为 `~/.omp/reports/vm2api-upgrade-1.3.49/`。

## 单账号容器内存提高到 2 GiB（2026-09-25）

北京时间 10:09:33，宿主内核日志明确记录 kin-01 达到 500 MiB cgroup 上限，OOM killer 杀掉 `cli-node`；请求停滞后，10:11:35 Cloudflare 返回 524。10:12:35 账号容器已重启。随后同一账号 Opus 5.5 实际请求成功，故障原因是容器内存限制。

用户只有一个运行账号，授权提高内存。通过 `docker update --memory 2g --memory-swap 2g kin-01` 在线增加上限；宿主约 4 GiB 内存。`.env` 保存 `KIN_VM_MEMORY=2g`，通过上游 routing API 将 `official_cc.memory` 改为 `2g`，并用上游 helper 保存 vm-01 的 `runtime.memory`。确认无在途请求后，仅重建控制面以加载环境变量；kin-01 的启动时间和宿主 PID 均保持不变，未重启账号容器或 CLI。vm-02 继续停止。

运行中 Docker 限额和控制面环境已确认均为 2 GiB，凭证 fresh、无卡死执行位，1h 缓存保留。实际 OMP → VM2 → Opus 5.5 请求返回 `VM2_MEMORY_OK`，新增缓存写入为 7,165 个 1h tokens。远端配置备份与验证记录位于 `/opt/vm2api/backups/single-account-memory-2g-20260925T022132Z/`，本机请求记录位于 `~/.omp/reports/vm2api-memory-2g-20260925/`。后续部署须保留 `.env` 的 `KIN_VM_MEMORY=2g` 和 `official_cc.memory=2g`，避免恢复为上游 500 MiB 默认值。应用代码仍采用上游原版。
