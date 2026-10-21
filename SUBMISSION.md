# 提交说明

- 姓名或称呼：待提供（公开提交前填写）
- 应聘方向：技术｜Agent 应用开发
- 打码手机号：待提供（仅前三位 + `****` + 后四位）
- 当前交付：本地实现与验证完成，尚未发布 PR

## 结果概览

原项目余额为 0 时仍调用模型，并发准入使用认证余额快照。现在在模型调用前用 PostgreSQL 条件更新预占额度，按请求记录确认消费或释放。模型失败不消费，结果保存和消费状态在同一事务确认，重复释放及提交应答丢失不会盲目退款。

前端显示可用额度、同步刷新，区分明确失败与结果不明，零额度禁用生成并说明下一步。保留本地模拟模型和原有任务语义。

## 交付入口

- [SPEC](SPEC.md)：规则、方案取舍、流程、边界、实现拆解与 AC01～AC23。
- [quotaService](src/demo-ai-app/quotaService.ts)、[planService](src/demo-ai-app/planService.ts)：原子预占、确认、释放与异常编排。
- [迁移](migrations/20261021133000_quota_reservation/migration.sql)、[页面](src/demo-ai-app/DemoAppPage.tsx)。
- [运行和重复验证](docs/QUOTA_RUN.md)：S/H/B/C 完整命令及环境前提。
- [逐项实际验收](docs/VERIFICATION.md)、[脱敏结果摘要](docs/verification/summary.json)。
- [数据库测试](tests/quota.integration.test.ts)、[HTTP 脚本](scripts/verify-quota.mjs)、[浏览器脚本](scripts/verify-browser.cjs)。

## 验证记录

AC01～AC23 的命令组、前置操作、预期、实际与证据逐项列在 VERIFICATION，避免在此重复。最终结果：真实 PostgreSQL 17/17 测试通过；HTTP 35 场景通过，包含两种余额各10轮五请求并发；浏览器12步通过、0页面异常；编译及两项 TypeScript 检查退出0。

环境原始基线单独保留在 [BASELINE_RUN](docs/BASELINE_RUN.md)。测试报告没有把设计预期当作通过记录；本机原始日志和截图受 Git 忽略，随提交保存的是脱敏快照和可重跑入口。

## 待办与交接

设计提交为 `99cecc9994a24fbd78549f54dd2f9166a6bf2cfd`，实现提交为 `da266a9146fc173213840f87328c36dafdc47f1c`，验证记录接续其后，未重写上游历史。

服务崩溃或数据库长期不可用可能留下暂占额度，自动崩溃补偿在题目范围外；未知状态不得自动重试或无条件返还。未做真实网络断电测试和多副本 HTTP 部署；对应替代验证及边界见 VERIFICATION。

本轮按要求只完成本地交付。发布前仍需补齐称呼、打码手机号、个人 Fork 和招聘通知时间，在48小时截止前按 `技术｜称呼｜138****5678` 填写 PR 标题；不填写完整手机号。
