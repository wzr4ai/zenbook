# 前端文档 (Uni-App + Vue 3)

## 1. 技术栈与规范
- **框架**：Uni-App（基于 Vue 3 Composition API，`<script setup>`）。
- **状态管理**：Pinia；`store/user` 与 `store/booking` 是强制依赖。
- **UI**：可选 uView/NutUI，确保样式支持微信小程序端。
- **HTTP**：统一在 `api/request.js` 封装 `uni.request`，注入 `Authorization: Bearer token`。
- **开发/调试工具**：统一使用 **HBuilderX CLI**（如 `hbx-cli dev` / `hbx-cli build mp-weixin`）进行本地调试与构建，确保与 CI 构建产物一致。

## 2. 目录结构
```
/frontend
├─ pages/          # 客户主包（预约核心链路）
│  ├─ index/       # F-06 入口（地点/技师/服务选择）
│  ├─ booking/     # F-07 可用时间日历
│  └─ confirm/     # F-08 预约确认（就诊人、备注）
├─ pages_sub/      # 客户分包（低频）
│  ├─ profile/     # F-01 登录/资料
│  ├─ patients/    # F-02 就诊人管理
│  ├─ appointments/# F-09 预约列表 + 详情 (含 F-10 取消)
│  └─ appointment_detail/
├─ pages_admin/    # 管理分包（B-01 ~ B-12）
│  ├─ dashboard/   # 日历看板，入口指向其它管理页面
│  ├─ appt_create/ # 手动预约
│  ├─ schedule_mgmt/
│  ├─ catalog_mgmt/
│  └─ user_mgmt/
├─ api/            # 按模块拆分的 API 调用
├─ components/     # 日历、时间槽、表单等复用组件
├─ store/          # Pinia
├─ static/
├─ main.js
├─ App.vue
├─ manifest.json
└─ pages.json
```

## 3. 状态与权限控制
### 3.1 User Store
```ts
// store/user.ts
export const useUserStore = defineStore('user', {
  state: () => ({ token: '', userInfo: null, impersonateRole: '' }),
  getters: {
    viewRole: (state) => state.impersonateRole || state.userInfo?.role || 'customer',
    hasStaffAccess: (state) => ['admin','technician'].includes(state.userInfo?.role ?? ''),
  },
  actions: {
    async login() {
      const { code } = await uni.login();
      const data = await authApi.login({ code });
      this.token = data.token;
      this.userInfo = data.userInfo; // 必须包含 role
      this.impersonateRole = '';
    },
    setImpersonateRole(role: string) {
      if (this.userInfo?.role === 'admin') this.impersonateRole = role;
    },
    logout() { this.$reset(); },
  },
});
```

### 3.2 Booking Store
- 暂存 `selectedLocation`, `selectedTechnician`, `selectedService`, `selectedDate`, `selectedSlot`, `selectedPatientId`。
- 将 `offerings` 与 `availability` 缓存在 store，便于确认页复用。

### 3.3 路由守卫
- `pages_admin` 中的所有页面路径注册到 `adminPages` 列表。
- 在 `main.js` 中使用 `uni.addInterceptor('navigateTo', ...)` 判断 `userStore.hasStaffAccess`（真实权限），未授权时阻断并 toast。
- Tab 第二页 `pages/me` 中提供管理员身份切换按钮，仅影响前端展示，后端权限仍依赖真实角色。

## 4. 核心页面交互
| 页面 | 功能点 | 说明 |
| --- | --- | --- |
| `pages/index` | 选择地点/技师/服务 或 管理面板 | 客户视角调用 `catalog` API；管理员/技师视角展示预约管理快捷入口 |
| `pages/me` | 个人中心 | “我的”Tab：客户展示资料与常用入口，管理员可切换前端身份 |
| `pages/booking` | 可用时间日历 | 调用 `scheduleApi.getAvailability`，展示时间槽；点击后设置 `bookingStore.selectedSlot` |
| `pages/confirm` | 预约确认 | 拉取 `patients`，允许备注；提交 `appointmentsApi.create` |
| `pages_sub/appointments` | 我的预约 | 展示 `scheduled/completed`，支持下拉刷新 |
| `pages_sub/appointment_detail` | 取消预约 | 调用 `appointmentsApi.cancel` |
| `pages_admin/dashboard` | 全局预约视图 | 显示所有技师、地点；提供快速入口到 `appt_create` |
| `pages_admin/catalog_mgmt` | CRUD 服务/技师/地点/offerings | 与后端 `/api/v1/admin/catalog/...` 对应 |
| `pages_admin/schedule_mgmt` | 批量排班 | 管理员按未来 7 天批量创建/编辑班次 |

## 5. API 封装
- `api/request.js`：
  - 设置 `BASE_URL`。
  - 请求拦截：若存在 token 则注入 header。
  - 响应拦截：401 -> 清空 store + 跳转登录；403 -> toast 无权限。
- 模块文件（如 `api/catalog.js`）只暴露语义化方法，便于 tree-shaking。

## 6. 组件建议
- `components/calendar`：客户与管理员可共用基础骨架，通过 props 控制多选、并发标记。
- `components/time-slot`：负责展示并发/禁用状态；接收 `{ time, disabled, reason }`。
- `components/booking-card`：管理员看板复用。

## 7. 开发与测试注意事项
- 使用 `pinia-plugin-persistedstate` 缓存 token & userInfo。
- 在 `pages.json` 中将管理功能放进单独分包，减少客户首屏包体。
- 小程序端需开启“分包异步化”以避免管理端代码阻塞启动。
- 本地调试统一通过 `HBuilderX CLI`，可在 `package.json` 增加 `scripts` 以调用 `hbx-cli`，便于自动化测试、CI 和联调环境保持一致。
- 若需在非微信容器中调试登录，可在 `.env.local` 中设置 `VITE_DEV_LOGIN_CODE`，配合后端的 WeChat 模拟服务即可获取稳定的 mock openid。
- UI/交互测试：建议在 `OPS_AND_TEST.md` 中记录关键路径（登录→预约→取消、管理员手动预约等）。

## 8. 与其他文档的映射
- API 契约：`docs/API.md`
- 业务逻辑/并发规则：`docs/BACKEND.md`
- 数据结构：`docs/DB.md`
