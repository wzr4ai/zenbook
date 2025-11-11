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

## 2. 客户端接口

### 2.1 用户与就诊人 `/users`
| 方法 | 路径 | 权限 | 功能 |
| --- | --- | --- | --- |
| GET | `/users/me` | customer | 获取登录账户信息 |
| PATCH | `/users/me` | customer | 更新昵称等资料 |
| GET | `/users/patients` | customer | 列出当前账户的就诊人 |
| POST | `/users/patients` | customer | 新增就诊人 |
| PUT | `/users/patients/{id}` | customer | 编辑就诊人 |
| DELETE | `/users/patients/{id}` | customer | 删除就诊人 |

### 2.2 服务目录 `/catalog`（公开）
| 方法 | 路径 | 描述 |
| --- | --- | --- |
| GET | `/catalog/locations` | 地点列表 |
| GET | `/catalog/technicians` | 技师列表（含简介、头像） |
| GET | `/catalog/services` | 服务列表（含默认时长/并发） |
| GET | `/catalog/offerings` | 技师+服务+地点组合及价格/时长 |

### 2.3 排班与预约
#### `GET /schedule/availability`
- **权限**：公开（前端仍需传 token 以便统计）
- **查询参数**：
  - `date` (YYYY-MM-DD)
  - `technician_id`
  - `service_id`
  - `location_id`
- **响应**：`[{ "start": "2024-05-20T08:30:00+08:00", "end": "2024-05-20T09:30:00+08:00", "reason": null }]`

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

#### `GET /appointments/me`
- 返回当前账户的预约（支持 `status` 过滤）。

#### `POST /appointments/{id}/cancel`
- 将预约状态置为 `cancelled`，仅允许预约所属账户操作。

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
| PUT | `/appointments/{id}` | 修改预约（时间、状态、备注） |
| DELETE | `/appointments/{id}` | 删除预约 |

### 3.4 用户与就诊人
- `GET /users`：查看全部账户
- `GET /patients`：查看全部就诊人

## 4. 错误码约定
| 状态码 | 场景 | 说明 |
| --- | --- | --- |
| 400 | 参数错误 | 例如缺少 `offering_id` |
| 401 | 未登录或 token 失效 | 前端需触发重新登录 |
| 403 | 权限不足 | 客户访问管理接口 |
| 409 | 业务冲突 | 时间槽被占用、父亲限额已满 |
| 422 | 语义校验失败 | 就诊人与账户不匹配 |

## 5. 版本与扩展
- 未来迭代可将客户端接口迁移至 `/api/v2`，新增队列/排班 AI 推荐等功能。
- 若要开放第三方对接，可在 `/api/v1/webhooks` 增加预约变更推送。

## 6. 文档关联
- 业务背景：`docs/PROJECT_PLAN.md`
- 后端实现：`docs/BACKEND.md`
- 前端调用说明：`docs/FRONTEND.md`
- 数据模型：`docs/DB.md`
