# ZenBook 🧘‍♀️

> **Simplicity for your customers, power for your operations.**
> 把清爽留给客户，把复杂留给 ZenBook。

ZenBook 是一个现代化的开源预约调度系统，专为需要精细化管理资源和服务行业的场景设计（如养生馆、心理咨询、高端沙龙等）。它不仅是一个简单的日历预约工具，更是一个能处理复杂现实业务规则的调度引擎。

### ✨ 核心特性 (Core Features)

* **灵活排班引擎**：支持常规营业时间与临时例外（Exception）的高优先级覆盖。
* **复杂并发控制**：基于服务的并发度设置（如某些服务独占技师，某些服务可同时进行）。
* **特殊额度管理**：独特的基于身份的预约限制功能（解决如“熟人/特定角色”每日预约上限等复杂业务痛点）。
* **双端一体化**：一套系统同时支撑客户自助预约端与管理端后台。

### 🏗️ 项目架构 (Architecture)

ZenBook 采用前后端分离架构，由以下核心子项目组成：

| 模块 | 仓库 | 技术栈 | 说明 |
| :--- | :--- | :--- | :--- |
| **Core API** | [**zenbook-backend**](./link-to-backend) | FastAPI, PostgreSQL, Redis | 核心调度算法、数据管理与 API 服务。 |
| **WeApp** | [**zenbook-weapp**](./link-to-weapp) | Uni-App (Vue3), WeChat MP | 微信小程序客户端，包含客户预约与管理员功能。 |
| *(Future)* | *zenbook-admin-web* | *(Vue3/React)* | *(规划中) 基于 Web 的专业管理后台。* |

### 🚀 快速开始

请访问上述子仓库查看具体的部署和开发指南。

### 📄 License

MIT License
