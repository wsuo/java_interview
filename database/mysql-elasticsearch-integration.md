# MySQL与ElasticSearch协作存储详解

## 1. 架构设计概述

### 1.1 为什么需要MySQL + ElasticSearch？

**MySQL优势：**
- 强一致性
- 事务支持
- 复杂查询（JOIN、子查询）
- 数据完整性约束

**ElasticSearch优势：**
- 全文搜索
- 实时分析
- 水平扩展
- 复杂聚合查询

### 1.2 典型架构模式

```
应用层 → MySQL（主存储） → 数据同步 → ElasticSearch（搜索引擎）
         ↓                              ↓
      事务处理                      全文搜索/分析
```

## 2. 数据同步策略

### 2.1 同步方式对比

| 方式 | 优点 | 缺点 | 适用场景 |
|------|------|------|----------|
| 双写 | 实时性好 | 一致性难保证 | 对实时性要求高 |
| 异步同步 | 性能好 | 延迟同步 | 大部分场景 |
| CDC | 准实时、可靠 | 复杂度高 | 企业级应用 |
| 定时同步 | 简单 | 延迟大 | 对实时性要求低 |

### 2.2 双写模式实现

```java
@Service
@Transactional
public class ProductService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    public void createProduct(Product product) {
        try {
            // 1. 写入MySQL
            Product savedProduct = productRepository.save(product);
            
            // 2. 写入ElasticSearch
            ProductDocument document = convertToDocument(savedProduct);
            elasticsearchTemplate.save(document);
            
        } catch (Exception e) {
            // 处理同步失败
            handleSyncFailure(product, e);
            throw new RuntimeException("Product creation failed", e);
        }
    }
    
    private void handleSyncFailure(Product product, Exception e) {
        // 记录同步失败的数据，用于后续补偿
        FailedSyncRecord record = new FailedSyncRecord();
        record.setEntityType("Product");
        record.setEntityId(product.getId());
        record.setOperation("CREATE");
        record.setRetryCount(0);
        
        failedSyncRepository.save(record);
        
        // 发送告警
        alertService.sendAlert("ElasticSearch sync failed", e.getMessage());
    }
}
```

### 2.3 异步同步实现

```java
@Component
public class DataSyncService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @Async
    @EventListener
    public void handleProductCreated(ProductCreatedEvent event) {
        try {
            Product product = productRepository.findById(event.getProductId())
                .orElseThrow(() -> new EntityNotFoundException("Product not found"));
            
            ProductDocument document = convertToDocument(product);
            elasticsearchTemplate.save(document);
            
        } catch (Exception e) {
            log.error("Failed to sync product to ElasticSearch", e);
            // 重试逻辑
            scheduleRetry(event);
        }
    }
    
    @Retryable(value = Exception.class, maxAttempts = 3, backoff = @Backoff(delay = 2000))
    public void syncToElasticsearch(Product product) {
        ProductDocument document = convertToDocument(product);
        elasticsearchTemplate.save(document);
    }
}
```

### 2.4 CDC（Change Data Capture）同步

**使用Canal同步：**
```java
@Component
public class CanalClient {
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @PostConstruct
    public void startCanal() {
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("localhost", 11111), 
            "example", 
            "", 
            ""
        );
        
        new Thread(() -> {
            while (true) {
                try {
                    connector.connect();
                    connector.subscribe(".*\\..*");
                    
                    while (true) {
                        Message message = connector.getWithoutAck(100);
                        processMessage(message);
                        connector.ack(message.getId());
                    }
                } catch (Exception e) {
                    log.error("Canal sync error", e);
                }
            }
        }).start();
    }
    
    private void processMessage(Message message) {
        for (CanalEntry.Entry entry : message.getEntries()) {
            if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
                CanalEntry.RowChange rowChange = CanalEntry.RowChange.parseFrom(entry.getStoreValue());
                
                switch (rowChange.getEventType()) {
                    case INSERT:
                        handleInsert(entry.getHeader(), rowChange);
                        break;
                    case UPDATE:
                        handleUpdate(entry.getHeader(), rowChange);
                        break;
                    case DELETE:
                        handleDelete(entry.getHeader(), rowChange);
                        break;
                }
            }
        }
    }
    
    private void handleInsert(CanalEntry.Header header, CanalEntry.RowChange rowChange) {
        String tableName = header.getTableName();
        
        for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
            if ("products".equals(tableName)) {
                syncProductToES(rowData.getAfterColumnsList(), "CREATE");
            }
        }
    }
}
```

## 3. 数据一致性保证

### 3.1 最终一致性实现

```java
@Component
@Scheduled(fixedDelay = 60000) // 每分钟检查一次
public class DataConsistencyChecker {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    public void checkDataConsistency() {
        // 1. 获取最近更新的数据
        List<Product> recentProducts = productRepository.findByUpdateTimeAfter(
            LocalDateTime.now().minusMinutes(5)
        );
        
        for (Product product : recentProducts) {
            try {
                // 2. 检查ES中是否存在
                ProductDocument document = elasticsearchTemplate.get(
                    product.getId().toString(), 
                    ProductDocument.class
                );
                
                if (document == null) {
                    // 3. 不存在则同步
                    syncToElasticsearch(product);
                } else {
                    // 4. 检查数据是否一致
                    if (!isDataConsistent(product, document)) {
                        syncToElasticsearch(product);
                    }
                }
            } catch (Exception e) {
                log.error("Consistency check failed for product {}", product.getId(), e);
            }
        }
    }
    
    private boolean isDataConsistent(Product product, ProductDocument document) {
        return product.getName().equals(document.getName()) &&
               product.getPrice().equals(document.getPrice()) &&
               product.getUpdateTime().equals(document.getUpdateTime());
    }
}
```

### 3.2 分布式事务实现

```java
@Service
public class ProductTransactionService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @Autowired
    private TransactionManager transactionManager;
    
    public void createProductWithTransaction(Product product) {
        TransactionStatus status = transactionManager.getTransaction(
            new DefaultTransactionDefinition()
        );
        
        try {
            // 1. 写入MySQL
            Product savedProduct = productRepository.save(product);
            
            // 2. 写入ElasticSearch
            ProductDocument document = convertToDocument(savedProduct);
            elasticsearchTemplate.save(document);
            
            // 3. 提交事务
            transactionManager.commit(status);
            
        } catch (Exception e) {
            // 4. 回滚事务
            transactionManager.rollback(status);
            
            // 5. 清理ElasticSearch中的数据
            try {
                elasticsearchTemplate.delete(ProductDocument.class, product.getId().toString());
            } catch (Exception cleanupException) {
                log.error("Failed to cleanup ElasticSearch data", cleanupException);
            }
            
            throw new RuntimeException("Transaction failed", e);
        }
    }
}
```

## 4. 事务问题解决方案

### 4.1 两阶段提交（2PC）

```java
@Component
public class TwoPhaseCommitManager {
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    public void executeTransaction(List<TransactionOperation> operations) {
        String transactionId = UUID.randomUUID().toString();
        
        // Phase 1: Prepare
        boolean allPrepared = true;
        for (TransactionOperation operation : operations) {
            try {
                operation.prepare(transactionId);
            } catch (Exception e) {
                allPrepared = false;
                log.error("Prepare phase failed for operation {}", operation.getClass().getSimpleName(), e);
                break;
            }
        }
        
        // Phase 2: Commit or Rollback
        if (allPrepared) {
            // All prepared, commit
            for (TransactionOperation operation : operations) {
                try {
                    operation.commit(transactionId);
                } catch (Exception e) {
                    log.error("Commit phase failed for operation {}", operation.getClass().getSimpleName(), e);
                    // 需要人工介入处理
                    notifyAdministrator(transactionId, e);
                }
            }
        } else {
            // Rollback all
            for (TransactionOperation operation : operations) {
                try {
                    operation.rollback(transactionId);
                } catch (Exception e) {
                    log.error("Rollback phase failed for operation {}", operation.getClass().getSimpleName(), e);
                }
            }
        }
    }
}

// 具体操作实现
@Component
public class ProductMySQLOperation implements TransactionOperation {
    
    @Autowired
    private ProductRepository productRepository;
    
    private Map<String, Product> pendingProducts = new ConcurrentHashMap<>();
    
    @Override
    public void prepare(String transactionId) {
        // 验证数据有效性，预留资源
        Product product = pendingProducts.get(transactionId);
        if (product == null) {
            throw new IllegalStateException("No product found for transaction " + transactionId);
        }
        
        // 检查约束条件
        if (productRepository.existsByName(product.getName())) {
            throw new IllegalStateException("Product name already exists");
        }
    }
    
    @Override
    public void commit(String transactionId) {
        Product product = pendingProducts.remove(transactionId);
        productRepository.save(product);
    }
    
    @Override
    public void rollback(String transactionId) {
        pendingProducts.remove(transactionId);
    }
}
```

### 4.2 Saga模式

```java
@Component
public class ProductSagaOrchestrator {
    
    @Autowired
    private ProductService productService;
    
    @Autowired
    private ElasticsearchSyncService elasticsearchSyncService;
    
    @Autowired
    private NotificationService notificationService;
    
    public void createProductSaga(Product product) {
        SagaTransaction saga = SagaTransaction.builder()
            .transactionId(UUID.randomUUID().toString())
            .build();
        
        try {
            // Step 1: Create product in MySQL
            Product savedProduct = productService.createProduct(product);
            saga.addCompensation(() -> productService.deleteProduct(savedProduct.getId()));
            
            // Step 2: Sync to ElasticSearch
            elasticsearchSyncService.syncProduct(savedProduct);
            saga.addCompensation(() -> elasticsearchSyncService.deleteProduct(savedProduct.getId()));
            
            // Step 3: Send notification
            notificationService.sendProductCreatedNotification(savedProduct);
            saga.addCompensation(() -> notificationService.sendProductDeletedNotification(savedProduct));
            
            saga.markAsCompleted();
            
        } catch (Exception e) {
            log.error("Saga transaction failed", e);
            saga.compensate();
            throw new RuntimeException("Product creation saga failed", e);
        }
    }
}

// Saga事务管理
public class SagaTransaction {
    private String transactionId;
    private List<Runnable> compensations = new ArrayList<>();
    private boolean completed = false;
    
    public void addCompensation(Runnable compensation) {
        compensations.add(compensation);
    }
    
    public void compensate() {
        if (!completed) {
            // 逆序执行补偿操作
            Collections.reverse(compensations);
            for (Runnable compensation : compensations) {
                try {
                    compensation.run();
                } catch (Exception e) {
                    log.error("Compensation failed", e);
                }
            }
        }
    }
    
    public void markAsCompleted() {
        this.completed = true;
    }
}
```

## 5. 查询策略

### 5.1 读写分离

```java
@Service
public class ProductQueryService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    // 精确查询走MySQL
    public Product findById(Long id) {
        return productRepository.findById(id)
            .orElseThrow(() -> new EntityNotFoundException("Product not found"));
    }
    
    // 全文搜索走ElasticSearch
    public List<ProductDocument> search(String keyword, int page, int size) {
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.multiMatchQuery(keyword, "name", "description"))
            .withPageable(PageRequest.of(page, size))
            .withHighlightFields(
                new HighlightBuilder.Field("name"),
                new HighlightBuilder.Field("description")
            );
        
        SearchHits<ProductDocument> searchHits = elasticsearchTemplate.search(
            queryBuilder.build(), 
            ProductDocument.class
        );
        
        return searchHits.stream()
            .map(SearchHit::getContent)
            .collect(Collectors.toList());
    }
    
    // 复杂聚合查询走ElasticSearch
    public Map<String, Long> getCategoryStatistics() {
        NativeSearchQueryBuilder queryBuilder = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.matchAllQuery())
            .withAggregations(
                AggregationBuilders.terms("category_stats").field("category.keyword")
            );
        
        SearchResponse response = elasticsearchTemplate.search(
            queryBuilder.build(), 
            ProductDocument.class
        ).getAggregations().get("category_stats");
        
        Terms terms = response.getAggregations().get("category_stats");
        return terms.getBuckets().stream()
            .collect(Collectors.toMap(
                bucket -> bucket.getKeyAsString(),
                bucket -> bucket.getDocCount()
            ));
    }
}
```

### 5.2 查询降级策略

```java
@Service
public class ProductSearchService {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @Autowired
    private CircuitBreaker circuitBreaker;
    
    public List<Product> searchProducts(String keyword) {
        return circuitBreaker.executeSupplier(() -> {
            // 优先使用ElasticSearch
            return searchFromElasticsearch(keyword);
        }).recover(throwable -> {
            // 降级到MySQL
            log.warn("ElasticSearch search failed, falling back to MySQL", throwable);
            return searchFromMySQL(keyword);
        });
    }
    
    private List<Product> searchFromElasticsearch(String keyword) {
        // ElasticSearch搜索实现
        List<ProductDocument> documents = elasticsearchTemplate.search(
            QueryBuilders.multiMatchQuery(keyword, "name", "description"),
            ProductDocument.class
        ).stream()
        .map(SearchHit::getContent)
        .collect(Collectors.toList());
        
        return documents.stream()
            .map(this::convertToProduct)
            .collect(Collectors.toList());
    }
    
    private List<Product> searchFromMySQL(String keyword) {
        // MySQL模糊查询作为降级方案
        return productRepository.findByNameContainingOrDescriptionContaining(keyword, keyword);
    }
}
```

## 6. 监控和运维

### 6.1 数据同步监控

```java
@Component
public class SyncMonitor {
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    private final Counter syncSuccessCounter;
    private final Counter syncFailureCounter;
    private final Timer syncTimer;
    
    public SyncMonitor(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.syncSuccessCounter = Counter.builder("sync.success")
            .description("Successful sync operations")
            .register(meterRegistry);
        this.syncFailureCounter = Counter.builder("sync.failure")
            .description("Failed sync operations")
            .register(meterRegistry);
        this.syncTimer = Timer.builder("sync.duration")
            .description("Sync operation duration")
            .register(meterRegistry);
    }
    
    public void recordSyncSuccess() {
        syncSuccessCounter.increment();
    }
    
    public void recordSyncFailure() {
        syncFailureCounter.increment();
    }
    
    public Timer.Sample startSyncTimer() {
        return Timer.start(meterRegistry);
    }
    
    public void stopSyncTimer(Timer.Sample sample) {
        sample.stop(syncTimer);
    }
}
```

### 6.2 健康检查

```java
@Component
public class DataSyncHealthIndicator implements HealthIndicator {
    
    @Autowired
    private ProductRepository productRepository;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @Override
    public Health health() {
        try {
            // 检查MySQL连接
            long mysqlCount = productRepository.count();
            
            // 检查ElasticSearch连接
            long esCount = elasticsearchTemplate.count(
                QueryBuilders.matchAllQuery(), 
                ProductDocument.class
            );
            
            // 检查数据一致性
            double consistencyRatio = (double) esCount / mysqlCount;
            
            if (consistencyRatio < 0.95) {
                return Health.down()
                    .withDetail("mysql_count", mysqlCount)
                    .withDetail("elasticsearch_count", esCount)
                    .withDetail("consistency_ratio", consistencyRatio)
                    .withDetail("message", "Data consistency issue detected")
                    .build();
            }
            
            return Health.up()
                .withDetail("mysql_count", mysqlCount)
                .withDetail("elasticsearch_count", esCount)
                .withDetail("consistency_ratio", consistencyRatio)
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withDetail("error", e.getMessage())
                .build();
        }
    }
}
```

## 7. 面试常见问题

### Q1: 如何保证MySQL和ElasticSearch的数据一致性？

**答案：**
1. **最终一致性**：通过定时任务检查和修复数据不一致
2. **分布式事务**：使用2PC或Saga模式
3. **补偿机制**：记录失败操作，后续重试
4. **监控告警**：及时发现数据不一致问题

### Q2: 双写模式的问题和解决方案？

**问题：**
- 数据不一致
- 性能影响
- 代码复杂度增加

**解决方案：**
```java
@Transactional
public void createProduct(Product product) {
    // 1. 先写MySQL（主存储）
    Product savedProduct = productRepository.save(product);
    
    // 2. 异步写ElasticSearch
    applicationEventPublisher.publishEvent(
        new ProductCreatedEvent(savedProduct.getId())
    );
}
```

### Q3: 如何处理ElasticSearch写入失败？

**解决方案：**
1. **重试机制**：使用@Retryable注解
2. **死信队列**：失败消息进入死信队列
3. **补偿任务**：定时任务重新同步失败数据
4. **监控告警**：及时发现并处理失败

### Q4: 查询时如何选择MySQL还是ElasticSearch？

**选择策略：**
- **精确查询**：MySQL（根据ID、状态查询）
- **全文搜索**：ElasticSearch（关键词搜索）
- **聚合分析**：ElasticSearch（统计分析）
- **复杂关联**：MySQL（多表JOIN）

### Q5: 如何优化同步性能？

**优化方案：**
1. **批量同步**：批量写入ElasticSearch
2. **异步处理**：使用消息队列解耦
3. **数据压缩**：减少网络传输
4. **连接池优化**：复用连接资源

```java
@Service
public class BatchSyncService {
    
    private final List<ProductDocument> buffer = new ArrayList<>();
    private final int BATCH_SIZE = 1000;
    
    @Scheduled(fixedDelay = 5000) // 每5秒批量同步
    public void batchSync() {
        if (!buffer.isEmpty()) {
            List<ProductDocument> toSync = new ArrayList<>(buffer);
            buffer.clear();
            
            elasticsearchTemplate.save(toSync);
        }
    }
}
```