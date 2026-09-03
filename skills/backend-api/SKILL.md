---
name: "backend-api"
description: "后端 API 工程规范：Web 框架选型（FastAPI、Flask、Django、沿用既有栈）、分层结构、统一响应 ApiResponse、JWT 认证、数据库设计与事务、三场景测试。开发任何 Web API 后端时使用。"
---

# 后端 API 工程规范

后端工程模式。覆盖安全、验收、分层等工程约束的具体落地写法。**约束框架无关**——先按场景选技术栈，再套用本规范。

**本规范下的所有 Python 代码同时遵循 python-style skill**（snake_case、中文注释、公开函数 docstring 含 Args/Returns）。

## 1. 技术栈选型（按场景，先选栈再写码）

| 场景特征 | 推荐方案 | 理由 |
|----------|---------|------|
| 纯 REST API、高并发异步 IO、类型驱动 | FastAPI | async 原生、Pydantic 入参校验、OpenAPI 自动文档 |
| 传统同步、小服务、依赖的生态库多 | Flask | 轻量、生态最广、迁移老代码成本低 |
| 全功能站点：模板渲染 + admin 后台 + ORM + 权限 | Django | admin/ORM/认证开箱即用，中后台 CRUD 效率最高 |
| 已有项目迭代 | 沿用既有栈 | 选定技术栈后不随意更换 |

- 选型输出物：一行选型理由写进架构文档，说明被排除方案及原因（对照表见 eng-thinking skill）
- 工程最优 ≠ 技术最新——团队熟悉度是正当的选型理由
- 同一项目禁止混用两个 Web 框架；ORM 同理（SQLModel / SQLAlchemy / Django ORM 按栈选一个）

## 2. 分层目录结构（约束框架无关，文件名按栈适配）

```
project/
├── run.py / manage.py     # 初始化数据库、创建默认管理员
├── 入口文件               # 只做路由注册、健康检查，无业务逻辑
├── app/
│   ├── 连接层             # 引擎/Session/建表——只管连接
│   ├── models 层          # 数据表模型 + CRUD 方法
│   ├── schema 层          # 请求/响应契约模型
│   └── routes 层          # 按业务域拆分路由（auth/tickets/comments）
└── tests/
```

- 每层职责单一：连接层不含业务，models 管模型和 CRUD，schema 管出入参，路由只编排
- 路由按业务域拆分文件，入口只做注册

## 3. 统一响应格式（强制，任何栈一致）

所有接口返回统一 `ApiResponse`，`code=0` 成功、`code=1` 业务错误：

```python
class ApiResponse(Generic[T]):
    code: int      # 0 成功 / 1 业务错误
    message: str
    data: T | None
```

- 业务错误（重复注册、无权操作）返回 HTTP 200 + `code=1`；参数校验、鉴权失败用框架对应的 4xx
- 健康检查独立：`GET /` 或 `/health` 返回 `{"status": "healthy"}`

## 4. RESTful 接口设计（框架无关）

- 动词映射：`GET` 查询 / `POST` 创建 / `PUT` 更新 / `DELETE` 删除
- 路径按资源组织：`/auth/register`、`/tickets/`、`/tickets/{ticket_id}`
- 列表接口支持 `skip`/`limit` 分页，禁止无限制全量查询
- OAuth2 密码模式登录接收 form 数据（FastAPI 用 `OAuth2PasswordRequestForm`，客户端须 `data=` 而非 `json=`）

## 5. 认证与安全落地（约束框架无关）

| 约束 | 落地 |
|------|------|
| 密码存储 | bcrypt（或 argon2）哈希，**禁止明文/MD5/SHA1**；响应模型排除 `hashed_password` 字段 |
| JWT 鉴权 | token 解析 → 查库验证用户存在 → 失败 401；受保护接口统一经鉴权依赖/中间件注入当前用户 |
| 越权防御 | 操作前校验资源归属：`if obj.user_id != current_user.id: return ApiResponse(code=1, message="无权操作")`——前端传 ID 不代表有权改 |
| SQL 注入 | 一律参数化查询/ORM 占位符，禁止字符串拼接 SQL |

## 6. 数据库设计规范（框架无关）

- 状态、类型、优先级用枚举常量（`TicketStatus.OPEN`），禁止魔法值
- 外键显式声明（`orders.user_id REFERENCES users(id)`）；约束用 `FOREIGN KEY`、`UNIQUE KEY`、`CHECK`
- 表选项：`utf8mb4`、InnoDB（MySQL），字段写注释
- 索引：`WHERE`/`ORDER BY`/`JOIN` 高频列建索引，命名 `idx_表_列`
- 事务：多表变动必须原子——ORM 用 session 事务块，裸 SQL 用 `try/except/finally` 包裹 `begin/commit/rollback`
- 批量插入用批量接口（`executemany`/ORM bulk）；连接走连接池（`PooledDB`/引擎内置池）

## 7. 测试规范（三场景起步，框架无关）

接口测试至少覆盖三个场景：

```python
def test_create_ticket(client):              # 正常请求
def test_create_ticket_bad_params(client):   # 参数错误
def test_unauthorized(client):               # 未登录/无权限
```

- 测试客户端按栈选：FastAPI `TestClient` / Flask `app.test_client()` / Django `TestCase`，统一 pytest 组织
- 验收六条对照：正常能用、出错会拦、没权限过不去、数据不越界、数据库结果对得上、关键过程日志可查

## 8. 部署顺序（框架无关）

容器化用 Docker Compose；入口脚本先执行初始化（建库、默认管理员）再启动服务——环境准备优先于服务启动。排查连通性问题从内到外：程序端口 → 安全组 → 防火墙；数据库、Redis 不用默认端口。
