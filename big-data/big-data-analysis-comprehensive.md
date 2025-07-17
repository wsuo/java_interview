# 大数据分析技术栈全面解析

## 1. 大数据分析概述

### 1.1 大数据的5V特征

- **Volume（体量）**：数据规模庞大（TB、PB级别）
- **Velocity（速度）**：数据产生和处理速度快
- **Variety（多样性）**：数据类型多样（结构化、半结构化、非结构化）
- **Veracity（真实性）**：数据质量和准确性
- **Value（价值）**：数据的商业价值

### 1.2 大数据分析架构

```
数据源 → 数据采集 → 数据存储 → 数据处理 → 数据分析 → 数据可视化 → 数据应用
  ↓         ↓         ↓         ↓         ↓         ↓         ↓
业务系统   Flume     HDFS      Spark     Spark     Zeppelin   BI系统
日志文件   Kafka     HBase     Flink     Flink     Grafana    推荐系统
物联网     Sqoop     Hive      Storm     Hive      Tableau    风控系统
```

## 2. 数据采集层

### 2.1 Apache Kafka

**核心概念：**
- **Topic**：消息主题
- **Partition**：分区，提高并发性
- **Producer**：生产者
- **Consumer**：消费者
- **Broker**：Kafka服务器节点

**Java实现示例：**
```java
// Producer示例
@Service
public class KafkaProducerService {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    public void sendMessage(String topic, String message) {
        kafkaTemplate.send(topic, message)
            .addCallback(new ListenableFutureCallback<SendResult<String, String>>() {
                @Override
                public void onSuccess(SendResult<String, String> result) {
                    System.out.println("Message sent successfully: " + result.getRecordMetadata().offset());
                }
                
                @Override
                public void onFailure(Throwable ex) {
                    System.err.println("Failed to send message: " + ex.getMessage());
                }
            });
    }
}

// Consumer示例
@Component
public class KafkaConsumerService {
    
    @KafkaListener(topics = "user-events", groupId = "analytics-group")
    public void consumeUserEvents(String message) {
        try {
            UserEvent event = JSON.parseObject(message, UserEvent.class);
            processUserEvent(event);
        } catch (Exception e) {
            System.err.println("Error processing message: " + e.getMessage());
        }
    }
    
    private void processUserEvent(UserEvent event) {
        // 处理用户事件
        System.out.println("Processing user event: " + event.getEventType());
    }
}
```

### 2.2 Apache Flume

**核心组件：**
- **Source**：数据源
- **Channel**：数据通道
- **Sink**：数据目的地

**配置示例：**
```properties
# Flume配置
agent.sources = r1
agent.sinks = k1
agent.channels = c1

# Source配置
agent.sources.r1.type = spooldir
agent.sources.r1.spoolDir = /data/logs
agent.sources.r1.channels = c1

# Channel配置
agent.channels.c1.type = memory
agent.channels.c1.capacity = 10000
agent.channels.c1.transactionCapacity = 1000

# Sink配置
agent.sinks.k1.type = hdfs
agent.sinks.k1.hdfs.path = /data/warehouse/logs
agent.sinks.k1.hdfs.fileType = DataStream
agent.sinks.k1.channel = c1
```

### 2.3 Apache Sqoop

**数据迁移工具：**
```bash
# 从MySQL导入到HDFS
sqoop import \
  --connect jdbc:mysql://localhost:3306/ecommerce \
  --username root \
  --password password \
  --table orders \
  --target-dir /data/warehouse/orders \
  --num-mappers 4

# 从HDFS导出到MySQL
sqoop export \
  --connect jdbc:mysql://localhost:3306/ecommerce \
  --username root \
  --password password \
  --table order_summary \
  --export-dir /data/warehouse/order_summary
```

## 3. 数据存储层

### 3.1 Hadoop HDFS

**核心概念：**
- **NameNode**：元数据管理
- **DataNode**：数据存储
- **Block**：数据块（默认128MB）
- **Replication**：数据副本（默认3份）

**Java API示例：**
```java
@Service
public class HDFSService {
    
    private FileSystem fileSystem;
    
    @PostConstruct
    public void init() throws IOException {
        Configuration conf = new Configuration();
        conf.set("fs.defaultFS", "hdfs://localhost:9000");
        this.fileSystem = FileSystem.get(conf);
    }
    
    public void uploadFile(String localPath, String hdfsPath) throws IOException {
        Path src = new Path(localPath);
        Path dst = new Path(hdfsPath);
        
        fileSystem.copyFromLocalFile(src, dst);
        System.out.println("File uploaded to HDFS: " + hdfsPath);
    }
    
    public void downloadFile(String hdfsPath, String localPath) throws IOException {
        Path src = new Path(hdfsPath);
        Path dst = new Path(localPath);
        
        fileSystem.copyToLocalFile(src, dst);
        System.out.println("File downloaded from HDFS: " + hdfsPath);
    }
    
    public List<String> listFiles(String hdfsPath) throws IOException {
        Path path = new Path(hdfsPath);
        FileStatus[] statuses = fileSystem.listStatus(path);
        
        return Arrays.stream(statuses)
            .map(status -> status.getPath().getName())
            .collect(Collectors.toList());
    }
}
```

### 3.2 Apache HBase

**NoSQL数据库特点：**
- **列族存储**：按列族组织数据
- **稀疏矩阵**：支持稀疏数据
- **时间戳**：多版本数据
- **自动分片**：Region自动分裂

**Java API示例：**
```java
@Service
public class HBaseService {
    
    private Connection connection;
    
    @PostConstruct
    public void init() throws IOException {
        Configuration conf = HBaseConfiguration.create();
        conf.set("hbase.zookeeper.quorum", "localhost");
        conf.set("hbase.zookeeper.property.clientPort", "2181");
        this.connection = ConnectionFactory.createConnection(conf);
    }
    
    public void createTable(String tableName, String... columnFamilies) throws IOException {
        Admin admin = connection.getAdmin();
        
        TableName table = TableName.valueOf(tableName);
        if (!admin.tableExists(table)) {
            HTableDescriptor descriptor = new HTableDescriptor(table);
            
            for (String cf : columnFamilies) {
                descriptor.addFamily(new HColumnDescriptor(cf));
            }
            
            admin.createTable(descriptor);
            System.out.println("Table created: " + tableName);
        }
    }
    
    public void putData(String tableName, String rowKey, String columnFamily, 
                       String column, String value) throws IOException {
        Table table = connection.getTable(TableName.valueOf(tableName));
        
        Put put = new Put(Bytes.toBytes(rowKey));
        put.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(column), Bytes.toBytes(value));
        
        table.put(put);
        table.close();
    }
    
    public String getData(String tableName, String rowKey, String columnFamily, 
                         String column) throws IOException {
        Table table = connection.getTable(TableName.valueOf(tableName));
        
        Get get = new Get(Bytes.toBytes(rowKey));
        get.addColumn(Bytes.toBytes(columnFamily), Bytes.toBytes(column));
        
        Result result = table.get(get);
        byte[] value = result.getValue(Bytes.toBytes(columnFamily), Bytes.toBytes(column));
        
        table.close();
        return value != null ? Bytes.toString(value) : null;
    }
}
```

### 3.3 Apache Hive

**数据仓库工具：**
```sql
-- 创建外部表
CREATE EXTERNAL TABLE user_events (
    user_id BIGINT,
    event_type STRING,
    timestamp BIGINT,
    properties MAP<STRING, STRING>
)
PARTITIONED BY (dt STRING)
STORED AS PARQUET
LOCATION '/data/warehouse/user_events';

-- 添加分区
ALTER TABLE user_events ADD PARTITION (dt='2024-01-01') 
LOCATION '/data/warehouse/user_events/dt=2024-01-01';

-- 分析查询
SELECT 
    event_type,
    COUNT(*) as event_count,
    COUNT(DISTINCT user_id) as unique_users
FROM user_events
WHERE dt >= '2024-01-01' AND dt <= '2024-01-31'
GROUP BY event_type
ORDER BY event_count DESC;
```

## 4. 数据处理层

### 4.1 Apache Spark

**核心概念：**
- **RDD**：弹性分布式数据集
- **DataFrame**：结构化数据抽象
- **Dataset**：类型安全的DataFrame
- **Spark SQL**：结构化数据处理

**Java示例：**
```java
@Service
public class SparkService {
    
    private SparkSession spark;
    
    @PostConstruct
    public void init() {
        this.spark = SparkSession.builder()
            .appName("BigDataAnalysis")
            .master("local[*]")
            .config("spark.sql.adaptive.enabled", "true")
            .getOrCreate();
    }
    
    public void analyzeUserBehavior() {
        // 读取数据
        Dataset<Row> events = spark.read()
            .format("parquet")
            .load("/data/warehouse/user_events");
        
        // 用户行为分析
        Dataset<Row> result = events
            .filter(col("event_type").equalTo("page_view"))
            .groupBy(col("user_id"))
            .agg(
                count("*").alias("page_views"),
                countDistinct("page_id").alias("unique_pages"),
                max("timestamp").alias("last_activity")
            )
            .filter(col("page_views").gt(10));
        
        // 写入结果
        result.write()
            .mode(SaveMode.Overwrite)
            .format("parquet")
            .save("/data/warehouse/user_behavior_analysis");
    }
    
    public void realTimeProcessing() {
        // 读取Kafka流
        Dataset<Row> kafkaStream = spark.readStream()
            .format("kafka")
            .option("kafka.bootstrap.servers", "localhost:9092")
            .option("subscribe", "user-events")
            .load();
        
        // 处理流数据
        Dataset<Row> processedStream = kafkaStream
            .select(
                from_json(col("value").cast("string"), getEventSchema()).alias("event")
            )
            .select("event.*")
            .withWatermark("timestamp", "10 minutes")
            .groupBy(
                window(col("timestamp"), "5 minutes"),
                col("event_type")
            )
            .count();
        
        // 输出到控制台
        StreamingQuery query = processedStream.writeStream()
            .outputMode("update")
            .format("console")
            .trigger(Trigger.ProcessingTime("30 seconds"))
            .start();
    }
}
```

### 4.2 Apache Flink

**流处理框架：**
```java
@Component
public class FlinkProcessor {
    
    public void processUserEvents() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        
        // 配置Kafka消费者
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "localhost:9092");
        props.setProperty("group.id", "flink-consumer");
        
        FlinkKafkaConsumer<String> consumer = new FlinkKafkaConsumer<>(
            "user-events", 
            new SimpleStringSchema(), 
            props
        );
        
        // 创建数据流
        DataStream<UserEvent> eventStream = env.addSource(consumer)
            .map(new MapFunction<String, UserEvent>() {
                @Override
                public UserEvent map(String value) throws Exception {
                    return JSON.parseObject(value, UserEvent.class);
                }
            });
        
        // 实时统计
        DataStream<EventCount> countStream = eventStream
            .keyBy(UserEvent::getEventType)
            .window(TumblingProcessingTimeWindows.of(Time.minutes(5)))
            .aggregate(new EventCountAggregator());
        
        // 输出到Kafka
        FlinkKafkaProducer<EventCount> producer = new FlinkKafkaProducer<>(
            "event-counts",
            new EventCountSerializer(),
            props
        );
        
        countStream.addSink(producer);
        
        env.execute("User Event Processing");
    }
}
```

## 5. 数据分析层

### 5.1 机器学习

**Spark MLlib示例：**
```java
@Service
public class MLService {
    
    @Autowired
    private SparkSession spark;
    
    public void userRecommendation() {
        // 读取用户行为数据
        Dataset<Row> ratings = spark.read()
            .format("parquet")
            .load("/data/warehouse/user_ratings");
        
        // 构建推荐模型
        ALS als = new ALS()
            .setMaxIter(10)
            .setRegParam(0.1)
            .setUserCol("user_id")
            .setItemCol("item_id")
            .setRatingCol("rating");
        
        ALSModel model = als.fit(ratings);
        
        // 生成推荐
        Dataset<Row> recommendations = model.recommendForAllUsers(10);
        
        // 保存推荐结果
        recommendations.write()
            .mode(SaveMode.Overwrite)
            .format("parquet")
            .save("/data/warehouse/user_recommendations");
    }
    
    public void fraudDetection() {
        // 读取交易数据
        Dataset<Row> transactions = spark.read()
            .format("parquet")
            .load("/data/warehouse/transactions");
        
        // 特征工程
        VectorAssembler assembler = new VectorAssembler()
            .setInputCols(new String[]{"amount", "merchant_category", "time_of_day"})
            .setOutputCol("features");
        
        Dataset<Row> featureData = assembler.transform(transactions);
        
        // 训练随机森林模型
        RandomForestClassifier rf = new RandomForestClassifier()
            .setLabelCol("is_fraud")
            .setFeaturesCol("features")
            .setNumTrees(100);
        
        RandomForestClassificationModel model = rf.fit(featureData);
        
        // 预测
        Dataset<Row> predictions = model.transform(featureData);
        
        // 评估模型
        BinaryClassificationEvaluator evaluator = new BinaryClassificationEvaluator()
            .setLabelCol("is_fraud")
            .setRawPredictionCol("rawPrediction");
        
        double auc = evaluator.evaluate(predictions);
        System.out.println("AUC: " + auc);
    }
}
```

### 5.2 实时分析

**实时仪表板：**
```java
@RestController
@RequestMapping("/api/analytics")
public class AnalyticsController {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Autowired
    private ElasticsearchTemplate elasticsearchTemplate;
    
    @GetMapping("/realtime/metrics")
    public ResponseEntity<Map<String, Object>> getRealtimeMetrics() {
        Map<String, Object> metrics = new HashMap<>();
        
        // 从Redis获取实时指标
        metrics.put("online_users", redisTemplate.opsForValue().get("online_users"));
        metrics.put("current_pv", redisTemplate.opsForValue().get("current_pv"));
        metrics.put("current_uv", redisTemplate.opsForValue().get("current_uv"));
        
        // 从Elasticsearch获取趋势数据
        NativeSearchQuery query = new NativeSearchQueryBuilder()
            .withQuery(QueryBuilders.rangeQuery("timestamp").gte("now-1h"))
            .withAggregations(
                AggregationBuilders.dateHistogram("trend")
                    .field("timestamp")
                    .interval(DateHistogramInterval.MINUTE)
            )
            .build();
        
        SearchResponse response = elasticsearchTemplate.search(query, Object.class);
        
        // 处理聚合结果
        Histogram histogram = response.getAggregations().get("trend");
        List<Map<String, Object>> trend = histogram.getBuckets().stream()
            .map(bucket -> {
                Map<String, Object> point = new HashMap<>();
                point.put("timestamp", bucket.getKey());
                point.put("count", bucket.getDocCount());
                return point;
            })
            .collect(Collectors.toList());
        
        metrics.put("trend", trend);
        
        return ResponseEntity.ok(metrics);
    }
    
    @GetMapping("/reports/daily")
    public ResponseEntity<DailyReport> getDailyReport(@RequestParam String date) {
        // 从数据仓库获取日报数据
        Dataset<Row> dailyData = spark.read()
            .format("parquet")
            .load("/data/warehouse/daily_reports")
            .filter(col("date").equalTo(date));
        
        DailyReport report = new DailyReport();
        report.setDate(date);
        report.setTotalUsers(dailyData.filter(col("metric").equalTo("total_users")).first().getLong(2));
        report.setTotalOrders(dailyData.filter(col("metric").equalTo("total_orders")).first().getLong(2));
        report.setRevenue(dailyData.filter(col("metric").equalTo("revenue")).first().getDouble(2));
        
        return ResponseEntity.ok(report);
    }
}
```

## 6. 数据可视化

### 6.1 Apache Zeppelin

**交互式数据分析：**
```scala
// Zeppelin Notebook示例
%spark2.sql
SELECT 
    DATE(timestamp) as date,
    event_type,
    COUNT(*) as event_count
FROM user_events
WHERE timestamp >= '2024-01-01'
GROUP BY DATE(timestamp), event_type
ORDER BY date, event_count DESC

%spark2.scala
// 用户留存分析
val userEvents = spark.read.parquet("/data/warehouse/user_events")

val retention = userEvents
  .filter($"event_type" === "login")
  .groupBy("user_id")
  .agg(
    min("timestamp").alias("first_login"),
    max("timestamp").alias("last_login"),
    count("*").alias("login_count")
  )
  .withColumn("retention_days", 
    datediff(col("last_login"), col("first_login")))

retention.show()
```

### 6.2 Grafana集成

**监控配置：**
```yaml
# docker-compose.yml
version: '3.8'
services:
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana-storage:/var/lib/grafana
    depends_on:
      - prometheus
      - elasticsearch

  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

## 7. 面试常见问题

### Q1: 什么是Lambda架构和Kappa架构？

**Lambda架构：**
- **批处理层**：处理历史数据
- **流处理层**：处理实时数据
- **服务层**：合并批处理和流处理结果

**Kappa架构：**
- 只有流处理层
- 所有数据都作为流处理
- 简化了架构复杂性

### Q2: Spark和Flink的区别？

| 特性 | Spark | Flink |
|------|-------|-------|
| 处理模型 | 微批处理 | 流处理 |
| 延迟 | 秒级 | 毫秒级 |
| 吞吐量 | 高 | 中等 |
| 状态管理 | 简单 | 复杂 |
| 易用性 | 高 | 中等 |

### Q3: 如何保证数据质量？

**数据质量保证措施：**
1. **数据校验**：格式、范围、逻辑检查
2. **数据清洗**：去重、填充、标准化
3. **数据监控**：实时监控数据质量指标
4. **数据血缘**：追踪数据来源和流向

```java
@Component
public class DataQualityChecker {
    
    public DataQualityReport checkDataQuality(Dataset<Row> data) {
        DataQualityReport report = new DataQualityReport();
        
        // 完整性检查
        long totalCount = data.count();
        long nullCount = data.filter(col("user_id").isNull()).count();
        report.setCompletenessScore(1.0 - (double) nullCount / totalCount);
        
        // 唯一性检查
        long distinctCount = data.select("user_id").distinct().count();
        report.setUniquenessScore((double) distinctCount / totalCount);
        
        // 有效性检查
        long validCount = data.filter(col("email").rlike("^[\\w\\.-]+@[\\w\\.-]+\\.[\\w]+$")).count();
        report.setValidityScore((double) validCount / totalCount);
        
        return report;
    }
}
```

### Q4: 如何处理数据倾斜？

**数据倾斜解决方案：**
1. **重新分区**：使用合适的分区键
2. **预聚合**：Map端预聚合
3. **加盐技术**：给key添加随机前缀
4. **广播变量**：小表广播

```java
// 处理数据倾斜示例
public Dataset<Row> handleDataSkew(Dataset<Row> data) {
    // 方法1：重新分区
    Dataset<Row> repartitioned = data.repartition(200, col("user_id"));
    
    // 方法2：加盐处理
    Dataset<Row> salted = data.withColumn("salted_key", 
        concat(col("user_id"), lit("_"), 
               (rand().multiply(100)).cast("int")));
    
    return salted.groupBy("salted_key")
        .agg(sum("amount").alias("total_amount"))
        .groupBy(regexp_replace(col("salted_key"), "_\\d+$", "").alias("user_id"))
        .agg(sum("total_amount").alias("final_amount"));
}
```

### Q5: 如何设计实时数据仓库？

**实时数据仓库架构：**
```java
@Component
public class RealtimeDataWarehouse {
    
    // 1. 数据接入层
    @KafkaListener(topics = "raw-events")
    public void handleRawEvents(String event) {
        // 数据清洗和标准化
        StandardEvent standardEvent = cleanAndStandardize(event);
        
        // 发送到不同的topic
        kafkaTemplate.send("cleaned-events", standardEvent);
    }
    
    // 2. 实时计算层
    @StreamListener("cleaned-events")
    public void processCleanedEvents(StandardEvent event) {
        // 实时聚合
        updateRealTimeMetrics(event);
        
        // 写入实时存储
        saveToRedis(event);
        saveToElasticsearch(event);
    }
    
    // 3. 存储层
    private void updateRealTimeMetrics(StandardEvent event) {
        // 更新Redis中的实时指标
        redisTemplate.opsForValue().increment("pv_count", 1);
        redisTemplate.opsForHyperLogLog().add("uv_count", event.getUserId());
        
        // 更新时间窗口统计
        String timeWindow = getTimeWindow(event.getTimestamp());
        redisTemplate.opsForZSet().incrementScore("event_count:" + timeWindow, 
                                                 event.getEventType(), 1);
    }
}
```

### Q6: 大数据性能优化策略？

**性能优化方案：**
1. **存储优化**：选择合适的存储格式（Parquet、ORC）
2. **计算优化**：合理使用缓存、广播变量
3. **资源优化**：动态资源分配
4. **并行度调优**：合理设置并行度

```java
// Spark性能优化配置
SparkConf conf = new SparkConf()
    .setAppName("BigDataOptimization")
    .set("spark.sql.adaptive.enabled", "true")
    .set("spark.sql.adaptive.coalescePartitions.enabled", "true")
    .set("spark.sql.adaptive.skewJoin.enabled", "true")
    .set("spark.sql.execution.arrow.pyspark.enabled", "true")
    .set("spark.sql.parquet.columnarReaderBatchSize", "10000")
    .set("spark.sql.files.maxPartitionBytes", "134217728")
    .set("spark.sql.shuffle.partitions", "200");
```

## 8. 项目实战案例

### 8.1 电商实时推荐系统

**架构设计：**
```java
@Service
public class RealtimeRecommendationService {
    
    @Autowired
    private KafkaTemplate<String, String> kafkaTemplate;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    // 用户行为实时处理
    public void processUserBehavior(UserBehavior behavior) {
        // 1. 更新用户画像
        updateUserProfile(behavior);
        
        // 2. 实时推荐计算
        List<Item> recommendations = calculateRecommendations(behavior.getUserId());
        
        // 3. 缓存推荐结果
        cacheRecommendations(behavior.getUserId(), recommendations);
        
        // 4. 推送给用户
        pushToUser(behavior.getUserId(), recommendations);
    }
    
    private void updateUserProfile(UserBehavior behavior) {
        String userKey = "user_profile:" + behavior.getUserId();
        
        // 更新用户兴趣标签
        redisTemplate.opsForZSet().incrementScore(userKey + ":interests", 
                                                 behavior.getCategory(), 1.0);
        
        // 更新用户活跃度
        redisTemplate.opsForValue().increment(userKey + ":activity_score", 1);
    }
    
    private List<Item> calculateRecommendations(String userId) {
        // 基于协同过滤的实时推荐
        Set<String> similarUsers = findSimilarUsers(userId);
        
        return similarUsers.stream()
            .flatMap(user -> getUserRecentItems(user).stream())
            .collect(Collectors.groupingBy(Item::getId, Collectors.counting()))
            .entrySet().stream()
            .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
            .limit(10)
            .map(entry -> getItemById(entry.getKey()))
            .collect(Collectors.toList());
    }
}
```

### 8.2 用户行为分析系统

**分析模块：**
```java
@Component
public class UserBehaviorAnalyzer {
    
    public UserAnalysisReport analyzeUserBehavior(String userId, LocalDate startDate, LocalDate endDate) {
        UserAnalysisReport report = new UserAnalysisReport();
        
        // 1. 基础统计
        report.setTotalSessions(calculateTotalSessions(userId, startDate, endDate));
        report.setTotalPageViews(calculateTotalPageViews(userId, startDate, endDate));
        report.setAverageSessionDuration(calculateAverageSessionDuration(userId, startDate, endDate));
        
        // 2. 行为模式分析
        report.setBehaviorPatterns(analyzeBehaviorPatterns(userId, startDate, endDate));
        
        // 3. 用户价值评估
        report.setUserValue(calculateUserValue(userId, startDate, endDate));
        
        // 4. 流失风险预测
        report.setChurnRisk(predictChurnRisk(userId));
        
        return report;
    }
    
    private List<BehaviorPattern> analyzeBehaviorPatterns(String userId, LocalDate startDate, LocalDate endDate) {
        // 使用Spark进行行为序列分析
        Dataset<Row> userEvents = spark.read()
            .format("parquet")
            .load("/data/warehouse/user_events")
            .filter(col("user_id").equalTo(userId))
            .filter(col("date").between(startDate.toString(), endDate.toString()));
        
        // 序列模式挖掘
        return userEvents.select("event_type", "timestamp")
            .orderBy("timestamp")
            .collect()
            .stream()
            .map(row -> new BehaviorPattern(row.getString(0), row.getLong(1)))
            .collect(Collectors.toList());
    }
}
```

这个大数据分析技术栈文档涵盖了从数据采集到分析应用的完整链路，包括了主要技术组件的原理和实战代码。对于面试准备，建议重点关注：

1. **基础概念**：大数据特征、架构模式
2. **核心技术**：Hadoop、Spark、Kafka、HBase等
3. **实战经验**：数据质量、性能优化、故障处理
4. **业务场景**：推荐系统、用户分析、实时监控

这些知识点在大数据相关的面试中经常被问到。