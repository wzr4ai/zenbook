# API 文档 (v1)

> 所有接口默认路径前缀 `/api/v1`，返回 JSON。采用 JWT Bearer 鉴权：
> - `customer`：访问客户端接口。
> - `admin` / `technician`：可访问管理端接口。
> - 所有 `id` 字段统一使用 **ULID**（示例：`01HPCTSZ1A2B3C4D5E6FGHJKL8`），便于前后端比对。

## 1. 认证模块 `/auth`
### `POST /auth/login`
- **描述**：前端提交微信 `code`，后端换取 `openid` 并签发 JWT。
- **请求体**：`{ "code": "wx123" }`
- **响应**：
```json
{
  "token": "jwt",
  "userInfo": { "id": "01HPCTSZ1A2B3C4D5E6FGHJKL8", "name": "魏师傅", "role": "admin" }
}
```

### `POST /auth/sms`
- **描述**：H5 客户端请求发送一次性验证码，用于后续手机号登录，默认 5 分钟有效（当前仅模拟发送）。
- **请求体**：`{ "phone": "+8613711112222" }`
- **响应**：`{ "status": "sent" }`

### `POST /auth/login/phone`
- **描述**：H5 Web 端提供手机号登录（必须附带短信验证码）并复用相同的 JWT 响应模板。
- **请求体**：`{ "phone": "+8613711112222", "code": "123456" }`
- **响应**：与 `/auth/login` 相同，`userInfo` 会包含 `phone` 字段以便前端展示。

## 2. 客户端接口

### 2.1 用户与顾客 `/users`
| 方法 | 路径 | 权限 | 功能 |
| --- | --- | --- | --- |
| GET | `/users/me` | customer | 获取登录账户信息 |
| PATCH | `/users/me` | customer | 更新昵称等资料 |
| GET | `/users/patients` | customer | 列出当前账户的顾客 |
| POST | `/users/patients` | customer | 新增顾客 |
| PUT | `/users/patients/{id}` | customer | 编辑顾客 |
| DELETE | `/users/patients/{id}` | customer | 删除顾客 |

> `/users/me` 额外返回 `default_location_id` 字段，用于记录客户最近一次完成预约时的地点，前端可据此预填城市/门店。

### 2.2 服务目录 `/catalog`（公开）
| 方法 | 路径 | 描述 |
| --- | --- | --- |
| GET | `/catalog/locations` | 地点列表 |
| GET | `/catalog/technicians` | 技师列表（含简介、头像） |
| GET | `/catalog/services` | 服务列表（含默认时长/并发/权重） |
| GET | `/catalog/offerings` | 技师+服务+地点组合及价格/时长 |

> 服务对象新增 `weight` 字段，数值越大越靠前，前端会在无选择时默认取权重最高的服务。

### 2.3 排班与预约
#### `GET /schedule/availability`
- **权限**：公开（前端仍需传 token 以便统计）
- **查询参数**：
  - `date` (YYYY-MM-DD)
  - `technician_id`
  - `service_id`
  - `location_id`
- **响应**：
```json
[
  { "start": "2024-05-20T08:30:00+08:00", "end": "2024-05-20T09:30:00+08:00", "reason": null },
  { "start": "2024-05-20T09:30:00+08:00", "end": "2024-05-20T10:30:00+08:00", "reason": "已被预约" }
]
```
> `reason` 非空表示该时间段不可预约（配额已满或已被占用），前端需灰显并禁止点击。

#### `POST /appointments`
- **权限**：customer
- **请求体**：
```json
{
  "offering_id": "01HPD4XZ6MPTC0N8R1S2T3UVWY",
  "patient_id": "01HPD4XZ6MPTC0N8R1S2T3UVWX",
  "start_time": "2024-05-20T08:30:00+08:00",
  "notes": "可选"
}
```
- **响应**：`{ "appointment_id": "01HPD4XZ6MPTC0N8R1S2T3UVWX", "status": "scheduled" }`
- **状态含义**：
  - `scheduled`：待服务。
  - `completed`：管理员/技师在预约开始后标记完成。
  - `no_show`：预约开始后由管理员/技师标记违约（爽约）。
- **地点偏好**：若本次预约的地点与账户 `default_location_id` 不一致，创建成功后系统会自动写入新的地点，供下次进入预约页时默认选中。

#### `GET /appointments/me`
- 返回当前账户的预约（支持 `status` 过滤）。

#### `DELETE /appointments/{id}`
- **权限**：customer
- **说明**：预约开始前客户可自行删除记录；一旦到达预约时间则返回 `403`，需联系管理员处理。

## 3. 管理端接口 `/admin`
> 需 `admin` 或 `technician` 角色；路由建议在 FastAPI 中注册为 `APIRouter(prefix="/api/v1/admin")`。

### 3.1 目录管理 `/catalog`
- `(CRUD) /catalog/locations`
- `(CRUD) /catalog/technicians`
- `(CRUD) /catalog/services`
- `(CRUD) /catalog/offerings`

### 3.2 排班管理 `/schedule`
- `(CRUD) /schedule/business-hours`

### 3.3 预约管理 `/appointments`
| 方法 | 路径 | 功能 |
| --- | --- | --- |
| GET | `/appointments` | 列出全部预约，支持筛选技师/日期/状态 |
| POST | `/appointments` | 管理员手动添加，`booked_by_role='admin'`，可绕过父亲限额 |
| PUT | `/appointments/{id}` | 修改预约（时间、备注、状态）。`status` 仅能在预约开始后更新为 `completed` 或 `no_show` |
| DELETE | `/appointments/{id}` | 删除预约，管理员/技师可在任何时间执行 |

### 3.4 用户与顾客
- `GET /users`：查看全部账户
- `GET /patients`：查看全部顾客

## 4. 错误码约定
| 状态码 | 场景 | 说明 |
| --- | --- | --- |
| 400 | 参数错误 | 例如缺少 `offering_id` |
| 401 | 未登录或 token 失效 | 前端需触发重新登录 |
| 403 | 权限不足 | 客户访问管理接口 |
| 409 | 业务冲突 | 时间槽被占用、父亲限额已满 |
| 422 | 语义校验失败 | 顾客与账户不匹配 |

## 5. 版本与扩展
- 未来迭代可将客户端接口迁移至 `/api/v2`，新增队列/排班 AI 推荐等功能。
- 若要开放第三方对接，可在 `/api/v1/webhooks` 增加预约变更推送。

## 6. 文档关联
- 业务背景：`docs/PROJECT_PLAN.md`
- 后端实现：`docs/BACKEND.md`
- 前端调用说明：`docs/FRONTEND.md`
- 数据模型：`docs/DB.md`
