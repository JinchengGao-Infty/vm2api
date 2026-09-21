# 自用维护记录

## 分支与更新方式

- 维护分支：`infty/main`，远端 `fork` 指向 `JinchengGao-Infty/vm2api`。
- 上游来源：`origin` 指向 `dofastted/vm2api`。
- 2026-09-20 从已保留缓存部署修复的 `a283a61` 建立维护分支。该提交已合入上游 `30c06a9`，版本字段为 1.2.9。
- 上游 PR #18 已按用户要求关闭；修复继续保留在本分支。

接收上游更新时，先查看更新涉及哪些代码，再合并到 `infty/main`，解决冲突时保留下面列出的修复。推送到 `fork`。维护分支和生产部署分别记录，不把合并完成当作服务器已升级。

上游面板更新和一键安装脚本会选择上游 release tag，不能用它们覆盖这个维护分支。部署应使用我们选定的维护分支提交，并保留现场修改和可恢复的旧版本。

## 必须保留的修复

`src/lib/vm/wrap-cli-runtime.mjs` 中的 `kernelPayloadPath()` 统一内核来源：

1. `KIN_KERNEL_BIN` 指向的文件存在时，优先采用它。
2. 否则保留模板或槽位的 `kin-kernel.bin`，再回退到已有 wrapper。
3. 安装/同步、制作模板、从槽位晋升模板都使用这一规则；CLI 和 glibc 文件仍来自所选模板。

原因：旧模板和旧槽位会跨版本保留，原来的来源优先级能把已经升级的主内核覆盖回旧内核。单纯重启控制面不能让正在运行的槽内进程加载新内核；相应部署需要同步并重启目标槽。

这项修复此前已在现有部署回放三条连续真实 OMP 请求：缓存读取为 0、40,645、42,333 tokens，新增历史能进入缓存。原始请求和响应留在仓库之外。

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
