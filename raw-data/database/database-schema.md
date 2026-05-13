# MES 数据库全景架构

> 版本：1.0 | 2026-04-15
> 来源：熊旺提供 + n8n工作流交叉分析
> 数据库：PostgreSQL 18 + MongoDB 6.0 + SQL Server 10.50

---

## 🗺️ 全局架构总览

```
┌─────────────────────────────────────────────────────────────────┐
│                        数据流全景图                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────┐    n8n工作流     ┌─────────────────────────────┐  │
│  │ 日本生产  │ ──────────────→ │  mes_jp.machine_data        │  │
│  │ SharePoint│  CSV上传        │  (日本代工厂生产资料)          │  │
│  └──────────┘                 └──────────────┬──────────────┘  │
│                                              │                 │
│  ┌──────────┐    n8n工作流     ┌──────────────▼──────────────┐  │
│  │ 国内发货  │ ──────────────→ │  warehouseDelivery.public   │  │
│  │ Lowcoder │  前端录入       │  mes_delivery_order_details │  │
│  └──────────┘                 └──────────────┬──────────────┘  │
│                                              │                 │
│  ┌──────────┐    n8n工作流     ┌──────────────▼──────────────┐  │
│  │ 旧MES    │ ──────────────→ │  YFConsumer.dbo             │  │
│  │ SQLServer│  历史数据       │  consm_barcodebindinfo      │  │
│  └──────────┘                 └──────────────┬──────────────┘  │
│                                              │                 │
│  ┌──────────┐    n8n工作流     ┌──────────────▼──────────────┐  │
│  │ OMS系统  │ ←────────────── │  MongoDB oms-service        │  │
│  │ 库存管理  │  拉取日本数据    │  sninventories/bins/        │  │
│  └──────────┘                 │  snoutbounds                │  │
│                               └─────────────────────────────┘  │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  mes主库 (pgsql.mes.coros.team:5432)                     │   │
│  │  ├── mes          (主库表 - 待补充)                       │   │
│  │  ├── mes_jp       (日本生产数据)                          │   │
│  │  ├── public       (品质应用/激活/不良记录)                 │   │
│  │  └── sync         (同步任务/SAP缓存/工单)                  │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📊 数据库清单

| # | 数据库 | 类型 | 地址 | Schema | 用途 |
|---|--------|------|------|--------|------|
| 1 | **mes** | PostgreSQL 18 | pgsql.mes.coros.team:5432 | mes | 主库（生产核心） |
| 2 | **mes** | PostgreSQL 18 | pgsql.mes.coros.team:5432 | mes_jp | 日本生产数据 |
| 3 | **mes** | PostgreSQL 18 | pgsql.mes.coros.team:5432 | public | 品质应用/激活记录 |
| 4 | **mes** | PostgreSQL 18 | pgsql.mes.coros.team:5432 | sync | 同步任务/SAP缓存 |
| 5 | **warehouseDelivery** | PostgreSQL 18 | pgsql.mes.coros.team:5432 | public | 国内发货/Lowcoder |
| 6 | **YFConsumer** | SQL Server 10.50 | 192.168.128.7:1433 | dbo | 旧MES绑定数据 |
| 7 | **oms-service** | MongoDB 6.0 | localhost:27017 | - | OMS库存/发货 |

---

## 1️⃣ mes_jp — 日本生产数据

### 连接信息
- **主机**: `pgsql.mes.coros.team:5432`
- **数据库**: `mes`
- **Schema**: `mes_jp`
- **数据量**: ~10,846 条

### 表：machine_data（日本代工厂生产资料）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK, 自增) | 主键 |
| retroid | varchar(10) NOT NULL | 设备ID（10位） |
| data | json NOT NULL | 生产数据（JSON格式） |
| job_number | varchar(25) NOT NULL | 工单号 |
| createtime | timestamp NOT NULL | 创建时间 |
| uploadtime | timestamp NOT NULL | 上传时间 |

### n8n 关联流程
- **CSV上传流程**: `https://n8n.mes.coros.team/workflow/kRCqKO9jDbPn09W9`
  - 数据源：SharePoint `https://jems01.sharepoint.com/sites/tm206-COROS-JEMS/...`
  - 操作：CSV → 转换 → 插入 `mes_jp.machine_data`
  - 供给：OMS系统拉取日本生产数据

---

## 2️⃣ sync — 同步任务

### 连接信息
- **主机**: `pgsql.mes.coros.team:5432`
- **数据库**: `mes`
- **Schema**: `sync`

### 表清单

#### government_subsidy（政府补贴）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| retroid | char(10) | 设备ID |
| create_time | timestamp | 创建时间 |

#### jobs（同步任务）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| name | varchar(200) | 任务名称 |
| last_nosql_id | varchar(24) | 最后处理的NoSQL ID |
| last_n8n_execution_id | int4 | 最后n8n执行ID |
| status | int4 | 状态（默认1） |

#### pending_task（待处理任务）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| job_id | int4 NOT NULL | 关联任务ID |
| current_id | varchar(24) | 当前处理ID |
| task_start_time | timestamp | 开始时间 |
| task_end_time | timestamp | 结束时间 |
| task_result | text | 执行结果 |
| task_state | int4 | 状态（默认0） |
| work_count | int4 | 重试次数 |

#### sap_job_info_cache（SAP工单缓存）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| job_number | varchar | 工单号（唯一） |
| material_code | varchar | 物料号 |
| crete_time | timestamp | 创建时间 |
| item_code | varchar | 商品规格代码 |
| sale_location | int2 | 1=国内，2=海外 |

#### sn_material_code（SN物料映射）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| sn | varchar NOT NULL (唯一) | 序列号 |
| material_code | varchar | 物料号 |
| create_time | timestamp | 创建时间 |
| job_number | varchar | 工单号 |

#### task（任务执行记录）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| job_id | int4 NOT NULL | 关联任务ID |
| previous_id | varchar(24) NOT NULL | 上游ID |
| current_id | varchar(24) | 当前ID |
| task_start_time | timestamp | 开始时间 |
| task_end_time | timestamp | 结束时间 |
| task_result | text | 执行结果 |
| n8n_execution_id | int4 | n8n执行ID |

#### yf_pcba_data（PCBA生产数据）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| material_code | varchar NOT NULL | 物料号 |
| material_description | varchar | 物料描述 |
| production_quantity | varchar | 生产数量 |
| type | char(10) | 类型 |
| production_date | varchar | 生产日期 |
| order_status | int2 | 订单状态 |
| create_time | timestamp | 创建时间 |
| production_end_date | varchar | 生产结束日期 |
| work_order_number | int8 (唯一) | 工单号 |
| order_create_time | varchar | 订单创建时间 |
| base_bom_code | varchar | 基准BOM料号 |
| is_update | int2 | 是否更新 |
| zecnl4 | varchar | 扩展字段 |

---

## 3️⃣ public — 品质应用（mes主库）

### 连接信息
- **主机**: `pgsql.mes.coros.team:5432`
- **数据库**: `mes`
- **Schema**: `public`
- **关联**: Lowcoder前端页面

### 核心表

#### activation_records（激活记录）— 2,430,231 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| activatetime | int8 | 激活时间戳 |
| city | text | 城市 |
| country_code | text | 国家代码 |
| email | text | 邮箱 |
| expirestime | date | 过期时间 |
| firmware_type | text | 固件类型 |
| ip_address | text | IP地址 |
| remain_warranty_days | numeric | 剩余保修天数 |
| user_id | text | 用户ID |
| deactivestatus | text | 1=已删除激活记录 |
| retroid | varchar | 设备ID |
| uuid | varchar | UUID |

#### deactivation_records（注销记录）— 94,339 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| active_id | int4 | 关联激活记录ID |
| deletetime | timestamp | 注销时间 |
| deletetype | numeric | 注销类型 |

#### defective_records（不良记录）— 44,347 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| retroid | varchar | 设备ID |
| defective_date | timestamp | 不良日期 |
| defective_type | text | 不良类型 |
| defective_reason | text | 不良原因 |
| active_id | int4 | 关联激活记录ID |
| defective_duration_days | numeric | 不良持续天数（自动计算） |

#### historydata（历史数据）
| 字段 | 类型 | 说明 |
|------|------|------|
| region_id | int4 | 区域ID |
| id | int8 (PK) | 主键 |
| user_id | int8 | 用户ID |
| device_id | varchar(50) | 设备ID |
| mac | varchar(50) | MAC地址 |
| uuid | varchar(50) | UUID |
| firmware_type | varchar(50) | 固件类型 |
| firmware_version | varchar(50) | 固件版本 |
| activate_time | timestamp | 激活时间 |
| activate_day | int4 | 激活日 |
| create_date | timestamp | 创建日期 |
| timezone | int4 | 时区 |
| ip_address | varchar(50) | IP地址 |
| longitude | float8 | 经度 |
| latitude | float8 | 纬度 |
| expires_time | timestamp | 过期时间 |
| type | int4 | 类型 |
| channel | varchar(20) | 渠道 |
| phase | varchar(20) | 阶段 |
| update_time | timestamp | 更新时间 |
| city | text | 城市 |
| country | varchar(100) | 国家 |
| province | varchar(100) | 省份 |
| region | varchar(100) | 区域 |
| retro_id | varchar(50) | 设备ID |
| product_code | varchar(50) | 产品代码 |
| machine_type | varchar(50) | 机型 |
| device_mac | varchar(50) | 设备MAC |
| device_uuid | varchar(50) | 设备UUID |
| channel_his | int4 | 历史渠道 |

---

## 4️⃣ warehouseDelivery.public — 国内发货

### 连接信息
- **主机**: `pgsql.mes.coros.team:5432`
- **数据库**: `warehouseDelivery`
- **Schema**: `public`
- **关联**: Lowcoder前端页面

### 核心表

#### mes_delivery_order_details（MES发货订单明细）— 739,675 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| retroid | varchar(36) NOT NULL | 设备ID |
| uuid | varchar(200) | UUID |
| order_number | varchar(200) | 订单号 |
| express_no | varchar(50) | 快递单号 |
| create_time | timestamp | 创建时间 |
| delivery_status | int2 | 发货状态 |
| uuid_unique | varchar(50) | 唯一UUID |
| channel | varchar(10) | 渠道 |
| sku_code | varchar(30) | SKU代码 |
| box_id | varchar | 箱号 |
| record_create_time | timestamp | 记录创建时间 |

#### delivery_order_details（发货订单明细）— 10,020 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| retroid | varchar NOT NULL | 设备ID |
| uuid | varchar | UUID |
| watchtype | varchar | 手表类型 |
| price | numeric | 价格 |
| order_number | varchar | 订单号 |
| create_time | timestamptz | 创建时间 |
| watch_status | int2 | 手表状态（默认1） |
| sku_code | varchar | SKU代码 |
| uuid_unique | varchar(50) | 唯一UUID |
| sku_name | varchar(30) | SKU名称 |

#### delivery_log（发货日志）— 578,392 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| jobnumber | varchar(20) | 工单号 |
| ordernumber | varchar(20) | 订单号 |
| productcode | varchar(20) | 产品代码 |
| clientcode | varchar(20) | 客户代码 |
| retroid | varchar(50) | 设备ID |
| baseline | varchar(50) | 基线 |
| uuid | varchar(200) | UUID |
| baselinebatch | varchar(50) | 基线批次 |
| batterybatch | varchar(200) | 电池批次 |
| procedurevalue | int4 | 工序值 |
| isstateok | bool | 状态是否正常 |
| ischeck | bool | 是否检查 |
| createtime | timestamp | 创建时间 |
| modifytime | timestamp | 修改时间 |
| channel | int4 | 渠道 |
| uuid_unique | varchar(31) | 唯一UUID |
| order_number | varchar(200) | 订单号或快递单号 |

#### delivery_order（发货订单）— 4,935 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| order_number | varchar NOT NULL (唯一) | 订单号 |
| trial_status | int2 | 试用状态（默认1） |
| payment_amount | numeric | 支付金额 |
| create_time | timestamp | 创建时间 |
| order_details | text[] | 订单明细数组 |
| channel_id | varchar | 渠道ID |
| express_no | varchar(50) | 快递单号 |

#### delivery_order_caches（发货订单缓存）— 456,898 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| express_no | varchar | 快递单号 |
| data | text | 缓存数据 |

#### refund（退款）— 5,276 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| order_number | varchar NOT NULL | 订单号 |
| retroid | varchar | 设备ID |
| uuid | varchar | UUID |
| refund_price | numeric NOT NULL | 退款金额 |
| refund_status | int2 | 退款状态 |
| sku_code | varchar | SKU代码 |
| refund_time | timestamp | 退款时间 |
| create_time | timestamp | 创建时间 |

#### prototype_trial（原型试用）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | varchar(36) (PK) | 主键ID |
| order_phonenumber | text[] | 下单人手机号 |
| trial_expiration_date | date | 试用到期时间 |
| is_remind | int2 | 是否提醒（默认1） |
| status | int2 | 审批实例状态（默认0） |
| step | int2 | 审批调用接口步骤（默认0） |
| channel | varchar(30) | 渠道号 |
| partners | varchar(100) | 合作伙伴名称 |
| order_number | text[] | 订单号 |
| create_time | timestamp | 创建时间 |

#### api_log（API日志）— 9,923 条
| 字段 | 类型 | 说明 |
|------|------|------|
| id | int4 (PK) | 主键 |
| instance_id | varchar NOT NULL | 实例ID |
| api_name | varchar NOT NULL | API名称 |
| transfer_args | json | 传输参数 |
| transfer_result | json | 传输结果 |
| step | int2 | 步骤 |
| create_time | timestamp | 创建时间 |

#### approval_info（审批信息）
| 字段 | 类型 | 说明 |
|------|------|------|
| uuid | uuid (PK) | 主键 |
| app_id | text | 应用ID |
| approval_code | varchar | 审批码 |
| custom_key | text | 自定义键 |
| def_key | text | 默认键 |
| generate_type | text | 生成类型 |
| instance_code | varchar | 实例码 |
| open_id | text | OpenID |
| operate_time | int8 | 操作时间 |
| status | text | 状态 |
| task_id | int8 | 任务ID |
| tenant_key | text | 租户键 |
| type | text | 类型 |
| user_id | text | 用户ID |
| token | text | 令牌 |
| ts | float8 | 时间戳 |
| event | jsonb | 事件 |
| info_type | int2 | 信息类型 |
| info_status | int2 | 信息状态 |

#### approval_records（审批记录）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uuid (PK) | 主键 |
| entity_type | varchar(50) NOT NULL | 实体类型 |
| entity_id | uuid NOT NULL | 实体ID |
| approval_type | varchar(20) NOT NULL | 审批类型 |
| approval_no | varchar(50) | 审批编号 |
| instance_code | varchar(100) | 实例码 |
| status | varchar(30) | 状态（默认pending） |
| result | varchar(30) | 结果 |
| submitted_by | varchar(100) | 提交人 |
| submitted_by_name | varchar(100) | 提交人姓名 |
| submitted_at | timestamptz | 提交时间 |
| completed_at | timestamptz | 完成时间 |
| remark | text | 备注 |

#### suppliers（供应商）
| 字段 | 类型 | 说明 |
|------|------|------|
| id | uuid (PK) | 主键 |
| supplier_code | varchar | 供应商代码 |
| supplier_name | varchar(200) NOT NULL | 供应商名称 |
| supplier_category | varchar(20) NOT NULL | 供应商类别 |
| country | varchar(100) NOT NULL | 国家 |
| address | text | 地址 |
| vat_no | varchar | VAT号 |
| w9 | varchar(10) | W9表格 |
| contact_person | varchar(100) | 联系人 |
| contact_email | varchar(200) | 联系邮箱 |
| pay_via | varchar(50) NOT NULL | 支付方式 |
| pay_to | varchar(200) | 收款方 |
| payment_bank | varchar(200) | 付款银行 |
| account | varchar(100) | 账号 |
| paypal_account | varchar(200) | PayPal账号 |
| routing | varchar(50) | 路由号 |
| swift | varchar(50) | SWIFT代码 |
| iban | varchar(50) | IBAN |
| is_sap_one_time_vendor | bool | 是否SAP一次性供应商 |
| approval_status | varchar(30) | 审批状态（默认pending） |
| submitted_by | varchar(100) | 提交人 |
| submitted_by_name | varchar(100) | 提交人姓名 |
| created_by | varchar(100) | 创建人 |
| created_by_name | varchar(100) | 创建人姓名 |
| updated_by | varchar(100) | 更新人 |
| updated_by_name | varchar(100) | 更新人姓名 |
| created_at | timestamptz | 创建时间 |
| updated_at | timestamptz | 更新时间 |
| deleted_at | timestamptz | 删除时间 |

#### users / roles / user_roles（用户权限）
标准RBAC权限模型，包含用户、角色、权限规则。

---

## 5️⃣ YFConsumer.dbo — 旧MES

### 连接信息
- **主机**: `192.168.128.7:1433`
- **数据库**: `YFConsumer`
- **Schema**: `dbo`
- **类型**: SQL Server 10.50

### 表：consm_barcodebindinfo（条码绑定信息）— 2,760,483 条

| 字段 | 类型 | 说明 |
|------|------|------|
| id | int (PK, 自增) | 主键 |
| jobnumber | nvarchar(20) | 工单号 |
| ordernumber | nvarchar(20) | 订单号 |
| productcode | nvarchar(20) | 产品代码 |
| clientcode | nvarchar(20) | 客户代码 |
| retroid | nvarchar(50) | 设备ID（唯一索引） |
| baseline | nvarchar(50) | 基线 |
| uuid | nvarchar(200) | UUID（索引） |
| BaseLineBatch | nvarchar(50) | 基线批次 |
| BatteryBatch | nvarchar(200) | 电池批次 |
| procedurevalue | int | 工序值 |
| isstateok | bit | 状态是否正常 |
| ischeck | bit | 是否检查 |
| createtime | datetime | 创建时间（索引） |
| modifytime | datetime | 修改时间 |
| Channel | int | 渠道（索引） |
| uuid_unique | nvarchar(18) | 唯一UUID |

### 索引
- `retroid` — 唯一索引
- `uuid` — 普通索引
- `createtime` — 普通索引
- `Channel` — 普通索引

---

## 6️⃣ oms-service — OMS库存管理

### 连接信息
- **主机**: `localhost:27017`
- **数据库**: `oms-service`
- **类型**: MongoDB 6.0.6

### 核心集合

#### sninventories（SN库存）
| 索引 | 字段 | 说明 |
|------|------|------|
| 唯一索引 | isDelete + workOrderNo + status + sn + material | SN库存唯一标识 |
| 普通索引 | sn | SN查询 |
| 普通索引 | material | 物料查询 |
| 普通索引 | workOrderNo | 工单查询 |

#### bins（仓库）
| 索引 | 字段 | 说明 |
|------|------|------|
| 唯一索引 | isDelete + sapBinCode + sapFactoryCode | 仓库唯一标识 |

#### snoutbounds（SN出库记录）
| 索引 | 字段 | 说明 |
|------|------|------|
| 普通索引 | sn | SN查询 |

#### snoutboundresets（SN出库重置）
| 索引 | 字段 | 说明 |
|------|------|------|
| 普通索引 | dnId | 发货单ID |
| 普通索引 | dnNo | 发货单号 |

#### sns（SN主集合）
| 索引 | 字段 | 说明 |
|------|------|------|
| 唯一索引 | sn | SN唯一 |
| 普通索引 | material | 物料查询 |

#### snmaps（SN映射）
| 索引 | 字段 | 说明 |
|------|------|------|
| 唯一索引 | sn | SN唯一映射 |

#### snimportjobs（SN导入任务）
| 索引 | 字段 | 说明 |
|------|------|------|
| 唯一索引 | key | 任务键唯一 |

#### retailsnimportrecords（零售SN导入记录）
| 索引 | 字段 | 说明 |
|------|------|------|
| 普通索引 | sn | SN查询 |
| 普通索引 | lineItemId | 行项目ID |
| 普通索引 | variantId | 变体ID |

### Shopify 相关集合
- `rawshopifyorders` — Shopify原始订单
- `rawshopifyproducts` — Shopify原始产品
- `rawshopifycustomers` — Shopify原始客户
- `rawshopifyfulfillments` — Shopify原始履约
- `shopifyorders` — Shopify订单
- `shopifyproducts` — Shopify产品
- `shopifyfulfillments` — Shopify履约
- `shopifyfulfillmentorders` — Shopify履约订单
- `shopifylocations` — Shopify门店
- `shopifyrefunds` — Shopify退款
- `shopifyreturns` — Shopify退货
- `shopifytransactions` — Shopify交易

### OMS 业务集合
- `customerorders` — 客户订单
- `customerdelivernotes` — 客户发货单
- `customerrmas` — 客户RMA
- `suppliers` — 供应商
- `supplierorders` — 供应商订单
- `supplierdelivernotes` — 供应商发货单
- `companies` — 公司
- `goods` — 商品
- `prices` — 价格
- `inventories` — 库存
- `inventorycaches` — 库存缓存
- `plannedinventories` — 计划库存
- `logisticstracks` — 物流追踪
- `events` — 事件
- `agendaJobs` / `agendaJobNew` — 定时任务

---

## 🔗 跨库关联关系

### retroid（设备ID）— 核心关联键
```
consm_barcodebindinfo.retroid (旧MES)
    ↓
mes_jp.machine_data.retroid (日本生产)
    ↓
public.activation_records.retroid (激活记录)
    ↓
warehouseDelivery.mes_delivery_order_details.retroid (发货明细)
    ↓
oms-service.sns.sn / sninventories.sn (OMS库存)
```

### order_number（订单号）— 订单关联键
```
warehouseDelivery.delivery_order.order_number
    ↓
warehouseDelivery.delivery_order_details.order_number
    ↓
warehouseDelivery.mes_delivery_order_details.order_number
    ↓
oms-service.customerorders.orderNo
```

### job_number / workOrderNo（工单号）— 生产关联键
```
sync.sap_job_info_cache.job_number
    ↓
sync.sn_material_code.job_number
    ↓
mes_jp.machine_data.job_number
    ↓
oms-service.sninventories.workOrderNo
```

### uuid / uuid_unique — 设备唯一标识
```
consm_barcodebindinfo.uuid / uuid_unique
    ↓
public.activation_records.uuid
    ↓
warehouseDelivery.delivery_log.uuid / uuid_unique
    ↓
warehouseDelivery.mes_delivery_order_details.uuid / uuid_unique
```

---

## 📋 n8n 工作流 × 数据库 交叉映射

| n8n 流程 | 操作数据库 | 操作表 | 说明 |
|----------|-----------|--------|------|
| 日本CSV上传 | mes_jp | machine_data | SharePoint CSV → 转换 → 插入 |
| get_channel | warehouseDelivery | mes_delivery_order_details | 渠道拉取 |
| get_channel | public | delivery_log | 国内发货数据 |
| get_channel | oms-service | sninventories | OMS库存数据 |
| get_mespackingdata | warehouseDelivery | delivery_order_details | 包装数据 |
| get_mesProductionData | sync | yf_pcba_data | PCBA生产数据 |
| get_mesProductionData | sync | sap_job_info_cache | SAP工单缓存 |
| 每日数据同步 | mes_daily_data | - | 同步给软件部 |
| 激活记录同步 | public | activation_records | 激活数据 |
| 不良记录处理 | public | defective_records | 不良品处理 |

---

*持续更新，每次新增表或流程后补充。*
