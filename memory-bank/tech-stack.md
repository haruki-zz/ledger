# 技术栈方案（最简且健壮）

## 客户端（iOS / macOS）
- 语言与框架：Swift 5.9+，SwiftUI 多端共享视图；Swift Concurrency（async/await）+ Observation/Combine 处理状态与数据流。
- 本地存储：SQLite + GRDB（事务/迁移可控，支持行级并发）；本地表与服务端结构对齐，含变更队列与软删除字段。
- 网络层：URLSession + supabase-swift SDK（Auth/REST 调用 PostgREST）；统一请求封装，内置重试与超时。
- 图表与视觉：Apple Charts 框架（折线/柱状/饼/进度条样式定制）；系统等宽字体与 TUI 风格组件。
- 同步与后台：前台/下拉触发 + BGTaskScheduler 定时任务（iOS），macOS 使用 App life-cycle 定时；变更队列按顺序推送，24h 自动重试。
- 导出：本地生成 CSV（按年全字段），使用 CodableCSV 或等价轻量 CSV 写入库，文件分享/保存由系统分享面板处理。
- 监控与调试：os_log 分级日志；XCTest + Swift Testing（单元/集成）+ Xcode Instruments 做性能与内存检查。
- 依赖管理：Swift Package Manager；不引入多余三方 UI 组件以保持轻量。

## 后端（Supabase）
- 数据库：Supabase Postgres，表结构与 PRD/设计文档一致；使用 RLS 以 user_id 限定访问。
- API：PostgREST + supabase-swift 客户端；无额外自建 API 层，减少复杂度。
- 认证：Supabase Auth（邮箱/Apple 登录）；客户端持短期访问令牌。
- 实时与消息：暂不启用 Realtime，避免多余复杂度；同步依赖轮询/触发式请求。
- 迁移与运维：Supabase CLI 管理 SQL 迁移；监控使用 Supabase 内置 Metrics/Logs。

## 其他
- 构建与发布：Xcode 15+，多方案（Debug/Release）；CI 可用 GitHub Actions (xcodebuild + fastlane 导出)。
- 国际化与格式：仅日语/中文界面可选，金额默认日元符号与千分位，无小数。
- 安全：使用系统 Keychain 存储 token；本地数据默认明文（符合需求，无额外加密）。***
