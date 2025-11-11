# 数据库文档

## 1. 设计原则
- **范式**：采用高度规范化结构，确保预约、定价、排班逻辑各自独立。
- **时间一致性**：所有时间字段采用 `TIMESTAMP WITH TIME ZONE`（PostgreSQL）或 UTC 存储，前端再转时区。
- **主键策略**：所有主键统一使用 **ULID**（可排序、便于分布式生成），在数据库中以 `CHAR(26)` 或 `VARCHAR(26)` 存储。
- **审计性**：预约记录保留 `booked_by_role`、`price_at_booking`，避免管理员操作引发歧义。

## 2. 实体关系概览
```
Users (1) ──< Patients
Users (1) ──< Technicians
Technicians (1) ──< Offerings >── (1) Services
Offerings (N) ── BusinessHours (按 technician+location)
Offerings (1) ──< Appointments >── (1) Patients
```
> `Offerings` 将技师、服务、地点三者绑定，是价格与时长的唯一来源；预约只引用 `offering_id`。

## 3. 表定义

### 3.1 用户域
| 表 | 关键字段 | 备注 |
| --- | --- | --- |
| `Users` | `user_id (ULID, PK)`, `wechat_openid (Unique)`, `role (Enum)`, `default_location_id (FK Locations)` | 记录客户最近预约的地点，进入预约页时作为默认值 |
| `Patients` | `patient_id (ULID, PK)`, `managed_by_user_id (FK Users)` | 允许同一账户管理多位就诊人 |
| `Technicians` | `technician_id (ULID, PK)`, `user_id (FK Users)`, `display_name`, `is_active` | `user_id` 可为空（便于仅供展示） |

### 3.2 服务目录域
| 表 | 关键字段 | 备注 |
| --- | --- | --- |
| `Locations` | `location_id (ULID, PK)`, `name`, `address` | 可扩展城市或门店 |
| `Services` | `service_id (ULID, PK)`, `name`, `default_duration`, `concurrency_level`, `weight` | `weight` 越大，前端默认展示越靠前，默认 0 |
| `Offerings` | `offering_id (ULID, PK)`, `technician_id`, `service_id`, `location_id`, `price`, `duration`, `is_available` | duration 可覆盖 `default_duration` |

### 3.3 排班域
| 表 | 关键字段 | 备注 |
| --- | --- | --- |
| `BusinessHours` | `rule_id (ULID, PK)`, `technician_id`, `location_id`, `day_of_week (monday~sunday)`, `start_time`, `end_time` | 周期性排班规则 |

### 3.4 预约域
| 表 | 关键字段 | 备注 |
| --- | --- | --- |
| `Appointments` | `appointment_id (ULID, PK)`, `patient_id`, `booked_by_user_id`, `offering_id`, `start_time`, `end_time`, `status`, `booked_by_role`, `price_at_booking`, `notes` | `status` 枚举：`scheduled/completed/no_show` |

## 4. 约束与索引建议
- `Users.wechat_openid`：唯一索引，保证单账号绑定。
- `Patients (managed_by_user_id, name)`：唯一索引，避免重复就诊人。
- `Offerings`：唯一索引 `(technician_id, service_id, location_id)`，确保同一组合只有一条定价策略；可增加 `is_available` 过滤索引。
- `BusinessHours`：唯一索引 `(technician_id, location_id, day_of_week, start_time)`，其中 `day_of_week` 直接存储 `monday`~`sunday`。
- `Appointments`：组合索引 `(technician_id via offering, start_time)` 以加速可用时间查询；在实现层可通过物化视图/缓存。

## 5. 关键约束逻辑
1. **父亲预约限制**：可在应用层依据 `Technicians.display_name` 或单独字段标记 `restricted_by_customer_quota`，并在查询 `Appointments` 时统计。
2. **并发控制**：基于 `Services.concurrency_level`，可在数据库中为 Appointments 添加 `EXCLUDE USING GIST` 约束（Postgres）避免锁死服务的时间重叠。
3. **价格锁定**：`price_at_booking` 必须 NOT NULL，并默认 `Offerings.price`。

## 6. 迁移与初始化
- 使用 Alembic/Soda 等工具管理迁移脚本；初始 schema 可放入 `/backend/db/versions`.
- 初始数据：
  - 管理员账号（role=admin）
  - 两位技师（含父亲标记）
  - 核心地点/服务/offerings
  - 基础 BusinessHours

## 7. 数据治理
- 每日导出 `Appointments` 作为备份，可直接基于 `start_time` 分区表（Postgres）。
- 敏感信息（手机号）需加密或至少脱敏。
- 日志表/审计表可在 Phase 3 评估后另建，避免影响 MVP。

## 8. 文档关联
- 业务流程：`docs/PROJECT_PLAN.md`
- 后端逻辑：`docs/BACKEND.md`
- API 字段映射：`docs/API.md`
