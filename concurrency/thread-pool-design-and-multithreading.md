# Java线程池设计与多线程编程实战

> 🧵 深入理解Java并发编程的核心机制和最佳实践

## 🎯 线程池核心概念

### 为什么需要线程池？
1. **降低资源消耗**：重复利用已创建的线程，减少线程创建和销毁的开销
2. **提高响应速度**：任务到达时，无需等待线程创建即可立即执行
3. **提高线程的可管理性**：统一分配、调优和监控线程资源

### 线程池的工作原理
```java
// 线程池执行流程
public class ThreadPoolWorkFlow {
    /*
     * 1. 提交任务到线程池
     * 2. 判断核心线程池是否已满
     *    - 未满：创建核心线程执行任务
     *    - 已满：进入步骤3
     * 3. 判断工作队列是否已满
     *    - 未满：任务加入队列等待
     *    - 已满：进入步骤4
     * 4. 判断最大线程池是否已满
     *    - 未满：创建非核心线程执行任务
     *    - 已满：执行拒绝策略
     */
}
```

## 🏗️ ThreadPoolExecutor详解

### 核心参数解析
```java
public ThreadPoolExecutor(
    int corePoolSize,              // 核心线程数
    int maximumPoolSize,           // 最大线程数
    long keepAliveTime,            // 非核心线程空闲存活时间
    TimeUnit unit,                 // 时间单位
    BlockingQueue<Runnable> workQueue,  // 工作队列
    ThreadFactory threadFactory,   // 线程工厂
    RejectedExecutionHandler handler     // 拒绝策略
) {
    // 构造函数实现
}
```

### 参数配置实战
```java
@Configuration
public class ThreadPoolConfig {
    
    /**
     * CPU密集型任务线程池
     * 核心线程数 = CPU核心数 + 1
     */
    @Bean("cpuIntensivePool")
    public ThreadPoolExecutor cpuIntensivePool() {
        int coreSize = Runtime.getRuntime().availableProcessors() + 1;
        return new ThreadPoolExecutor(
            coreSize,                    // 核心线程数
            coreSize,                    // 最大线程数（CPU密集型不需要更多线程）
            60L,                         // 空闲存活时间
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(100),  // 有界队列防止OOM
            new CustomThreadFactory("cpu-pool"),
            new ThreadPoolExecutor.CallerRunsPolicy()  // 调用者运行策略
        );
    }
    
    /**
     * IO密集型任务线程池
     * 核心线程数 = CPU核心数 * 2
     */
    @Bean("ioIntensivePool")
    public ThreadPoolExecutor ioIntensivePool() {
        int coreSize = Runtime.getRuntime().availableProcessors() * 2;
        return new ThreadPoolExecutor(
            coreSize,                    // 核心线程数
            coreSize * 2,               // 最大线程数
            300L,                       // 较长的空闲时间
            TimeUnit.SECONDS,
            new LinkedBlockingQueue<>(200),
            new CustomThreadFactory("io-pool"),
            new ThreadPoolExecutor.AbortPolicy()
        );
    }
    
    /**
     * 自定义线程工厂
     */
    public static class CustomThreadFactory implements ThreadFactory {
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;
        
        public CustomThreadFactory(String namePrefix) {
            this.namePrefix = namePrefix + "-thread-";
        }
        
        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, namePrefix + threadNumber.getAndIncrement());
            t.setDaemon(false);  // 设置为用户线程
            t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
}
```

### 工作队列选择策略
```java
public class QueueTypeExample {
    
    // 1. ArrayBlockingQueue - 有界阻塞队列
    BlockingQueue<Runnable> arrayQueue = new ArrayBlockingQueue<>(1000);
    // 特点：基于数组，FIFO，有界，可防止OOM
    
    // 2. LinkedBlockingQueue - 链表阻塞队列
    BlockingQueue<Runnable> linkedQueue = new LinkedBlockingQueue<>(1000);
    // 特点：基于链表，FIFO，可设置容量，默认Integer.MAX_VALUE
    
    // 3. SynchronousQueue - 同步队列
    BlockingQueue<Runnable> syncQueue = new SynchronousQueue<>();
    // 特点：不存储元素，每个插入操作必须等待移除操作
    
    // 4. PriorityBlockingQueue - 优先级队列
    BlockingQueue<Runnable> priorityQueue = new PriorityBlockingQueue<>();
    // 特点：支持优先级排序，无界队列
    
    // 5. DelayQueue - 延迟队列
    BlockingQueue<Runnable> delayQueue = new DelayQueue<>();
    // 特点：支持延迟获取元素，元素必须实现Delayed接口
}
```

### 拒绝策略详解
```java
public class RejectionPolicyExample {
    
    // 1. AbortPolicy（默认）- 抛出异常
    RejectedExecutionHandler abortPolicy = new ThreadPoolExecutor.AbortPolicy();
    
    // 2. CallerRunsPolicy - 调用者运行
    RejectedExecutionHandler callerRunsPolicy = new ThreadPoolExecutor.CallerRunsPolicy();
    
    // 3. DiscardPolicy - 静默丢弃
    RejectedExecutionHandler discardPolicy = new ThreadPoolExecutor.DiscardPolicy();
    
    // 4. DiscardOldestPolicy - 丢弃最老的任务
    RejectedExecutionHandler discardOldestPolicy = new ThreadPoolExecutor.DiscardOldestPolicy();
    
    // 5. 自定义拒绝策略
    RejectedExecutionHandler customPolicy = new RejectedExecutionHandler() {
        @Override
        public void rejectedExecution(Runnable r, ThreadPoolExecutor executor) {
            // 记录日志
            log.warn("Task {} rejected from {}", r.toString(), executor.toString());
            
            // 可以选择：
            // 1. 放入备用队列
            // 2. 持久化到数据库
            // 3. 发送到消息队列
            // 4. 降级处理
            
            try {
                // 尝试放入备用队列，设置超时时间
                boolean success = backupQueue.offer(r, 100, TimeUnit.MILLISECONDS);
                if (!success) {
                    log.error("Backup queue is also full, task discarded: {}", r);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                log.error("Interrupted while offering to backup queue", e);
            }
        }
    };
}
```

## 🛠️ JDK并发工具类详解

### 1. CountDownLatch - 倒计时门闩
```java
@Service
public class CountDownLatchExample {
    
    /**
     * 等待多个服务启动完成
     */
    public void waitForServicesStartup() throws InterruptedException {
        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);
        
        // 启动数据库服务
        executor.submit(() -> {
            try {
                startDatabaseService();
                log.info("Database service started");
            } finally {
                latch.countDown();  // 计数器减1
            }
        });
        
        // 启动缓存服务
        executor.submit(() -> {
            try {
                startCacheService();
                log.info("Cache service started");
            } finally {
                latch.countDown();
            }
        });
        
        // 启动消息队列服务
        executor.submit(() -> {
            try {
                startMessageQueueService();
                log.info("Message queue service started");
            } finally {
                latch.countDown();
            }
        });
        
        // 等待所有服务启动完成
        boolean allStarted = latch.await(30, TimeUnit.SECONDS);
        if (allStarted) {
            log.info("All services started successfully");
        } else {
            log.error("Some services failed to start within timeout");
        }
    }
}
```

### 2. CyclicBarrier - 循环屏障
```java
@Service
public class CyclicBarrierExample {
    
    /**
     * 多线程并行计算，等待所有线程完成后汇总结果
     */
    public void parallelCalculation() {
        int threadCount = 4;
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
            // 所有线程到达屏障后执行的任务
            log.info("All threads completed calculation, starting result aggregation");
            aggregateResults();
        });
        
        List<Future<Integer>> futures = new ArrayList<>();
        
        for (int i = 0; i < threadCount; i++) {
            final int taskId = i;
            Future<Integer> future = executor.submit(() -> {
                try {
                    // 执行计算任务
                    int result = performCalculation(taskId);
                    log.info("Thread {} completed calculation: {}", taskId, result);
                    
                    // 等待其他线程完成
                    barrier.await();
                    
                    return result;
                } catch (InterruptedException | BrokenBarrierException e) {
                    log.error("Thread {} interrupted", taskId, e);
                    return 0;
                }
            });
            futures.add(future);
        }
        
        // 获取所有结果
        futures.forEach(future -> {
            try {
                Integer result = future.get();
                log.info("Final result: {}", result);
            } catch (Exception e) {
                log.error("Error getting result", e);
            }
        });
    }
}
```

### 3. Semaphore - 信号量
```java
@Service
public class SemaphoreExample {
    
    // 限制同时访问数据库的连接数
    private final Semaphore dbConnectionSemaphore = new Semaphore(10);
    
    /**
     * 数据库连接池管理
     */
    public void accessDatabase() {
        try {
            // 获取许可证
            boolean acquired = dbConnectionSemaphore.tryAcquire(5, TimeUnit.SECONDS);
            if (!acquired) {
                log.warn("Failed to acquire database connection within timeout");
                return;
            }
            
            try {
                // 执行数据库操作
                performDatabaseOperation();
                log.info("Database operation completed");
            } finally {
                // 释放许可证
                dbConnectionSemaphore.release();
            }
            
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Interrupted while waiting for database connection", e);
        }
    }
    
    /**
     * 限流器实现
     */
    @Component
    public static class RateLimiter {
        private final Semaphore semaphore;
        private final ScheduledExecutorService scheduler;
        
        public RateLimiter(int permitsPerSecond) {
            this.semaphore = new Semaphore(permitsPerSecond);
            this.scheduler = Executors.newScheduledThreadPool(1);
            
            // 每秒释放指定数量的许可证
            scheduler.scheduleAtFixedRate(() -> {
                semaphore.release(permitsPerSecond - semaphore.availablePermits());
            }, 1, 1, TimeUnit.SECONDS);
        }
        
        public boolean tryAcquire() {
            return semaphore.tryAcquire();
        }
        
        public boolean tryAcquire(long timeout, TimeUnit unit) throws InterruptedException {
            return semaphore.tryAcquire(timeout, unit);
        }
    }
}
```

### 4. Exchanger - 数据交换器
```java
public class ExchangerExample {
    
    /**
     * 生产者消费者数据交换
     */
    public void producerConsumerExchange() {
        Exchanger<List<String>> exchanger = new Exchanger<>();
        
        // 生产者线程
        executor.submit(() -> {
            List<String> producerData = new ArrayList<>();
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // 生产数据
                    for (int i = 0; i < 10; i++) {
                        producerData.add("data-" + System.currentTimeMillis());
                    }
                    
                    log.info("Producer generated {} items", producerData.size());
                    
                    // 与消费者交换数据
                    List<String> emptyList = exchanger.exchange(producerData);
                    producerData = emptyList;  // 获取空列表继续生产
                    
                    Thread.sleep(1000);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                log.info("Producer interrupted");
            }
        });
        
        // 消费者线程
        executor.submit(() -> {
            List<String> consumerData = new ArrayList<>();
            try {
                while (!Thread.currentThread().isInterrupted()) {
                    // 与生产者交换数据
                    List<String> dataToProcess = exchanger.exchange(consumerData);
                    
                    // 处理数据
                    log.info("Consumer received {} items", dataToProcess.size());
                    dataToProcess.forEach(this::processData);
                    
                    // 清空列表准备下次交换
                    consumerData.clear();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                log.info("Consumer interrupted");
            }
        });
    }
}
```

## 🔒 线程安全问题与解决方案

### 1. 可见性问题
```java
public class VisibilityProblem {
    
    // 问题：没有volatile修饰，可能存在可见性问题
    private boolean running = true;
    
    // 解决方案1：使用volatile
    private volatile boolean runningVolatile = true;
    
    // 解决方案2：使用AtomicBoolean
    private final AtomicBoolean runningAtomic = new AtomicBoolean(true);
    
    public void stopTask() {
        runningVolatile = false;
        // 或者
        runningAtomic.set(false);
    }
    
    public void taskLoop() {
        while (runningVolatile) {  // 使用volatile变量
            // 执行任务
            doWork();
        }
    }
}
```

### 2. 原子性问题
```java
public class AtomicityProblem {
    
    // 问题：非原子操作
    private int counter = 0;
    
    // 解决方案1：使用synchronized
    public synchronized void incrementSync() {
        counter++;
    }
    
    // 解决方案2：使用AtomicInteger
    private final AtomicInteger atomicCounter = new AtomicInteger(0);
    
    public void incrementAtomic() {
        atomicCounter.incrementAndGet();
    }
    
    // 解决方案3：使用Lock
    private final ReentrantLock lock = new ReentrantLock();
    
    public void incrementLock() {
        lock.lock();
        try {
            counter++;
        } finally {
            lock.unlock();
        }
    }
}
```

### 3. 有序性问题
```java
public class OrderingProblem {
    
    private volatile boolean initialized = false;
    private String data;
    
    // 写线程
    public void initializeData() {
        data = "some data";        // 1
        initialized = true;        // 2 - volatile写，建立happens-before关系
    }
    
    // 读线程
    public String readData() {
        if (initialized) {         // 3 - volatile读
            return data;           // 4 - 由于happens-before，保证能看到data的写入
        }
        return null;
    }
}
```

## 🎯 实际项目应用案例

### 1. 异步任务处理系统
```java
@Service
public class AsyncTaskService {
    
    @Autowired
    @Qualifier("ioIntensivePool")
    private ThreadPoolExecutor ioPool;
    
    @Autowired
    @Qualifier("cpuIntensivePool")
    private ThreadPoolExecutor cpuPool;
    
    /**
     * 异步处理用户上传的文件
     */
    public CompletableFuture<String> processFileAsync(MultipartFile file) {
        return CompletableFuture
            .supplyAsync(() -> {
                // IO密集型：保存文件
                String filePath = saveFile(file);
                log.info("File saved to: {}", filePath);
                return filePath;
            }, ioPool)
            .thenApplyAsync(filePath -> {
                // CPU密集型：处理文件内容
                String processedContent = processFileContent(filePath);
                log.info("File processed: {}", filePath);
                return processedContent;
            }, cpuPool)
            .thenApplyAsync(content -> {
                // IO密集型：保存处理结果
                String resultId = saveProcessResult(content);
                log.info("Result saved with ID: {}", resultId);
                return resultId;
            }, ioPool)
            .exceptionally(throwable -> {
                log.error("Error processing file", throwable);
                return "ERROR";
            });
    }
}
```

### 2. 批量数据处理
```java
@Service
public class BatchDataProcessor {
    
    private final ThreadPoolExecutor executor;
    private final int batchSize = 1000;
    
    /**
     * 并行处理大量数据
     */
    public void processBigData(List<DataItem> dataList) {
        int totalSize = dataList.size();
        int threadCount = Math.min(
            (totalSize + batchSize - 1) / batchSize,  // 计算需要的线程数
            Runtime.getRuntime().availableProcessors()
        );
        
        CountDownLatch latch = new CountDownLatch(threadCount);
        AtomicInteger processedCount = new AtomicInteger(0);
        
        // 分批处理数据
        for (int i = 0; i < threadCount; i++) {
            int startIndex = i * batchSize;
            int endIndex = Math.min(startIndex + batchSize, totalSize);
            List<DataItem> batch = dataList.subList(startIndex, endIndex);
            
            executor.submit(() -> {
                try {
                    processBatch(batch);
                    int processed = processedCount.addAndGet(batch.size());
                    log.info("Processed {}/{} items", processed, totalSize);
                } catch (Exception e) {
                    log.error("Error processing batch", e);
                } finally {
                    latch.countDown();
                }
            });
        }
        
        try {
            // 等待所有批次处理完成
            boolean completed = latch.await(30, TimeUnit.MINUTES);
            if (completed) {
                log.info("All batches processed successfully");
            } else {
                log.error("Processing timeout");
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            log.error("Processing interrupted", e);
        }
    }
}
```

## 🎯 面试高频问题

### Q1: 线程池的核心参数有哪些？如何合理设置？
**答案：**
- **corePoolSize**：CPU密集型设为CPU核数+1，IO密集型设为CPU核数×2
- **maximumPoolSize**：根据系统负载能力设置，通常为核心线程数的2倍
- **keepAliveTime**：非核心线程空闲时间，IO密集型可设置较长时间
- **workQueue**：有界队列防止OOM，大小根据内存和响应时间要求设置
- **threadFactory**：自定义线程名称便于调试
- **handler**：根据业务需求选择合适的拒绝策略

### Q2: volatile关键字的作用是什么？
**答案：**
1. **保证可见性**：一个线程修改volatile变量，其他线程立即可见
2. **禁止指令重排序**：在volatile变量读写前后建立内存屏障
3. **不保证原子性**：volatile不能保证复合操作的原子性

**使用场景：**
- 状态标志位
- 双重检查锁定中的实例变量
- 读多写少的场景

### Q3: synchronized和ReentrantLock的区别？
**答案：**

| 特性 | synchronized | ReentrantLock |
|------|-------------|---------------|
| 实现方式 | JVM内置关键字 | JDK类库实现 |
| 锁的获取 | 自动获取和释放 | 手动获取和释放 |
| 中断响应 | 不可中断 | 可中断 |
| 超时机制 | 不支持 | 支持 |
| 公平性 | 非公平锁 | 支持公平锁和非公平锁 |
| 条件变量 | 单一条件 | 支持多个条件变量 |
| 性能 | JVM优化好 | 功能更丰富 |

### Q4: 如何避免死锁？
**答案：**
1. **避免嵌套锁**：尽量避免一个线程同时获取多个锁
2. **锁顺序**：多个线程获取锁的顺序要一致
3. **超时机制**：使用tryLock(timeout)避免无限等待
4. **死锁检测**：定期检测死锁并处理

```java
// 避免死锁的示例
public class DeadlockAvoidance {
    private static final Object lock1 = new Object();
    private static final Object lock2 = new Object();
    
    // 按照锁的hashCode排序获取锁
    public void transferMoney(Account from, Account to, int amount) {
        Account firstLock = from.hashCode() < to.hashCode() ? from : to;
        Account secondLock = from.hashCode() < to.hashCode() ? to : from;
        
        synchronized (firstLock) {
            synchronized (secondLock) {
                from.debit(amount);
                to.credit(amount);
            }
        }
    }
}
```

### Q5: 什么是线程池的拒绝策略？各有什么特点？
**答案：**
1. **AbortPolicy**：抛出RejectedExecutionException异常
2. **CallerRunsPolicy**：调用者线程执行任务，提供负反馈
3. **DiscardPolicy**：静默丢弃任务
4. **DiscardOldestPolicy**：丢弃队列中最老的任务

**选择建议：**
- 重要任务：使用CallerRunsPolicy或自定义策略
- 可丢失任务：使用DiscardPolicy
- 需要监控：自定义策略记录日志

---

> 💡 **学习建议**：多线程编程的关键在于理解并发的本质问题（可见性、原子性、有序性），熟练掌握各种并发工具的使用场景，在实际项目中多实践和总结。
