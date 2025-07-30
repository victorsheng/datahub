# DataHub MySQL 数据库操作详解

## 概述

DataHub使用MySQL作为主要的元数据存储后端，通过Ebean ORM框架实现对MySQL的数据访问。本文档详细分析DataHub中MySQL数据库的插入和更新操作机制。

## 核心组件

### 1. EbeanAspectDao - 数据访问层

`EbeanAspectDao` 是DataHub访问MySQL的核心DAO组件，实现了：
- **Aspect存储**：元数据的增删改查
- **版本控制**：支持多版本数据存储
- **批量操作**：高效的批量数据处理
- **事务管理**：ACID事务保证

### 2. 数据模型

DataHub使用统一的Aspect模型存储所有元数据：

```mermaid
erDiagram
    METADATA_ASPECT_V2 {
        varchar urn PK "实体URN"
        varchar aspect PK "Aspect类型"
        bigint version PK "版本号"
        text metadata "JSON格式的元数据"
        text systemMetadata "系统元数据"
        timestamp createdOn "创建时间"
        varchar createdBy "创建者"
        varchar createdFor "创建目标"
    }
```

**关键字段说明**：
- `urn`: 实体的唯一标识符
- `aspect`: 元数据类型（如 DatasetProperties, SchemaMetadata）
- `version`: 版本号，0表示最新版本
- `metadata`: JSON格式存储的业务元数据
- `systemMetadata`: 系统级元数据（审计信息等）

## 插入操作流程

### 1. 单个Aspect插入

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant EntityService as EntityServiceImpl
    participant AspectDao as EbeanAspectDao
    participant Ebean as Ebean ORM
    participant MySQL as MySQL数据库
    participant EventProducer as EventProducer
    participant Kafka as Kafka
    
    Client->>EntityService: ingestProposal(mcp)
    EntityService->>EntityService: 数据验证和转换
    EntityService->>AspectDao: insertAspect(aspect, version)
    
    Note over AspectDao: 开始事务
    AspectDao->>AspectDao: validateConnection()
    AspectDao->>AspectDao: 转换为EbeanAspectV2
    AspectDao->>Ebean: saveEbeanAspect()
    
    alt 首次插入
        Ebean->>MySQL: INSERT INTO metadata_aspect_v2
    else 版本更新
        Ebean->>MySQL: INSERT新版本 + UPDATE latest
    end
    
    MySQL-->>Ebean: 插入结果
    Ebean-->>AspectDao: 返回实体
    AspectDao-->>EntityService: 返回EntityAspect
    
    Note over EntityService: 提交事务
    EntityService->>EventProducer: 发送变更事件
    EventProducer->>Kafka: 发布MCL事件
    EntityService-->>Client: 返回结果
```

**关键代码流程**：

1. **数据验证**：
```java
// EntityServiceImpl.applyUpsert()
static SystemAspect applyUpsert(ChangeMCP changeMCP, SystemAspect latestAspect) {
    long rowNextVersion = Math.max(1, changeMCP.getNextAspectVersion());
    // 版本控制和审计信息设置
    SystemMetadata changeSystemMetadata = new SystemMetadata(changeMCP.getSystemMetadata().copy().data());
    changeSystemMetadata.setVersion(String.valueOf(rowNextVersion));
}
```

2. **数据库插入**：
```java
// EbeanAspectDao.insertAspect()
public Optional<EntityAspect> insertAspect(TransactionContext txContext, SystemAspect aspect, long version) {
    validateConnection();
    if (!canWrite) return Optional.empty();
    
    EbeanAspectV2 ebeanAspectV2 = EbeanAspectV2.fromEntityAspect(aspect.asLatest());
    saveEbeanAspect(txContext, ebeanAspectV2, true);
    return Optional.of(ebeanAspectV2.toEntityAspect());
}
```

### 2. 批量插入操作

```mermaid
flowchart TD
    A[批量MCP请求] --> B[分组和验证]
    B --> C[创建AspectsBatch]
    C --> D[批量数据转换]
    D --> E[事务开始]
    E --> F[批量INSERT操作]
    F --> G[更新最新版本]
    G --> H[事务提交]
    H --> I[发布批量事件]
    I --> J[返回结果]
    
    style E fill:#e1f5fe
    style F fill:#e1f5fe
    style G fill:#e1f5fe
    style H fill:#e1f5fe
```

**批量操作优化**：
- 使用JDBC批处理减少网络往返
- 智能分批，避免内存溢出
- 优化SQL生成，减少数据库负载

## 更新操作流程

### 1. Upsert操作（插入或更新）

DataHub采用Upsert模式，根据数据是否存在自动选择插入或更新：

```mermaid
flowchart TD
    A[接收更新请求] --> B{检查数据是否存在}
    B -->|存在| C[执行更新逻辑]
    B -->|不存在| D[执行插入逻辑]
    
    C --> E[版本比较]
    E --> F{是否需要新版本}
    F -->|是| G[创建新版本]
    F -->|否| H[更新现有记录]
    
    G --> I[插入新版本记录]
    H --> J[原地更新记录]
    
    D --> K[直接插入新记录]
    
    I --> L[更新索引]
    J --> L
    K --> L
    L --> M[发布变更事件]
```

### 2. 版本控制机制

DataHub实现了sophisticated的版本控制：

```mermaid
graph LR
    subgraph "版本存储策略"
        V0[Version 0<br/>最新版本]
        V1[Version 1<br/>历史版本]
        V2[Version 2<br/>历史版本]
        VN[Version N<br/>历史版本]
    end
    
    V0 -.->|更新时| V1
    V1 -.->|保留历史| V2
    V2 -.->|历史链| VN
    
    style V0 fill:#4caf50
    style V1 fill:#ff9800
    style V2 fill:#ff9800
    style VN fill:#ff9800
```

**版本控制规则**：
- **Version 0**：始终表示最新版本
- **Version > 0**：表示历史版本
- 更新时创建新的历史版本，然后更新version 0
- 支持版本回滚和历史查询

## 事务管理

### 1. 事务隔离级别

DataHub使用`READ_COMMITTED`隔离级别，配合`SELECT FOR UPDATE`实现乐观锁：

```java
public static final TxIsolation TX_ISOLATION = TxIsolation.READ_COMMITED;
```

### 2. 并发控制

```mermaid
sequenceDiagram
    participant T1 as 事务1
    participant T2 as 事务2
    participant DB as 数据库
    
    T1->>DB: SELECT FOR UPDATE (获取行锁)
    T2->>DB: SELECT FOR UPDATE (等待锁)
    Note over DB: T2被阻塞
    
    T1->>DB: UPDATE/INSERT
    T1->>DB: COMMIT (释放锁)
    Note over DB: T2获得锁
    
    T2->>DB: 重新读取最新数据
    T2->>DB: UPDATE/INSERT
    T2->>DB: COMMIT
```

### 3. 重试机制

```java
private static final int DEFAULT_MAX_TRANSACTION_RETRY = 3;

// 自动重试事务冲突
public TransactionResult<IngestResult> ingestAspects(/* ... */) {
    for (int retry = 0; retry < maxRetries; retry++) {
        try {
            return executeTransaction(/* ... */);
        } catch (OptimisticLockException e) {
            if (retry == maxRetries - 1) throw e;
            // 指数退避重试
            Thread.sleep((1L << retry) * 100);
        }
    }
}
```

## 查询操作优化

### 1. 批量查询

DataHub实现了智能的批量查询机制：

```java
// 自动分批查询，避免SQL参数过多
private int queryKeysCount = 375; // 单次查询最大key数量

public Map<EntityAspectIdentifier, EntityAspect> batchGet(Set<EntityAspectIdentifier> keys) {
    if (queryKeysCount <= 0) {
        return batchGetIn(keys); // 单次查询
    } else {
        return batchGetPaginated(keys, queryKeysCount); // 分页查询
    }
}
```

### 2. SQL优化

**复合主键查询优化**：
```sql
-- 优化前：多个单独查询
SELECT * FROM metadata_aspect_v2 WHERE urn = ? AND aspect = ? AND version = ?

-- 优化后：复合IN查询
SELECT * FROM metadata_aspect_v2 
WHERE (urn, aspect, version) IN (
    ('urn1', 'aspect1', 0),
    ('urn2', 'aspect2', 0),
    ('urn3', 'aspect3', 1)
)
```

### 3. 缓存策略

```mermaid
flowchart LR
    A[查询请求] --> B{检查L1缓存}
    B -->|命中| C[返回缓存结果]
    B -->|未命中| D{检查L2缓存}
    D -->|命中| E[更新L1缓存]
    D -->|未命中| F[查询数据库]
    
    E --> C
    F --> G[更新所有缓存层]
    G --> C
    
    style B fill:#e8f5e8
    style D fill:#e8f5e8
    style F fill:#e1f5fe
```

## 性能监控和指标

### 1. 关键性能指标

DataHub集成了全面的性能监控：

```java
// 数据库操作耗时
MetricUtils.timer(MetricRegistry.TIMER_ASPECT_GET_TIME)
    .time(() -> executeQuery());

// 批量操作大小
MetricUtils.histogram(BATCH_SIZE_ATTR, batchSize);

// 错误率监控  
MetricUtils.counter(ASPECT_DAO_ERROR_COUNTER)
    .increment();
```

### 2. 监控维度

- **操作类型**：INSERT, UPDATE, SELECT, DELETE
- **批量大小**：单次操作的记录数量
- **响应时间**：数据库操作耗时分布
- **错误率**：失败操作的比例
- **并发度**：同时进行的数据库连接数

## 数据库优化建议

### 1. 索引策略

```sql
-- 主键索引（自动创建）
PRIMARY KEY (urn, aspect, version)

-- 查询优化索引
CREATE INDEX idx_metadata_aspect_v2_urn_aspect 
ON metadata_aspect_v2 (urn, aspect);

-- 时间范围查询索引
CREATE INDEX idx_metadata_aspect_v2_created_on 
ON metadata_aspect_v2 (createdOn);
```

### 2. 分区策略

```sql
-- 按时间分区（针对大量历史数据）
CREATE TABLE metadata_aspect_v2 (
    -- 字段定义
) PARTITION BY RANGE (YEAR(createdOn)) (
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);
```

### 3. 连接池优化

```yaml
# 数据库连接池配置
ebean:
  datasource:
    maximumPoolSize: 20
    minimumIdle: 5
    connectionTimeout: 30000
    idleTimeout: 600000
    maxLifetime: 1800000
```

## 故障处理和恢复

### 1. 常见故障场景

1. **死锁检测**：
   - 自动重试机制
   - 指数退避策略
   - 最大重试次数限制

2. **连接池耗尽**：
   - 连接泄漏检测
   - 自动连接回收
   - 监控告警

3. **大事务超时**：
   - 批量操作分片
   - 事务超时配置
   - 异步处理

### 2. 数据一致性保证

```mermaid
flowchart TD
    A[数据写入MySQL] --> B[事务提交]
    B --> C[发送Kafka事件]
    C --> D{Kafka发送成功？}
    D -->|是| E[完成]
    D -->|否| F[重试发送]
    F --> G{达到重试上限？}
    G -->|否| C
    G -->|是| H[记录失败日志]
    H --> I[人工干预]
```

## 总结

DataHub的MySQL操作设计体现了以下特点：

1. **高性能**：批量操作、连接池、缓存优化
2. **高可靠**：ACID事务、重试机制、故障恢复
3. **可扩展**：分片策略、异步处理、监控告警
4. **易维护**：清晰的分层架构、完善的日志记录

这种设计确保了DataHub能够高效、可靠地处理大规模元数据的存储和查询需求。