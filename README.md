# COROS MES 系统架构文档

MES（制造执行系统）的完整技术文档体系。

## 📚 文档目录

| # | 文档 | 说明 | 飞书链接 |
|---|------|------|---------|
| 1 | [MES系统全景图](01-system-overview.md) | 入口文档，10分钟了解全貌 | [飞书](https://www.feishu.cn/wiki/Z4Lwwp9V7iUIIJk3Zt8c94sUnNg) |
| 2 | [业务架构](02-business-architecture.md) | SN生命周期、工单、工序、各业务流程 | [飞书](https://www.feishu.cn/wiki/OYNbwY6oCizoZ7ke3wqcKdDknHr) |
| 3 | [技术架构](03-technical-architecture.md) | NestJS、n8n分工、接口标准、部署 | [飞书](https://www.feishu.cn/wiki/FrxXwoGNDiqjHbkOweMcgLy9nHg) |
| 4 | [数据库参考手册](04-database-reference.md) | ER图、8个Schema、表字段详解 | [飞书](https://www.feishu.cn/wiki/UUB7wS45GixSO9kKC3Ncjh2Jn4c) |
| 5 | [n8n工作流参考手册](05-n8n-workflow-reference.md) | 322个工作流分类、排查、迁移状态 | [飞书](https://www.feishu.cn/wiki/Oo6GwpRlSiWKJdklI4dcBo18n9c) |
| 6 | [API接口参考手册](06-api-reference.md) | NestJS API + n8n Webhook 接口清单 | [飞书](https://www.feishu.cn/wiki/Mt3QwHpsYiEmSikx88wciQKWnTe) |

## 📂 原始数据

| 目录 | 内容 |
|------|------|
| `raw-data/n8n-workflows/` | 322个n8n工作流JSON文件 |
| `raw-data/database/` | 数据库表结构文件 |

## 阅读顺序

```
第一层：全景图（了解MES是什么）
  ↓
第二层：业务架构 + 技术架构（了解怎么运转）
  ↓
第三层：数据库/n8n/API参考手册（需要时查阅）
```

---

*本仓库由虾王（MES产品助手）自动维护。文档源头在飞书知识库「MES全览」。*
