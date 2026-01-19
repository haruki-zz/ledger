# Implementation Plan

1. 创建并提交 `memory-bank/architecture.md` 草稿：列出所有模块边界（客户端 UI、领域模型、SQLite/GRDB 存储、Supabase 同步、变更队列、CSV 导出、设置开关），并完整抄写当前数据库表与字段/约束列表，标明主/副标签规则与软删除字段。
   - Test: 对照 `prd.md` 与 `design-document.md` 勾选核对每个表（categories、tags、locations、entries、entry_tags、budgets、budget_rollovers、fixed_expense_templates、fixed_expense_runs）是否在文档中出现且字段/约束齐全。

2. 初始化 SwiftPM 多模块工程（如 `AppCore`, `DataStore`, `Sync`, `UI`, `Export`），配置 iOS/macOS target，接入 SwiftUI 与 async/await 基础依赖，不引入多余第三方。
   - Test: 运行 `swift test` 与 `xcodebuild -scheme Ledger -configuration Debug build`，确认空模块可编译。

3. 建立领域模型定义（Entry、EntryTag、Budget、BudgetRollover、FixedExpenseTemplate/Run、Category、Tag、Location），使用值类型并包含软删除/同步字段，主副标签角色明确。
   - Test: 编写模型序列化/反序列化单元测试，验证 currency 固定为 JPY、金额整数、role(primary/secondary) 正确映射、软删除字段可选存在。

4. 用 GRDB 编写本地 SQLite 迁移，创建所有表、唯一约束与索引（如 budgets 唯一 user_id+month+tag_id、entry_tags 复合主键 entry_id+tag_id+role、entries 按 user_id+occurred_on 索引）。
   - Test: 在测试中运行迁移到内存数据库，验证创建的约束/索引存在；对违反唯一约束的插入进行失败断言。

5. 搭建本地仓储接口（Repository）和 DAO，提供 CRUD、软删除、批量查询与按月聚合入口，所有操作纳入事务。
   - Test: 仓储集成测试覆盖插入/更新/删除/软删除后查询结果正确，金额累加不重复计算，主标签冗余 main_tag_id 与 entry_tags 校验一致。

6. 实现本地变更队列（按时间戳顺序）持久化，记录操作类型、实体快照、rev/device_id，并提供幂等标记。
   - Test: 队列单元测试验证入队顺序、重复操作合并规则、软删除入队后实体不可被再次创建；模拟离线期间入队多操作，确认重启后顺序保留。

7. 集成 Supabase Auth（邮箱/Apple 登录），封装 token 刷新与 Keychain 存储，限制到 user_id 作用域。
   - Test: 使用 supabase-swift stub/假响应模拟登录、token 过期与刷新流程，断言 Keychain 持久化成功且请求头带上有效 token。

8. 封装 Supabase 数据访问层（PostgREST 调用），对每张表提供增量拉取/推送接口，带 rev/version 和 device_id；禁止直接操作 PostgREST 之外的通道。
   - Test: 针对每个表的 API 调用编写契约测试，校验请求 payload 字段、RLS user_id 限制、失败重试逻辑与超时处理。

9. 实现同步引擎：登录/前台/手动/定时触发 → 读取变更队列按顺序推送 → 拉取远端差异 → 写入本地并更新 rev → 清理已推送项。
   - Test: 同步集成测试在内存数据库 + stub 网络下，验证顺序推送、失败重试间隔、成功后队列清空；断言拉取后本地数据与远端快照一致。

10. 冲突检测与解决：发现远端 rev 更新且字段差异时，生成本地/远端版本 diff，阻断自动合并，要求用户选择一侧覆盖整条记录。
    - Test: 构造本地修改与远端修改同时存在的 entry/budget 案例，触发冲突后断言 UI/状态机返回需要决策的状态；选择“保留本地”或“接受远端”后，验证数据库记录与 rev 更新符合选择。

11. 记账核心流程：快速新增 entry（金额、主标签、可选副标签、日期、地点/备注），写入 entries 与 entry_tags，保证总支出只计一次并打上 is_fixed 标识（默认为 false）。
    - Test: 用用例测试覆盖单主多副、无副标签、重复副标签去重；验证金额存为整数、千分位展示格式化、主标签与 entry_tags 一致性校验。

12. 预算管理：空白月份自动复制上月草稿；当月出现第一条 entry 时将对应月份 budgets.locked=true；锁定后禁止编辑，仅可查看。
    - Test: 预算模块测试验证复制逻辑（缺失时创建新草稿）、首笔记账触发锁定、锁定后更新操作抛出错误或被拒绝；>80% 与 >100% 预警状态计算正确。

13. 预算结转：依据设置开关决定是否将上一月结余写入 budget_rollovers 并计入下月预算合计。
    - Test: 构造上月预算/实际支出数据，验证结余计算、写入 rollover 记录；切换开关后检查结余是否被包含在下月总预算计算中。

14. 固定开销模板与生成：管理模板（amount、tag、day_of_month、active），每月初检查 fixed_expense_runs，如未生成则创建 entry + run(status=created/posted) 并标记 is_fixed=true。
    - Test: 定时任务单元测试模拟月初触发，验证已生成月份不重复；模板停用后不生成；生成的 entry 在列表/图表中按固定开销样式标记。

15. 转账与充值规则：默认充值为转账、不计支出；开关开启时充值计支出；还款始终为转账且不重复计支出。
    - Test: 规则测试覆盖三种场景（默认、开关开启、还款），断言 entries 分类、预算影响与统计口径符合设定；确保转账不出现在支出合计中。

16. Dashboard 计算与缓存：按月聚合 entries/entry_tags、预算进度、固定开销总额、每日折线、分类占比/趋势、活跃度 heat map；必要时维护月度缓存以减少重算。
    - Test: 数据聚合测试在固定数据集下验证各模块输出（预算进度、固定开销淡化标记、热力图计数/金额），并验证缓存命中与失效策略在变更后即时更新。

17. 列表与详情：按日期倒序展示，固定开销淡色，超支分类用红色；详情可编辑或软删除。
    - Test: UI/状态测试确认排序正确、软删除后列表隐藏且数据库标记 deleted_at、编辑后预算/统计实时更新，超支条目呈现红色。

18. 导出与设置：设置页提供 CSV 导出入口（按年全字段）和两个开关（预算结转、充值计支出）。
    - Test: 导出测试生成 CSV，断言包含所有字段与主/副标签展开；切换开关后复查预算结转与充值规则逻辑是否随设置生效。

19. 监控与错误处理：统一 os_log 分级日志，关键路径（同步、队列、预算锁定、固定开销生成）记录；错误提示明确且不泄露敏感信息。
    - Test: 日志与错误处理测试验证失败场景（网络、迁移冲突、RLS 拒绝）被捕获并输出期望日志；UI 呈现对应错误状态且可重试。

20. 回归与文档更新：执行全量 `swift test`、关键 E2E 用例（快速记账、预算锁定、固定开销生成、冲突解决、导出）；更新 `design-document.md`/`prd.md`/`memory-bank/architecture.md` 反映实现与开关默认值。
    - Test: CI 中运行测试套件与静态分析（如 SwiftLint/Format 如有），确认全部通过；人工检查文档与实现一致，尤其是数据库 schema 与开关默认值。
