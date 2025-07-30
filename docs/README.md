# DataHub 存储架构文档

本目录包含了DataHub存储架构的详细文档，通过阅读源码整理而成。

## 文档列表

### 1. [存储架构](./1-storage-architecture.md)
详细介绍了DataHub的混合存储架构，包括：
- 整体架构图（Mermaid）
- 核心组件说明
- 数据流程
- 存储选择策略

### 2. [MySQL数据库和图数据的插入更新机制](./2-data-insert-update.md)
深入分析了数据的插入和更新流程，包括：
- 数据更新流程图
- MySQL数据库的插入和更新实现
- 图数据的插入和更新实现
- 差异化更新模式
- 性能优化策略

### 3. [图数据使用场景](./3-graph-data-usage.md)
全面介绍了图数据在DataHub中的应用场景，包括：
- 数据血缘追踪
- 影响分析
- 数据依赖关系
- 数据所有权和团队协作
- 数据质量传播
- 相似数据发现
- 访问路径分析

## 核心概念

### 实体(Entity)
DataHub中的基本数据单元，如Dataset、DataJob、Dashboard等。

### 方面(Aspect)
实体的不同维度信息，如Schema、Ownership、Documentation等。

### URN (Uniform Resource Name)
实体的唯一标识符，格式如：`urn:li:dataset:(urn:li:dataPlatform:hive,SampleHiveDataset,PROD)`

### MCP (Metadata Change Proposal)
元数据变更提议，客户端提交的变更请求。

### MCL (Metadata Change Log)
元数据变更日志，记录实际发生的变更事件。

## 技术栈

- **MySQL**: 通过Ebean ORM存储实体元数据
- **Neo4j/ElasticSearch**: 存储和查询图关系
- **Kafka**: 事件流处理
- **ElasticSearch**: 全文搜索和部分图存储

## 架构特点

1. **混合存储**: 不同类型数据使用最适合的存储引擎
2. **事件驱动**: 通过Kafka实现异步更新和最终一致性
3. **可扩展性**: 支持多种图数据库实现
4. **高性能**: 批量操作、缓存、异步处理等优化
