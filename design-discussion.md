## 对话摘要
- 目标：开发以日元为主的记账软件，突出美观丰富的 dashboard 和多样统计。
- 价值：帮助用户养成良好、诚实的记账习惯。
- 用户与平台：个人自用；iOS 作为记账入口，macOS 同步支持录入、编辑与查看；需要跨端数据同步。
- 数据来源：当前仅手动输入。
- 账户与分类：常用账户类型（现金、信用卡、电子钱包等）+ 分类/标签体系；币种主日元，远期可能增加人民币，不做汇率换算。
- Dashboard 重点：本月支出/收入；支出方式/类别/地点明细；支出占比；预算 vs 实际进度；按月支出曲线；月度结余/超支。
- 可视化：折线图、柱状图、扇形图、进度条等常用统计图形。
- 诚实习惯机制：主要依靠丰富、直观的 dashboard 反馈支出/结余。
- 同步/存储：初期尝试 Supabase 数据库（结构化多维数据），若开发受阻回退 iCloud Numbers；需支持离线编辑后自动同步。
- 预算：按月为每个分类设预算，默认稳定可修改；超支时分类数据用红色标示；结余全额结转到下月总预算。
- 分类/标签：分类模板（饮食、出行、运动、购物、娱乐、水电燃气；通信/住房等固定开销内置自动记）；地点模板（超市、咖啡店、便利店、餐厅、网站、车站）；支持自定义。
- 习惯促进：希望用活跃度 heat map（类似 GitHub contribution 方格）展示记账活跃度。
- 多标签真实性：采用“主标签 + 副标签”模式；总支出只计一次，主标签用于归类与占比，副标签参与该标签视图与预算但在视觉上区分。
- 账户与转账：需区分消费 vs 转账。信用卡给交通 IC 充值默认视为账户间转账（信用卡负债增加、IC 余额增加），不计作支出；实际乘车/消费时才记支出。可提供配置切换为“充值即计支出”以贴合现金制偏好。
- 信用卡还款：记作“银行账户 → 信用卡”的转账，用于清偿负债，不重复计作支出；支出早在刷卡/实际消费时已记账。若用户采用“充值即计支出”模式，则还款仍作为转账，不再计支出。
- 说明：充值/还款都是转账；支出在消费发生时记账。若选择“充值即计支出”，充值计支出但还款仍为转账，避免重复。
- categories/tags：需添加唯一约束（同名不重复）与排序字段（自定义展示顺序）。

## Supabase 数据模型草案
- 核心实体（按需开启 accounts；默认不处理实体账户）：categories（含固定开销标记、排序、唯一约束）、tags（主/副类型字段、排序、唯一约束）、locations、entries（单笔记账）、entry_tags（role=primary/secondary）、budgets（按月-标签预算）、budget_rollovers（结余结转）、fixed_expense_templates（固定开销模板）、fixed_expense_runs（生成记录）。
- 同步字段：各表含 id(uuid)、user_id、created_at、updated_at、deleted_at、rev/version、device_id；本地离线缓存 + 变更队列，冲突时弹差异提示。
- 多标签计数：entry_tags 中 role 区分主/副；统计/预算按主标签和副标签各算一次但总支出只计一次；可保留 entry_allocations 作为精细分摊后备。

## 主要表字段（草案）
- categories：id, user_id, name, is_fixed(bool), sort_index, unique(name,user_id), created_at, updated_at.
- tags：id, user_id, name, tag_type(enum primary/secondary), sort_index, unique(name,user_id), created_at, updated_at.
- locations：id, user_id, name, created_at, updated_at.
- entries：id, user_id, amount, currency(JPY), occurred_on(date), note, location_id, main_tag_id（可与 entry_tags role=primary 冗余校验）, created_at, updated_at.
- entry_tags：entry_id, tag_id, role(enum primary/secondary), created_at。
- budgets：id, user_id, month(YYYY-MM), tag_id（主/副均可设预算），amount, locked(bool), created_at, updated_at。
- budget_rollovers：id, user_id, month(YYYY-MM), amount, source_month, created_at。
- fixed_expense_templates：id, user_id, name, amount, tag_id, day_of_month, active(bool), created_at, updated_at。
- fixed_expense_runs：id, template_id, month(YYYY-MM), entry_id（生成的记账记录），status(enum created/posted), created_at。

## 视图与查询示例（设计思路）
- 当月总支出/收入：按 entries.occurred_on 当月汇总，总支出仅计一次；主/副标签细分通过 entry_tags role 区分。
- 分类/标签占比：主标签直接汇总；副标签汇总时标记为 secondary 以淡色显示。
- 预算 vs 实际：按 budgets（标签维度）对比 entries 汇总；预算锁定后仅读；结余通过 budget_rollovers 纳入下月总预算。
- 固定开销：fixed_expense_templates 在每月初生成 entries，标记为固定开销（可在 entries 增加 is_fixed 标记）。
- 活跃度 heat map：按 entries.occurred_on 计数/金额映射至日历格。

## Supabase DDL 草案（SQL 示意）
```sql
-- categories
create table categories (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  is_fixed boolean not null default false,
  sort_index int not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  rev bigint not null default 0,
  device_id text,
  constraint uq_categories_name unique (user_id, name)
);

-- tags
create table tags (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  tag_type text not null check (tag_type in ('primary','secondary')),
  sort_index int not null default 0,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  rev bigint not null default 0,
  device_id text,
  constraint uq_tags_name unique (user_id, name)
);

-- locations
create table locations (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  rev bigint not null default 0,
  device_id text,
  constraint uq_locations_name unique (user_id, name)
);

-- entries
create table entries (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  amount numeric(12,2) not null check (amount >= 0),
  currency text not null default 'JPY',
  occurred_on date not null,
  note text,
  location_id uuid references locations(id),
  main_tag_id uuid references tags(id),
  is_fixed boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  deleted_at timestamptz,
  rev bigint not null default 0,
  device_id text
);
create index idx_entries_user_date on entries(user_id, occurred_on);

-- entry_tags
create table entry_tags (
  entry_id uuid not null references entries(id) on delete cascade,
  tag_id uuid not null references tags(id),
  role text not null check (role in ('primary','secondary')),
  created_at timestamptz not null default now(),
  primary key (entry_id, tag_id, role)
);

-- budgets
create table budgets (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  month date not null, -- 当月第一天
  tag_id uuid not null references tags(id),
  amount numeric(12,2) not null check (amount >= 0),
  locked boolean not null default false,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  rev bigint not null default 0,
  device_id text,
  constraint uq_budgets unique (user_id, month, tag_id)
);

-- budget_rollovers
create table budget_rollovers (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  month date not null,         -- 结余进入的月份（当月第一天）
  source_month date not null,  -- 结余来源月份
  amount numeric(12,2) not null,
  created_at timestamptz not null default now()
);

-- fixed expense templates
create table fixed_expense_templates (
  id uuid primary key default gen_random_uuid(),
  user_id uuid not null,
  name text not null,
  amount numeric(12,2) not null check (amount >= 0),
  tag_id uuid not null references tags(id),
  day_of_month int not null check (day_of_month between 1 and 28),
  active boolean not null default true,
  created_at timestamptz not null default now(),
  updated_at timestamptz not null default now(),
  rev bigint not null default 0,
  device_id text
);

-- fixed expense runs
create table fixed_expense_runs (
  id uuid primary key default gen_random_uuid(),
  template_id uuid not null references fixed_expense_templates(id) on delete cascade,
  month date not null, -- 当月第一天
  entry_id uuid references entries(id),
  status text not null check (status in ('created','posted')),
  created_at timestamptz not null default now()
);
create index idx_fixed_runs_template_month on fixed_expense_runs(template_id, month);
```

## 同步与冲突处理流程（草案）
- 本地缓存：APP 本地存储所有表数据，记录 rev/version 和 device_id。
- 变更队列：离线也可记账，操作入队；恢复联网后按时间顺序同步。
- 冲突检测：上传时若服务器 rev 更新且有字段差异，则拉取差异，展示对比，让用户选择“保留本地覆盖”或“接受远端覆盖”；合并后不支持撤销，但可再编辑。
- 软删除：deleted_at 标记；同步时按 rev 处理，防止删除被覆盖。
- 固定开销生成：每月初触发 fixed_expense_templates → fixed_expense_runs → entries（标记 is_fixed=true）；如失败可重试。
- 预算锁定：当月有 entries 即锁 budgets.locked，防止修改；当月无记账时可编辑预算（默认复制上月）。

## UI 交互草案
- Dashboard：默认当月，可切换历史；模块优先级：1) 本月支出 + 总结余/超支（突出非固定，与固定区分）；2) 本月各分类预算进度；3) 本月各分类占比；4) 每日开销折线趋势；5) 各分类按月趋势；6) 活跃度 heat map；展示固定开销总额。
- 列表视图：固定开销用淡色标记；副标签贡献用淡色/图例说明。
- 预算管理：进入当月若已有记账显示锁定；可查看但不可改；当月为空时默认加载上月预算，可编辑后锁定随记账生效。
- 冲突合并：弹出差异对比（左右两列），用户选“保留本地”或“保留远端”整条覆盖；合并后提示可继续编辑但不支持撤销。
- 固定开销：设置模板（金额、标签、月日），每月初自动生成；在列表中可见生成的固定开销记录，标记为固定。
- 副标签可视化：同色系淡化（降低透明度）展示副标签贡献，无斜线/描边。
- 预算提醒：单分类预算进度>80% 用黄色/橙色预警；>100% 用红色。
- 历史月份导航：支持顶部月份选择器和左右滑动手势切换。
- 记录详情：显示主/副标签图例说明，帮助理解副标签统计方式。
- 货币格式：默认使用日元符号，千分位逗号分隔，会计格式，精度到个位（无小数）。

## 后续可讨论的产品项（提案）
- 快捷输入：近期常用金额/标签的快捷按钮，或“上次同地点/同标签”的一键复用。
- 导出/备份：是否需要 CSV/Excel 导出；Supabase 数据是否需本地加密备份。
- 今日状态提示：是否在首页显示“今日已记账/未记账”以鼓励习惯。
- 通知策略：保持零提醒，或仅在预算预警时提供非打扰提示（与 >80% 进度条颜色一致）。
- 待覆盖的重要主题：
  - 搜索与筛选：按金额区间、标签（主/副）、地点、时间范围的过滤需求。
  - 数据一致性与错误恢复：网络中断/同步失败的提示与重试策略，离线编辑冲突的用户反馈。
  - 空状态与引导：初次使用的教程/示例数据，空白 dashboard 展示。
  - 可访问性与配色：色盲友好（预算预警色是否需额外标识）、字号/对比度。
  - 导出/备份策略：定期备份频率、导出格式与位置。
  - 体验细节：输入效率（数字键盘、默认金额光标）、撤销/恢复（除冲突合并外的日常操作）、手势操作。

## 最新决策补充
- 搜索/筛选：暂不提供搜索筛选功能。
- 同步失败处理：网络导致同步失败时后台隔一段时间自动重试（如隔一天）；前端不提示，用户无感。
- 空白 Dashboard：各数值置零展示，无示例数据。
- 可访问性：暂不考虑色盲友好设计。
- 导出：支持手动导出 CSV；无自动备份。

## 核心用户流（待定/需敲定细节）
- 首次上手：创建账户 → 载入默认分类/标签/预算 → 固定开销模板启用/关闭。
- 快速记一笔：输入金额（数字键盘、光标默认在金额）→ 选主/副标签 → 选地点/备注（可选）→ 保存。
- 编辑/删除：修改金额/标签后预算与统计实时更新；删除采用软删除并同步。
- Dashboard 浏览：按优先级模块查看；切换历史月份（顶部选择器或左右滑动）；固定 vs 非固定区分、预算预警色反馈。
- 预算设置：空月自动复制上月 → 可编辑 → 有记账后锁定。
- 固定开销：模板管理（金额/标签/月日）→ 每月初自动生成 → 列表中淡色显示，可编辑。
- 冲突合并：展示差异 → 选择一侧覆盖 → 合并后可编辑（无撤销）。
- 导出：手动导出 CSV（当月或指定范围）的入口与步骤。

## UI 风格
- 基调：极简，仅提供必要信息/指示。
- 字体：初版纯英文，使用等宽（monospace）编程风格字体，保持科技感与对齐，偏 TUI 风格。
- 已定局部：副标签同色系淡化；预算预警色 80% 黄/橙、100% 红；固定开销淡色；货币格式日元千分位到个位。
- 细化方案：
  - 色板（建议）：背景浅色方案（#F9FAFB）、文本深灰（#1F2933）、分隔线/边框（#E5E7EB）；强调色选单一冷色（如青绿 #0EA5E9 或蓝绿 #22D3EE）；警告色与预警色使用标准黄/橙/红。
  - 卡片/容器：无阴影，1px 细边框（#E5E7EB），小圆角（4px），保持 TUI 扁平感。
  - 按钮：扁平边框，轻微圆角（4px），主色填充 + 白字；次按钮为描边透明；禁用态降低透明度。
  - 图表主题：简洁线型/柱状，线宽 2px；副标签用主色的 30–40% 透明度；固定开销在占比/柱状中用浅灰或主色的 20% 透明度。
  - 布局：留白适中，分区采用细线或间距分隔；避免装饰性元素。
  - 图例：必须标识副标签（淡色）与固定开销（浅色）的含义，确保可读。
  - 预算进度条：<80% 用绿色，80–100% 用黄/橙，>100% 用红色。

## 多标签真实性方案（已选 + 设计细化）
- 已选方案：主标签 + 副标签。每笔仅一个主标签（用于归类、占比），可有多个副标签（用于视图/预算参与）；总支出只计一次。
- 统计规则：总支出/收入只算一次；主标签视图与单标签无差异；副标签视图参与统计，但用视觉弱化（淡色/虚线）标记为副标签贡献。
- 预算规则：金额计入主标签预算；副标签也计入其预算（预算视角允许多重归属）；总预算仍按实际总支出计算，不重复。
- 示例：5000 日元，主=饮食，副=娱乐。总支出+5000；饮食统计+5000；娱乐统计+5000（标注副标签贡献）；饮食与娱乐预算各+5000。
- 后备方案：保留 entry_allocations/子项拆分（需要精细分摊时可启用）。

## 交互与业务规则（新增要点）
- 冲突合并：检测到冲突时先展示差异，再让用户选一侧覆盖；合并后不支持撤销，但可继续编辑。
- 预算：每月默认复制上月预算；当月有记账后预算锁定，仅当月无记账时可修改。
- 固定开销：每月初自动生成并入总支出，列表/图表以淡色区分，无专门视图。
- 时间范围：Dashboard 默认当月，可切换历史月份。

## 待办与确认
- 平台：仅 iOS/macOS，短期不做 Android/Web。
- 同步与存储：检测冲突时提示用户合并（不采用纯 last-write-wins）。
- 多标签展示：副标签用淡色/弱化视觉；图例需标注副标签贡献。
- 习惯辅助：目前仅 heat map，无额外提醒/小结。
- 时间范围：Dashboard 默认展示当月，可切换查看历史月份。

## 当前待澄清点（问题释义与进展）
- Supabase 表结构细节：是否给 `categories/tags` 增加唯一约束（同名不重复）和排序字段（自定义展示顺序）；现已确定 tags 有主/副类型字段、entry_tags 有 role 标记副标签。
- 冲突合并交互：发现冲突时先展示差异，再让用户选择“用 A 覆盖 B”或“用 B 覆盖 A”；合并后不允许撤销，但用户可继续编辑记录。
- 固定开销视图：固定开销每月初自动入账计入总支出；不单独视图，仅在列表/图表以淡色区分固定 vs 非固定。
- 预算快捷操作：默认复制上月预算；预算仅在当月无记账时允许修改（锁定后随记账不可改，除非清空当月记录）。
- 账户体系细节：本软件不涉及实体账户，仅记录用户输入；（转账、余额、账单日等账户属性可简化或不处理）。
- 安全与登录：已选择 Supabase Auth（邮箱/Apple 登录），确认无额外本地加密/隐私保护需求。
- 附加字段（待确认）：entries 是否需要附件/照片、地理位置，budgets 是否需要备注；若需要，可在 schema 中扩展。
