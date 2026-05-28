# 数据质量监控平台 MVP 实现计划

## Context

构建一个面向数据工程师的数据质量监控平台 MVP，目标是 2-4 周跑通核心链路：用 YAML 定义质量规则 → 对数据库/文件执行检查 → 发现异常通过 IM 告警。

**收敛后的 MVP 范围：**
- 质量维度：完整性（空值率、行数波动）+ 准确性（值域范围、格式校验）
- 数据源：PostgreSQL/MySQL + CSV/Parquet 文件
- 通知：IM webhook（飞书/钉钉/企微，选一个）
- 配置：YAML 文件定义规则
- 触发：手动 CLI 命令 + 简单事件触发（文件到达/webhook 接收）
- 无外部系统集成

---

## 技术架构

```
┌─────────────────────────────────────────────────┐
│                   CLI / API                       │
│              (手动触发 / Webhook接收)              │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│              Rule Engine (规则引擎)               │
│   解析 YAML → 生成检查任务 → 调度执行             │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            Connector Layer (连接器层)             │
│   DatabaseConnector  │  FileConnector            │
│   (SQLAlchemy)       │  (pandas/polars)          │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│            Check Executor (检查执行器)            │
│   completeness_check │  accuracy_check           │
└──────────────────────┬──────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────┐
│         Result Store + Alert (结果 & 告警)        │
│   SQLite存储历史  │  IM Webhook推送告警           │
└─────────────────────────────────────────────────┘
```

---

## 项目结构

```
data-quality-monitor/
├── pyproject.toml              # 项目配置 (Poetry/uv)
├── README.md
├── config/
│   └── rules/                  # 质量规则 YAML 文件目录
│       └── example.yaml
├── src/
│   └── dqm/                    # 主包
│       ├── __init__.py
│       ├── cli.py              # CLI 入口 (Typer/Click)
│       ├── api.py              # FastAPI 服务 (webhook触发)
│       ├── models.py           # 数据模型 (Pydantic)
│       ├── rule_engine.py      # YAML解析 + 规则调度
│       ├── connectors/
│       │   ├── __init__.py
│       │   ├── base.py         # Connector 抽象基类
│       │   ├── database.py     # 数据库连接器
│       │   └── file.py         # 文件连接器
│       ├── checks/
│       │   ├── __init__.py
│       │   ├── base.py         # Check 抽象基类
│       │   ├── completeness.py # 完整性检查
│       │   └── accuracy.py     # 准确性检查
│       ├── store.py            # 检查结果存储 (SQLite)
│       └── alerting/
│           ├── __init__.py
│           └── im_webhook.py   # IM 告警推送
└── tests/
    ├── test_rule_engine.py
    ├── test_checks.py
    └── test_connectors.py
```

---

## 规则 YAML 设计

```yaml
# config/rules/order_table.yaml
source:
  type: database
  connection: postgresql://user:pass@host:5432/db
  table: orders

checks:
  - type: completeness
    name: "订单金额不为空"
    column: amount
    rule: not_null
    threshold: 0.99  # 允许1%空值

  - type: completeness
    name: "每日订单数波动"
    rule: row_count
    expect:
      min_ratio: 0.7   # 不低于昨日70%
      max_ratio: 1.5   # 不超过昨日150%

  - type: accuracy
    name: "金额范围合理"
    column: amount
    rule: range
    expect:
      min: 0
      max: 1000000

  - type: accuracy
    name: "手机号格式"
    column: phone
    rule: regex
    pattern: "^1[3-9]\\d{9}$"

alert:
  channel: feishu  # feishu / dingtalk / wecom
  webhook_url: "https://open.feishu.cn/open-apis/bot/v2/hook/xxx"
  on: failure  # failure / always
```

---

## 实现步骤

### Phase 1：基础框架（第 1 周）

1. **项目初始化**
   - 用 uv 初始化 Python 项目
   - 配置依赖：fastapi, sqlalchemy, pydantic, typer, pyyaml, httpx

2. **数据模型定义** (`models.py`)
   - RuleConfig: YAML 解析后的规则对象
   - CheckResult: 单次检查结果（pass/fail + 详情）
   - AlertMessage: 告警消息结构

3. **Connector 层** (`connectors/`)
   - BaseConnector 抽象：`connect()`, `query()`, `get_column()`, `get_row_count()`
   - DatabaseConnector: 基于 SQLAlchemy，支持 PG/MySQL
   - FileConnector: 基于 pandas，支持 CSV/Parquet

4. **规则引擎** (`rule_engine.py`)
   - 解析 YAML 文件
   - 根据 check type 分发到对应检查器

### Phase 2：检查逻辑（第 2 周）

5. **完整性检查** (`checks/completeness.py`)
   - `not_null`: 空值率检查
   - `row_count`: 行数波动检查（与历史对比）

6. **准确性检查** (`checks/accuracy.py`)
   - `range`: 数值范围检查
   - `regex`: 正则格式校验
   - `enum`: 枚举值校验

7. **结果存储** (`store.py`)
   - SQLite 存储每次检查结果
   - 支持查询历史（供行数波动对比用）

### Phase 3：触发与告警（第 3 周）

8. **CLI 入口** (`cli.py`)
   - `dqm run config/rules/` — 执行所有规则
   - `dqm run config/rules/order_table.yaml` — 执行单个文件
   - `dqm history --table orders --last 7d` — 查看历史

9. **API 服务** (`api.py`)
   - `POST /webhook/trigger` — 接收外部事件触发检查
   - `GET /results` — 查询检查结果

10. **IM 告警** (`alerting/im_webhook.py`)
    - 封装飞书/钉钉/企微 webhook 格式
    - 检查失败时推送结构化卡片消息

### Phase 4：打磨与验证（第 4 周）

11. **端到端测试**
    - 准备测试数据库 + 测试 CSV
    - 完整跑通：规则定义 → 执行检查 → 存储结果 → 触发告警

12. **文档与示例**
    - README：快速开始指南
    - 示例规则文件

---

## 关键技术选型

| 组件 | 选型 | 理由 |
|------|------|------|
| Web框架 | FastAPI | 异步、自带文档、类型安全 |
| ORM/DB连接 | SQLAlchemy 2.0 | 多数据库支持、成熟稳定 |
| 数据处理 | pandas (或 polars) | 文件读取 + 统计计算 |
| CLI | Typer | 基于类型提示、开发体验好 |
| 配置解析 | PyYAML + Pydantic | 校验 + 类型安全 |
| 结果存储 | SQLite | 零部署、MVP 够用 |
| HTTP客户端 | httpx | 异步支持、用于 webhook |
| 包管理 | uv | 快速、现代 |

---

## 验证方式

1. **单元测试**：`pytest tests/` 覆盖规则解析、检查逻辑、连接器
2. **集成测试**：准备一个 Docker Compose（PG + 测试数据），端到端验证
3. **手动验证**：
   - 写一份规则 YAML → `dqm run` → 确认 IM 收到告警
   - 故意制造脏数据 → 确认检查能正确识别并告警
   - 调用 `POST /webhook/trigger` → 确认 API 触发正常工作

---

## MVP 之后的迭代方向（参考）

- V2：Web UI（规则管理 + 结果看板）
- V2：增加时效性 + 一致性维度
- V2：邮件 + 平台内通知
- V3：与 Airflow/dbt 集成
- V3：自动建议规则（基于数据 profiling）
