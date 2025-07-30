# DataHub 存储架构分析

## 概述

DataHub 是一个现代化的数据发现和治理平台，其存储架构采用了多层次、多后端的设计模式。系统主要使用两种类型的存储后端来满足不同的数据访问需求：

1. **关系型数据库（MySQL）** - 用于存储元数据的结构化数据
2. **图数据库（ElasticSearch/Neo4j）** - 用于存储和查询实体间的关系数据
3. **搜索引擎（ElasticSearch）** - 用于全文搜索和复杂查询

## 整体存储架构

```mermaid
graph TB
    subgraph "DataHub 存储层架构"
        subgraph "应用层"
            Web[DataHub Web UI]
            API[REST/GraphQL API]
            Ingestion[数据摄取]
        end
        
        subgraph "服务层"
            EntityService[EntityService<br/>实体服务]
            GraphService[GraphService<br/>图服务]
            SearchService[SearchService<br/>搜索服务]
        end
        
        subgraph "数据访问层"
            AspectDao[EbeanAspectDao<br/>MySQL数据访问层]
            ESGraphDAO[ESGraphDAO<br/>ES图数据访问层]
            Neo4jDAO[Neo4jDAO<br/>Neo4j数据访问层]
            SearchDAO[SearchDAO<br/>搜索数据访问层]
        end
        
        subgraph "存储后端"
            MySQL[(MySQL Database<br/>元数据存储)]
            ElasticSearch[(ElasticSearch<br/>图数据+搜索)]
            Neo4j[(Neo4j<br/>图数据库)]
            Kafka[Kafka<br/>消息队列]
        end
        
        subgraph "缓存层"
            Cache[Hazelcast/Redis<br/>分布式缓存]
        end
    end
    
    %% 连接关系
    Web --> API
    API --> EntityService
    API --> GraphService
    API --> SearchService
    Ingestion --> EntityService
    
    EntityService --> AspectDao
    GraphService --> ESGraphDAO
    GraphService --> Neo4jDAO
    SearchService --> SearchDAO
    
    AspectDao --> MySQL
    ESGraphDAO --> ElasticSearch
    Neo4jDAO --> Neo4j
    SearchDAO --> ElasticSearch
    
    EntityService --> Kafka
    Kafka --> ElasticSearch
    Kafka --> Neo4j
    
    EntityService --> Cache
    GraphService --> Cache
    SearchService --> Cache
    
    style MySQL fill:#e1f5fe
    style ElasticSearch fill:#f3e5f5
    style Neo4j fill:#e8f5e8
    style Kafka fill:#fff3e0
```

## 核心组件详解

### 1. EntityService - 实体服务

`EntityServiceImpl` 是DataHub的核心服务，负责：

- **实体生命周期管理**：创建、更新、删除实体
- **版本控制**：管理实体的多版本存储
- **批量操作**：支持高效的批量数据处理
- **事务管理**：确保数据一致性
- **事件发布**：向Kafka发送变更事件

**关键特性**：
- 使用 `EbeanAspectDao` 访问MySQL
- 支持乐观锁并发控制
- 实现了完整的ACID事务
- 集成缓存机制提升性能

### 2. GraphService - 图服务

图服务提供两种实现：

#### ElasticSearchGraphService
- 基于ElasticSearch的图数据存储
- 适合大规模数据和复杂查询
- 支持全文搜索和聚合分析
- 默认的图服务实现

#### Neo4jGraphService  
- 基于Neo4j的原生图数据库
- 专为图遍历和关系查询优化
- 支持Cypher查询语言
- 可选的图服务实现

### 3. 存储后端选择

```mermaid
flowchart LR
    subgraph "存储决策"
        Data[数据类型判断]
        Data --> Metadata{元数据？}
        Data --> Relationship{关系数据？}
        Data --> Search{搜索数据？}
        
        Metadata -->|是| MySQL[MySQL存储]
        Relationship -->|是| GraphChoice{图实现选择}
        Search -->|是| ES[ElasticSearch]
        
        GraphChoice --> ES_Graph[ElasticSearch图服务]
        GraphChoice --> Neo4j_Graph[Neo4j图服务]
    end
    
    style MySQL fill:#e1f5fe
    style ES fill:#f3e5f5
    style ES_Graph fill:#f3e5f5
    style Neo4j_Graph fill:#e8f5e8
```

## 数据流转模式

### 写入流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant EntityService as EntityService
    participant AspectDao as EbeanAspectDao
    participant MySQL as MySQL
    participant Producer as EventProducer
    participant Kafka as Kafka
    participant GraphService as GraphService
    participant ES as ElasticSearch
    
    Client->>EntityService: 提交元数据变更
    EntityService->>AspectDao: 数据验证和转换
    AspectDao->>MySQL: 事务性写入
    MySQL-->>AspectDao: 写入确认
    AspectDao-->>EntityService: 返回结果
    EntityService->>Producer: 发送变更事件
    Producer->>Kafka: 异步消息
    Kafka->>GraphService: 消费关系变更
    GraphService->>ES: 更新图数据
    EntityService-->>Client: 返回成功
```

### 读取流程

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant EntityService as EntityService
    participant Cache as 缓存
    participant AspectDao as EbeanAspectDao
    participant MySQL as MySQL
    
    Client->>EntityService: 查询实体
    EntityService->>Cache: 检查缓存
    alt 缓存命中
        Cache-->>EntityService: 返回缓存数据
    else 缓存未命中
        EntityService->>AspectDao: 查询数据库
        AspectDao->>MySQL: SQL查询
        MySQL-->>AspectDao: 返回数据
        AspectDao-->>EntityService: 返回实体
        EntityService->>Cache: 更新缓存
    end
    EntityService-->>Client: 返回实体数据
```

## 关键设计特点

### 1. 分层架构
- **表现层**：Web UI和API
- **业务层**：各种服务组件
- **数据访问层**：DAO组件
- **存储层**：多种后端存储

### 2. 多存储后端
- **MySQL**：ACID事务，结构化数据
- **ElasticSearch**：搜索和图数据
- **Neo4j**：专业图数据库
- **Kafka**：异步消息处理

### 3. 异步处理
- 使用Kafka实现存储后端间的数据同步
- 分离写入和索引更新，提升性能
- 支持最终一致性模型

### 4. 扩展性设计
- 支持水平扩展
- 可插拔的存储实现
- 微服务架构支持

## 配置管理

DataHub使用Spring Boot的配置管理，通过不同的Factory类实现存储后端的选择：

- `ElasticSearchGraphServiceFactory` - ElasticSearch图服务配置
- `Neo4jGraphServiceFactory` - Neo4j图服务配置
- `EbeanAspectDaoFactory` - MySQL数据访问配置

配置示例：
```yaml
# 图服务类型选择
graphService:
  type: elasticsearch  # 或 neo4j

# MySQL配置
ebean:
  datasource:
    url: jdbc:mysql://localhost:3306/datahub
    username: datahub
    password: datahub

# ElasticSearch配置
elasticsearch:
  host: localhost
  port: 9200
```

## 总结

DataHub的存储架构通过多后端存储、分层设计和异步处理，实现了高性能、高可用和强扩展性的数据管理平台。这种架构能够有效支撑大规模元数据管理的需求，同时保持了良好的维护性和扩展性。