# 后端文档 (FastAPI)

## 1. 技术栈与运行环境
- **框架**：FastAPI + Pydantic + SQLAlchemy (或 Prisma/SQLModel，保持 async 能力)。
- **数据库**：PostgreSQL（推荐，支持并发事务和 JSON 字段），兼容 MySQL。
- **部署**：Uvicorn，建议容器化；使用 uv 管理依赖。
- **环境变量**：
  - `DATABASE_URL`：标准 DSN，可直接书写 `postgresql://...` 或 `mysql://...`，服务会自动补齐异步驱动。
  - `JWT_SECRET`, `JWT_EXPIRES_IN`, `WECHAT_APPID/SECRET`。
  - `DEFAULT_TIMEZONE`：用于生成时间槽（建议 `Asia/Shanghai`）。

## 2. 项目结构（`backend/`）
```
/backend
├─ main.py              # FastAPI 入口，挂载路由与中间件
├─ requirements.txt
└─ src/
   ├─ core/             # 配置、DB 会话、依赖注入、异常处理
   ├─ shared/           # 工具、常量、基础 schema（分页、响应包装）
   └─ modules/
      ├─ auth/          # 登录/Token 刷新
      ├─ users/         # 用户 & 顾客 CRUD
      ├─ catalog/       # 技师/服务/地点/定价
      ├─ schedule/      # 常规排班与可用时间计算
      └─ appointments/  # 客户/管理员预约、状态流转
```

## 3. 模块职责与 API 边界
| 模块 | 主要模型 | 公开服务 | 侧重 |
| --- | --- | --- | --- |
| `auth` | Users | 微信 code 换 token、JWT 验证 | 统一鉴权中间件 |
| `users` | Users, Patients | 我的账户、顾客 CRUD、管理员查看全部 | 绑定 managed_by_user_id，并在预约成功时更新 `default_location_id` 供前端默认选址 |
| `catalog` | Technicians, Locations, Services, Offerings | 公开查询 + 管理端 CRUD | price/duration 源 |
| `schedule` | BusinessHours | 可用时间、排班 CRUD | 处理并发/限额 |
| `appointments` | Appointments (+ Offerings/Patients join) | 客户预约/撤销、管理员增删改 | 维护 `booked_by_role` 逻辑 |

- `auth` 模块也提供 `/auth/login/phone` 供 H5 客户端提交手机号（可选短信验证码）获取 JWT，因此 `User.wechat_openid` 被设计为可空，允许非微信账号登录。

所有模块通过 `router = APIRouter(prefix="/api/v1/...")` 暴露接口，统一在 `main.py` 注册。管理端路由放在 `/api/v1/admin/...`，在依赖中校验 `role in {'admin','technician'}`。

## 4. 核心业务流程
### 4.1 可用时间计算 (`GET /schedule/availability`)
1. 读取 Offerings（技师/服务/地点）与 duration。
2. 拉取当天 `BusinessHours`，按 `day_of_week`（直接存储 `monday`~`sunday`）生成初始时间块。
3. 按配置的 BusinessHours 生成时间块（后续由配额和预约冲突裁剪）。
4. 查询该技师当天相关 `Appointments`，按 `status='scheduled'` 排除。
5. 根据 `Services.concurrency_level` 判断是否可叠加；单线程服务需确保无 overlap。
6. 若目标技师为“父亲”，统计 `Appointments` 中 `booked_by_role='customer'` 且在本周/当日的数量，超限则返回空集合。
7. 返回全部时间槽，并在被配额或预约冲突挡住时写入 `reason` 字段（例如“客户配额已满”“已被预约”），供前端灰显显示。

### 4.2 客户预约 (`POST /appointments`)
1. 校验 JWT，确认 `role = customer`。
2. 验证 `patient_id` 归属当前账户。
3. 调用与可用时间相同的校验器，强制串联并发/限额逻辑。
4. 写入 `Appointments`，`booked_by_role='customer'`，`price_at_booking=offer.price`，`end_time = start_time + duration`。

### 4.3 管理员预约 (`POST /admin/appointments`)
1. 允许 `role in {admin, technician}`。
2. 跳过“父亲限额”校验，但保留时间冲突检测。
3. 可覆盖 `price`, `duration`, `notes`，并写入 `booked_by_role='admin'` 以供审计。

## 5. 代码约定
- **Schema**：输入使用 `schemas.py`，输出统一 `ResponseModel`，并在路由层调用 `from_orm`。
- **数据库事务**：使用 `async_sessionmaker` + `async with session.begin()`；可用时间查询可落在 `READ COMMITTED` 隔离级别。
- **异常处理**：自定义 `BusinessLogicError`，统一转为 422/409，便于前端提示。
- **主键生成**：所有表的 PK 均使用 **ULID**；建议通过 `ulid-py` 或等价库在服务层生成，再写入数据库，保证排序与分布式唯一性。
- **权限**：在 `core/deps.py` 定义 `get_current_user`, `require_admin`, `require_customer` 依赖，所有路由显式声明。

## 6. 测试与质量
- **单元测试**：针对并发计算、预约创建、管理员绕过逻辑编写 pytest。
- **集成测试**：利用 FastAPI TestClient + SQLite (或 PostgreSQL 容器) 复现全流程。
- **监控**：建议在 Phase 3 的 `OPS_AND_TEST.md` 中记录日志格式、Prometheus 指标（请求耗时、预约成功率）。

## 7. 与其他文档的链接
- API 协议详见 `docs/API.md`。
- 表结构与字段约束详见 `docs/DB.md`。
- 前端路由/状态需求详见 `docs/FRONTEND.md`。
