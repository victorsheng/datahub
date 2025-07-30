# DataHub 图数据使用场景

## 概述

DataHub 使用图数据库来存储和查询实体之间的关系。图数据在DataHub中扮演着核心角色，主要用于表示和查询数据资产之间的复杂关系网络。

## 主要使用场景

### 1. 数据血缘(Data Lineage)

数据血缘是图数据最重要的应用场景，用于追踪数据的来源、转换和去向。

```mermaid
graph LR
    subgraph "上游数据源"
        A[原始数据表A]
        B[原始数据表B]
    end
    
    subgraph "数据处理"
        C[ETL作业1]
        D[ETL作业2]
    end
    
    subgraph "下游数据"
        E[数据仓库表]
        F[数据集市表]
        G[报表/仪表板]
    end
    
    A -->|输入| C
    B -->|输入| C
    C -->|输出| E
    E -->|输入| D
    D -->|输出| F
    F -->|数据源| G
    
    style A fill:#f9f,stroke:#333,stroke-width:2px
    style B fill:#f9f,stroke:#333,stroke-width:2px
    style E fill:#bbf,stroke:#333,stroke-width:2px
    style F fill:#bbf,stroke:#333,stroke-width:2px
    style G fill:#bfb,stroke:#333,stroke-width:2px
```

#### 血缘查询示例

```java
// GraphService 血缘查询接口
public EntityLineageResult getLineage(
    Urn entityUrn,
    LineageDirection direction,  // UPSTREAM 或 DOWNSTREAM
    int maxDepth,
    int maxHops
) {
    // Neo4j Cypher查询示例
    String query = direction == LineageDirection.UPSTREAM ?
        "MATCH path=(start:Dataset {urn: $urn})<-[:DownstreamOf*1.." + maxHops + "]-(upstream) " +
        "RETURN path" :
        "MATCH path=(start:Dataset {urn: $urn})-[:DownstreamOf*1.." + maxHops + "]->(downstream) " +
        "RETURN path";
}
```

### 2. 影响分析(Impact Analysis)

当需要修改或删除某个数据资产时，通过图查询可以快速了解会影响哪些下游系统。

```mermaid
graph TD
    A[要修改的表] -->|影响| B[ETL作业1]
    A -->|影响| C[ETL作业2]
    B -->|影响| D[数据集市表1]
    C -->|影响| E[数据集市表2]
    D -->|影响| F[报表1]
    D -->|影响| G[报表2]
    E -->|影响| H[API服务]
    
    style A fill:#f99,stroke:#333,stroke-width:3px
    style B fill:#fbb,stroke:#333,stroke-width:2px
    style C fill:#fbb,stroke:#333,stroke-width:2px
    style D fill:#fdd,stroke:#333,stroke-width:1px
    style E fill:#fdd,stroke:#333,stroke-width:1px
```

### 3. 数据依赖关系

#### 3.1 表与字段级别的依赖

```java
// 细粒度血缘关系
FineGrainedLineage fineGrainedLineage = new FineGrainedLineage();
// 字段级别的血缘：tableA.column1 -> tableB.column2
fineGrainedLineage.setUpstreams(
    new UrnArray(ImmutableList.of(
        Urn.createFromString("urn:li:schemaField:(urn:li:dataset:tableA,column1)")
    ))
);
fineGrainedLineage.setDownstreams(
    new UrnArray(ImmutableList.of(
        Urn.createFromString("urn:li:schemaField:(urn:li:dataset:tableB,column2)")
    ))
);
```

#### 3.2 数据作业的输入输出关系

```mermaid
graph LR
    subgraph "输入数据集"
        A[Dataset A]
        B[Dataset B]
    end
    
    subgraph "数据作业"
        J[DataJob: daily_etl]
    end
    
    subgraph "输出数据集"
        C[Dataset C]
        D[Dataset D]
    end
    
    A -->|Consumes| J
    B -->|Consumes| J
    J -->|Produces| C
    J -->|Produces| D
```

### 4. 数据所有权和团队协作

图数据用于表示数据资产的所有权关系和团队协作网络。

```mermaid
graph TD
    subgraph "团队"
        T1[数据团队]
        T2[分析团队]
        U1[用户Alice]
        U2[用户Bob]
    end
    
    subgraph "数据资产"
        D1[核心数据表]
        D2[分析数据集]
        D3[报表]
    end
    
    T1 -->|拥有| D1
    T2 -->|拥有| D2
    T2 -->|拥有| D3
    U1 -->|属于| T1
    U2 -->|属于| T2
    U1 -->|管理| D1
    U2 -->|使用| D1
    U2 -->|管理| D2
```

### 5. 数据质量传播

通过图关系追踪数据质量问题的传播路径。

```java
// 查找受数据质量问题影响的下游资产
public Set<Urn> findAffectedDownstreams(Urn problematicDataset) {
    // 使用图查询找出所有下游依赖
    EntityLineageResult lineageResult = graphService.getLineage(
        problematicDataset,
        LineageDirection.DOWNSTREAM,
        Integer.MAX_VALUE,
        1000
    );
    
    // 返回所有可能受影响的下游资产
    return extractUrns(lineageResult);
}
```

### 6. 相似数据发现

基于图结构发现相似或相关的数据资产。

```mermaid
graph TD
    subgraph "共同上游"
        A[源数据表]
    end
    
    subgraph "相似数据集"
        B[衍生表1]
        C[衍生表2]
        D[衍生表3]
    end
    
    A --> B
    A --> C
    A --> D
    
    B -.相似.- C
    C -.相似.- D
    B -.相似.- D
```

### 7. 访问路径分析

分析用户如何通过不同路径访问数据。

```java
// 查找两个实体之间的所有路径
public List<Path> findPaths(Urn source, Urn target, int maxLength) {
    String query = 
        "MATCH path = shortestPath((source {urn: $sourceUrn})-[*.."+maxLength+"]-(target {urn: $targetUrn})) " +
        "RETURN path";
    
    // 执行查询并返回路径
    return executeCypherQuery(query, source, target);
}
```

## 图数据的优势

### 1. 性能优势
- **快速遍历**: O(1)时间复杂度的关系遍历
- **深度查询**: 支持多跳查询而不影响性能
- **模式匹配**: 高效的图模式匹配

### 2. 灵活性
- **动态模式**: 无需预定义固定的关系模式
- **复杂关系**: 轻松表示多对多关系
- **属性丰富**: 节点和边都可以有属性

### 3. 直观性
- **可视化**: 图结构天然适合可视化展示
- **易理解**: 关系网络直观易懂

## 实现细节

### Neo4j实现
```cypher
// 创建索引加速查询
CREATE INDEX ON :Dataset(urn);
CREATE INDEX ON :DataJob(urn);
CREATE INDEX ON :Dashboard(urn);

// 血缘关系查询
MATCH path = (start:Dataset {urn: $urn})-[:DownstreamOf*1..3]->(end)
WHERE end:Dataset OR end:Dashboard
RETURN path;
```

### ElasticSearch实现
```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "relationshipType": "DownstreamOf" } },
        { "term": { "source.urn": "urn:li:dataset:example" } }
      ]
    }
  }
}
```

## 最佳实践

1. **合理设置查询深度**: 避免无限深度查询
2. **使用索引**: 在关键属性上创建索引
3. **批量操作**: 批量更新关系以提高性能
4. **缓存策略**: 缓存常用的血缘查询结果
5. **异步更新**: 使用事件驱动方式异步更新图数据