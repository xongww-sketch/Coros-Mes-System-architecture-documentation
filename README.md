# COROS MES 系统架构文档

MES（制造执行系统）的完整技术文档体系。覆盖系统架构、业务架构、数据库架构、n8n工作流架构、API接口。

## 📚 文档目录

按**三层递进**阅读：

### 第一层：全貌

| 文档 | 说明 |
|------|------|
| [MES系统全景图](01-system-overview.md) | **从这里开始**。10分钟了解MES是什么、有几个部分、跟谁对接 |

### 第二层：业务 + 技术

| 文档 | 说明 |
|------|------|
| [业务架构](02-business-architecture.md) | SN全生命周期、工单模型、24道工序、维修/作废/补料、模块联动 |
| [技术架构](03-technical-architecture.md) | NestJS与n8n分工、接口分类标准、认证授权、部署架构 |

### 第三层：参考手册（需要时查阅）

| 文档 | 说明 |
|------|------|
| [数据库参考手册](04-database-reference.md) | 8个Schema、ER图、表字段详解、跨库关联、常用SQL |
| [n8n工作流参考手册](05-n8n-workflow-reference.md) | 322个工作流分类、Top15详解、风险登记、故障排查 |
| [API接口参考手册](06-api-reference.md) | NestJS后台API + n8n Webhook接口清单、迁移状态 |

## 📂 原始数据

供Agent解析和开发参考：

| 目录 | 内容 | 数量 |
|------|------|------|
| [raw-data/n8n-workflows/](raw-data/n8n-workflows/) | n8n工作流JSON文件（已脱敏） | 322个 |
| [raw-data/database/](raw-data/database/) | 数据库表结构文件 | 2个 |

## 🏗️ 系统技术栈

| 层 | 技术 |
|----|------|
| 前端 | Vue3 + Vite（RuoYi-Vue3） |
| 低代码 | Lowcoder |
| 后端 | NestJS（carole-admin） |
| 工作流 | n8n（自托管，322个工作流） |
| 数据库 | PostgreSQL 18（8个Schema） |
| 缓存 | Redis |
| ORM | Prisma |
| 队列 | BullMQ |
| 设备通信 | MQTT |

## 📎 飞书知识库

文档同步维护在飞书知识库「MES全览」节点下：

| 文档 | 飞书链接 |
|------|---------|
| MES系统全景图 | [飞书](https://www.feishu.cn/wiki/Z4Lwwp9V7iUIIJk3Zt8c94sUnNg) |
| 业务架构 | [飞书](https://www.feishu.cn/wiki/OYNbwY6oCizoZ7ke3wqcKdDknHr) |
| 技术架构 | [飞书](https://www.feishu.cn/wiki/FrxXwoGNDiqjHbkOweMcgLy9nHg) |
| 数据库参考手册 | [飞书](https://www.feishu.cn/wiki/UUB7wS45GixSO9kKC3Ncjh2Jn4c) |
| n8n工作流参考手册 | [飞书](https://www.feishu.cn/wiki/Oo6GwpRlSiWKJdklI4dcBo18n9c) |
| API接口参考手册 | [飞书](https://www.feishu.cn/wiki/Mt3QwHpsYiEmSikx88wciQKWnTe) |

---

*本仓库由虾王（MES产品助手）维护。飞书知识库为源头，GitHub为同步备份和Agent数据源。*
