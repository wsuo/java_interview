# Elasticsearch查询语句与实际应用场景

> 📊 深入理解Elasticsearch的查询机制和实际业务应用场景

## 🎯 核心概念

### 基础概念
- **Index（索引）**：类似于数据库中的database
- **Type（类型）**：类似于数据库中的table（7.x版本后已废弃）
- **Document（文档）**：类似于数据库中的row
- **Field（字段）**：类似于数据库中的column
- **Mapping（映射）**：定义文档结构和字段类型

### 查询上下文 vs 过滤上下文
```json
{
  "query": {
    "bool": {
      "must": [     // 查询上下文，计算相关性得分
        {"match": {"title": "elasticsearch"}}
      ],
      "filter": [   // 过滤上下文，不计算得分，可缓存
        {"term": {"status": "published"}},
        {"range": {"publish_date": {"gte": "2023-01-01"}}}
      ]
    }
  }
}
```

## 🔍 常用查询语句详解

### 1. 全文搜索查询

#### Match查询（最常用）
```json
// 基础match查询
{
  "query": {
    "match": {
      "title": "Java面试"
    }
  }
}

// match查询with操作符
{
  "query": {
    "match": {
      "content": {
        "query": "Spring Boot微服务",
        "operator": "and",        // 默认是or
        "minimum_should_match": "75%"
      }
    }
  }
}
```

#### Multi-Match查询
```json
{
  "query": {
    "multi_match": {
      "query": "Java开发",
      "fields": ["title^2", "content", "tags"],  // ^2表示title字段权重为2
      "type": "best_fields"  // 可选：most_fields, cross_fields, phrase, phrase_prefix
    }
  }
}
```

#### Match Phrase查询（短语匹配）
```json
{
  "query": {
    "match_phrase": {
      "content": {
        "query": "Spring Boot应用",
        "slop": 2  // 允许词语间隔2个位置
      }
    }
  }
}
```

### 2. 精确匹配查询

#### Term查询（精确匹配）
```json
{
  "query": {
    "term": {
      "status.keyword": "published"  // 注意：对于text字段需要使用.keyword
    }
  }
}

// Terms查询（多值匹配）
{
  "query": {
    "terms": {
      "tags.keyword": ["java", "spring", "elasticsearch"]
    }
  }
}
```

#### Range查询（范围查询）
```json
{
  "query": {
    "range": {
      "publish_date": {
        "gte": "2023-01-01",
        "lte": "2023-12-31",
        "format": "yyyy-MM-dd"
      }
    }
  }
}

// 数值范围查询
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lt": 1000
      }
    }
  }
}
```

### 3. 复合查询

#### Bool查询（最重要的复合查询）
```json
{
  "query": {
    "bool": {
      "must": [                    // 必须匹配，影响得分
        {"match": {"title": "Java"}}
      ],
      "must_not": [               // 必须不匹配
        {"term": {"status": "deleted"}}
      ],
      "should": [                 // 应该匹配，影响得分但不是必须的
        {"match": {"tags": "spring"}},
        {"range": {"views": {"gte": 1000}}}
      ],
      "filter": [                 // 必须匹配，但不影响得分
        {"term": {"category": "technology"}},
        {"range": {"publish_date": {"gte": "2023-01-01"}}}
      ],
      "minimum_should_match": 1   // should子句中至少匹配1个
    }
  }
}
```

### 4. 聚合查询

#### 基础聚合
```json
{
  "size": 0,  // 不返回文档，只返回聚合结果
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword",
        "size": 10
      }
    },
    "avg_price": {
      "avg": {
        "field": "price"
      }
    },
    "date_histogram": {
      "date_histogram": {
        "field": "publish_date",
        "calendar_interval": "month"
      }
    }
  }
}
```

#### 嵌套聚合
```json
{
  "size": 0,
  "aggs": {
    "categories": {
      "terms": {
        "field": "category.keyword"
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        },
        "top_products": {
          "top_hits": {
            "size": 3,
            "_source": ["title", "price"],
            "sort": [{"price": {"order": "desc"}}]
          }
        }
      }
    }
  }
}
```

## 🏢 实际业务应用场景

### 1. 电商搜索系统

#### 商品搜索实现
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "苹果手机",
            "fields": ["title^3", "description", "brand^2", "category"],
            "type": "best_fields",
            "tie_breaker": 0.3
          }
        }
      ],
      "filter": [
        {"range": {"price": {"gte": 1000, "lte": 8000}}},
        {"term": {"status": "available"}},
        {"terms": {"brand.keyword": ["Apple", "华为", "小米"]}}
      ]
    }
  },
  "sort": [
    {"_score": {"order": "desc"}},
    {"sales_count": {"order": "desc"}},
    {"price": {"order": "asc"}}
  ],
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [
          {"to": 2000},
          {"from": 2000, "to": 5000},
          {"from": 5000}
        ]
      }
    },
    "brands": {
      "terms": {
        "field": "brand.keyword",
        "size": 20
      }
    }
  }
}
```

### 2. 日志分析系统

#### 错误日志分析
```json
{
  "query": {
    "bool": {
      "must": [
        {"match": {"level": "ERROR"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  },
  "aggs": {
    "error_by_service": {
      "terms": {
        "field": "service.keyword",
        "size": 10
      },
      "aggs": {
        "error_timeline": {
          "date_histogram": {
            "field": "@timestamp",
            "fixed_interval": "5m"
          }
        }
      }
    },
    "top_errors": {
      "terms": {
        "field": "message.keyword",
        "size": 5
      }
    }
  }
}
```

### 3. 实时监控告警

#### 系统性能监控
```json
{
  "query": {
    "bool": {
      "filter": [
        {"range": {"@timestamp": {"gte": "now-5m"}}},
        {"term": {"metric_type": "system"}}
      ]
    }
  },
  "aggs": {
    "avg_cpu": {
      "avg": {
        "field": "cpu_usage"
      }
    },
    "max_memory": {
      "max": {
        "field": "memory_usage"
      }
    },
    "servers_over_threshold": {
      "filter": {
        "range": {"cpu_usage": {"gte": 80}}
      },
      "aggs": {
        "servers": {
          "terms": {
            "field": "hostname.keyword"
          }
        }
      }
    }
  }
}
```

## ⚡ 性能优化技巧

### 1. 索引优化

#### Mapping设计最佳实践
```json
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",
        "search_analyzer": "ik_smart",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "publish_date": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "tags": {
        "type": "keyword"  // 直接使用keyword，避免不必要的分析
      },
      "content": {
        "type": "text",
        "analyzer": "ik_max_word",
        "index_options": "positions"  // 支持短语查询
      }
    }
  }
}
```

### 2. 查询优化

#### 使用Filter Context
```json
// 优化前：全部使用must（查询上下文）
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "Java"}},
        {"term": {"status": "published"}},
        {"range": {"date": {"gte": "2023-01-01"}}}
      ]
    }
  }
}

// 优化后：合理使用filter（过滤上下文）
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "Java"}}  // 只有需要相关性得分的查询放在must中
      ],
      "filter": [
        {"term": {"status": "published"}},
        {"range": {"date": {"gte": "2023-01-01"}}}
      ]
    }
  }
}
```

#### 避免深度分页
```json
// 使用scroll API进行深度分页
POST /my_index/_search?scroll=1m
{
  "size": 1000,
  "query": {"match_all": {}},
  "sort": ["_doc"]
}

// 或使用search_after
{
  "size": 100,
  "query": {"match_all": {}},
  "sort": [
    {"timestamp": {"order": "desc"}},
    {"_id": {"order": "desc"}}
  ],
  "search_after": ["2023-01-01T00:00:00", "doc_id_123"]
}
```

### 3. 硬件和配置优化

#### JVM配置
```bash
# elasticsearch.yml
bootstrap.memory_lock: true

# jvm.options
-Xms4g
-Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

#### 索引设置优化
```json
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "30s",  // 降低刷新频率
    "index.translog.durability": "async",  // 异步事务日志
    "index.merge.policy.max_merged_segment": "5gb"
  }
}
```

## 🤝 与MySQL协作最佳实践

### 1. 数据同步策略

#### 实时同步方案
```java
@Service
public class DataSyncService {
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @Autowired
    private ProductRepository productRepository;
    
    // 新增产品时同步到ES
    @EventListener
    public void handleProductCreated(ProductCreatedEvent event) {
        Product product = event.getProduct();
        ProductDocument doc = convertToDocument(product);
        
        IndexRequest request = new IndexRequest("products")
            .id(product.getId().toString())
            .source(objectMapper.writeValueAsString(doc), XContentType.JSON);
            
        try {
            elasticsearchTemplate.index(request);
        } catch (Exception e) {
            // 记录失败，后续补偿
            log.error("Failed to sync product to ES: {}", product.getId(), e);
            // 可以发送到消息队列进行重试
        }
    }
    
    // 批量同步
    @Scheduled(fixedDelay = 300000) // 5分钟执行一次
    public void batchSync() {
        List<Product> products = productRepository.findModifiedSince(
            LocalDateTime.now().minusMinutes(5)
        );
        
        if (!products.isEmpty()) {
            BulkRequest bulkRequest = new BulkRequest();
            
            products.forEach(product -> {
                ProductDocument doc = convertToDocument(product);
                IndexRequest request = new IndexRequest("products")
                    .id(product.getId().toString())
                    .source(objectMapper.writeValueAsString(doc), XContentType.JSON);
                bulkRequest.add(request);
            });
            
            elasticsearchTemplate.bulk(bulkRequest);
        }
    }
}
```

### 2. 读写分离策略

#### 查询路由
```java
@Service
public class SearchService {
    
    // 复杂搜索使用ES
    public SearchResult searchProducts(SearchRequest request) {
        if (isComplexSearch(request)) {
            return searchFromElasticsearch(request);
        } else {
            // 简单查询使用MySQL
            return searchFromMySQL(request);
        }
    }
    
    private boolean isComplexSearch(SearchRequest request) {
        return request.hasFullTextSearch() || 
               request.hasAggregations() || 
               request.hasFacetFilters();
    }
}
```

### 3. 数据一致性保证

#### 最终一致性方案
```java
@Component
public class DataConsistencyChecker {
    
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点执行
    public void checkDataConsistency() {
        // 比较MySQL和ES中的数据
        List<Long> mysqlIds = productRepository.findAllIds();
        List<Long> esIds = getProductIdsFromES();
        
        // 找出不一致的数据
        Set<Long> missingInES = Sets.difference(
            new HashSet<>(mysqlIds), 
            new HashSet<>(esIds)
        );
        
        Set<Long> extraInES = Sets.difference(
            new HashSet<>(esIds), 
            new HashSet<>(mysqlIds)
        );
        
        // 修复不一致的数据
        repairMissingData(missingInES);
        removeExtraData(extraInES);
    }
}
```

## 🎯 面试常见问题与答案

### Q1: Elasticsearch的倒排索引是什么？
**答案：**
倒排索引是Elasticsearch的核心数据结构，它将文档中的每个词映射到包含该词的文档列表。

**结构示例：**
```
词项(Term) -> 文档列表(Document List)
"java"    -> [doc1, doc3, doc5]
"spring"  -> [doc1, doc2, doc4]
"boot"    -> [doc2, doc4, doc5]
```

**优势：**
- 快速的全文搜索
- 支持复杂的查询组合
- 高效的聚合计算

### Q2: 如何优化Elasticsearch的查询性能？
**答案：**
1. **合理使用Filter Context**：不需要相关性得分的查询使用filter
2. **避免深度分页**：使用scroll或search_after
3. **优化Mapping**：合理设置字段类型和分析器
4. **控制返回字段**：使用_source filtering
5. **预热常用查询**：使用warmer API

### Q3: Elasticsearch如何保证数据一致性？
**答案：**
Elasticsearch是最终一致性系统：
- **写一致性**：通过wait_for_active_shards参数控制
- **读一致性**：通过preference参数指定读取特定分片
- **版本控制**：使用_version字段进行乐观锁控制

### Q4: 如何设计Elasticsearch的分片策略？
**答案：**
1. **分片数量**：通常每个分片20-40GB
2. **副本策略**：至少1个副本保证高可用
3. **路由策略**：使用自定义routing提高查询效率
4. **时间序列数据**：使用索引模板和别名

### Q5: Elasticsearch集群如何进行容量规划？
**答案：**
1. **数据量评估**：原始数据大小 × 1.45（索引开销）
2. **内存规划**：堆内存不超过32GB，系统内存的50%给ES
3. **磁盘规划**：考虑副本和预留空间，实际需要2-3倍存储
4. **网络规划**：节点间通信带宽要充足

---

> 💡 **学习建议**：Elasticsearch的学习重点在于理解其分布式搜索的原理，多练习不同场景下的查询语句编写，关注性能优化和与其他系统的集成方案。
