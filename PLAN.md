# Multica `start_date` + 只读甘特图 MVP 计划

## Summary

- 目标分两批完成：先给 issue 加 `start_date`，再在 Issues 页面增加只读 `gantt` 视图。
- 本期不做拖拽改排期、不做依赖线、不做自动推算开始/截止日期。
- 甘特图只展示有明确日期的数据：`start_date` 和 `due_date` 都存在的 issue 画时间条；缺任一日期的 issue 放到“未排期”区域，不用 `created_at` 或 `due_date` 做兜底。

## Key Changes

- 后端新增 issue 字段：`start_date TIMESTAMPTZ NULL`，迁移文件使用下一个编号 `069_issue_start_date`。
- API 增加 `start_date`：`Issue` 响应、`CreateIssueRequest`、`UpdateIssueRequest`、批量更新、CLI `issue create/update` 都支持该字段，格式沿用 `due_date` 的 RFC3339 字符串。
- 更新 SQL 和生成代码：在 issue list/get/create/update 相关查询中加入 `start_date`，然后运行 `make sqlc`。
- 前端类型增加 `start_date: string | null`，日期选择器复用现有 `DueDatePicker` 思路，抽出通用日期 picker 或新增 `StartDatePicker`，详情页属性区展示“开始日期”和“截止日期”。
- activity/通知本期只记录 `start_date_changed` 事件并显示“开始日期设为/移除了开始日期”，不新增复杂排期事件语义。

## Gantt MVP

- Issues 页面新增第三种视图模式：`board | list | gantt`，入口放在现有视图切换菜单。
- 甘特图使用当前 Issues 页已有筛选结果，不单独新增后端接口；状态、负责人、项目、标签等筛选继续生效。
- 甘特图按状态分组展示 issue，时间范围根据可排期 issue 的最早 `start_date` 和最晚 `due_date` 自动计算，并至少覆盖当前周。
- 每条 issue 显示：identifier、标题、负责人头像、状态颜色条、起止跨度。
- `start_date > due_date` 的异常 issue 不画正常条，放入“日期异常”区域，提示需要修正日期。
- 无日期或缺一端日期的 issue 放入“未排期”区域，支持点击进入详情页修改日期。
- 桌面端和 Web 共用 `packages/views` 中的甘特图组件，不在 app 层重复实现。

## Test Plan

- 后端：补充 create/update issue 的 `start_date` 成功、清空、非法格式返回 400、响应字段正确的 Go 测试。
- SQL/codegen：运行 `make sqlc` 后确认生成代码包含 `StartDate`，再跑 `make test` 或至少相关 handler 测试。
- 前端：补充类型与组件测试，覆盖详情页展示/更新开始日期、视图菜单出现 Gantt、甘特图分类“已排期/未排期/日期异常”。
- 校验：运行 `pnpm typecheck`、`pnpm test`；必要时启动 `make dev` 后手动检查 Issues 三种视图切换。

## Assumptions

- `start_date` 是手动维护字段，不由 agent 开始执行时间或 `first_executed_at` 自动填充。
- 本期不改变 issue 排序默认值；只在显示设置里新增按 `start_date` 排序。
- 本期不引入新的甘特图库，优先用 React + CSS 实现只读时间轴，避免公开仓库后续依赖维护压力。
- 兼容旧数据：历史 issue 的 `start_date` 默认为 `NULL`，不会迁移填充。
