# SFA CRM 项目操作技能手册

**版本**: 1.0.0 | **更新日期**: 2026-05-02

本文档汇总了 SFA CRM 项目的核心操作流程、开发规范和最佳实践，基于项目内部文档（宪法、Spec、Memory）提炼而成。

---

## 目录

- [一、项目核心理念](#一项目核心理念)
- [二、快速启动流程](#二快速启动流程)
- [三、用户操作流程](#三用户操作流程)
- [四、开发工作流程](#四开发工作流程)
- [五、AI Copilot 配置与使用](#五ai-copilot-配置与使用)
- [六、演示数据管理](#六演示数据管理)
- [七、关键业务规则](#七关键业务规则)
- [八、技术架构要点](#八技术架构要点)
- [九、常见问题排查](#九常见问题排查)

---

## 一、项目核心理念

### 1.1 六大宪法原则

所有开发和操作必须遵循以下核心原则（详见 `.specify/memory/constitution.md`）：

| 原则 | 核心要求 | 原因 |
|------|---------|------|
| **Ontology 优先** | 业务实体显式建模为对象、关系和动作；状态变更必须由事件触发 | 客户归因、客保机制复杂，平铺表结构会导致规则隐含在代码里 |
| **API 优先** | GUI 和 AI Agent 共用同一套 API；禁止绕过 API 直接访问数据层 | Copilot 与人工操作必须执行同样的业务规则校验 |
| **配置驱动** | 大区规则、公共池策略等必须存储为配置，不得硬编码 | 5个大区规则各不相同且会持续变化 |
| **数据完整性** | 客户唯一性强制保证；竞争性操作实施速率限制 | 重复客户和外挂抢池是活跃问题 |
| **最小化录入负担** | 可推导的内容必须自动化处理（日报从拜访记录生成） | 销售对 CRM 的抵触与录入负担直接相关 |
| **显式优于隐式** | 每条业务规则必须在 spec 中明确声明 | AI 辅助实现要求规格无歧义 |

### 1.2 技术约束

- **数据层**: SQLite WAL 模式 + SQLModel ORM
- **后端**: FastAPI (Python 3.11+)
- **前端**: Next.js 14 (Node.js 20+)
- **部署**: Docker Compose + 腾讯云轻量服务器
- **AI Agent**: Vercel AI SDK + 多 LLM Provider（Admin 可切换）
- **速率限制**: SlowAPI + MemoryStorage，针对竞争性操作

---

## 二、快速启动流程

### 2.1 本地开发环境启动

#### 方式一：手动启动（推荐调试时使用）

**步骤 1: 启动后端**

```bash
cd src/backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e ".[dev]"

# 初始化数据库（建表 + 种入测试数据）
python -c "from app.core.init_db import init_db; init_db()"

# 启动服务
uvicorn app.main:app --reload --port 8000
```

**步骤 2: 启动前端**

```bash
cd src/frontend
npm install
npm run dev
```

**验证启动成功：**

| 地址 | 预期响应 |
|------|---------|
| http://localhost:8000 | `{"status": "ok", "service": "SFA CRM API"}` |
| http://localhost:8000/docs | Swagger API 文档页面 |
| http://localhost:3000 | 前端登录页 |

#### 方式二：Docker 一键启动（推荐生产部署）

```bash
cd src
docker-compose up --build        # 前台运行（查看日志）
docker-compose up -d             # 后台运行
```

停止服务：

```bash
docker-compose down
```

### 2.2 配置 AI 助手（首次使用必做）

**前置条件**: 准备一个 LLM API Key（Anthropic / OpenAI / DeepSeek 均可）

**操作步骤**:

1. 用 `admin` 账号登录系统（密码: `12345`）
2. 侧边栏 → **系统配置** → 「AI 模型配置」区域
3. 选择 Provider：
   - **Anthropic**: Model 填 `claude-sonnet-4-20250514`
   - **OpenAI**: Model 填 `gpt-4o`
   - **DeepSeek**: Model 填 `deepseek-chat`
4. 填入 API Key
5. 点击「保存 LLM 配置」
6. 刷新页面，右下角蓝色 AI 聊天气泡即可使用

**注意事项**:
- API Key 存储在 `src/backend/.env` 文件中（已在 `.gitignore` 中，不会提交到 Git）
- 配置对所有用户生效，无需每个用户单独配置
- 更换 Provider 后建议重启后端服务

### 2.3 重置演示数据

当需要恢复初始演示状态时：

```bash
# Windows
reset-demo.bat

# macOS/Linux
./reset-demo.sh
```

**重置内容**:
- 清空所有业务数据（线索、客户、跟进记录等）
- 重新种入标准演示数据（sales01/02/03 和 manager01 的数据分布）
- 保留系统配置（包括 LLM API Key）

---

## 三、用户操作流程

### 3.1 测试账号速查

| 账号 | 密码 | 角色 | 数据权限 | 典型使用场景 |
|------|------|------|---------|-------------|
| `admin` | `12345` | 系统管理员 | 全部数据 | 系统配置、组织管理、用户管理、审计日志 |
| `manager01` | `12345` | 战队队长 | 当前节点及下属 | 团队线索、团队客户、团队日报、队员分析 |
| `sales01` | `12345` | 销售（王小明） | 仅自己 | 我的线索、公共线索库、我的客户、我的日报 |
| `sales02` | `12345` | 销售（李思远） | 仅自己 | 同上，活跃度中等 |
| `sales03` | `12345` | 销售（张磊） | 仅自己 | 同上，活跃度低 |

### 3.2 销售日常操作流程

#### 流程 1: 创建新线索

1. 登录系统（使用 sales 账号）
2. 侧边栏 → **我的线索** → 点击右上角「新建」按钮
3. 填写表单：
   - **公司名称**（必填）：系统会自动去重检测
   - **所属大区**（必填）：华东/华南/华北/华西/华中
   - **线索来源**（必填）：转介绍/自然流量/KOC投放/外呼
   - **联系人姓名**（可选）
   - **联系人微信/手机**（可选，用于后续去重预警）
4. 点击「保存」

**系统自动行为**:
- 检查公司名是否重复（有组织机构代码时精确匹配；无时模糊匹配 + 联系人预警）
- 如果重复，推送主管确认
- 新线索自动进入当前销售的私有池

#### 流程 2: 从公共线索库抢占线索

1. 侧边栏 → **公共线索库**
2. 浏览或搜索可用线索（显示各线索的大区、来源、最后跟进时间）
3. 找到目标线索 → 点击「抢占」按钮
4. 确认抢占操作

**系统自动行为**:
- 检查当前销售私有池是否已满（默认上限 100 条 active 线索）
- 检查该线索是否已被其他人抢占（并发控制）
- 将线索 owner 更新为当前销售，pool 从 public 改为 private
- 记录抢占事件到 audit_log

**注意**: 不同大区可能有不同的抢占规则（配置驱动），如每日抢占次数限制

#### 流程 3: 添加跟进记录

1. 进入线索详情页（从「我的线索」列表点击进入）
2. 滚动到「跟进记录」区域
3. 点击「添加跟进」
4. 选择跟进类型：电话 / 微信 / 拜访 / 其他
5. 填写跟进内容（支持富文本）
6. 可选：标记为关键事件（如"送书"、"邀约成功"）
7. 点击「保存」

**系统自动行为**:
- 更新线索的 `last_followup_at` 字段
- 如果标记为关键事件，同步创建 KeyEvent 记录
- 当天如有日报草稿，自动追加到日报中

#### 流程 4: 转化线索为客户

1. 进入线索详情页
2. 确认该线索已购买小课（2万元课程）
3. 点击顶部「转化」按钮
4. 系统提示确认转化操作
5. 确认后，线索状态变为 `converted`，同时创建 Customer 记录

**系统自动行为**:
- Lead.stage 更新为 `converted`，填入 `converted_at`
- 创建 Customer 记录，继承 Lead 的公司名、大区、owner 等信息
- Lead 归档（不再出现在销售列表中，但保留历史记录）
- 联系人迁移到 Customer 下
- 启动 14 天大课转化窗口计时

#### 流程 5: 查看/编辑日报

1. 侧边栏 → **我的日报**
2. 系统自动生成当天日报草稿（基于当天的跟进记录）
3. 可手动补充内容（如"今日工作总结"、"明日计划"）
4. 点击「提交」完成日报

**系统自动行为**:
- 每天 00:00 自动为每个销售创建空白日报模板
- 每次添加跟进记录时，自动追加到当日日报草稿
- 提交后的日报不可修改（审计要求）

### 3.3 管理者操作流程

#### 流程 1: 分配线索给销售

1. 用 manager 或 admin 账号登录
2. 侧边栏 → **团队线索**（manager）或 **全部线索**（admin）
3. 找到未分配的线索（owner 为空）
4. 点击「分配」按钮
5. 选择目标销售
6. 可选：填写分配理由
7. 点击「确认分配」

**系统自动行为**:
- 更新线索的 owner_id 和 pool 字段
- 记录分配事件到 audit_log
- 向被分配的销售发送通知（站内消息）

#### 流程 2: 查看团队偷懒情况（AI 辅助）

1. 用 manager 账号登录
2. 点击右下角 AI 聊天气泡
3. 输入："谁在偷懒？" 或 "分析团队成员的跟进活跃度"
4. AI 会自动：
   - 调用 `search_leads` 工具获取团队线索列表
   - 按 owner 分组统计跟进频率
   - 识别长期未跟进的线索及其负责人
   - 输出分析报告，标注异常低活跃度的销售

**示例输出**:
```
团队跟进活跃度分析（过去7天）：

📊 王小明（sales01）：
  - 负责线索：45条
  - 新增跟进：23次
  - 平均每条线索跟进间隔：1.8天 ✅ 活跃

📊 李思远（sales02）：
  - 负责线索：38条
  - 新增跟进：12次
  - 平均每条线索跟进间隔：3.2天 ⚠️ 一般

📊 张磊（sales03）：
  - 负责线索：42条
  - 新增跟进：3次
  - 平均每条线索跟进间隔：9.5天 ❌ 低活跃
  - 风险提示：15条线索超过10天未跟进，即将自动释放
```

#### 流程 3: 调整系统配置

1. 用 admin 账号登录
2. 侧边栏 → **系统配置**
3. 可配置项包括：
   - **私有池上限**: 默认 100 条
   - **自动释放规则**: 未跟进天数（默认10天）、未成单天数（默认30天）
   - **大区抢占规则**: 各区域的特殊规则
   - **LLM 配置**: AI 助手的 Provider 和 API Key
4. 修改后点击「保存」

**注意**: 配置修改立即生效，影响所有用户的后续操作

---

## 四、开发工作流程

### 4.1 Spec Coding 工作流

本项目采用 **Spec First** 开发模式，严格遵循以下流程：

```
业务讨论 → Ontology 设计 → Spec 编写 → Plan 制定 → Tasks 拆解 → 编码实现 → 测试验证
```

**核心规则**:

1. **先写 spec，再写代码**: 每个功能在完成对应 spec 之前不得开始实现
2. **唯一真相源**: 业务逻辑只存在于 API 层；UI 和 Agent 是薄客户端
3. **状态从事件推导**: 派生状态始终计算得出，不作为可变字段存储
4. **逐任务提交**: 每完成 `tasks.md` 一个任务必须立即 commit，不允许跨任务积压

### 4.2 目录结构与职责划分

```
src/
├── backend/
│   └── app/
│       ├── api/           # API 路由层（参数校验 + 权限检查）
│       ├── models/        # SQLModel ORM 模型（数据表定义）
│       ├── services/      # 业务逻辑层（Ontology Actions 实现）
│       ├── core/          # 基础设施（数据库、认证、配置、依赖注入）
│       └── tools/         # AI Tool Use 定义（供 LLM 调用）
├── frontend/
│   └── src/
│       ├── app/           # Next.js App Router 页面
│       ├── components/    # UI 组件（nav/chat/leads/...）
│       └── lib/           # API 客户端、AI SDK 配置
└── docker-compose.yml
```

**职责边界**:

| 层级 | 职责 | 禁止事项 |
|------|------|---------|
| `api/` | 接收请求、参数校验、权限检查、调用 service | 不写业务逻辑、不直接操作数据库 |
| `services/` | 实现 Ontology Actions、执行业务规则、数据库事务 | 不做 HTTP 响应格式化、不处理认证 |
| `models/` | 定义数据表结构、字段类型、索引 | 不包含业务逻辑、不执行查询 |
| `tools/` | 封装可供 LLM 调用的工具函数 | 不直接暴露给用户界面 |

### 4.3 新增功能的标准流程

#### 步骤 1: 更新 Spec

在 `specs/master/spec.md` 中添加：
- Ontology 对象定义（如有新对象）
- Actions 定义（操作接口）
- 用户故事（User Story）
- 验收条件

#### 步骤 2: 更新 Data Model

在 `specs/master/data-model.md` 中添加：
- SQL DDL 语句（CREATE TABLE）
- 索引设计
- 种子数据（如需要）

#### 步骤 3: 更新 Tasks

在 `specs/master/tasks.md` 中拆解任务：
- 按 Phase 组织
- 每个任务标注 `[P]`（可并行）或 `[Story]`（对应用户故事）
- 任务粒度：单个文件修改或小功能实现

#### 步骤 4: 编码实现

按 tasks.md 顺序逐个实现，**每完成一个任务立即 commit**：

```bash
git add <modified files>
git commit -m "feat: T0XX - <task description>

🤖 Generated with [Lingma][https://lingma.aliyun.com]"
```

**Commit 规范**:
- 标题格式: `type: TXXX - 简短描述`
- type 可选值: `feat`（新功能）、`fix`（修复）、`refactor`（重构）、`docs`（文档）
- 必须包含 Lingma 署名

#### 步骤 5: 测试验证

```bash
cd src/backend
pytest tests/integration/ -v     # 运行集成测试
```

**必测关键路径**:
- 客户去重
- 池分配规则
- 速率限制
- 拆单识别

### 4.4 权限双检模式

所有涉及数据读写的 API 必须实施**双重权限检查**：

```python
# 示例：获取可见线索列表
@router.get("/leads")
def list_leads(
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    # 检查 1: 功能权限（是否有"查看线索"权限）
    require_permission(current_user, "lead:view")

    # 检查 2: 数据范围（能看到哪些用户的线索）
    visible_user_ids = get_visible_user_ids(
        current_user,
        scope=current_user.data_scope  # self_only / current_node / ...
    )

    # 查询时过滤
    leads = db.query(Lead).filter(
        Lead.owner_id.in_(visible_user_ids)
    ).all()

    return leads
```

**DataScope 五档**:

| Scope | 含义 | 适用角色 |
|-------|------|---------|
| `self_only` | 仅自己 | 普通销售 |
| `current_node` | 当前组织节点 | 战队队长（只看本战队） |
| `current_and_below` | 当前节点及所有子节点 | 大区总（看本大区所有战队） |
| `selected_nodes` | 指定的多个节点 | 特殊授权场景 |
| `all` | 全部数据 | 系统管理员 |

### 4.5 状态变更只能通过 Action

**错误做法**（直接修改字段）:

```python
# ❌ 禁止：直接修改线索 owner
lead.owner_id = new_user_id
lead.pool = "private"
db.commit()
```

**正确做法**（通过 Action 函数）:

```python
# ✅ 正确：通过 assign_lead Action
from app.services.lead_actions import assign_lead

assign_lead(
    db=db,
    lead_id=lead_id,
    target_user_id=new_user_id,
    operator=current_user,
    reason="主管分配"
)
```

**Action 函数的职责**:
1. 校验前置条件（如目标用户是否有空余私有池名额）
2. 执行状态变更
3. 记录事件到 audit_log
4. 触发副作用（如发送通知）
5. 返回操作结果

---

## 五、AI Copilot 配置与使用

### 5.1 Copilot 架构

```
用户输入自然语言
    ↓
Next.js 前端（Vercel AI SDK）
    ↓
LLM（Claude/GPT/DeepSeek）解析意图
    ↓
LLM 决定调用哪个 Tool（tool use）
    ↓
FastAPI 后端执行 Tool 对应的 Action
    ↓
返回结果给 LLM
    ↓
LLM 生成自然语言回复
    ↓
前端展示给用户
```

**关键设计**:
- **LLM 不负责业务逻辑**：只做意图识别和工具调用决策
- **Tool 映射 Ontology Actions**：`search_leads` → `GET /leads`、`assign_lead` → `POST /leads/{id}/assign`
- **DataScope 过滤**：Copilot 调用的工具同样受用户数据权限限制

### 5.2 预填表单机制

当用户通过 Copilot 导航到某个页面时，支持自动预填表单：

**流程**:

1. 用户对 Copilot 说："帮我创建一个华为的线索，联系人张三，微信 zhangsan123"
2. LLM 调用 `navigate` 工具，传递参数：
   ```json
   {
     "path": "/leads/new",
     "prefill": {
       "company_name": "华为",
       "contact_name": "张三",
       "wechat_id": "zhangsan123"
     }
   }
   ```
3. 前端将预填数据存入 `sessionStorage('copilot_prefill')`
4. 跳转到 `/leads/new` 页面
5. 页面加载时读取 sessionStorage，自动填充表单
6. 用户确认无误后点击「保存」

**优势**:
- 减少销售手动录入工作量
- 避免 LLM 编造 URL 或 ID
- 保持"人确认、系统执行"的安全模式

### 5.3 常用 Copilot 指令示例

#### 查询类

```
"我有多少条线索快要超时了？"
→ LLM 调用 search_leads，过滤 last_followup_at > 8天的线索

"显示华东大区最近一周新增的客户"
→ LLM 调用 list_customers，按 region 和 created_at 过滤

"谁是我的下属？"
→ LLM 调用 get_org_tree，展示当前用户的下属节点
```

#### 操作类

```
"把线索 LD-123 分配给王小明"
→ LLM 调用 assign_lead 工具，传入 lead_id 和 target_user_id

"帮我抢占公共池里那条华为的线索"
→ LLM 调用 search_leads（public pool），然后 claim_lead

"标记线索 LD-456 为流失，原因是预算不足"
→ LLM 调用 mark_lost_lead，传入 reason
```

#### 分析类

```
"谁在偷懒？"
→ LLM 按 owner 分组统计跟进频率，识别低活跃度销售

"分析我团队的转化漏斗"
→ LLM 统计各阶段线索数量，计算转化率

"哪些线索应该优先跟进？"
→ LLM 根据最后跟进时间、来源质量、联系人数量排序
```

### 5.4 Copilot 调试技巧

**问题 1: LLM 不调用工具，直接编造 URL**

**原因**: System prompt 不够明确，或 LLM 温度过高

**解决**:
- 检查 `src/backend/app/tools/system_prompt.py`，确保工作流程清晰
- 降低 LLM 温度（在系统配置中调整）
- 在 prompt 中强调："必须先调用工具获取真实数据，禁止编造 URL 或 ID"

**问题 2: Copilot 返回"无权访问"**

**原因**: 用户数据权限不足

**解决**:
- 确认当前用户的 `data_scope` 配置
- 如需扩大权限，用 admin 账号在「用户管理」中调整
- Copilot 遵循与 GUI 相同的权限规则，无法绕过

**问题 3: 工具调用失败，报错"参数缺失"**

**原因**: LLM 提取的参数不完整

**解决**:
- 优化用户指令，提供更明确的信息
- 检查 tool schema 定义，确保必填字段标注清楚
- 在 system prompt 中添加参数提取示例

---

## 六、演示数据管理

### 6.1 演示账号数据分布

项目预置了差异化活跃度的销售数据，用于演示 AI 分析能力：

| 账号 | 姓名 | 私有池线索数 | 近7天跟进次数 | 特点 |
|------|------|------------|--------------|------|
| `sales01` | 王小明 | 45 | 23 | 高活跃，跟进及时 |
| `sales02` | 李思远 | 38 | 12 | 中等活跃，偶有延迟 |
| `sales03` | 张磊 | 42 | 3 | 低活跃，多条线索即将释放 |
| `manager01` | 陈队长 | - | - | 可查看上述3人的数据 |

### 6.2 演示案例集

完整的演示案例见 `src/demo/copilot-cases.md`，共 8 个独立案例：

**第一幕：基础能力**
- 案例 1: 自然语言查询（"我有多少条线索？"）
- 案例 2: 跨对象关联（"显示华为的所有联系人"）

**第二幕：AI + 人协同**
- 案例 3: 一句话多步操作 + 表单预填（核心案例）

**第三幕：AI 洞察与决策**
- 案例 4: 策略建议（"哪些线索应该优先跟进？"）
- 案例 5: 转化决策（"LD-789 是否值得继续投入？"）
- 案例 6: 团队偷懒检测（"谁在偷懒？"）

**第四幕：权限体系**
- 案例 7: 数据权限对比（sales vs manager 看到的不同数据）
- 案例 8: 队长评估（"陈队长的团队表现如何？"）

**演示路线**（约 15 分钟）:
1. 用 `sales01` 登录 → 演示案例 1-3（5分钟）
2. 切换到 `manager01` → 演示案例 6-8（5分钟）
3. 用 `admin` 登录 → 展示系统配置和审计日志（5分钟）

### 6.3 重置演示数据

当演示数据被污染或需要重新演示时：

```bash
# Windows
reset-demo.bat

# macOS/Linux
chmod +x reset-demo.sh
./reset-demo.sh
```

**重置脚本执行内容**:
1. 删除现有 SQLite 数据库文件（`src/backend/sfa_crm.db`）
2. 重新运行 `init_db()`，种入标准演示数据
3. 保留 `.env` 文件（LLM API Key 不丢失）
4. 输出重置完成提示

**注意**: 重置后所有自定义数据（新建的线索、跟进记录等）将丢失

---

## 七、关键业务规则

### 7.1 线索去重规则

**场景**: 销售录入新线索时，系统需检测是否已存在相同企业

**规则**:

1. **有统一社会信用代码时**:
   - 精确匹配 `unified_code` 字段
   - 如匹配到，阻止创建，提示"该企业已存在"

2. **无统一社会信用代码时**:
   - 使用 `rapidfuzz` 对公司名进行模糊匹配（阈值 85 分）
   - 同时检查联系人微信/手机号是否与已有联系人重复
   - 如匹配到，推送主管确认（不直接阻止，允许特殊情况）

**代码位置**: `src/backend/app/services/lead_deduplication.py`

### 7.2 公共池抢占规则

**场景**: 多个销售同时点击"抢占"同一条公共池线索

**规则**:

1. **速率限制**:
   - 单个销售每分钟最多抢占 5 条线索
   - 使用 SlowAPI 在 API 层实施，客户端不可绕过

2. **并发控制**:
   - 数据库层面使用乐观锁（检查 `pool == 'public' AND owner_id IS NULL`）
   - 只有第一个请求能成功，后续请求返回"该线索已被抢占"

3. **大区差异化配置**:
   - 各区域可在 `SystemConfig` 表中配置特殊规则
   - 例如：华东区每日最多抢占 20 条，华南区无限制

**代码位置**: `src/backend/app/services/lead_actions.py::claim_lead()`

### 7.3 自动释放规则

**场景**: 销售长期不跟进线索，系统自动释放回公共池

**触发条件**（满足任一即释放）:

1. **10天未跟进**: `last_followup_at < now - 10 days`
2. **30天未成单**: `created_at < now - 30 days AND stage == 'active'`

**执行机制**:

- 使用 APScheduler 定时任务，每小时扫描一次
- 符合条件的线索：
  - `owner_id` 设为 `NULL`
  - `pool` 设为 `'public'`
  - 记录释放事件到 `audit_log`
  - 向原 owner 发送站内通知

**配置项**:

| 配置键 | 默认值 | 说明 |
|--------|--------|------|
| `auto_release.days_no_followup` | 10 | 未跟进天数阈值 |
| `auto_release.days_no_deal` | 30 | 未成单天数阈值 |
| `auto_release.enabled` | true | 是否启用自动释放 |

**代码位置**: `src/backend/app/core/scheduler.py`

### 7.4 14天转化窗口规则

**场景**: 线索转化为客户后，跟踪大课（20万）转化情况

**规则**:

1. **窗口起点**: Customer.created_at（即线索转化时间）
2. **窗口终点**: Customer.created_at + 14天
3. **窗口内状态**:
   - 查询课时订单系统，检查是否有大课订单
   - 如无订单，标记为"转化窗口期内，待观察"
   - 如有订单，标记为"大课转化成功"
4. **窗口外状态**:
   - 如无订单，标记为"大课转化失败"
   - 可选择重新激活（延长窗口期，需主管审批）

**实现方式**:

- **不存库**：窗口状态实时计算（`now - customer.created_at < 14天`）
- **API 端点**: `GET /customers/{id}/conversion_window` 返回窗口状态

**代码位置**: `src/backend/app/services/customer_service.py`

### 7.5 拆单识别规则

**场景**: 销售为规避提成阶梯，将一个大单拆成多个小单

**检测逻辑**:

1. **时间窗口**: 同一客户在 7 天内产生多笔订单
2. **金额特征**: 多笔订单金额之和接近下一个提成阶梯阈值
3. **支付方式**: 多笔订单使用相同支付方式或付款账户

**告警机制**:

- 检测到疑似拆单时，向主管发送告警
- 告警包含：客户名称、订单列表、疑似原因
- 主管可在 Admin 后台查看和处理告警

**代码位置**: `src/backend/app/services/fraud_detection.py`

---

## 八、技术架构要点

### 8.1 数据库设计

#### SQLite WAL 模式配置

为解决 FastAPI 并发写入问题，启用 WAL（Write-Ahead Logging）模式：

```python
# src/backend/app/core/database.py
engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False}
)

# 执行 PRAGMA 命令
with engine.connect() as conn:
    conn.execute(text("PRAGMA journal_mode=WAL"))
    conn.execute(text("PRAGMA synchronous=NORMAL"))
    conn.execute(text("PRAGMA cache_size=-64000"))  # 64MB
    conn.execute(text("PRAGMA temp_store=MEMORY"))
```

**效果**:
- 读写不互斥，提升并发性能
- 适合演示环境的中小规模数据量

#### 关键索引设计

```sql
-- 线索表索引
CREATE INDEX idx_lead_owner ON lead(owner_id);
CREATE INDEX idx_lead_pool ON lead(pool);
CREATE INDEX idx_lead_stage ON lead(stage);
CREATE INDEX idx_lead_last_followup ON lead(last_followup_at);

-- 跟进记录表索引
CREATE INDEX idx_followup_lead ON followup(lead_id);
CREATE INDEX idx_followup_created ON followup(created_at);

-- 审计日志表索引
CREATE INDEX idx_audit_action ON audit_log(action);
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

**设计原则**:
- 高频查询字段必建索引
- 复合查询考虑联合索引
- 避免过度索引（影响写入性能）

### 8.2 API 设计规范

#### 统一响应格式

**成功响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": { ... }
}
```

**错误响应**:
```json
{
  "code": 400,
  "message": "参数错误：公司名称不能为空",
  "data": null
}
```

**分页响应**:
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "items": [...],
    "total": 100,
    "page": 1,
    "page_size": 20
  }
}
```

#### RESTful 路由规范

| 操作 | 方法 | 路径 | 说明 |
|------|------|------|------|
| 列表查询 | GET | `/leads` | 支持 query params 过滤 |
| 详情查询 | GET | `/leads/{id}` | 返回单个对象 |
| 创建 | POST | `/leads` | Body 传入字段 |
| 更新 | PUT | `/leads/{id}` | 全量更新 |
| 部分更新 | PATCH | `/leads/{id}` | 部分字段更新 |
| 删除 | DELETE | `/leads/{id}` | 软删除（标记 stage=lost） |
| 动作 | POST | `/leads/{id}/assign` | 业务动作（分配/抢占/转化等） |

**注意**: 业务动作不使用 PUT/PATCH，统一用 POST + 动作名

### 8.3 认证与授权

#### JWT Token 机制

**登录流程**:

1. 用户提交用户名密码到 `POST /auth/login`
2. 后端验证通过后，生成 JWT Token：
   ```python
   payload = {
       "sub": user.id,
       "username": user.username,
       "role": user.role,
       "exp": datetime.utcnow() + timedelta(hours=24)
   }
   token = jwt.encode(payload, SECRET_KEY, algorithm="HS256")
   ```
3. 前端存储 Token 到 `localStorage`
4. 后续请求在 Header 中携带：`Authorization: Bearer <token>`

**Token 验证**:

```python
# src/backend/app/core/auth.py
def get_current_user(token: str = Depends(oauth2_scheme)) -> User:
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        user_id = payload["sub"]
        user = db.query(User).filter(User.id == user_id).first()
        if not user:
            raise HTTPException(status_code=401, detail="用户不存在")
        return user
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token 已过期")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="无效 Token")
```

#### 权限装饰器

```python
def require_permission(user: User, permission_code: str):
    """检查用户是否拥有指定权限"""
    permissions = db.query(Permission).join(RolePermission).join(Role).join(UserRole).filter(
        UserRole.user_id == user.id,
        Permission.code == permission_code
    ).all()

    if not permissions:
        raise HTTPException(status_code=403, detail=f"缺少权限：{permission_code}")
```

**使用示例**:

```python
@router.post("/leads/{lead_id}/assign")
def assign_lead_api(
    lead_id: str,
    request: AssignLeadRequest,
    current_user: User = Depends(get_current_user),
    db: Session = Depends(get_db)
):
    # 权限检查
    require_permission(current_user, "lead:assign")

    # 业务逻辑
    assign_lead(db=db, lead_id=lead_id, target_user_id=request.target_user_id, operator=current_user)

    return {"message": "分配成功"}
```

### 8.4 速率限制实现

**技术选型**: SlowAPI + MemoryStorage

**配置示例**:

```python
# src/backend/app/core/rate_limiter.py
from slowapi import Limiter
from slowapi.util import get_remote_address

limiter = Limiter(key_func=get_remote_address)

# 在 API 路由中使用
@router.post("/leads/{lead_id}/claim")
@limiter.limit("5/minute")  # 每分钟最多 5 次
def claim_lead_api(request: Request, lead_id: str, ...):
    ...
```

**限制策略**:

| 端点 | 限制 | 原因 |
|------|------|------|
| `POST /leads/{id}/claim` | 5次/分钟 | 防止外挂抢池 |
| `POST /auth/login` | 10次/分钟 | 防止暴力破解 |
| `POST /chat/completions` | 20次/分钟 | 防止滥用 AI 助手 |

### 8.5 审计追踪

**审计日志表结构**:

```sql
CREATE TABLE audit_log (
    id UUID PRIMARY KEY,
    action VARCHAR(50) NOT NULL,        -- 动作类型（assign_lead/claim_lead/...）
    resource_type VARCHAR(50) NOT NULL, -- 资源类型（lead/customer/...）
    resource_id UUID NOT NULL,          -- 资源 ID
    user_id UUID NOT NULL,              -- 操作人
    old_values JSON,                    -- 变更前数据
    new_values JSON,                    -- 变更后数据
    ip_address VARCHAR(45),             -- 客户端 IP
    user_agent TEXT,                    -- 客户端 UA
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**记录时机**:

- 所有 Ontology Actions 执行时
- 用户登录/登出
- 系统配置变更
- 权限分配/撤销

**查询审计日志**:

```python
# Admin 后台可查看审计日志
@router.get("/audit-logs")
def list_audit_logs(
    action: str = None,
    resource_type: str = None,
    user_id: str = None,
    start_date: datetime = None,
    end_date: datetime = None,
    db: Session = Depends(get_db)
):
    query = db.query(AuditLog)

    if action:
        query = query.filter(AuditLog.action == action)
    if resource_type:
        query = query.filter(AuditLog.resource_type == resource_type)
    if user_id:
        query = query.filter(AuditLog.user_id == user_id)
    if start_date:
        query = query.filter(AuditLog.created_at >= start_date)
    if end_date:
        query = query.filter(AuditLog.created_at <= end_date)

    return query.order_by(AuditLog.created_at.desc()).all()
```

---

## 九、常见问题排查

### 9.1 启动问题

#### 问题 1: 后端启动失败，报错"ModuleNotFoundError"

**原因**: Python 虚拟环境未激活或依赖未安装

**解决**:

```bash
cd src/backend
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -e ".[dev]"
```

#### 问题 2: 前端启动失败，报错"Cannot find module"

**原因**: node_modules 未安装

**解决**:

```bash
cd src/frontend
npm install
```

#### 问题 3: 数据库初始化失败，报错"Table already exists"

**原因**: 数据库文件已存在，重复执行 `init_db()`

**解决**:

- 方案 1: 删除数据库文件后重新初始化
  ```bash
  rm src/backend/sfa_crm.db
  python -c "from app.core.init_db import init_db; init_db()"
  ```

- 方案 2: 使用幂等版 `init_db()`（2026-04-28 后版本已支持）
  ```python
  # init_db() 内部已增加表存在性检查，可直接多次调用
  ```

### 9.2 运行时问题

#### 问题 4: AI 助手无法使用，报错"API Key 未配置"

**原因**: 未在系统配置中设置 LLM API Key

**解决**:

1. 用 `admin` 账号登录
2. 系统配置 → AI 模型配置 → 填入 API Key
3. 或使用环境变量（推荐）：
   ```bash
   # 编辑 src/backend/.env 文件
   ANTHROPIC_API_KEY=sk-ant-xxxxx
   ```
4. 重启后端服务

#### 问题 5: 线索去重不准确，重复录入

**原因**: 公司名相似度阈值过低或未填写统一社会信用代码

**解决**:

- **短期**: 录入时尽量填写完整信息（公司全称、统一社会信用代码）
- **长期**: 调整去重阈值
  ```python
  # src/backend/app/services/lead_deduplication.py
  SIMILARITY_THRESHOLD = 90  # 默认 85，可调高到 90
  ```

#### 问题 6: 公共池抢占失败，提示"已达速率限制"

**原因**: 短时间内频繁抢占触发速率限制

**解决**:

- 等待 1 分钟后重试
- 如需调整限制，修改配置：
  ```python
  # src/backend/app/core/rate_limiter.py
  @limiter.limit("10/minute")  # 从 5/minute 调整为 10/minute
  ```

#### 问题 7: 定时任务未执行，线索未自动释放

**原因**: APScheduler 未启动或后端服务未持续运行

**解决**:

1. 检查后端日志，确认调度器已启动：
   ```
   INFO:apscheduler.scheduler:Scheduler started
   ```

2. 确认定时任务已注册：
   ```python
   # 在后端 Python REPL 中执行
   from app.core.scheduler import scheduler
   print(scheduler.get_jobs())
   ```

3. 如未启动，检查 `src/backend/app/main.py` 中是否正确初始化调度器

### 9.3 数据问题

#### 问题 8: 销售看不到应看到的线索

**原因**: 数据权限配置不正确

**解决**:

1. 用 `admin` 账号登录 → 用户管理
2. 找到该销售，检查 `data_scope` 字段
3. 如需调整：
   - 普通销售应为 `self_only`
   - 战队队长应为 `current_node`
   - 大区总应为 `current_and_below`
4. 修改后刷新页面

#### 问题 9: 日报未自动生成

**原因**: 当天无跟进记录或定时任务未执行

**解决**:

1. 确认当天已添加跟进记录
2. 检查定时任务日志：
   ```bash
   # 查看后端日志
   tail -f src/backend/logs/app.log | grep "daily_report"
   ```
3. 手动触发日报生成（临时方案）：
   ```python
   from app.services.daily_report_service import generate_daily_report
   generate_daily_report(db=db, user_id=user_id, date=today)
   ```

#### 问题 10: 转化窗口状态显示错误

**原因**: 课时订单系统未集成或数据同步延迟

**解决**:

- **演示环境**: 手动在数据库中插入测试订单数据
  ```sql
  INSERT INTO course_order (customer_id, course_type, amount, paid_at)
  VALUES ('<customer_id>', 'big_course', 200000, '2026-05-01 10:00:00');
  ```

- **生产环境**: 检查 Webhook 接收端点是否正常
  ```bash
  curl -X POST http://localhost:8000/webhooks/course-payment \
    -H "Content-Type: application/json" \
    -d '{"customer_id": "...", "amount": 200000}'
  ```

### 9.4 性能问题

#### 问题 11: 线索列表加载缓慢

**原因**: 数据量大且未优化查询

**解决**:

1. 检查是否正确使用索引：
   ```sql
   EXPLAIN QUERY PLAN SELECT * FROM lead WHERE owner_id = 'xxx';
   ```

2. 避免 N+1 查询：
   ```python
   # ❌ 错误：循环查询
   for lead in leads:
       lead.owner = db.query(User).filter(User.id == lead.owner_id).first()

   # ✅ 正确：一次性加载
   owner_ids = [lead.owner_id for lead in leads if lead.owner_id]
   owners = db.query(User).filter(User.id.in_(owner_ids)).all()
   owner_map = {owner.id: owner for owner in owners}
   for lead in leads:
       lead.owner = owner_map.get(lead.owner_id)
   ```

3. 增加分页：
   ```python
   @router.get("/leads")
   def list_leads(page: int = 1, page_size: int = 20):
       offset = (page - 1) * page_size
       leads = db.query(Lead).offset(offset).limit(page_size).all()
       return leads
   ```

#### 问题 12: AI 助手响应缓慢

**原因**: LLM API 调用延迟或网络问题

**解决**:

1. 检查 API Key 是否正确
2. 切换更快的 LLM Provider（如从 Claude 切换到 DeepSeek）
3. 优化 System Prompt，减少 Token 消耗
4. 增加前端 loading 状态提示，改善用户体验

---

## 附录

### A. 术语表

| 术语 | 英文 | 说明 |
|------|------|------|
| 线索 | Lead | 尚未购买小课的目标企业 |
| 客户 | Customer | 已购买小课的企业（线索转化而来） |
| 联系人 | Contact | 挂在线索或客户下的自然人 |
| 私有池 | Private Pool | 销售个人负责的线索集合 |
| 公共池 | Public Pool | 未被分配的线索集合 |
| 跟进记录 | FollowUp | 销售与客户的沟通记录 |
| 关键事件 | KeyEvent | 重要的业务节点（如送书、邀约） |
| 日报 | DailyReport | 销售每日工作总结 |
| 组织节点 | OrgNode | 公司组织架构树节点 |
| 数据范围 | DataScope | 用户可见数据的范围限制 |
| Ontology | Ontology | 业务对象的显式建模（对象+关系+动作） |
| Spec | Specification | 功能规格文档 |
| Action | Action | 业务动作（如 assign_lead、convert_lead） |

### B. 参考文档

| 文档 | 路径 | 用途 |
|------|------|------|
| 项目宪法 | `.specify/memory/constitution.md` | 核心原则和技术约束 |
| 功能规格 | `specs/master/spec.md` | Ontology 定义和用户故事 |
| 实现计划 | `specs/master/plan.md` | 技术选型和目录结构 |
| 任务清单 | `specs/master/tasks.md` | 110 个实现任务 |
| 数据模型 | `specs/master/data-model.md` | 数据库表结构 SQL |
| API 契约 | `specs/master/contracts/api-contracts.md` | 接口请求/响应格式 |
| 快速指南 | `specs/master/quickstart.md` | 用户和开发者快速入门 |
| 演示案例 | `src/demo/copilot-cases.md` | AI Copilot 演示脚本 |
| 项目记忆 | `memory/project_main.md` | 里程碑和重大决策记录 |
| PRD 文档 | `docs/SFA-CRM-PRD.md` | 产品需求文档（51.9 KB） |

### C. 常用命令速查

```bash
# 后端
cd src/backend
source .venv/bin/activate
uvicorn app.main:app --reload --port 8000

# 前端
cd src/frontend
npm run dev

# 测试
cd src/backend
pytest tests/integration/ -v

# Docker
cd src
docker-compose up -d
docker-compose down

# 重置演示数据
./reset-demo.sh   # macOS/Linux
reset-demo.bat    # Windows

# Git 提交
git add .
git commit -m "feat: T0XX - <description>

🤖 Generated with [Lingma][https://lingma.aliyun.com]"
```

### D. 联系方式

- **GitHub**: https://github.com/pmYangKun/sfa-crm
- **Check PRD Skill**: https://github.com/pmYangKun/check-prd-skill
- **项目作者**: Yang Kun

---

**文档版本**: 1.0.0 | **最后更新**: 2026-05-02

*本文档基于项目内部文档（宪法、Spec、Memory、PRD）自动生成，如有疑问请参考原始文档。*
