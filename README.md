# LLM Customer Service

一个基于大语言模型的电商智能客服 Agent 工程。项目主体是 `atguigu_ai` 对话框架，配套 `ecs_demo` 电商客服示例，覆盖订单查询、收货信息修改、订单取消、物流查询、售后申请，以及基于 Neo4j 图数据库的企业知识检索。

本项目的核心目标不是简单调用 LLM 生成回复，而是把大模型能力工程化地接入客服系统：LLM 负责理解用户输入并生成受控命令，后续由 Flow 引擎、策略模块、Action 插件和对话状态管理器完成可控、可追踪、可扩展的业务执行。

> 说明：当前仓库更偏教学、实验和项目展示，不是开箱即用的生产级客服平台。完整运行 `ecs_demo` 需要配置大模型 API、MySQL、Neo4j 和本地 embedding 模型。

## 目录

- [项目亮点](#项目亮点)
- [系统架构](#系统架构)
- [核心处理链路](#核心处理链路)
- [技术栈](#技术栈)
- [目录结构](#目录结构)
- [环境要求](#环境要求)
- [快速开始](#快速开始)
- [配置说明](#配置说明)
- [REST API](#rest-api)
- [核心概念](#核心概念)
- [电商客服 Demo](#电商客服-demo)
- [GraphRAG 企业知识检索](#graphrag-企业知识检索)
- [新增业务流程](#新增业务流程)
- [CLI 命令](#cli-命令)
- [模型打包与加载](#模型打包与加载)
- [调试与排障](#调试与排障)
- [适合阅读的代码入口](#适合阅读的代码入口)

## 项目亮点

- **LLM 命令化输出**：不让大模型直接操作业务，而是让模型输出受控命令，例如启动 Flow、设置槽位、触发知识检索、转人工等。
- **LangGraph 编排 Agent 执行链路**：将一次用户输入拆分为理解、策略、动作、保护和响应多个节点，提升复杂多轮任务的可控性。
- **配置化多轮业务流程**：通过 YAML 定义订单查询、修改收货信息、取消订单、物流查询、售后申请等流程，方便扩展新业务。
- **对话状态统一管理**：使用 `DialogueStateTracker` 和 `DialogueStack` 记录用户消息、机器人回复、槽位、当前 Flow、执行历史和上下文。
- **Action 插件机制**：业务项目只需在 `actions/` 下新增 Action 类，框架会自动扫描、注册并执行。
- **GraphRAG 检索增强**：结合 Neo4j、BGE embedding、Hybrid Retrieval 和 LLM 生成 Cypher，实现商品知识图谱问答。
- **可运行、可调试、可观察**：提供 CLI、FastAPI、WebSocket、Inspect 页面和 Tracker 查询接口，便于调试多轮 Agent 行为。

## 系统架构

项目分为两层：

1. `atguigu_ai`：通用对话框架层，负责 Agent 加载、消息处理、状态管理、流程编排、策略决策、动作执行、API 服务和命令行工具。
2. `ecs_demo`：电商客服业务层，负责领域配置、业务 Flow、自定义 Action、数据库访问和 GraphRAG 检索扩展。

整体架构如下：

```text
用户输入
  |
  v
Channel / API / CLI
  |
  v
Agent.handle_message()
  |
  v
LangGraph Message Processing Graph
  |
  +--> understand_node
  |      - 解析 /SetSlots(...) 按钮 payload
  |      - 调用 LLMCommandGenerator
  |      - 生成并处理 Command
  |
  +--> policy_node
  |      - 调用 PolicyEnsemble
  |      - FlowPolicy 优先处理业务流程
  |      - EnterpriseSearchPolicy 处理知识检索、闲聊、兜底
  |
  +--> action_node
  |      - 查找并执行 Action
  |      - 写入 BotMessage
  |      - 累积响应
  |
  +--> guard_node
  |      - 限制最大动作次数
  |      - 防止死循环
  |
  +--> response_node
         - 结束本轮处理
         - 返回响应
```

## 核心处理链路

一条用户消息会经过以下阶段：

1. **理解阶段**  
   `LLMCommandGenerator` 读取当前对话历史、槽位、Flow 定义和用户最新输入，让大模型输出受控命令。

2. **命令处理阶段**  
   `CommandParser` 将 LLM 输出解析为命令对象，`CommandProcessor` 执行命令并更新 Tracker，例如启动 Flow、设置槽位、压入搜索栈或转人工栈。

3. **策略决策阶段**  
   `PolicyEnsemble` 按优先级选择下一步动作。`FlowPolicy` 处理确定性业务流程，`EnterpriseSearchPolicy` 处理知识检索、闲聊、无法处理、流程完成后的追问等场景。

4. **动作执行阶段**  
   `Action` 执行业务逻辑，例如查询订单数据库、返回按钮、修改收货信息、提交售后申请、触发搜索或发送文本。

5. **保护与响应阶段**  
   `guard_node` 检查动作执行次数，避免配置错误导致无限循环；`response_node` 收集最终响应并结束本轮。

核心链路：

```text
用户消息
  -> understand_node
  -> policy_node
  -> action_node
  -> guard_node
  -> policy_node / response_node
  -> 机器人响应
```

## 技术栈

| 类型 | 技术 |
| --- | --- |
| 语言 | Python 3.10+ |
| Web 服务 | FastAPI, Uvicorn |
| CLI | Click |
| Agent 编排 | LangGraph |
| LLM 集成 | LangChain, DashScope/Qwen, OpenAI-compatible client |
| 状态管理 | DialogueStateTracker, DialogueStack, Slot |
| 流程配置 | YAML Flow DSL |
| 数据库 | MySQL, SQLAlchemy, PyMySQL |
| 图数据库 | Neo4j |
| RAG / GraphRAG | neo4j-graphrag, HybridRetriever, Cypher generation |
| Embedding | sentence-transformers, BGE base zh v1.5 |
| 模板 | Jinja2 |
| 通信 | REST, WebSocket, Socket.IO, Console |
| 工程工具 | dotenv, pydantic, rich, tqdm |

## 目录结构

```text
.
├── atguigu_ai/                     # 对话系统框架源码
│   ├── agent/                      # Agent 入口、Action 系统、LangGraph 图
│   │   └── graph/
│   │       ├── builder.py          # 构建消息处理图
│   │       ├── edges.py            # 条件边路由
│   │       ├── state.py            # LangGraph 状态定义
│   │       └── nodes/              # understand / policy / action / guard / response
│   ├── api/                        # FastAPI 服务与 Inspect 页面
│   ├── channels/                   # REST、SocketIO、Console 通道抽象
│   ├── cli/                        # atguigu 命令行入口
│   ├── core/                       # Domain、Slot、Tracker、TrackerStore
│   ├── dialogue_understanding/     # 命令生成、命令解析、Flow、对话栈
│   ├── policies/                   # FlowPolicy、EnterpriseSearchPolicy、PolicyEnsemble
│   ├── retrieval/                  # 检索器抽象与动态创建
│   ├── shared/                     # 配置、常量、LLM 客户端、YAML 工具
│   ├── training/                   # 配置校验与模型打包
│   └── nlg/                        # 模板回复和回复重述
├── ecs_demo/                       # 电商客服 Demo 工程
│   ├── actions/                    # 订单、物流、售后业务动作
│   ├── addons/                     # GraphRAG 检索扩展
│   ├── data/flows/                 # 电商客服流程定义
│   ├── domain/                     # 槽位、回复模板、动作声明
│   ├── models/bge-base-zh-v1.5/    # 本地中文 embedding 模型
│   ├── config.yml                  # Pipeline 与策略配置
│   └── endpoints.yml               # LLM、Neo4j、MySQL、Tracker 配置
├── scripts/                        # 辅助脚本
├── requirements-atguigu.txt        # Python 依赖
├── setup.py                        # 包安装配置，注册 atguigu CLI
└── README.md
```

## 环境要求

- Python 3.10+
- 可访问的大模型服务，示例默认使用 DashScope/Qwen
- 如运行完整 `ecs_demo`，还需要：
  - MySQL，默认数据库名为 `ecommerce`
  - Neo4j，默认地址为 `bolt://localhost:7687`
  - 本地 embedding 模型目录：`ecs_demo/models/bge-base-zh-v1.5`

## 快速开始

### 1. 创建虚拟环境

```bash
python -m venv .venv
```

Windows PowerShell：

```powershell
.\.venv\Scripts\Activate.ps1
```

macOS / Linux：

```bash
source .venv/bin/activate
```

### 2. 安装依赖

```bash
pip install -r requirements-atguigu.txt
pip install -e .
```

安装完成后会注册命令行工具：

```bash
atguigu --help
python -m atguigu_ai --help
```

### 3. 配置环境变量

在项目根目录或 `ecs_demo` 目录下创建 `.env`：

```dotenv
DASHSCOPE_API_KEY=your_dashscope_api_key
MYSQL_PASSWORD=your_mysql_password
NEO4J_PASSWORD=your_neo4j_password

# 可选：覆盖 GraphRAG 的本地 embedding 模型路径
EMBEDDING_MODEL=./models/bge-base-zh-v1.5
```

### 4. 进入 Demo 工程

```bash
cd ecs_demo
```

### 5. 校验配置

```bash
atguigu train --dry-run
```

在本框架中，`train` 不是传统机器学习训练，而是用于：

- 加载并校验 Domain
- 加载并校验 Flow
- 检查 Flow 引用的 Action
- 打包可部署配置

### 6. 打包模型配置

```bash
atguigu train
```

默认输出：

```text
models/model-YYYYMMDD-HHMMSS.tar.gz
```

### 7. 命令行对话测试

```bash
atguigu shell
```

常用 Shell 命令：

```text
/help      显示帮助
/reset     重置当前会话
/slots     查看槽位
/history   查看对话历史
/debug     切换调试日志
/quit      退出
```

### 8. 启动 HTTP 服务

```bash
atguigu run --port 5005
```

启动后可访问：

- API 文档：http://localhost:5005/docs
- Inspect 调试页：http://localhost:5005/inspect
- 健康检查：http://localhost:5005/health

## 配置说明

### `ecs_demo/config.yml`

该文件定义对话管线和策略：

```yaml
recipe: default.v1
language: zh

pipeline:
  - name: LLMCommandGenerator
    llm: default

policies:
  - name: FlowPolicy
  - name: EnterpriseSearchPolicy
    llm: default
    vector_store: addons.information_retrieval.GraphRAG
```

含义：

- `LLMCommandGenerator`：使用大模型生成对话命令。
- `FlowPolicy`：优先执行配置化业务流程。
- `EnterpriseSearchPolicy`：处理知识检索、闲聊、无法处理、流程完成追问等逻辑。
- `vector_store`：动态加载 GraphRAG 检索器。

### `ecs_demo/endpoints.yml`

该文件定义模型、数据库、图数据库和 Tracker 存储：

```yaml
models:
  default:
    type: qwen
    model: qwen-plus
    api_key: ${DASHSCOPE_API_KEY}
    temperature: 0.1

vector_store:
  uri: bolt://localhost:7687
  user: neo4j
  password: ${NEO4J_PASSWORD}

database:
  url: mysql+pymysql://root:${MYSQL_PASSWORD}@localhost:3306/ecommerce

tracker_store:
  type: memory

nlg:
  rephrase_enabled: false
```

## REST API

### 健康检查

```http
GET /health
```

响应示例：

```json
{
  "status": "ok",
  "version": "0.1.0",
  "agent_ready": true
}
```

### 发送消息

```http
POST /api/messages
Content-Type: application/json
```

请求：

```json
{
  "sender": "user_001",
  "message": "我想查一下订单",
  "metadata": {}
}
```

响应：

```json
[
  {
    "recipient_id": "user_001",
    "text": "请选择订单",
    "buttons": [
      {
        "title": "[待发货]订单ID：xxx",
        "payload": "/SetSlots(order_id=xxx)"
      }
    ],
    "image": null,
    "custom": null
  }
]
```

### 查看会话

```http
GET /api/sessions/{session_id}
GET /api/tracker/{session_id}/full
```

### 重置会话

```http
POST /api/sessions/{session_id}/reset
```

### 查看配置

```http
GET /api/domain
GET /api/flows
```

### WebSocket

```text
ws://localhost:5005/api/stream
```

消息示例：

```json
{"type": "connect", "session_id": "user_001"}
{"type": "message", "sender_id": "user_001", "message": "查物流"}
{"type": "ping"}
```

## 核心概念

### Domain

Domain 描述机器人能够处理的领域信息，包括：

- `slots`：对话槽位，例如 `order_id`、`user_id`、`receive_city`
- `responses`：模板回复，例如 `utter_ask_order_id`
- `actions`：可执行动作声明
- `forms`：预留表单配置
- `session_config`：会话配置

`ecs_demo/domain/` 将 Domain 拆成多个 YAML 文件：

```text
domain_order.yml      订单相关槽位、回复和动作
domain_logistics.yml  物流相关槽位和动作
domain_postsale.yml   售后相关槽位、回复和动作
domain_patterns.yml   通用兜底、转人工、完成提示等模式
```

### Flow

Flow 定义多轮业务流程，常见步骤类型包括：

- `action`：执行业务动作
- `collect`：收集槽位
- `set_slot` / `set_slots`：设置槽位
- `condition`：条件分支
- `link`：跳转到其他 Flow
- `call`：调用子 Flow
- `END`：结束流程

典型流程：

- `switch_user_id`：切换用户
- `query_order_detail`：查询订单详情
- `modify_order_receive_info`：修改订单收货信息
- `cancel_order`：取消订单
- `query_logistics_companys`：查询支持的快递公司
- `query_shipping_order_logistics`：查询订单物流
- `apply_postsale`：申请售后

### Slot

Slot 保存多轮对话中的结构化信息，来源包括：

- `from_llm`：由 LLM 从用户输入中提取
- `controlled`：由 Action 或按钮 payload 设置

按钮 payload 支持直接设置槽位：

```text
/SetSlots(order_id=123456)
/SetSlots(if_cancel_order=true)
/SetSlots(modify_content=收货地址)
```

`understand_node` 会识别 `/SetSlots(...)`，绕过 LLM，直接转换为 `SetSlotCommand`，提升按钮交互的稳定性和速度。

### Command

Command 是 LLM 与确定性业务系统之间的中间层。模型输出的不是最终业务结果，而是可解析、可校验、可执行的命令。

常见命令：

```text
start flow query_order_detail
set slot order_id "123456"
knowledge_answer
chitchat
cannot_handle
human_handoff
```

### Policy

Policy 决定下一步执行什么 Action。

- `FlowPolicy`：当存在活跃 Flow 时，根据 Flow 当前步骤决定下一步动作。
- `EnterpriseSearchPolicy`：处理搜索、闲聊、无法处理、转人工、流程完成后的追问。
- `PolicyEnsemble`：按优先级组合多个策略，选择最佳预测结果。

### Action

Action 是业务逻辑执行单元。自定义 Action 需要继承 `atguigu_ai.agent.actions.Action` 并实现 `run()`：

```python
from typing import Any, Optional
from atguigu_ai.agent.actions import Action, ActionResult


class ActionDemo(Action):
    @property
    def name(self) -> str:
        return "action_demo"

    async def run(
        self,
        tracker: Any,
        domain: Optional[Any] = None,
        **kwargs: Any,
    ) -> ActionResult:
        result = ActionResult()
        result.add_response("这是一个自定义动作")
        return result
```

`Agent.load()` 会自动扫描业务工程的 `actions/` 目录，导入所有 `.py` 文件，发现并注册 `Action` 子类。

### Tracker

`DialogueStateTracker` 是单个用户会话的状态容器，记录：

- 用户消息和机器人回复
- 当前槽位值
- 当前活跃 Flow
- 对话栈 `dialogue_stack`
- Flow 执行历史
- 最近执行的 Action
- LLM 生成过的命令

Tracker 默认存储方式由 `endpoints.yml` 的 `tracker_store` 决定，示例使用内存：

```yaml
tracker_store:
  type: memory
```

也可以扩展为 JSON 或 MySQL 存储。

## 电商客服 Demo

### 订单模块

相关文件：

```text
ecs_demo/domain/domain_order.yml
ecs_demo/data/flows/flow_order.yml
ecs_demo/actions/action_order.py
```

能力：

- 切换用户 ID
- 查询订单详情
- 修改订单收货信息
- 取消订单

设计要点：

- 使用 `goto` 槽位控制订单查询范围。
- `action_ask_order_id` 根据不同业务场景筛选订单。
- 用户通过按钮选择订单，按钮 payload 直接设置 `order_id`。
- 修改收货信息流程支持选择已有地址、修改并新建地址、确认后更新订单。

### 物流模块

相关文件：

```text
ecs_demo/domain/domain_logistics.yml
ecs_demo/data/flows/flow_logistics.yml
ecs_demo/actions/action_logistics.py
```

能力：

- 查询支持的快递公司
- 查询已发货订单物流轨迹

### 售后模块

相关文件：

```text
ecs_demo/domain/domain_postsale.yml
ecs_demo/data/flows/flow_postsale.yml
ecs_demo/actions/action_postsale.py
```

能力：

- 申请退款、退货或换货
- 选择售后订单
- 校验售后资格
- 收集售后类型和售后原因
- 提交售后申请

业务约束示例：

- 订单需处于签收后状态才支持售后。
- 可根据签收时间判断是否超过售后期限。
- 售后申请创建后更新订单状态为售后中。

## GraphRAG 企业知识检索

相关文件：

```text
ecs_demo/addons/information_retrieval.py
ecs_demo/addons/create_indexing.py
```

`GraphRAG` 继承 `InformationRetrieval`，核心流程：

1. LLM 根据用户问题识别图谱入口节点类型和实体，例如 SKU、SPU、品牌、分类、属性、用户。
2. 对实体进行中文分词和 embedding。
3. 使用 Neo4j Hybrid Retrieval 获取候选入口节点。
4. LLM 根据 schema、入口节点和用户问题生成 Cypher。
5. 使用 `EXPLAIN` 和 LLM 校验 Cypher 语法与逻辑。
6. 如有问题，调用 LLM 修正 Cypher。
7. 使用 `CypherQueryCorrector` 校正关系方向。
8. 执行 Cypher 查询。
9. 将查询结果格式化为 `SearchResult`。
10. `EnterpriseSearchPolicy` 基于检索结果生成最终客服回答。

该模块重点解决自然语言问题到图数据库查询之间的映射问题。

## 新增业务流程

推荐按以下顺序扩展：

1. 在 `domain/*.yml` 中声明需要的槽位、回复模板和动作名。
2. 在 `data/flows/*.yml` 中新增 Flow，描述业务步骤。
3. 在 `actions/*.py` 中实现业务 Action。
4. 在 `config.yml` 中确认策略可用。
5. 执行 `atguigu train --dry-run` 校验配置。
6. 使用 `atguigu shell` 或 `atguigu inspect` 调试多轮流程。

示例 Flow：

```yaml
flows:
  query_coupon:
    name: 查询优惠券
    description: 查询用户当前可用优惠券
    steps:
      - action: action_query_coupon
        next: END
```

示例 Domain：

```yaml
actions:
  - action_query_coupon

responses:
  utter_default:
    - text: "抱歉，我暂时没有理解您的意思。"
```

示例 Action：

```python
from typing import Any, Optional
from atguigu_ai.agent.actions import Action, ActionResult


class ActionQueryCoupon(Action):
    @property
    def name(self) -> str:
        return "action_query_coupon"

    async def run(
        self,
        tracker: Any,
        domain: Optional[Any] = None,
        **kwargs: Any,
    ) -> ActionResult:
        result = ActionResult()
        user_id = tracker.get_slot("user_id")
        result.add_response(f"用户 {user_id} 当前暂无可用优惠券。")
        return result
```

## CLI 命令

### 初始化新工程

```bash
atguigu init --path ./my_bot
```

生成结构：

```text
config.yml
domain.yml
endpoints.yml
data/flows.yml
actions/actions.py
models/
tests/
```

### 训练 / 校验 / 打包

```bash
atguigu train
atguigu train --dry-run
atguigu train --domain domain --data data --output models
```

### 运行服务

```bash
atguigu run
atguigu run --model ./models/latest.tar.gz
atguigu run --port 8080 --disable-inspect
```

### 交互测试

```bash
atguigu shell
atguigu shell --sender-id test_user
```

### 调试页面

```bash
atguigu inspect
atguigu inspect --no-browser
```

### 导出

```bash
atguigu export --output ./export/model.tar.gz
atguigu export --format zip --output ./export/model.zip
atguigu export --format dir --output ./export/model_dir
```

## 模型打包与加载

`Agent.load(project_path)` 支持三种输入：

- 模型压缩包：`models/model-xxx.tar.gz`
- 包含 `models/` 子目录的工程目录：自动选择最新模型包
- 直接包含 `config.yml`、`domain.yml` 或 `domain/`、`data/flows` 的项目目录

加载时会：

1. 加载 Domain。
2. 加载 Flows。
3. 自动发现 `actions/` 中的 Action 类。
4. 读取 `endpoints.yml` 中的 LLM、Tracker、vector store 配置。
5. 读取 `config.yml` 中的 pipeline 和 policies。
6. 创建 `LLMCommandGenerator`、`FlowPolicy`、`EnterpriseSearchPolicy`。
7. 构建 LangGraph 消息处理图。

## 工程落地设计

### 为什么不让 LLM 直接操作业务？

订单修改、取消订单、售后申请等操作对准确性要求较高，如果让 LLM 直接生成业务结果，容易出现幻觉、越权、状态污染或错误操作。

本项目将 LLM 限定在理解层：

```text
自然语言 -> 受控命令 -> 状态机 / Flow / Action -> 业务结果
```

这样既能利用大模型的语义理解能力，又能保持业务执行的确定性。

### 如何处理多轮状态污染？

项目通过以下机制控制状态：

- `DialogueStateTracker` 保存当前会话状态。
- `DialogueStack` 管理 Flow、搜索、闲聊、兜底、转人工等上下文。
- `FlowPolicy` 在流程结束后重置 scoped slots。
- `collect` 阶段启用 force slot filling，只保留当前需要收集的槽位命令。
- 按钮 payload 使用 `/SetSlots(...)` 直接设置槽位，绕过 LLM。

### 如何防止死循环？

LangGraph 图中加入 `guard_node`，每轮消息设置最大动作数：

```python
max_actions = 10
```

当策略或 Flow 配置异常导致动作不断执行时，guard 节点会终止流程并返回响应。

## 调试与排障

### 推荐调试方式

1. 使用 `atguigu train --dry-run` 检查 Domain、Flow 和 Action 引用。
2. 使用 `atguigu shell` 进行命令行多轮对话。
3. 使用 `/slots` 查看槽位是否正确设置。
4. 使用 `/history` 查看历史对话。
5. 使用 `atguigu inspect` 查看 Tracker、Flow 栈、执行历史和响应。
6. 查看服务日志定位 Action、LLM、数据库或检索异常。

### 常见问题

#### `ModuleNotFoundError`

优先检查：

- 当前工作目录是否正确。
- 是否执行过 `pip install -e .`。
- 是否在 `ecs_demo` 目录下运行 Demo。
- `actions/` 或 `addons/` 是否能被动态导入。

#### `DASHSCOPE_API_KEY` 相关错误

检查：

- `.env` 是否在当前工作目录或 `ecs_demo` 目录。
- 环境变量名是否正确。
- `endpoints.yml` 是否引用 `${DASHSCOPE_API_KEY}`。

#### MySQL 连接失败

检查：

- MySQL 服务是否启动。
- 数据库名是否为 `ecommerce`。
- `MYSQL_PASSWORD` 是否正确。
- `ecs_demo/endpoints.yml` 中 `database.url` 是否正确。
- 表结构和测试数据是否已准备。

#### Neo4j 连接失败

检查：

- Neo4j 服务是否启动。
- `vector_store.uri` 是否正确。
- `NEO4J_PASSWORD` 是否正确。
- 图谱节点、关系、全文索引和向量索引是否已创建。

#### Flow 不继续执行

排查：

- 当前是否有活跃 Flow。
- 当前步骤是否需要收集槽位。
- `/slots` 中目标槽位是否已经设置。
- 按钮 payload 是否为 `/SetSlots(slot=value)` 格式。
- Flow 的 `next`、`condition`、`END` 是否配置正确。

#### LLM 乱设置槽位或启动新流程

项目已在 collect 阶段做 force slot filling。如果仍有问题，可以检查：

- Prompt 中当前 `slot_to_collect` 是否正确。
- FlowStackFrame 中 `slot_to_collect` 是否被提前预设。
- Slot 描述和 allowed values 是否清晰。

#### GraphRAG 返回空结果

检查：

- LLM 是否正确识别入口节点 label 和 entity。
- Neo4j 中是否有对应节点。
- 全文索引和向量索引是否存在。
- embedding 模型路径是否正确。
- 生成的 Cypher 是否能在 Neo4j Browser 中执行。

## 适合阅读的代码入口

- `atguigu_ai/agent/agent.py`：Agent 加载和消息处理主流程
- `atguigu_ai/agent/graph/builder.py`：LangGraph 图结构
- `atguigu_ai/agent/graph/nodes/understand.py`：LLM 命令生成与按钮 payload 解析
- `atguigu_ai/agent/graph/nodes/policy.py`：策略预测节点
- `atguigu_ai/agent/graph/nodes/action.py`：Action 执行节点
- `atguigu_ai/agent/actions.py`：Action 基类、内置 Action 和注册机制
- `atguigu_ai/dialogue_understanding/generator/llm_generator.py`：LLM 命令生成器
- `atguigu_ai/dialogue_understanding/generator/command_parser.py`：命令解析器
- `atguigu_ai/dialogue_understanding/processor/command_processor.py`：命令执行与状态更新
- `atguigu_ai/dialogue_understanding/flow/flow_executor.py`：Flow 步骤执行逻辑
- `atguigu_ai/policies/flow_policy.py`：业务流程策略
- `atguigu_ai/policies/enterprise_search_policy.py`：知识检索、闲聊和兜底策略
- `atguigu_ai/core/tracker.py`：对话状态管理
- `atguigu_ai/api/server.py`：FastAPI 服务接口
- `ecs_demo/actions/action_order.py`：订单业务 Action
- `ecs_demo/actions/action_logistics.py`：物流业务 Action
- `ecs_demo/actions/action_postsale.py`：售后业务 Action
- `ecs_demo/addons/information_retrieval.py`：GraphRAG 检索实现

## 项目总结

本项目展示了一个 LLM 客服 Agent 从自然语言理解、多轮流程编排、状态管理、业务 Action 执行、知识图谱检索到 Web 服务化的完整工程链路。它重点解决了大模型输出不稳定、多轮状态难管理、业务流程难扩展、知识检索难落地和 Agent 调试困难等问题，适合作为学习 LLM 应用工程化、LangGraph 编排、RAG/GraphRAG 和智能客服系统设计的参考项目。
