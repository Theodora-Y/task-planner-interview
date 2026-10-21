# 额度功能运行与验证

2026-10-21：额度功能已实现并完成本地验收。业务规则与边界见 [SPEC](../SPEC.md)，逐项实际结果见 [VERIFICATION](VERIFICATION.md)。本机环境、启动/停止脚本见 [BASELINE_RUN](BASELINE_RUN.md)；无需重新安装环境或升级依赖。

## 1. 更新已有开发数据库

在 Ubuntu 正式源码目录执行（本机已执行成功，不必重复建库）：

```sh
source /home/ying/实习小作业/env.sh
cd /home/ying/实习小作业/task-planner-interview
wasp compile
wasp db migrate-dev
```

新增迁移为 `20261021133000_quota_reservation`；保留旧用户、任务、结果和余额，旧结果的 attempt 关联为 NULL。若历史余额小于 0，迁移拒绝执行，需要先人工核实，不会自动改成 0。

仍使用现有 `.env.server` 和 PostgreSQL 55432。本机 Windows PowerShell 日常操作：

```powershell
wsl -d Ubuntu -- bash /home/ying/实习小作业/start.sh
wsl -d Ubuntu -- bash /home/ying/实习小作业/status.sh
wsl -d Ubuntu -- bash /home/ying/实习小作业/stop.sh
```

前端 [localhost:3000](http://localhost:3000)，API 3001。新注册账号 3 个可用额度。生成时暂占 1 个，成功后消费；添加包含 `[fail]` 的任务会明确失败并释放，删除该任务后可继续。处理中看到 0 可能是暂占，完成后点击“刷新额度”。没有充值入口。

## 2. 真实 PostgreSQL 服务测试（S）

先编译生成当前 Prisma 客户端，再创建独立测试库：

```sh
node scripts/prepare-test-db.mjs --name task_planner_quota_test
npx vitest run --config vitest.server.config.ts tests/quota.integration.test.ts
```

准备脚本读取本地 `.env.server`，要求数据库角色有 CREATEDB 权限。仅接受显式指定、名称以 `_quota_test` 结尾的数据库；写入受 Git 忽略、权限 0600 的 `.env.quota-test`。测试连接显式使用 TEST_DATABASE_URL，并再次检查库名。

S 覆盖真实事务、模型调用、SQL 错误回滚、提交成功后丢失应答、屏障并发、跨账号与五个独立 Node 进程。AC21 另建临时隔离库，验证旧结构升级及负余额中止迁移；最后删除这些临时库。其余测试仅清理本轮 fixture ID，不清空其他数据。

报告位于 `test-results/quota/vitest.json`、`service-observations.json`。测试故意注入的失败会输出 `quota.settlement-unconfirmed`，以最终断言和退出码判断通过，不按 stderr 是否为空判断。

跨进程 helper 使用本项目要求的 Node 24 类型转换功能，可能显示 ExperimentalWarning；未新增运行时依赖。失败注入只在内部依赖参数中，HTTP 不提供故障开关。

## 3. 真实 HTTP 验证（H）

应用启动后执行：

```sh
node scripts/verify-quota.mjs \
  --base-url http://localhost:3001 \
  --server-log /home/ying/实习小作业/.environment/logs/app.log \
  --rounds 10
```

H 使用真实注册、登录、任务和生成操作。读取 `.env.server` 的 DATABASE_URL 对自己创建的账号做只读检查；**这个连接必须与 API 连接同一个数据库**，不能拿另一个测试库核对开发 API 的账号。S 的隔离库和 H 的数据库用途不同。可用 `--database-env <env文件>` 指向另一套与被测 API 一致的配置。

每个正式场景记录响应、前后可用额度/预占数、结果增量及真实模型 started/succeeded/failed 和 callId。额度 1/0 的准备调用不计入场景统计。并发请求一起创建并通过 Promise.allSettled 等待；余额 1 和 3 各重复 10 轮。

执行期间不要混入人工生成请求，也不要轮转/截断指定日志。缺日志、非 JSON、超时和断言失败均非零退出。报告默认写入 `test-results/quota/http.json`。参数说明：`node scripts/verify-quota.mjs --help`。

H 留下独立 `quota_*` 账号、任务和结果，以便复核；密码只保存在进程内、不写报告。S 不接触这些账号。删除测试账号或清理结果不是产品能力，本次不提供对整个开发库的清空命令。

## 4. 浏览器验证（B）

脚本依赖已安装的 Playwright 和 Chromium/Edge，不加入应用依赖。本机 Windows 已有工具，以下命令已实际执行：

```powershell
$env:PLAYWRIGHT_MODULE_PATH='D:\YYY\找实习\实习小作业\.environment\browser\node_modules\playwright'
$env:SERVER_LOG='\\wsl.localhost\Ubuntu\home\ying\实习小作业\.environment\logs\app.log'
node '\\wsl.localhost\Ubuntu\home\ying\实习小作业\task-planner-interview\scripts\verify-browser.cjs'
```

跨机器需提供自己的 Playwright 安装位置；支持 FRONTEND_URL、API_URL、BROWSER_CHANNEL。本机使用 msedge。B 与 H 不同时运行，避免日志区间混入别的模型调用。

B 覆盖真实注册、正常生成、明确失败、恢复、余额/按钮同步、双标签页和旧凭据直接请求。双标签页调用 bringToFront，并分发 focus 事件，保证无头窗口管理器也能复现焦点刷新路径。错误响应注入案例在名字中明确标为 injected：仅证明页面能显示保存失败、状态不明、服务故障和非 JSON 错误；不能替代 S 的数据库故障验收。

报告和截图位于 `test-results/quota/browser/`；账号密码不落盘。B 留下 `browser_*`、`ui_errors_*` 测试账号。

## 5. 编译与类型检查（C）

```sh
wasp compile
npx tsc --project tsconfig.src.json --noEmit
npx tsc --project tsconfig.wasp.json --noEmit
```

本机为避免生成目录与开发服务同时写入，先停止应用、保持数据库运行再检查，完成后用 start.sh 恢复服务。详见原始环境记录中的前台/后台进程管理方式。

## 6. 故障边界与人工核对

模型在数据库事务之外。T1 预占、T2 保存并确认、T3 释放均为短事务。同一个 attempt 只能从 RESERVED 进入一个终态；重复释放或提交应答丢失不会无条件返还额度。

进程崩溃、模型永不返回或数据库持续不可用可能遗留 RESERVED；自动崩溃补偿在题目范围外。页面显示状态不明时不要自动重试生成，保留请求编号。人工释放前必须停止相关业务进程，核对该账号、attempt 和关联结果，再使用同一条件释放事务；不得直接加余额，也不得只按记录年龄判断。

这些边界并不表示本次测试存在未结算请求。实际测试结果、未执行事项及招聘提交待办分别见 VERIFICATION 和 SUBMISSION。
