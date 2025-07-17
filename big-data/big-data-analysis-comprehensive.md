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

## 8. 实际业务应用场景

### 8.1 电商推荐系统

#### 系统架构设计
```
用户行为数据 → Kafka → Spark Streaming → Redis/HBase → 推荐算法 → API服务
     ↓              ↓           ↓            ↓          ↓        ↓
  点击/购买/浏览   实时采集    实时计算     特征存储    协同过滤   推荐结果
  收藏/评分/搜索   日志收集    用户画像     商品画像    深度学习   个性化展示
```

#### 技术选型决策
```java
@Configuration
public class RecommendationSystemConfig {

    /**
     * 实时推荐架构
     * - 用户行为实时采集：Kafka
     * - 实时特征计算：Spark Streaming
     * - 特征存储：Redis + HBase
     * - 推荐算法：Spark MLlib + TensorFlow
     */

    // 用户行为数据模型
    @Data
    public static class UserBehavior {
        private String userId;
        private String itemId;
        private String behavior;  // click, buy, view, collect
        private Long timestamp;
        private String category;
        private Double rating;
        private Map<String, Object> context;  // 上下文信息
    }

    // 实时特征计算
    @Component
    public class RealTimeFeatureCalculator {

        public void processUserBehavior(JavaDStream<UserBehavior> behaviorStream) {
            // 计算用户实时特征
            JavaPairDStream<String, UserFeature> userFeatures = behaviorStream
                .mapToPair(behavior -> new Tuple2<>(behavior.getUserId(), behavior))
                .groupByKey()
                .mapValues(behaviors -> {
                    UserFeature feature = new UserFeature();
                    // 计算用户偏好类别
                    Map<String, Long> categoryCount = StreamSupport.stream(behaviors.spliterator(), false)
                        .collect(Collectors.groupingBy(
                            UserBehavior::getCategory,
                            Collectors.counting()
                        ));
                    feature.setPreferredCategories(categoryCount);

                    // 计算活跃度
                    long recentBehaviorCount = StreamSupport.stream(behaviors.spliterator(), false)
                        .filter(b -> System.currentTimeMillis() - b.getTimestamp() < 3600000) // 1小时内
                        .count();
                    feature.setActivityLevel(recentBehaviorCount);

                    return feature;
                });

            // 存储到Redis
            userFeatures.foreachRDD(rdd -> {
                rdd.foreach(tuple -> {
                    String userId = tuple._1();
                    UserFeature feature = tuple._2();
                    redisTemplate.opsForValue().set(
                        "user_feature:" + userId,
                        feature,
                        Duration.ofHours(24)
                    );
                });
            });
        }
    }
}
```

#### 推荐算法实现
```java
@Service
public class RecommendationService {

    /**
     * 混合推荐策略
     * 1. 协同过滤（CF）- 基于用户行为相似性
     * 2. 内容推荐（CB）- 基于商品特征相似性
     * 3. 深度学习（DL）- 基于神经网络模型
     */

    public List<RecommendationItem> getRecommendations(String userId, int count) {
        // 获取用户特征
        UserFeature userFeature = getUserFeature(userId);

        // 多路召回
        List<RecommendationItem> cfItems = collaborativeFiltering(userId, count * 2);
        List<RecommendationItem> cbItems = contentBasedFiltering(userFeature, count * 2);
        List<RecommendationItem> dlItems = deepLearningRecommendation(userId, count * 2);

        // 融合排序
        List<RecommendationItem> candidates = new ArrayList<>();
        candidates.addAll(cfItems);
        candidates.addAll(cbItems);
        candidates.addAll(dlItems);

        // 去重和重排序
        return rankAndFilter(candidates, userFeature, count);
    }

    private List<RecommendationItem> collaborativeFiltering(String userId, int count) {
        // 使用Spark MLlib的ALS算法
        JavaRDD<Rating> ratingsRDD = getUserRatings();
        ALS als = new ALS()
            .setRank(50)
            .setIterations(10)
            .setLambda(0.01);

        MatrixFactorizationModel model = als.run(ratingsRDD.rdd());

        // 为用户推荐商品
        Rating[] recommendations = model.recommendProducts(
            Integer.parseInt(userId), count
        );

        return Arrays.stream(recommendations)
            .map(rating -> new RecommendationItem(
                String.valueOf(rating.product()),
                rating.rating(),
                "CF"
            ))
            .collect(Collectors.toList());
    }
}
```

### 8.2 金融风控系统

#### 实时风控架构
```
交易请求 → 规则引擎 → 机器学习模型 → 风险评分 → 决策引擎 → 处理结果
   ↓         ↓          ↓           ↓        ↓        ↓
 用户信息   黑名单检查   特征工程     风险概率   阈值判断   通过/拒绝/人工审核
 交易信息   规则匹配     模型预测     评分计算   策略执行   风险监控
```

#### 实时风控实现
```java
@Service
public class RealTimeRiskControlService {

    @Autowired
    private RuleEngine ruleEngine;

    @Autowired
    private MLModelService mlModelService;

    /**
     * 实时风控决策
     */
    public RiskDecision evaluateTransaction(TransactionRequest request) {
        // 1. 基础信息验证
        if (!validateBasicInfo(request)) {
            return RiskDecision.reject("基础信息验证失败");
        }

        // 2. 黑名单检查
        if (isInBlacklist(request.getUserId(), request.getDeviceId())) {
            return RiskDecision.reject("用户在黑名单中");
        }

        // 3. 规则引擎检查
        RuleResult ruleResult = ruleEngine.evaluate(request);
        if (ruleResult.isReject()) {
            return RiskDecision.reject(ruleResult.getReason());
        }

        // 4. 机器学习模型评分
        TransactionFeature feature = extractFeatures(request);
        double riskScore = mlModelService.predict(feature);

        // 5. 综合决策
        return makeDecision(ruleResult, riskScore, request);
    }

    private TransactionFeature extractFeatures(TransactionRequest request) {
        TransactionFeature feature = new TransactionFeature();

        // 用户特征
        UserProfile userProfile = getUserProfile(request.getUserId());
        feature.setUserAge(userProfile.getAge());
        feature.setUserLevel(userProfile.getLevel());
        feature.setHistoryTransactionCount(userProfile.getTransactionCount());

        // 交易特征
        feature.setTransactionAmount(request.getAmount());
        feature.setTransactionTime(request.getTimestamp());
        feature.setMerchantCategory(request.getMerchantCategory());

        // 设备特征
        DeviceInfo deviceInfo = getDeviceInfo(request.getDeviceId());
        feature.setDeviceType(deviceInfo.getType());
        feature.setDeviceLocation(deviceInfo.getLocation());

        // 行为特征
        BehaviorFeature behaviorFeature = calculateBehaviorFeature(request.getUserId());
        feature.setRecentTransactionFrequency(behaviorFeature.getFrequency());
        feature.setAverageTransactionAmount(behaviorFeature.getAvgAmount());

        return feature;
    }

    private RiskDecision makeDecision(RuleResult ruleResult, double riskScore, TransactionRequest request) {
        // 综合评分
        double finalScore = ruleResult.getScore() * 0.3 + riskScore * 0.7;

        if (finalScore > 0.8) {
            return RiskDecision.reject("风险评分过高: " + finalScore);
        } else if (finalScore > 0.6) {
            return RiskDecision.review("需要人工审核: " + finalScore);
        } else {
            return RiskDecision.approve("风险可控: " + finalScore);
        }
    }
}
```

#### 风控规则引擎
```java
@Component
public class RuleEngine {

    private List<RiskRule> rules = Arrays.asList(
        new AmountLimitRule(),
        new FrequencyLimitRule(),
        new LocationRule(),
        new TimeRule(),
        new DeviceRule()
    );

    public RuleResult evaluate(TransactionRequest request) {
        double totalScore = 0.0;
        List<String> triggeredRules = new ArrayList<>();

        for (RiskRule rule : rules) {
            RuleResult result = rule.evaluate(request);
            totalScore += result.getScore() * rule.getWeight();

            if (result.isTriggered()) {
                triggeredRules.add(rule.getName() + ": " + result.getReason());
            }

            // 如果有规则直接拒绝，立即返回
            if (result.isReject()) {
                return RuleResult.reject(result.getReason());
            }
        }

        return new RuleResult(totalScore, triggeredRules);
    }

    // 金额限制规则
    public static class AmountLimitRule implements RiskRule {
        @Override
        public RuleResult evaluate(TransactionRequest request) {
            double amount = request.getAmount();
            UserProfile profile = getUserProfile(request.getUserId());

            // 单笔限额检查
            if (amount > profile.getSingleTransactionLimit()) {
                return RuleResult.reject("超过单笔交易限额");
            }

            // 日累计限额检查
            double todayTotal = getTodayTransactionTotal(request.getUserId());
            if (todayTotal + amount > profile.getDailyTransactionLimit()) {
                return RuleResult.reject("超过日累计交易限额");
            }

            // 风险评分
            double score = Math.min(amount / profile.getSingleTransactionLimit(), 1.0);
            return new RuleResult(score, score > 0.8, "大额交易");
        }

        @Override
        public double getWeight() { return 0.3; }

        @Override
        public String getName() { return "金额限制规则"; }
    }
}
```

### 8.3 物联网数据处理

#### IoT数据处理架构
```
传感器设备 → MQTT/CoAP → Kafka → Spark Streaming → 时序数据库 → 监控告警
    ↓           ↓         ↓         ↓            ↓          ↓
  温度/湿度    协议转换   消息队列   实时计算      InfluxDB    异常检测
  压力/流量    数据采集   数据缓冲   聚合分析      TimescaleDB  预警通知
  位置/状态    边缘计算   负载均衡   模式识别      OpenTSDB    可视化展示
```

#### 实时数据处理
```java
@Component
public class IoTDataProcessor {

    /**
     * 处理IoT传感器数据流
     */
    public void processIoTDataStream() {
        // 创建Spark Streaming上下文
        JavaStreamingContext jssc = new JavaStreamingContext(sparkConf, Durations.seconds(5));

        // 从Kafka读取IoT数据
        JavaInputDStream<ConsumerRecord<String, String>> kafkaStream =
            KafkaUtils.createDirectStream(
                jssc,
                LocationStrategies.PreferConsistent(),
                ConsumerStrategies.<String, String>Subscribe(
                    Arrays.asList("iot-sensor-data"), kafkaParams
                )
            );

        // 解析传感器数据
        JavaDStream<SensorData> sensorDataStream = kafkaStream
            .map(record -> JSON.parseObject(record.value(), SensorData.class))
            .filter(data -> data != null && data.isValid());

        // 数据清洗和转换
        JavaDStream<SensorData> cleanedDataStream = sensorDataStream
            .filter(this::isValidSensorData)
            .map(this::normalizeSensorData);

        // 实时聚合计算
        JavaPairDStream<String, SensorAggregation> aggregatedStream = cleanedDataStream
            .mapToPair(data -> new Tuple2<>(data.getDeviceId(), data))
            .groupByKeyAndWindow(Durations.minutes(5), Durations.minutes(1))
            .mapValues(this::aggregateSensorData);

        // 异常检测
        JavaDStream<Alert> alertStream = aggregatedStream
            .map(tuple -> detectAnomalies(tuple._1(), tuple._2()))
            .filter(alert -> alert != null);

        // 存储到时序数据库
        aggregatedStream.foreachRDD(rdd -> {
            rdd.foreach(tuple -> {
                String deviceId = tuple._1();
                SensorAggregation aggregation = tuple._2();
                saveToTimeSeriesDB(deviceId, aggregation);
            });
        });

        // 发送告警
        alertStream.foreachRDD(rdd -> {
            rdd.foreach(alert -> sendAlert(alert));
        });

        jssc.start();
        jssc.awaitTermination();
    }

    private SensorAggregation aggregateSensorData(Iterable<SensorData> sensorDataList) {
        List<SensorData> dataList = StreamSupport.stream(sensorDataList.spliterator(), false)
            .collect(Collectors.toList());

        if (dataList.isEmpty()) {
            return null;
        }

        SensorAggregation aggregation = new SensorAggregation();
        aggregation.setDeviceId(dataList.get(0).getDeviceId());
        aggregation.setTimestamp(System.currentTimeMillis());

        // 计算统计指标
        DoubleSummaryStatistics tempStats = dataList.stream()
            .mapToDouble(SensorData::getTemperature)
            .summaryStatistics();

        aggregation.setAvgTemperature(tempStats.getAverage());
        aggregation.setMaxTemperature(tempStats.getMax());
        aggregation.setMinTemperature(tempStats.getMin());

        DoubleSummaryStatistics humidityStats = dataList.stream()
            .mapToDouble(SensorData::getHumidity)
            .summaryStatistics();

        aggregation.setAvgHumidity(humidityStats.getAverage());
        aggregation.setMaxHumidity(humidityStats.getMax());
        aggregation.setMinHumidity(humidityStats.getMin());

        // 计算数据质量指标
        long validDataCount = dataList.stream()
            .filter(SensorData::isValid)
            .count();
        aggregation.setDataQuality((double) validDataCount / dataList.size());

        return aggregation;
    }

    private Alert detectAnomalies(String deviceId, SensorAggregation aggregation) {
        // 获取设备的正常范围配置
        DeviceConfig config = getDeviceConfig(deviceId);

        Alert alert = null;

        // 温度异常检测
        if (aggregation.getMaxTemperature() > config.getMaxTemperature() ||
            aggregation.getMinTemperature() < config.getMinTemperature()) {
            alert = new Alert();
            alert.setDeviceId(deviceId);
            alert.setType("TEMPERATURE_ANOMALY");
            alert.setMessage(String.format("温度异常: %.2f°C (正常范围: %.2f-%.2f°C)",
                aggregation.getAvgTemperature(),
                config.getMinTemperature(),
                config.getMaxTemperature()));
            alert.setSeverity(calculateSeverity(aggregation, config));
        }

        // 湿度异常检测
        if (aggregation.getMaxHumidity() > config.getMaxHumidity() ||
            aggregation.getMinHumidity() < config.getMinHumidity()) {
            if (alert == null) {
                alert = new Alert();
                alert.setDeviceId(deviceId);
                alert.setType("HUMIDITY_ANOMALY");
            }
            alert.setMessage(alert.getMessage() + String.format(" 湿度异常: %.2f%% (正常范围: %.2f-%.2f%%)",
                aggregation.getAvgHumidity(),
                config.getMinHumidity(),
                config.getMaxHumidity()));
        }

        // 数据质量检测
        if (aggregation.getDataQuality() < 0.8) {
            if (alert == null) {
                alert = new Alert();
                alert.setDeviceId(deviceId);
                alert.setType("DATA_QUALITY_ISSUE");
            }
            alert.setMessage(alert.getMessage() + String.format(" 数据质量异常: %.2f%%",
                aggregation.getDataQuality() * 100));
        }

        return alert;
    }
}
```

## 9. 技术选型决策过程

### 9.1 技术选型矩阵

#### 消息队列选型
| 特性 | Kafka | RabbitMQ | RocketMQ | Pulsar |
|------|-------|----------|----------|--------|
| 吞吐量 | 极高 | 中等 | 高 | 极高 |
| 延迟 | 低 | 极低 | 低 | 低 |
| 可靠性 | 高 | 极高 | 极高 | 高 |
| 运维复杂度 | 中等 | 低 | 中等 | 高 |
| 生态成熟度 | 极高 | 高 | 中等 | 中等 |
| 适用场景 | 大数据、日志 | 业务消息 | 业务消息 | 云原生 |

#### 计算引擎选型
| 特性 | Spark | Flink | Storm | Hadoop MapReduce |
|------|-------|-------|-------|------------------|
| 处理模式 | 批+流 | 流为主 | 流处理 | 批处理 |
| 延迟 | 秒级 | 毫秒级 | 毫秒级 | 分钟级 |
| 吞吐量 | 极高 | 高 | 中等 | 高 |
| 容错性 | 高 | 极高 | 高 | 极高 |
| 易用性 | 高 | 中等 | 低 | 低 |
| 内存使用 | 高 | 中等 | 低 | 低 |

### 9.2 选型决策流程
```java
@Component
public class TechnologySelectionFramework {

    /**
     * 技术选型决策框架
     */
    public TechStack selectTechnology(ProjectRequirement requirement) {
        // 1. 需求分析
        RequirementAnalysis analysis = analyzeRequirement(requirement);

        // 2. 技术评估
        List<TechnologyOption> options = evaluateTechnologies(analysis);

        // 3. 权重计算
        Map<String, Double> weights = calculateWeights(requirement);

        // 4. 综合评分
        TechnologyOption bestOption = options.stream()
            .max(Comparator.comparing(option -> calculateScore(option, weights)))
            .orElseThrow();

        // 5. 风险评估
        RiskAssessment risk = assessRisk(bestOption);

        return new TechStack(bestOption, risk);
    }

    private RequirementAnalysis analyzeRequirement(ProjectRequirement requirement) {
        RequirementAnalysis analysis = new RequirementAnalysis();

        // 数据量分析
        if (requirement.getDataVolume() > 1_000_000_000) {  // 10亿条记录
            analysis.setDataScale(DataScale.BIG_DATA);
        } else if (requirement.getDataVolume() > 1_000_000) {  // 100万条记录
            analysis.setDataScale(DataScale.MEDIUM_DATA);
        } else {
            analysis.setDataScale(DataScale.SMALL_DATA);
        }

        // 实时性要求分析
        if (requirement.getLatencyRequirement() < 100) {  // 100ms
            analysis.setLatencyLevel(LatencyLevel.REAL_TIME);
        } else if (requirement.getLatencyRequirement() < 1000) {  // 1s
            analysis.setLatencyLevel(LatencyLevel.NEAR_REAL_TIME);
        } else {
            analysis.setLatencyLevel(LatencyLevel.BATCH);
        }

        // 一致性要求分析
        analysis.setConsistencyLevel(requirement.getConsistencyRequirement());

        // 可用性要求分析
        analysis.setAvailabilityLevel(requirement.getAvailabilityRequirement());

        return analysis;
    }
}
```

## 10. 性能调优实战经验

### 10.1 Spark性能调优

#### 内存调优
```scala
// Spark配置优化
val sparkConf = new SparkConf()
  .setAppName("BigDataProcessing")
  // 执行器内存配置
  .set("spark.executor.memory", "8g")
  .set("spark.executor.cores", "4")
  .set("spark.executor.instances", "20")

  // 内存分配比例
  .set("spark.sql.adaptive.enabled", "true")
  .set("spark.sql.adaptive.coalescePartitions.enabled", "true")
  .set("spark.sql.adaptive.skewJoin.enabled", "true")

  // 序列化优化
  .set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
  .set("spark.kryo.registrationRequired", "false")

  // 网络优化
  .set("spark.network.timeout", "300s")
  .set("spark.sql.broadcastTimeout", "300")

  // 动态资源分配
  .set("spark.dynamicAllocation.enabled", "true")
  .set("spark.dynamicAllocation.minExecutors", "5")
  .set("spark.dynamicAllocation.maxExecutors", "50")
```

#### 数据倾斜处理
```java
@Component
public class DataSkewHandler {

    /**
     * 处理数据倾斜的策略
     */
    public Dataset<Row> handleDataSkew(Dataset<Row> dataset, String keyColumn) {
        // 策略1: 加盐处理
        Dataset<Row> saltedDataset = addSalt(dataset, keyColumn);

        // 策略2: 预聚合
        Dataset<Row> preAggregated = preAggregate(saltedDataset, keyColumn);

        // 策略3: 广播小表
        if (isSmallTable(dataset)) {
            return broadcast(dataset);
        }

        return preAggregated;
    }

    private Dataset<Row> addSalt(Dataset<Row> dataset, String keyColumn) {
        // 添加随机盐值
        return dataset.withColumn("salted_key",
            concat(col(keyColumn), lit("_"),
                   (rand().multiply(100)).cast(DataTypes.IntegerType)));
    }

    private Dataset<Row> preAggregate(Dataset<Row> dataset, String keyColumn) {
        // 预聚合减少数据量
        return dataset
            .groupBy("salted_key")
            .agg(
                sum("amount").as("total_amount"),
                count("*").as("record_count"),
                max("timestamp").as("latest_timestamp")
            );
    }
}
```

### 10.2 Kafka性能调优

#### 生产者优化
```java
@Configuration
public class KafkaProducerOptimization {

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> props = new HashMap<>();

        // 基础配置
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // 性能优化配置
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);  // 64KB批次大小
        props.put(ProducerConfig.LINGER_MS_CONFIG, 10);      // 等待10ms收集更多消息
        props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");  // 压缩算法
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);  // 64MB缓冲区

        // 可靠性配置
        props.put(ProducerConfig.ACKS_CONFIG, "1");          // 等待leader确认
        props.put(ProducerConfig.RETRIES_CONFIG, 3);         // 重试次数
        props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);

        // 幂等性配置
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

#### 消费者优化
```java
@Configuration
public class KafkaConsumerOptimization {

    @Bean
    public ConsumerFactory<String, String> consumerFactory() {
        Map<String, Object> props = new HashMap<>();

        // 基础配置
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "big-data-consumer");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        // 性能优化配置
        props.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024);      // 最小拉取1KB
        props.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);     // 最大等待500ms
        props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 1000);     // 每次拉取1000条
        props.put(ConsumerConfig.RECEIVE_BUFFER_CONFIG, 65536);      // 64KB接收缓冲区

        // 偏移量管理
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);  // 手动提交偏移量

        return new DefaultKafkaConsumerFactory<>(props);
    }
}
```

## 11. 常见架构模式

### 11.1 Lambda架构
```
实时数据流 → 流处理层 → 实时视图
    ↓           ↓         ↓
批处理数据 → 批处理层 → 批处理视图 → 服务层 → 查询结果
    ↓           ↓         ↓        ↓
 历史数据   离线计算   准确结果   结果合并
```

#### Lambda架构实现
```java
@Service
public class LambdaArchitectureService {

    @Autowired
    private StreamProcessingService streamService;

    @Autowired
    private BatchProcessingService batchService;

    @Autowired
    private ServingLayerService servingService;

    /**
     * Lambda架构查询接口
     */
    public QueryResult query(QueryRequest request) {
        // 从实时视图获取最新数据
        RealtimeResult realtimeResult = streamService.queryRealtimeView(request);

        // 从批处理视图获取历史数据
        BatchResult batchResult = batchService.queryBatchView(request);

        // 在服务层合并结果
        return servingService.mergeResults(realtimeResult, batchResult);
    }

    /**
     * 流处理层实现
     */
    @Component
    public static class StreamProcessingService {

        public void processRealTimeData() {
            JavaStreamingContext jssc = createStreamingContext();

            // 从Kafka读取实时数据
            JavaInputDStream<ConsumerRecord<String, String>> kafkaStream =
                createKafkaStream(jssc);

            // 实时计算
            JavaDStream<ProcessedData> processedStream = kafkaStream
                .map(record -> parseData(record.value()))
                .filter(data -> data != null)
                .window(Durations.minutes(5), Durations.minutes(1))
                .map(this::processData);

            // 更新实时视图
            processedStream.foreachRDD(rdd -> {
                rdd.foreach(data -> updateRealtimeView(data));
            });

            jssc.start();
            jssc.awaitTermination();
        }

        private void updateRealtimeView(ProcessedData data) {
            // 更新Redis中的实时视图
            String key = "realtime:" + data.getKey();
            redisTemplate.opsForValue().set(key, data, Duration.ofMinutes(10));
        }
    }

    /**
     * 批处理层实现
     */
    @Component
    public static class BatchProcessingService {

        @Scheduled(cron = "0 0 2 * * ?")  // 每天凌晨2点执行
        public void processBatchData() {
            SparkSession spark = SparkSession.builder()
                .appName("BatchProcessing")
                .getOrCreate();

            // 读取历史数据
            Dataset<Row> historicalData = spark.read()
                .format("parquet")
                .load("hdfs://path/to/historical/data");

            // 批处理计算
            Dataset<Row> aggregatedData = historicalData
                .groupBy("key", "date")
                .agg(
                    sum("value").as("total_value"),
                    count("*").as("count"),
                    avg("value").as("avg_value")
                );

            // 保存到批处理视图
            aggregatedData.write()
                .mode(SaveMode.Overwrite)
                .format("parquet")
                .save("hdfs://path/to/batch/view");
        }
    }
}
```

### 11.2 Kappa架构
```
数据流 → 流处理引擎 → 存储层 → 查询层
  ↓         ↓          ↓       ↓
实时数据   Flink/Kafka  Kafka   API服务
历史数据   Streams     存储     实时查询
```

#### Kappa架构实现
```java
@Service
public class KappaArchitectureService {

    /**
     * 统一流处理
     */
    public void processUnifiedStream() {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

        // 配置Kafka消费者
        Properties kafkaProps = new Properties();
        kafkaProps.setProperty("bootstrap.servers", "localhost:9092");
        kafkaProps.setProperty("group.id", "kappa-processor");

        FlinkKafkaConsumer<String> kafkaConsumer = new FlinkKafkaConsumer<>(
            "input-topic", new SimpleStringSchema(), kafkaProps);

        // 设置从最早的记录开始消费（处理历史数据）
        kafkaConsumer.setStartFromEarliest();

        DataStream<String> inputStream = env.addSource(kafkaConsumer);

        // 数据解析和转换
        DataStream<Event> eventStream = inputStream
            .map(json -> JSON.parseObject(json, Event.class))
            .filter(event -> event != null)
            .assignTimestampsAndWatermarks(
                WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(10))
                    .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
            );

        // 窗口聚合
        DataStream<AggregatedResult> aggregatedStream = eventStream
            .keyBy(Event::getKey)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new EventAggregateFunction());

        // 输出到Kafka
        FlinkKafkaProducer<AggregatedResult> kafkaProducer = new FlinkKafkaProducer<>(
            "output-topic",
            new AggregatedResultSerializationSchema(),
            kafkaProps,
            FlinkKafkaProducer.Semantic.EXACTLY_ONCE
        );

        aggregatedStream.addSink(kafkaProducer);

        try {
            env.execute("Kappa Architecture Processing");
        } catch (Exception e) {
            log.error("Kappa processing failed", e);
        }
    }
}
```

### 11.3 微服务大数据架构
```
API网关 → 微服务集群 → 消息队列 → 大数据平台 → 数据服务
   ↓         ↓          ↓         ↓          ↓
用户请求   业务处理    异步消息   数据处理    结果查询
负载均衡   服务发现    事件驱动   批流一体    缓存加速
```

## 12. 面试中的系统设计题目

### 12.1 设计一个实时推荐系统

#### 系统需求分析
- **功能需求**：实时推荐、个性化、多样性、新颖性
- **非功能需求**：低延迟(<100ms)、高并发(10万QPS)、高可用(99.9%)
- **数据规模**：1亿用户、1000万商品、100亿行为记录

#### 系统设计方案
```java
/**
 * 实时推荐系统架构设计
 */
@Component
public class RealtimeRecommendationSystemDesign {

    /**
     * 系统架构组件
     */
    public class SystemArchitecture {
        // 数据采集层
        private KafkaCluster dataIngestion;

        // 实时计算层
        private SparkStreamingCluster realtimeComputing;

        // 特征存储层
        private RedisCluster featureStore;
        private HBaseCluster profileStore;

        // 模型服务层
        private TensorFlowServing modelServing;

        // 推荐服务层
        private RecommendationService recommendationAPI;

        // 缓存层
        private RedisCluster resultCache;
    }

    /**
     * 推荐算法设计
     */
    public class RecommendationAlgorithm {

        // 召回阶段：从海量商品中召回候选集
        public List<Item> recall(String userId) {
            List<Item> candidates = new ArrayList<>();

            // 协同过滤召回
            candidates.addAll(collaborativeFilteringRecall(userId));

            // 内容召回
            candidates.addAll(contentBasedRecall(userId));

            // 热门召回
            candidates.addAll(popularItemsRecall());

            // 新品召回
            candidates.addAll(newItemsRecall());

            return candidates;
        }

        // 排序阶段：对候选集进行精确排序
        public List<Item> rank(String userId, List<Item> candidates) {
            // 特征工程
            List<FeatureVector> features = candidates.stream()
                .map(item -> extractFeatures(userId, item))
                .collect(Collectors.toList());

            // 模型预测
            List<Double> scores = modelServing.predict(features);

            // 排序和过滤
            return IntStream.range(0, candidates.size())
                .boxed()
                .sorted((i, j) -> Double.compare(scores.get(j), scores.get(i)))
                .map(candidates::get)
                .limit(100)
                .collect(Collectors.toList());
        }
    }

    /**
     * 性能优化策略
     */
    public class PerformanceOptimization {

        // 多级缓存策略
        public List<Item> getRecommendations(String userId) {
            // L1缓存：本地缓存
            List<Item> result = localCache.get(userId);
            if (result != null) {
                return result;
            }

            // L2缓存：Redis缓存
            result = redisCache.get("rec:" + userId);
            if (result != null) {
                localCache.put(userId, result);
                return result;
            }

            // L3：实时计算
            result = computeRecommendations(userId);

            // 更新缓存
            redisCache.set("rec:" + userId, result, Duration.ofMinutes(10));
            localCache.put(userId, result);

            return result;
        }

        // 预计算策略
        @Scheduled(fixedDelay = 300000)  // 5分钟执行一次
        public void precomputeRecommendations() {
            // 为活跃用户预计算推荐结果
            List<String> activeUsers = getActiveUsers();

            activeUsers.parallelStream().forEach(userId -> {
                try {
                    List<Item> recommendations = computeRecommendations(userId);
                    redisCache.set("rec:" + userId, recommendations, Duration.ofHours(1));
                } catch (Exception e) {
                    log.error("Failed to precompute recommendations for user: " + userId, e);
                }
            });
        }
    }
}
```

### 12.2 设计一个日志分析系统

#### 系统架构设计
```java
/**
 * 分布式日志分析系统
 */
@Component
public class LogAnalysisSystemDesign {

    /**
     * 日志采集架构
     */
    public class LogCollectionArchitecture {

        // 日志采集配置
        public void configureLogCollection() {
            // Filebeat配置
            FilebeatConfig filebeatConfig = FilebeatConfig.builder()
                .inputPaths(Arrays.asList("/var/log/app/*.log"))
                .outputKafka(KafkaOutput.builder()
                    .brokers(Arrays.asList("kafka1:9092", "kafka2:9092"))
                    .topic("application-logs")
                    .build())
                .build();

            // Logstash配置
            LogstashConfig logstashConfig = LogstashConfig.builder()
                .input(KafkaInput.builder()
                    .topics(Arrays.asList("application-logs"))
                    .bootstrapServers("kafka1:9092,kafka2:9092")
                    .build())
                .filter(GrokFilter.builder()
                    .pattern("%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:message}")
                    .build())
                .output(ElasticsearchOutput.builder()
                    .hosts(Arrays.asList("es1:9200", "es2:9200"))
                    .index("logs-%{+YYYY.MM.dd}")
                    .build())
                .build();
        }
    }

    /**
     * 实时日志分析
     */
    public class RealTimeLogAnalysis {

        public void analyzeLogsInRealTime() {
            StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

            // 从Kafka读取日志
            DataStream<String> logStream = env.addSource(
                new FlinkKafkaConsumer<>("application-logs", new SimpleStringSchema(), kafkaProps)
            );

            // 解析日志
            DataStream<LogEvent> parsedLogs = logStream
                .map(this::parseLogEvent)
                .filter(Objects::nonNull);

            // 错误日志告警
            DataStream<Alert> errorAlerts = parsedLogs
                .filter(log -> "ERROR".equals(log.getLevel()))
                .keyBy(LogEvent::getService)
                .window(TumblingProcessingTimeWindows.of(Time.minutes(1)))
                .aggregate(new ErrorCountAggregator())
                .filter(count -> count.getErrorCount() > 10)  // 1分钟内超过10个错误
                .map(count -> new Alert("HIGH_ERROR_RATE", count.getService(), count.getErrorCount()));

            // 性能监控
            DataStream<PerformanceMetric> performanceMetrics = parsedLogs
                .filter(log -> log.getResponseTime() != null)
                .keyBy(LogEvent::getEndpoint)
                .window(TumblingProcessingTimeWindows.of(Time.minutes(5)))
                .aggregate(new PerformanceAggregator());

            // 输出告警
            errorAlerts.addSink(new AlertSink());

            // 输出性能指标
            performanceMetrics.addSink(new MetricsSink());

            try {
                env.execute("Real-time Log Analysis");
            } catch (Exception e) {
                log.error("Log analysis job failed", e);
            }
        }

        private LogEvent parseLogEvent(String logLine) {
            try {
                // 使用正则表达式或JSON解析日志
                Pattern pattern = Pattern.compile(
                    "(\\d{4}-\\d{2}-\\d{2} \\d{2}:\\d{2}:\\d{2}) (\\w+) \\[(\\w+)\\] (.+)"
                );
                Matcher matcher = pattern.matcher(logLine);

                if (matcher.matches()) {
                    LogEvent event = new LogEvent();
                    event.setTimestamp(LocalDateTime.parse(matcher.group(1),
                        DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss")));
                    event.setLevel(matcher.group(2));
                    event.setService(matcher.group(3));
                    event.setMessage(matcher.group(4));

                    // 提取响应时间（如果存在）
                    Pattern responseTimePattern = Pattern.compile(".*response_time=(\\d+)ms.*");
                    Matcher responseTimeMatcher = responseTimePattern.matcher(event.getMessage());
                    if (responseTimeMatcher.matches()) {
                        event.setResponseTime(Long.parseLong(responseTimeMatcher.group(1)));
                    }

                    return event;
                }
            } catch (Exception e) {
                log.warn("Failed to parse log line: " + logLine, e);
            }
            return null;
        }
    }

    /**
     * 批量日志分析
     */
    public class BatchLogAnalysis {

        @Scheduled(cron = "0 0 1 * * ?")  // 每天凌晨1点执行
        public void dailyLogAnalysis() {
            SparkSession spark = SparkSession.builder()
                .appName("Daily Log Analysis")
                .getOrCreate();

            // 读取昨天的日志数据
            String yesterday = LocalDate.now().minusDays(1).format(DateTimeFormatter.ofPattern("yyyy.MM.dd"));
            Dataset<Row> logs = spark.read()
                .format("org.elasticsearch.spark.sql")
                .option("es.nodes", "es1,es2")
                .option("es.port", "9200")
                .load("logs-" + yesterday);

            // 分析API调用统计
            Dataset<Row> apiStats = logs
                .filter(col("level").equalTo("INFO"))
                .filter(col("message").contains("API_CALL"))
                .groupBy("endpoint", "method")
                .agg(
                    count("*").as("call_count"),
                    avg("response_time").as("avg_response_time"),
                    max("response_time").as("max_response_time"),
                    expr("percentile_approx(response_time, 0.95)").as("p95_response_time")
                );

            // 分析错误统计
            Dataset<Row> errorStats = logs
                .filter(col("level").equalTo("ERROR"))
                .groupBy("service", "error_type")
                .agg(
                    count("*").as("error_count"),
                    collect_list("message").as("error_messages")
                );

            // 保存分析结果
            apiStats.write()
                .mode(SaveMode.Overwrite)
                .format("parquet")
                .save("hdfs://path/to/api-stats/" + yesterday);

            errorStats.write()
                .mode(SaveMode.Overwrite)
                .format("parquet")
                .save("hdfs://path/to/error-stats/" + yesterday);

            // 生成日报
            generateDailyReport(apiStats, errorStats, yesterday);
        }

        private void generateDailyReport(Dataset<Row> apiStats, Dataset<Row> errorStats, String date) {
            // 生成HTML报告
            StringBuilder report = new StringBuilder();
            report.append("<h1>日志分析报告 - ").append(date).append("</h1>");

            // API统计部分
            report.append("<h2>API调用统计</h2>");
            report.append("<table border='1'>");
            report.append("<tr><th>接口</th><th>调用次数</th><th>平均响应时间</th><th>P95响应时间</th></tr>");

            apiStats.collectAsList().forEach(row -> {
                report.append("<tr>")
                    .append("<td>").append(row.getString(0)).append("</td>")
                    .append("<td>").append(row.getLong(2)).append("</td>")
                    .append("<td>").append(String.format("%.2f", row.getDouble(3))).append("ms</td>")
                    .append("<td>").append(String.format("%.2f", row.getDouble(5))).append("ms</td>")
                    .append("</tr>");
            });
            report.append("</table>");

            // 错误统计部分
            report.append("<h2>错误统计</h2>");
            report.append("<table border='1'>");
            report.append("<tr><th>服务</th><th>错误类型</th><th>错误次数</th></tr>");

            errorStats.collectAsList().forEach(row -> {
                report.append("<tr>")
                    .append("<td>").append(row.getString(0)).append("</td>")
                    .append("<td>").append(row.getString(1)).append("</td>")
                    .append("<td>").append(row.getLong(2)).append("</td>")
                    .append("</tr>");
            });
            report.append("</table>");

            // 发送报告邮件
            emailService.sendReport("daily-log-report@company.com",
                "日志分析报告 - " + date, report.toString());
        }
    }
}
```

## 13. 大数据项目实施的挑战和解决方案

### 13.1 数据质量挑战

#### 数据质量监控框架
```java
@Component
public class DataQualityMonitor {

    /**
     * 数据质量检查规则
     */
    public class DataQualityRules {

        // 完整性检查
        public boolean checkCompleteness(Dataset<Row> dataset, List<String> requiredColumns) {
            for (String column : requiredColumns) {
                long nullCount = dataset.filter(col(column).isNull()).count();
                double nullRate = (double) nullCount / dataset.count();

                if (nullRate > 0.05) {  // 空值率超过5%
                    log.warn("Column {} has high null rate: {}", column, nullRate);
                    return false;
                }
            }
            return true;
        }

        // 一致性检查
        public boolean checkConsistency(Dataset<Row> dataset) {
            // 检查数据格式一致性
            long invalidEmailCount = dataset
                .filter(col("email").isNotNull())
                .filter(!col("email").rlike("^[A-Za-z0-9+_.-]+@[A-Za-z0-9.-]+\\.[A-Za-z]{2,}$"))
                .count();

            if (invalidEmailCount > 0) {
                log.warn("Found {} invalid email addresses", invalidEmailCount);
                return false;
            }

            // 检查数值范围
            long invalidAgeCount = dataset
                .filter(col("age").lt(0).or(col("age").gt(150)))
                .count();

            if (invalidAgeCount > 0) {
                log.warn("Found {} invalid age values", invalidAgeCount);
                return false;
            }

            return true;
        }

        // 及时性检查
        public boolean checkTimeliness(Dataset<Row> dataset, String timestampColumn) {
            Row latestRecord = dataset
                .select(max(col(timestampColumn)).as("latest_timestamp"))
                .first();

            Timestamp latestTimestamp = latestRecord.getTimestamp(0);
            long delayMinutes = (System.currentTimeMillis() - latestTimestamp.getTime()) / (1000 * 60);

            if (delayMinutes > 60) {  // 数据延迟超过1小时
                log.warn("Data is delayed by {} minutes", delayMinutes);
                return false;
            }

            return true;
        }
    }

    /**
     * 数据质量修复
     */
    public class DataQualityRepair {

        public Dataset<Row> repairData(Dataset<Row> dataset) {
            // 填充空值
            Dataset<Row> filledDataset = dataset
                .na().fill("unknown", new String[]{"category"})
                .na().fill(0, new String[]{"amount"})
                .na().fill("1900-01-01", new String[]{"birth_date"});

            // 数据标准化
            Dataset<Row> standardizedDataset = filledDataset
                .withColumn("email", lower(col("email")))
                .withColumn("phone", regexp_replace(col("phone"), "[^0-9]", ""));

            // 异常值处理
            Dataset<Row> cleanedDataset = removeOutliers(standardizedDataset, "amount");

            return cleanedDataset;
        }

        private Dataset<Row> removeOutliers(Dataset<Row> dataset, String column) {
            // 使用IQR方法去除异常值
            double q1 = dataset.stat().approxQuantile(column, new double[]{0.25}, 0.05)[0];
            double q3 = dataset.stat().approxQuantile(column, new double[]{0.75}, 0.05)[0];
            double iqr = q3 - q1;
            double lowerBound = q1 - 1.5 * iqr;
            double upperBound = q3 + 1.5 * iqr;

            return dataset.filter(
                col(column).geq(lowerBound).and(col(column).leq(upperBound))
            );
        }
    }
}
```

### 13.2 性能和扩展性挑战

#### 自动扩缩容方案
```java
@Component
public class AutoScalingManager {

    /**
     * 基于负载的自动扩缩容
     */
    public class LoadBasedAutoScaling {

        @Scheduled(fixedDelay = 60000)  // 每分钟检查一次
        public void checkAndScale() {
            // 获取当前集群状态
            ClusterMetrics metrics = getClusterMetrics();

            // CPU使用率检查
            if (metrics.getAvgCpuUsage() > 0.8) {
                scaleOut("CPU usage too high: " + metrics.getAvgCpuUsage());
            } else if (metrics.getAvgCpuUsage() < 0.3 && metrics.getNodeCount() > 3) {
                scaleIn("CPU usage low: " + metrics.getAvgCpuUsage());
            }

            // 内存使用率检查
            if (metrics.getAvgMemoryUsage() > 0.85) {
                scaleOut("Memory usage too high: " + metrics.getAvgMemoryUsage());
            }

            // 队列长度检查
            if (metrics.getQueueLength() > 1000) {
                scaleOut("Queue length too high: " + metrics.getQueueLength());
            }
        }

        private void scaleOut(String reason) {
            log.info("Scaling out cluster. Reason: {}", reason);

            // 增加Spark执行器
            sparkContext.requestTotalExecutors(
                sparkContext.getExecutorIds().size() + 2,
                0,
                Map.empty()
            );

            // 增加Kafka分区（如果需要）
            if (shouldIncreaseKafkaPartitions()) {
                increaseKafkaPartitions();
            }
        }

        private void scaleIn(String reason) {
            log.info("Scaling in cluster. Reason: {}", reason);

            // 减少Spark执行器
            List<String> executorIds = sparkContext.getExecutorIds();
            if (executorIds.size() > 3) {
                sparkContext.killExecutors(
                    executorIds.subList(0, Math.min(2, executorIds.size() - 3))
                );
            }
        }
    }

    /**
     * 预测性扩缩容
     */
    public class PredictiveAutoScaling {

        @Scheduled(cron = "0 */10 * * * ?")  // 每10分钟执行一次
        public void predictiveScaling() {
            // 获取历史负载数据
            List<LoadMetric> historicalLoad = getHistoricalLoad(Duration.ofDays(7));

            // 预测未来1小时的负载
            LoadPrediction prediction = predictLoad(historicalLoad);

            // 根据预测结果提前扩缩容
            if (prediction.getPredictedLoad() > getCurrentCapacity() * 0.8) {
                log.info("Predictive scaling out based on prediction: {}", prediction);
                scaleOut("Predictive scaling - expected high load");
            }
        }

        private LoadPrediction predictLoad(List<LoadMetric> historicalLoad) {
            // 简单的时间序列预测（实际项目中可以使用更复杂的机器学习模型）

            // 计算同一时间段的历史平均值
            LocalTime currentTime = LocalTime.now();
            double avgLoad = historicalLoad.stream()
                .filter(metric -> isSimilarTime(metric.getTimestamp().toLocalTime(), currentTime))
                .mapToDouble(LoadMetric::getLoad)
                .average()
                .orElse(0.0);

            // 考虑趋势因子
            double trendFactor = calculateTrendFactor(historicalLoad);

            double predictedLoad = avgLoad * (1 + trendFactor);

            return new LoadPrediction(predictedLoad, currentTime.plusHours(1));
        }

        private boolean isSimilarTime(LocalTime time1, LocalTime time2) {
            return Math.abs(time1.toSecondOfDay() - time2.toSecondOfDay()) < 1800; // 30分钟内
        }

        private double calculateTrendFactor(List<LoadMetric> historicalLoad) {
            if (historicalLoad.size() < 2) {
                return 0.0;
            }

            // 计算最近一周的负载增长趋势
            List<LoadMetric> recentLoad = historicalLoad.stream()
                .filter(metric -> metric.getTimestamp().isAfter(LocalDateTime.now().minusDays(1)))
                .sorted(Comparator.comparing(LoadMetric::getTimestamp))
                .collect(Collectors.toList());

            if (recentLoad.size() < 2) {
                return 0.0;
            }

            double firstLoad = recentLoad.get(0).getLoad();
            double lastLoad = recentLoad.get(recentLoad.size() - 1).getLoad();

            return (lastLoad - firstLoad) / firstLoad;
        }
    }
}
```

### 13.3 运维和监控挑战

#### 全链路监控系统
```java
@Component
public class ComprehensiveMonitoringSystem {

    /**
     * 应用性能监控
     */
    public class ApplicationPerformanceMonitoring {

        @EventListener
        public void onJobStart(JobStartEvent event) {
            // 记录作业开始时间
            meterRegistry.counter("job.started", "job_name", event.getJobName()).increment();

            // 创建作业执行时间计时器
            Timer.Sample sample = Timer.start(meterRegistry);
            jobTimers.put(event.getJobId(), sample);
        }

        @EventListener
        public void onJobComplete(JobCompleteEvent event) {
            // 记录作业完成
            meterRegistry.counter("job.completed",
                "job_name", event.getJobName(),
                "status", event.getStatus().toString()).increment();

            // 停止计时器
            Timer.Sample sample = jobTimers.remove(event.getJobId());
            if (sample != null) {
                sample.stop(Timer.builder("job.duration")
                    .tag("job_name", event.getJobName())
                    .register(meterRegistry));
            }

            // 记录处理的数据量
            meterRegistry.gauge("job.records_processed",
                Tags.of("job_name", event.getJobName()),
                event.getRecordsProcessed());
        }

        @Scheduled(fixedDelay = 30000)  // 每30秒收集一次指标
        public void collectMetrics() {
            // 收集JVM指标
            Runtime runtime = Runtime.getRuntime();
            meterRegistry.gauge("jvm.memory.used", runtime.totalMemory() - runtime.freeMemory());
            meterRegistry.gauge("jvm.memory.total", runtime.totalMemory());
            meterRegistry.gauge("jvm.memory.max", runtime.maxMemory());

            // 收集Spark指标
            if (sparkContext != null) {
                SparkStatusTracker statusTracker = sparkContext.statusTracker();
                meterRegistry.gauge("spark.executors.active", statusTracker.getExecutorInfos().length);

                for (SparkExecutorInfo executor : statusTracker.getExecutorInfos()) {
                    meterRegistry.gauge("spark.executor.memory.used",
                        Tags.of("executor_id", executor.executorId()),
                        executor.memoryUsed());
                }
            }

            // 收集Kafka指标
            collectKafkaMetrics();
        }

        private void collectKafkaMetrics() {
            // 收集Kafka消费者lag
            Map<TopicPartition, Long> consumerLag = getConsumerLag();
            consumerLag.forEach((tp, lag) -> {
                meterRegistry.gauge("kafka.consumer.lag",
                    Tags.of("topic", tp.topic(), "partition", String.valueOf(tp.partition())),
                    lag);
            });

            // 收集Kafka生产者指标
            meterRegistry.gauge("kafka.producer.record_send_rate", getProducerSendRate());
        }
    }

    /**
     * 异常监控和告警
     */
    public class ExceptionMonitoringAndAlerting {

        @EventListener
        public void onException(ExceptionEvent event) {
            // 记录异常指标
            meterRegistry.counter("exceptions.count",
                "exception_type", event.getException().getClass().getSimpleName(),
                "component", event.getComponent()).increment();

            // 判断是否需要告警
            if (shouldAlert(event)) {
                sendAlert(event);
            }
        }

        private boolean shouldAlert(ExceptionEvent event) {
            String exceptionType = event.getException().getClass().getSimpleName();

            // 获取最近5分钟内同类型异常的数量
            long recentExceptionCount = getRecentExceptionCount(exceptionType, Duration.ofMinutes(5));

            // 如果5分钟内同类型异常超过10次，则告警
            if (recentExceptionCount > 10) {
                return true;
            }

            // 对于严重异常，立即告警
            if (isCriticalException(event.getException())) {
                return true;
            }

            return false;
        }

        private void sendAlert(ExceptionEvent event) {
            Alert alert = Alert.builder()
                .title("大数据系统异常告警")
                .message(String.format("组件: %s, 异常: %s, 消息: %s",
                    event.getComponent(),
                    event.getException().getClass().getSimpleName(),
                    event.getException().getMessage()))
                .severity(calculateSeverity(event))
                .timestamp(LocalDateTime.now())
                .build();

            // 发送到多个渠道
            alertService.sendEmail(alert);
            alertService.sendSlack(alert);
            alertService.sendSMS(alert);  // 仅对严重告警发送短信
        }

        private boolean isCriticalException(Exception exception) {
            return exception instanceof OutOfMemoryError ||
                   exception instanceof StackOverflowError ||
                   exception instanceof NoClassDefFoundError ||
                   exception.getMessage().contains("Connection refused") ||
                   exception.getMessage().contains("Timeout");
        }
    }

    /**
     * 健康检查
     */
    public class HealthCheckService {

        @Scheduled(fixedDelay = 60000)  // 每分钟检查一次
        public void performHealthCheck() {
            HealthCheckResult result = new HealthCheckResult();

            // 检查Spark集群健康状态
            result.setSparkHealth(checkSparkHealth());

            // 检查Kafka集群健康状态
            result.setKafkaHealth(checkKafkaHealth());

            // 检查HDFS健康状态
            result.setHdfsHealth(checkHdfsHealth());

            // 检查数据库连接
            result.setDatabaseHealth(checkDatabaseHealth());

            // 更新健康状态指标
            updateHealthMetrics(result);

            // 如果有组件不健康，发送告警
            if (!result.isOverallHealthy()) {
                sendHealthAlert(result);
            }
        }

        private HealthStatus checkSparkHealth() {
            try {
                if (sparkContext == null || sparkContext.isStopped()) {
                    return HealthStatus.DOWN;
                }

                // 检查执行器状态
                SparkExecutorInfo[] executors = sparkContext.statusTracker().getExecutorInfos();
                long activeExecutors = Arrays.stream(executors)
                    .filter(executor -> executor.isActive())
                    .count();

                if (activeExecutors == 0) {
                    return HealthStatus.DOWN;
                } else if (activeExecutors < executors.length * 0.8) {
                    return HealthStatus.DEGRADED;
                } else {
                    return HealthStatus.UP;
                }
            } catch (Exception e) {
                log.error("Error checking Spark health", e);
                return HealthStatus.DOWN;
            }
        }

        private HealthStatus checkKafkaHealth() {
            try {
                // 尝试获取Kafka集群元数据
                Properties props = new Properties();
                props.put("bootstrap.servers", kafkaBootstrapServers);
                props.put("request.timeout.ms", "5000");

                try (AdminClient adminClient = AdminClient.create(props)) {
                    DescribeClusterResult clusterResult = adminClient.describeCluster();
                    clusterResult.nodes().get(5, TimeUnit.SECONDS);
                    return HealthStatus.UP;
                }
            } catch (Exception e) {
                log.error("Error checking Kafka health", e);
                return HealthStatus.DOWN;
            }
        }

        private void updateHealthMetrics(HealthCheckResult result) {
            meterRegistry.gauge("health.spark", result.getSparkHealth().ordinal());
            meterRegistry.gauge("health.kafka", result.getKafkaHealth().ordinal());
            meterRegistry.gauge("health.hdfs", result.getHdfsHealth().ordinal());
            meterRegistry.gauge("health.database", result.getDatabaseHealth().ordinal());
            meterRegistry.gauge("health.overall", result.isOverallHealthy() ? 1 : 0);
        }
    }
}
```

---

> 💡 **项目实施建议**：大数据项目的成功关键在于合理的架构设计、完善的监控体系、有效的数据质量管理和持续的性能优化。建议采用敏捷开发方式，先构建MVP（最小可行产品），然后逐步完善功能和性能。

## 14. 项目实战案例

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