# 运维与测试文档 (OPS & TEST)

本文件定义预约系统在不同阶段的部署、配置、监控与测试策略，覆盖后端 (FastAPI)、数据库、前端小程序三部分。

## 1. 环境矩阵
| 环境 | 目的 | 基础设施 | 数据 | 访问策略 |
| --- | --- | --- | --- | --- |
| 本地 Dev | 日常开发、单元/集成测试 | Docker Compose：PostgreSQL、FastAPI、模拟微信 API | 伪造/匿名化数据 | 仅开发者，使用 `.env.dev` |
| Staging | 回归测试、演示 | 云服务器 (与生产等价配置) + 独立 DB | 定期从生产脱敏恢复 | 管理员账号受白名单控制 |
| Production | 正式服务 | 云服务器/容器编排 (Kubernetes/ECI) + 托管 PostgreSQL | 实际用户数据 | 通过 VPN 或 VPC 访问管理端 |

## 2. 配置与环境变量
| 变量 | 描述 | 建议管理方式 |
| --- | --- | --- |
| `DATABASE_URL` | PostgreSQL 连接串 (含连接池参数) | 使用密钥管理服务 (KMS) 注入 |
| `JWT_SECRET`, `JWT_EXPIRES_IN` | JWT 签发密钥与过期时间 | 分环境生成，禁止硬编码 |
| `WECHAT_APPID`, `WECHAT_SECRET` | 微信登录凭证 | 使用小程序更多环境 (体验/生产) |
| `DEFAULT_TIMEZONE` | 预约计算时区 (`Asia/Shanghai`) | 仅当跨区域部署时修改 |
| `LOG_LEVEL`, `SENTRY_DSN` | 日志与告警 | 生产默认为 `INFO` |
| `RATE_LIMIT_CONFIG` | 可选：接口频率限制策略 | 生产必填，防止刷号 |

> 所有敏感配置写入 `config/{env}.env` 并通过 CI/CD 注入容器环境变量。

## 3. 部署流程
### 3.1 后端 & 数据库
1. **预检**：`pytest`, `ruff`, `mypy` 全通过；生成 `requirements.txt` 锁定版本。
2. **构建镜像**：`docker build -t registry/app-backend:$TAG .`，利用多阶段构建减小镜像。
3. **数据库迁移**：在部署前执行 `alembic upgrade head`；若失败立即回滚。
4. **发布**：
   - Dev/Staging：CI 自动滚动更新。
   - Production：分批滚动 (50%→100%)，期间监控 5xx 指标。
5. **健康检查**：FastAPI 提供 `/healthz` 返回 DB 连接状态；K8s/容器平台据此判定存活。

### 3.2 前端 (Uni-App 小程序)
1. 使用 **HBuilderX CLI** (`hbx-cli build mp-weixin --minimize`) 生成 `dist/build/mp-weixin`，同一 CLI 也用于日常 `hbx-cli dev` 调试以保证构建一致性。
2. 通过微信开发者工具 CLI 上传体验版，并附带 Git 提交信息。
3. 测试通过后，管理员在微信后台提交审核；审核通过后指定上线时间窗口。

### 3.3 配置变更
- 所有配置更新需在 `config/CHANGELOG.md` 记录“变更项/环境/负责人”。
- 重大配置（限流、预约限额）先在 Staging 验证 24h，再同步到生产。

## 4. 监控与日志
### 4.1 日志
- 统一使用 JSON 格式：`timestamp`, `level`, `trace_id`, `user_id`, `role`, `action`, `duration_ms`。
- 结构化日志写入 stdout，由容器平台收集到 ELK / Loki；敏感字段脱敏。

### 4.2 指标
| 指标 | 说明 | 采集方式 | 告警阈值 |
| --- | --- | --- | --- |
| `http_request_duration_seconds` | API 耗时 | Prometheus client in FastAPI | P95 > 800ms 持续 5 分钟告警 |
| `http_requests_total{status}` | 状态码分布 | 同上 | 5xx 比例 > 2% |
| `appointments_created_total` | 成功预约数 | 自定义 Counter | 低于日均 50% 触发预警 |
| `availability_cache_miss_total` | 可用时间缓存 MISS | 自定义 Counter | MISS 率 > 80% 需排查缓存 |

### 4.3 告警渠道
- 严重 (S1)：微信企业号 + 电话
- 一般 (S2)：企业微信群机器人
- 信息 (S3)：日报邮件

## 5. 备份与回滚
### 5.1 数据库
- **备份策略**：每日 02:00 全量备份 + 每 15 分钟 WAL 增量，保存 30 天。
- **恢复演练**：每季度在 Staging 进行一次全量恢复，验证脚本。

### 5.2 应用
- **后端**：保留最近 3 个稳定镜像，出现问题时通过 `kubectl rollout undo` 或重新部署上一版本。
- **前端**：若线上版本异常，立即将体验版回滚提交为新版本，或在微信后台切回上一稳定版本。

### 5.3 回滚流程
1. 触发告警并确认影响面。
2. 冻结新流量（关闭 CI 自动部署）。
3. 视情况回滚 DB 迁移（执行 `alembic downgrade` 或恢复备份）。
4. 回滚应用镜像/小程序版本。
5. 填写事故报告，更新 `OPS_AND_TEST.md` 附录。

## 6. 测试策略
| 层级 | 工具 | 覆盖内容 |
| --- | --- | --- |
| 单元测试 | pytest + coverage | 可用时间计算、限额校验、API 验证逻辑 |
| 集成测试 | FastAPI TestClient + 临时 DB | 模块间交互、JWT 权限、预约生命周期 |
| 合同测试 | Schemathesis / OpenAPI | 确保 API 与 `docs/API.md` 一致 |
| 前端组件测试 | Vitest + Vue Test Utils | Pinia store、日历/时间槽组件 |
| 端到端 (E2E) | uni-app 自动化 (或 wdio) + Mock WeChat | 登录→选择服务→下单→客户删除、管理员手动预约、排班调整 |

### 6.1 关键 E2E 用例
| 用例 ID | 场景 | 检查点 |
| --- | --- | --- |
| E2E-01 | 客户登录并绑定顾客 | Token 写入、Pinia 持久化、顾客列表更新 |
| E2E-02 | 查询可用时间并下单 | 可用时间与后端响应一致、预约成功提示 |
| E2E-03 | 客户删除预约 | 开始前允许删除（记录移除），可用时间回补 |
| E2E-04 | 管理员创建预约 | `booked_by_role='admin'`，父亲限额不受影响 |
| E2E-05 | 滚动批量排班 | 基于未来 7 天批量添加班次并在列表中可见 |

## 7. 发布前检查清单
1. CI 绿灯：单元/集成测试覆盖率 ≥ 80%，Lint 通过。
2. 数据库迁移已在 Staging 成功执行并验证。
3. 日志/监控配置（Sentry DSN、Prometheus）在新环境已校验。
4. 备份作业成功运行并有最近一次可用备份。
5. 微信体验版测试结果记录在 Issue/任务单中并获签字。
6. 发布窗口内有值班人员和回滚方案。

## 8. 文档关联
- 业务规划：`docs/PROJECT_PLAN.md`
- 后端实现：`docs/BACKEND.md`
- 前端规范：`docs/FRONTEND.md`
- 数据模型：`docs/DB.md`
- API 契约：`docs/API.md`
