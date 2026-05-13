# API 接口参考手册

## 1. 接口分布

| 类型 | 数量 | 说明 |
|------|------|------|
| NestJS 后台 API | ~50 个 | 管理后台调用，RESTful 风格 |
| n8n Webhook 接口 | ~244 个 | 产线设备调用，Webhook 触发 |
| **合计** | **~294 个** | - |

---

## 2. NestJS 后台 API

### 2.1 产品管理（/admin/product）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/product/list` | 产品列表查询 | Prisma ✅ |
| GET | `/admin/product/:id` | 产品详情 | Prisma ✅ |
| POST | `/admin/product` | 创建产品 | Prisma ✅ |
| PUT | `/admin/product/:id` | 更新产品 | Prisma ✅ |
| DELETE | `/admin/product/:id` | 删除产品 | Prisma ✅ |
| GET | `/admin/product/model/list` | 型号列表 | Prisma ✅ |
| POST | `/admin/product/model` | 创建型号 | Prisma ✅ |

### 2.2 工序管理（/admin/operation）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/operation/list` | 工序列表 | Prisma ✅ |
| GET | `/admin/operation/:id` | 工序详情 | Prisma ✅ |
| POST | `/admin/operation` | 创建工序 | Prisma ✅ |
| PUT | `/admin/operation/:id` | 更新工序 | Prisma ✅ |
| DELETE | `/admin/operation/:id` | 删除工序 | Prisma ✅ |
| GET | `/admin/operation/process/list` | 工艺流程列表 | Prisma ✅ |

### 2.3 工艺配置（/admin/config）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/config/list` | 工艺配置列表 | Prisma ✅ |
| GET | `/admin/config/:id` | 工艺配置详情 | Prisma ✅ |
| POST | `/admin/config` | 创建工艺配置 | Prisma ✅ |
| PUT | `/admin/config/:id` | 更新工艺配置 | Prisma ✅ |
| GET | `/admin/config/version/list` | 工艺版本列表 | Prisma ✅ |

### 2.4 工单管理（/admin/ticket）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/ticket/list` | 工单列表 | n8n（待迁移） |
| POST | `/admin/ticket` | 创建工单 | n8n（待迁移） |
| GET | `/admin/ticket/:id` | 工单详情 | n8n（待迁移） |

### 2.5 SN 管理（/admin/sn）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/sn/query` | SN 查询 | Prisma ✅ |

### 2.6 数据查询（/admin/data-query）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| GET | `/admin/data-query/production` | 生产数据查询 | Prisma ✅ |
| GET | `/admin/data-query/quality` | 质量数据查询 | Prisma ✅ |
| GET | `/admin/data-query/delivery` | 出库数据查询 | n8n（待迁移） |
| GET | `/admin/data-query/device` | 设备数据查询 | n8n（待迁移） |
| GET | `/admin/data-query/report` | 报表数据 | n8n（待迁移） |
| GET | `/admin/data-query/export` | 数据导出 | n8n（待迁移） |

### 2.7 系统管理（/system）

| 方法 | 路径 | 说明 | 实现 |
|------|------|------|------|
| POST | `/system/login` | 登录 | Prisma ✅ |
| GET | `/system/user/list` | 用户列表 | Prisma ✅ |
| POST | `/system/user` | 创建用户 | Prisma ✅ |
| PUT | `/system/user/:id` | 更新用户 | Prisma ✅ |
| GET | `/system/role/list` | 角色列表 | Prisma ✅ |
| GET | `/system/menu/list` | 菜单列表 | Prisma ✅ |
| GET | `/system/dept/list` | 部门列表 | Prisma ✅ |
| GET | `/system/datalog/list` | 操作日志 | Prisma ✅ |

---

## 3. n8n Webhook 接口

### 3.1 绑定（6 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/sn/bind` | SN 绑定 | 写：retroid, machine_data |
| `/webhook/sn/batch-bind` | 批量绑定 | 写：retroid |
| `/webhook/sn/bind/check` | 绑定校验 | 读：retroid |
| `/webhook/component/bind` | 组件绑定 | 写：machine_data |
| `/webhook/bind/verify` | 绑定验证 | 读：retroid, machine_data |
| `/webhook/bind/query` | 绑定查询 | 读：retroid |

### 3.2 解绑（1 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/sn/unbind` | SN 解绑 | 写：machine_data, retroid |

### 3.3 SN 管理（7 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/sn/generate` | SN 生成 | 写：retroid, retroid_generate |
| `/webhook/sn/query` | SN 查询 | 读：retroid, machine_data |
| `/webhook/sn/batch-query` | 批量查询 | 读：retroid |
| `/webhook/sn/status` | 状态更新 | 写：retroid |
| `/webhook/sn/check` | SN 校验 | 读：retroid |
| `/webhook/sn/progress` | 工序进度 | 读：machine_data |
| `/webhook/sn/history` | SN 历史记录 | 读：machine_data |

### 3.4 工单（4 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/joborder/create` | 工单创建 | 写：joborder |
| `/webhook/joborder/close` | 工单关闭 | 写：joborder |
| `/webhook/joborder/query` | 工单查询 | 读：joborder |
| `/webhook/joborder/status` | 状态更新 | 写：joborder |

### 3.5 维修（3 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/repair/checkin` | 维修进站 | 写：retroid, repair_flow_record |
| `/webhook/repair/message` | 维修信息 | 写：repair_flow_record |
| `/webhook/repair/checkout` | 维修出站 | 写：retroid, repair_flow_record |

### 3.6 校验（1 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/procedure/validate` | 工序校验 | 读：retroid, procedure_detail |

### 3.7 包装（2 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/pack/start` | 开始包装 | 读：retroid |
| `/webhook/pack/complete` | 完成包装 | 写：pack_detail, retroid |

### 3.8 质检（4 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/fqc/inspect` | FQC 检验 | 写：defective_records |
| `/webhook/oqc/inspect` | OQC 检验 | 写：defective_records |
| `/webhook/quality/record` | 不良记录 | 写：defective_records |
| `/webhook/quality/report` | 质量报表 | 读：defective_records |

### 3.9 数据供给（1 个）

| 路径 | 说明 | 读写表 |
|------|------|--------|
| `/webhook/sync/to-coros` | 同步到 COROS | 读：daily_info |

---

## 4. F3/F4/F5 新增接口

| 接口 | 实现 | 路径 | 说明 | 读写表 |
|------|------|------|------|--------|
| `repair_checkin_api` | n8n | `/webhook/repair/checkin` | 维修进站 | 写：retroid(status→4), repair_flow_record |
| `repair_message` | n8n | `/webhook/repair/message` | 维修信息录入 | 写：repair_flow_record |
| `sn_query_api` | NestJS | `/admin/sn/query` | SN 查询（含维修状态） | 读：retroid, repair_flow_record |
| `sn_void_api` | NestJS | `/admin/sn/void` | SN 作废 | 写：retroid(status→3), void_record |
| `sn_void_precheck_api` | NestJS | `/admin/sn/void/precheck` | 作废前置校验 | 读：retroid, machine_data |
| `sn_replenish_api` | NestJS | `/admin/sn/replenish` | 补料 | 写：retroid, retroid_generate, replenish_record |
| `replenish_precheck_api` | NestJS | `/admin/sn/replenish/precheck` | 补料前置校验 | 读：joborder, void_record |
| `sn_export_api` | NestJS | `/admin/sn/export` | SN 数据导出 | 读：retroid, machine_data |

---

## 5. 迁移状态速查

### 标记说明

| 标记 | 含义 |
|------|------|
| **Prisma ✅** | 已迁移到 NestJS + Prisma，使用 ORM 操作 |
| **n8n** | 仍在 n8n 中实现，通过 Webhook 调用 |
| **n8n（待迁移）** | 规划中，计划迁移到 NestJS + Prisma |

### 迁移优先级建议

| 优先级 | 模块 | 原因 |
|--------|------|------|
| P0 | 已迁移模块 | 保持现状，持续维护 |
| P1 | 工单管理（3 个） | 后台 CRUD，适合迁移 |
| P2 | 数据查询（4 个待迁移） | 查询复杂度高，Prisma 更优 |
| P3 | 产线过站 | 不建议迁移，留在 n8n |
