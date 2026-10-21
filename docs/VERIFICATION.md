# 本地额度验收记录

2026-10-21，Ubuntu / WSL2，Node 24.14.1、Wasp 0.25.0、PostgreSQL 16.15。**本轮本地实现与验证已完成；未发布 PR。** 原始环境基线不是本轮功能证据，仍单独保留在 [BASELINE_RUN](BASELINE_RUN.md)。

对应实现提交 `da266a9146fc173213840f87328c36dafdc47f1c`；设计提交 `99cecc9` 保留。

执行入口与完整命令见 [QUOTA_RUN](QUOTA_RUN.md)：S=真实 PostgreSQL 服务测试，H=真实 HTTP，B=Edge/Playwright，C=编译与类型检查。验收前置条件和操作按 [SPEC 第 6 节](../SPEC.md#6-验收案例)；下表列明预期和实际。S 中 HTTP 状态是业务错误到响应的映射结果，并非声称发生了真实 HTTP 数据库断网。

## 1. 汇总与证据

- S：完整运行 **17/17 测试通过，0 失败、0 跳过**，52 条状态观测；AC07/AC08 各 10 轮，并完成 5 个独立 Node 进程竞争。
- H：**35 个场景全部通过**；包含 20 轮五请求并发。每轮同时断言响应、数据库余额/预占、结果增量和模型日志。
- B：**12 步全部通过，0 个页面异常**；错误提示注入与真实模型失败分别记录。
- C：`wasp compile`、`npx tsc --project tsconfig.src.json --noEmit`、`npx tsc --project tsconfig.wasp.json --noEmit` 均退出 0。
- 新迁移已在开发库成功应用；最终只读核对为负余额 0、RESERVED 0，两条迁移均已完成。模型文件、原初始化迁移和依赖锁文件保持原样。

随 Git 提交的 [脱敏结果摘要](verification/summary.json) 包含逐场景的实际状态、模型次数和结果计数，供审阅。它是本次运行的快照，不代替重新执行。下列原始证据在本机存在但受 Git 忽略，**不随 PR 提交**：

- [S 测试报告](../test-results/quota/vitest.json)、[S 状态观测](../test-results/quota/service-observations.json)、[S 最终日志](../test-results/quota/service.log)。
- [H 报告](../test-results/quota/http.json)、[H 日志](../test-results/quota/http.log)。
- [B 报告](../test-results/quota/browser/report.json)、[成功截图](../test-results/quota/browser/success.png)、[失败释放截图](../test-results/quota/browser/failure-released.png)、[双标签页零额度截图](../test-results/quota/browser/zero-credit-two-tabs.png)。
- [编译日志](../test-results/quota/compile.log)、[源码类型检查](../test-results/quota/typescript-src.log)、[Wasp 类型检查](../test-results/quota/typescript-wasp.log)。两项类型检查无诊断输出，退出码保留在摘要中。

## 2. 逐项结果

C 为可用额度，P 为 RESERVED 数，M 为模型 started 数，ΔR 为本场景结果增量。S 的统计以当前场景为界；AC11 使用两个账号的聚合 M=2，分别各调用 1 次。

| 编号 | 命令组 / 前置与操作 | 预期 | 实际结果 / 证据 |
| --- | --- | --- | --- |
| AC01 | S/H/B；新注册查看 | C=3，P=0，M=0，ΔR=0 | 通过；数据库、auth 与页头均为 3，空任务按钮禁用；S/H AC01、B AC01-UI |
| AC02 | S/H/B；C=3 普通生成 | 200；C=2，M=1，ΔR=1 | 通过；显示日程与消费提示，P=0；S/H AC02、B AC02-UI |
| AC03 | S/H/B；C=1 生成 | 200；C=0，M=1，ΔR=1，按钮禁用 | 通过；页头同步到 0；H AC03、B AC03-AC06-UI |
| AC04 | S/H/B；C=0 直接请求 | 402；C=0，M=0，ΔR=0 | 通过；未调用模型、未新增结果；H AC04、B AC04-AC13-stale-direct-request |
| AC05 | S/H/B；C=1，包含 [fail] | 502；释放后 C=1，M=1/failed=1，ΔR=0 | 通过；页面明确已释放，保留旧结果并改标题；H AC05、B AC05-UI |
| AC06 | S/H/B；删除失败任务再生成 | 200；C=0，M=1，ΔR=1 | 通过；未被前次失败阻塞；H AC06、B AC03-AC06-UI |
| AC07 | S/H；C=1，5 并发×10 轮 | 每轮 1×200/4×402，C=0，M=1，ΔR=1 | 全部通过；S 10 轮及 H AC07-1～10 均符合，无负数余额或多发模型 |
| AC08 | S/H；C=3，5 并发×10 轮 | 每轮 3×200/2×402，C=0，M=3，ΔR=3 | 全部通过；S 10 轮及 H AC08-1～10 均符合 |
| AC09 | S 屏障保持失败预占；H 无屏障补测 | 屏障阶段 1×502/2×402，C=1，M=1，ΔR=0；随后可成功 | 通过；S 恢复为 200/C=0/M=1/ΔR=1；H 本次实际也是 [502,402,402]，但脚本依 SPEC 允许失败释放后再受理 |
| AC10 | S 在 T2 状态更新后注入真实 SQL 错误，再恢复 | 首次 500/C=1/M=1/ΔR=0；随后成功 | 通过；T2 回滚、T3 释放，第二次 200/C=0/ΔR=1；B 保存失败提示另用注入验证 |
| AC11 | S；两个账号各 C=1，模型返回屏障 | 两账号都进入模型再释放，各 200/C=0/ΔR=1 | 通过；聚合 M=2，不依赖耗时阈值；S AC11 A/B |
| AC12 | S/H/B；未登录、空任务、非法时长；合法边界/小数 | 401/400 均 M=0/ΔR=0；合法值可成功 | 通过；S 覆盖 NaN/±Infinity；H 覆盖认证、空任务、0/25/字符串及 1/24/1.25；B 非法输入禁用 |
| AC13 | B；双标签页 C=1，A 消费后聚焦 B，再直接请求 | B 显示 0；直接请求 402/M=0/ΔR=0 | 通过；实际页头与服务器一致，无新模型调用；B AC04-AC13-stale-direct-request |
| AC14 | S；数据库 0/认证快照 3，及数据库 1/快照 0 | 分别 402/M=0/ΔR=0，及 200/C=0/M=1/ΔR=1 | 通过；两种过期方向均以数据库为准；S stale high/low |
| AC15 | S；T1 扣减后插入前 SQL 错误，恢复后再生成 | 首次 503/C=1/P=0/M=0/ΔR=0；随后成功 | 通过；确认预占整体回滚；S rollback/recovery，B DB_UNAVAILABLE 文案注入通过 |
| AC16 | S；并发两次释放后再释放；成功后再尝试释放 | 只返还一次，成功不退款 | 通过；预占 fixture 最终 C=1/P=0/M=0/ΔR=0；成功 fixture C=0/M=1/ΔR=1，重复释放不改变 |
| AC17 | S；T2 真实提交后抛提交应答丢失，再确认 | 返回既有成功；C=0/P=0/M=1/ΔR=1 | 通过；未再次调用或保存、未退款；S recovered commit/repeated confirmation |
| AC18 | S；模型失败后持续阻断释放；另一个请求已提交但阻断确认 | 都 OUTCOME_UNKNOWN；保留各自真实状态，不承诺退款 | 通过；A C=0/P=1/M=1/ΔR=0；B C=0/P=0/M=1/ΔR=1。B 的 UI 注入显示编号、不声称未扣费、无自动重试 |
| AC19 | S；三份任务快照/预占，其中一份失败，屏障期间发第四请求 | 中途 C=0/P=3；终态 2×200/1×502，C=1/P=0/M=3/ΔR=2 | 通过；第四请求 402 且不新增调用；S AC19 |
| AC20 | S；5 个独立 Node 进程共用测试库 C=1 | 1×200/4×402，C=0/P=0/M=1/ΔR=1 | 通过；跨进程 IPC 聚合模型日志；未冒称多副本 HTTP 部署测试 |
| AC21 | S；旧结构库含历史记录；另建负余额库，应用迁移 | 普通库保留数据；负余额库拒绝，不修正金额 | 通过；正常退出 0，C=2、任务/结果各1、旧关联 NULL；负余额退出 3、C=-1 保留，无部分新表；M=0 |
| AC22 | S；T3 状态更新后退款 SQL 一次失败 | T3 全回滚后第二次清理成功，502/C=1/P=0/M=1/ΔR=0 | 通过；第二次释放前确认 C=0/P=1，有限重试共2次；S AC22 |
| AC23 | S；T1 真提交后丢失应答；可清理/持续不可清理两例 | 均零模型；已释放为 DB_UNAVAILABLE，否则 OUTCOME_UNKNOWN | 通过；前者 503/C=1/P=0/M=0/ΔR=0；后者 503/C=0/P=1/M=0/ΔR=0；S AC23 |

## 3. 失败定位与重跑

首轮 S 为 16 通过、1 失败：AC21 在创建旧结构 fixture 时，新客户端默认读取了尚不存在的 generationAttemptId。修正为只返回旧结构已有的 id，未修改迁移或削弱断言。AC21 单独重跑通过后，最终完整 S 重跑 17/17 通过。首轮与定点日志保留于本机 `service-first.log`、`migration-rerun.log`。

Wasp 客户端实际将服务端详情封装为 error.data.data，页面按生成代码适配，真实 [fail] 浏览器测试已验证。未修改模拟模型的延迟、失败触发或日志来通过测试。

## 4. 已知边界及未执行事项

- 服务崩溃自动补偿、模型永久挂起的取消、跨请求幂等不在题目范围内，未实现也未宣称验证。
- 未进行真实数据库网络故障/操作系统断电测试。提交应答丢失通过“真实数据库提交后抛出传输错误”注入验证；UI 错误响应通过 Playwright 路由注入验证。
- 未部署多副本 HTTP 服务；AC20 已验证独立进程共享数据库的原子准入。
- H/B 的专用本地账号保留；S 自己的 fixture 已清理。密码与连接配置不提交，原始大日志/截图仅本机保存。
- 个人 Fork、应聘称呼/打码手机号、招聘通知时间待提供；未创建或发布 PR。上述事项不阻塞本地实现交付。
