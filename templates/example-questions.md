# 示例题目

## 示例1：基础概念题

### 问题：什么是Java中的自动装箱和拆箱？
**难度等级：** 中级  
**知识点：** 包装类、自动装箱、性能优化  
**类型：** 概念题  

### 问题描述
请解释Java中的自动装箱(Autoboxing)和自动拆箱(Unboxing)机制，并说明它们的工作原理和可能的性能影响。

### 答案解析
自动装箱和拆箱是Java 5引入的特性：

**自动装箱（Autoboxing）**：
- 基本数据类型自动转换为对应的包装类对象
- 例如：`int` → `Integer`

**自动拆箱（Unboxing）**：
- 包装类对象自动转换为对应的基本数据类型
- 例如：`Integer` → `int`

**工作原理**：
编译器会自动调用`valueOf()`方法进行装箱，调用`xxxValue()`方法进行拆箱。

**性能影响**：
- 创建对象的开销
- 可能的空指针异常
- 缓存机制的影响

### 代码示例
```java
public class AutoBoxingExample {
    public static void main(String[] args) {
        // 自动装箱
        Integer num1 = 10;  // 等同于 Integer.valueOf(10)
        
        // 自动拆箱
        int num2 = num1;    // 等同于 num1.intValue()
        
        // 性能对比
        long start = System.currentTimeMillis();
        
        // 使用基本类型
        int sum1 = 0;
        for (int i = 0; i < 1000000; i++) {
            sum1 += i;
        }
        
        // 使用包装类型（会触发自动装箱拆箱）
        Integer sum2 = 0;
        for (int i = 0; i < 1000000; i++) {
            sum2 += i;  // 每次都有装箱拆箱操作
        }
        
        long end = System.currentTimeMillis();
        System.out.println("耗时: " + (end - start) + "ms");
    }
}
```

### 扩展问题
- Integer缓存机制的范围是什么？
- 什么情况下会出现空指针异常？
- 如何避免不必要的装箱拆箱？

### 参考资料
- Oracle官方文档：Autoboxing and Unboxing
- Effective Java：第61条

---

## 示例2：编程题

### 问题：实现一个线程安全的单例模式
**难度等级：** 中级  
**知识点：** 设计模式、线程安全、双重检查锁定  
**类型：** 编程题  

### 问题描述
请用Java实现一个线程安全的单例模式，要求：
1. 线程安全
2. 延迟初始化
3. 高性能（避免不必要的同步）

### 答案解析
推荐使用双重检查锁定（Double-Checked Locking）模式：

**关键点**：
1. 使用`volatile`关键字确保可见性
2. 双重检查减少同步开销
3. 静态内部类方式也是好的选择

### 代码示例
```java
/**
 * 双重检查锁定单例模式
 */
public class Singleton {
    // volatile确保多线程环境下的可见性
    private static volatile Singleton instance;
    
    // 私有构造函数
    private Singleton() {
        // 防止反射攻击
        if (instance != null) {
            throw new RuntimeException("请使用getInstance()方法");
        }
    }
    
    public static Singleton getInstance() {
        // 第一次检查
        if (instance == null) {
            synchronized (Singleton.class) {
                // 第二次检查
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}

/**
 * 静态内部类方式（推荐）
 */
public class SingletonInner {
    private SingletonInner() {}
    
    private static class SingletonHolder {
        private static final SingletonInner INSTANCE = new SingletonInner();
    }
    
    public static SingletonInner getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

/**
 * 枚举方式（最简单）
 */
public enum SingletonEnum {
    INSTANCE;
    
    public void doSomething() {
        System.out.println("执行操作");
    }
}
```

### 扩展问题
- 为什么需要使用volatile关键字？
- 静态内部类方式的优势是什么？
- 如何防止反射和序列化攻击？

### 参考资料
- Effective Java：第3条
- 《Java并发编程的艺术》