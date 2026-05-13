# MES 数据流全景图

> 版本：1.0 | 2026-04-15
> 来源：熊旺阐述 + 数据库结构分析 + n8n工作流交叉分析

---

## 🌊 数据流总览

```
                    ┌─────────────────────────────────────────────────────┐
                    │                    数据源头                          │
                    └─────────────────────────────────────────────────────┘
                                      │
          ┌───────────────────────────┼───────────────────────────┐
          ▼                           ▼                           ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│  日本代工厂       │      │  国内生产         │      │  旧MES历史       │
│  SharePoint      │      │  Lowcoder前端    │      │  SQL Server     │
│  CSV文件         │      │  手动录入        │      │  条码绑定        │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│ n8n: CSV上传     │      │ n8n: 发货流程    │      │ n8n: 数据迁移    │
│ kRCqKO9jDbPn09W9 │      │ get_channel等    │      │ 旧→新MES         │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         ▼                         ▼                         ▼
┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
│ mes_jp.          │      │ warehouseDelivery│      │ YFConsumer.      │
│ machine_data     │      │ .public          │      │ consm_           │
│ (日本生产)        │      │ (国内发货)        │      │ barcodebindinfo  │
│                  │      │                  │      │ (旧MES绑定)       │
└────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
         │                         │                         │
         │                         │                         │
         ▼                         ▼                         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        OMS 系统 (MongoDB)                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │ sninventories│  │    bins     │  │ snoutbounds │  │     sns     │ │
│  │ (SN库存)     │  │  (仓库)     │  │ (出库记录)   │  │ (SN主集合)  │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │                    Shopify 生态                                  │  │
│  │  orders → fulfillments → refunds → returns → transactions      │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    mes 主库 (PostgreSQL)                              │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐ │
│  │    mes      │  │   mes_jp    │  │   public    │  │    sync     │ │
│  │  (主表)     │  │ (日本数据)   │  │ (品质应用)   │  │ (同步任务)   │ │
│  └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘ │
│                                                                        │
│  public 核心表:                                                         │
│  - activation_records (243万条) — 设备激活                              │
│  - deactivation_records (9万条) — 激活注销                              │
│  - defective_records (4万条) — 不良记录                                 │
│  - historydata — 历史激活数据                                           │
│                                                                        │
│  sync 核心表:                                                           │
│  - sap_job_info_cache (24万条) — SAP工单缓存                            │
│  - sn_material_code (98万条) — SN物料映射                               │
│  - yf_pcba_data (32万条) — PCBA生产数据                                 │
│  - jobs / task / pending_task — 同步任务管理                            │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────┐
│                  warehouseDelivery (PostgreSQL)                       │
│  ┌─────────────────────────────────────────────────────────────────┐ │
│  │ 核心表:                                                          │ │
│  │ - mes_delivery_order_details (73万条) — MES发货明细               │ │
│  │ - delivery_order_details (1万条) — 发货明细                       │ │
│  │ - delivery_log (57万条) — 发货日志                                │ │
│  │ - delivery_order (4935条) — 发货订单                              │ │
│  │ - delivery_order_caches (45万条) — 发货缓存                       │ │
│  │ - refund (5276条) — 退款记录                                     │ │
│  │ - prototype_trial — 原型试用                                     │ │
│  │ - api_log (9923条) — API日志                                     │ │
│  │ - approval_* — 审批流程                                          │ │
│  │ - suppliers — 供应商管理                                         │ │
│  │ - users/roles — 用户权限                                         │ │
│  └─────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🔄 核心数据流详解

### 1. 日本生产数据流

```
SharePoint (JEMS)
    │
    │ CSV文件包含：
    │ - 机型生产数据
    │ - SN绑定数据
    │
    ▼
n8n: kRCqKO9jDbPn09W9
    │
    │ 操作：
    │ 1. 下载CSV
    │ 2. 解析转换
    │ 3. 插入数据库
    │
    ▼
mes_jp.machine_data
    │
    │ 字段：
    │ - retroid (设备ID)
    │ - data (JSON生产数据)
    │ - job_number (工单号)
    │ - createtime / uploadtime
    │
    ▼
OMS 拉取
    │
    │ 操作：
    │ 1. 读取 mes_jp.machine_data
    │ 2. 转换格式
    │ 3. 写入 sninventories
    │
    ▼
oms-service.sninventories
    │
    │ 字段：
    │ - sn (序列号)
    │ - material (物料号)
    │ - workOrderNo (工单号)
    │ - status (库存状态)
    │ - bin (仓库位置)
```

### 2. 国内发货数据流

```
Lowcoder 前端页面
    │
    │ 用户操作：
    │ - 录入发货信息
    │ - 扫描SN
    │ - 选择渠道
    │
    ▼
warehouseDelivery.public
    │
    │ 核心表：
    │ - mes_delivery_order_details (发货明细)
    │ - delivery_order_details (订单明细)
    │ - delivery_log (发货日志)
    │
    ▼
n8n: get_channel
    │
    │ 操作：
    │ 1. 从 warehouseDelivery 拉取发货数据
    │ 2. 从 public (mes主库) 拉取国内发货数据
    │ 3. 从 OMS 拉取库存数据
    │ 4. 合并渠道信息
    │
    ▼
软件部数据接口
    │
    │ 供给：
    │ - mes_daily_data (每日同步)
    │ - 生产数据接口
```

### 3. SN 全生命周期追踪

```
生产阶段                    发货阶段                    售后阶段
    │                          │                          │
    ▼                          ▼                          ▼
旧MES:                    warehouseDelivery:          public:
consm_barcodebindinfo     mes_delivery_order_details  activation_records
    │                          │                          │
    │ retroid                  │ retroid                  │ retroid
    │ uuid                     │ order_number             │ activatetime
    │ jobnumber                │ express_no               │ country_code
    │                          │ channel                  │ firmware_type
    │                          │ sku_code                 │
    ▼                          ▼                          ▼
OMS:                      n8n:                        n8n:
sninventories             get_channel                 激活记录同步
    │                          │                          │
    │ sn                       │ 渠道拉取                  │ 激活/注销
    │ material                 │ 库存查询                  │ 不良记录
    │ workOrderNo              │ 发货状态                  │ 保修计算
    │ status                   │                          │
    ▼                          ▼                          ▼
                    ┌─────────────────────────────────────────┐
                    │          SN 完整生命周期视图              │
                    │  生产 → 入库 → 发货 → 激活 → 售后        │
                    └─────────────────────────────────────────┘
```

### 4. 同步任务流

```
sync.jobs (任务定义)
    │
    │ 字段：
    │ - name (任务名称)
    │ - last_nosql_id (最后处理ID)
    │ - last_n8n_execution_id (最后n8n执行ID)
    │ - status (状态)
    │
    ▼
sync.pending_task (待处理队列)
    │
    │ 字段：
    │ - job_id (关联任务)
    │ - current_id (当前处理ID)
    │ - task_state (状态)
    │ - work_count (重试次数)
    │
    ▼
sync.task (执行记录)
    │
    │ 字段：
    │ - previous_id (上游ID)
    │ - current_id (当前ID)
    │ - task_result (结果)
    │ - n8n_execution_id (n8n执行ID)
    │
    ▼
具体同步任务：
├── sap_job_info_cache → SAP工单同步
├── sn_material_code → SN物料映射
├── yf_pcba_data → PCBA数据同步
└── government_subsidy → 政府补贴记录
```

---

## 📊 数据量统计

| 数据库/表 | 记录数 | 说明 |
|-----------|--------|------|
| **mes_jp.machine_data** | ~10,846 | 日本生产数据 |
| **sync.sap_job_info_cache** | ~246,743 | SAP工单缓存 |
| **sync.sn_material_code** | ~986,121 | SN物料映射 |
| **sync.yf_pcba_data** | ~326,319 | PCBA生产数据 |
| **sync.pending_task** | ~8,091 | 待处理任务 |
| **sync.task** | ~94,276 | 任务执行记录 |
| **sync.government_subsidy** | ~250,980 | 政府补贴 |
| **public.activation_records** | ~2,430,231 | 激活记录 |
| **public.deactivation_records** | ~94,339 | 注销记录 |
| **public.defective_records** | ~44,347 | 不良记录 |
| **warehouseDelivery.mes_delivery_order_details** | ~739,675 | MES发货明细 |
| **warehouseDelivery.delivery_log** | ~578,392 | 发货日志 |
| **warehouseDelivery.delivery_order_caches** | ~456,898 | 发货缓存 |
| **warehouseDelivery.refund** | ~5,276 | 退款记录 |
| **warehouseDelivery.delivery_order** | ~4,935 | 发货订单 |
| **warehouseDelivery.delivery_order_details** | ~10,020 | 发货明细 |
| **YFConsumer.consm_barcodebindinfo** | ~2,760,483 | 旧MES绑定数据 |
| **warehouseDelivery.api_log** | ~9,923 | API日志 |

---

## 🔑 关键关联字段

| 字段名 | 出现位置 | 用途 |
|--------|---------|------|
| **retroid** | 所有数据库 | 设备唯一标识（10位） |
| **uuid / uuid_unique** | 旧MES、public、warehouseDelivery | 设备UUID |
| **sn** | OMS (MongoDB) | 序列号 |
| **order_number / orderNo** | warehouseDelivery、OMS | 订单号 |
| **job_number / workOrderNo** | mes_jp、sync、OMS | 工单号 |
| **material_code / material** | sync、OMS | 物料号 |
| **channel** | warehouseDelivery、public | 渠道号 |
| **express_no** | warehouseDelivery | 快递单号 |
| **sku_code** | warehouseDelivery | SKU代码 |

---

*持续更新，每次新增数据流后补充。*
