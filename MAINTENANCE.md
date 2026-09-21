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
