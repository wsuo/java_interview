# 题目组织指南

## 文件命名规范
- 使用英文命名，小写字母加连字符
- 格式：`[序号]-[主题]-[难度].md`
- 例子：`01-autoboxing-medium.md`、`02-singleton-hard.md`

## 难度分级
- **初级（Basic）**：基础语法和概念
- **中级（Medium）**：需要理解原理和实现
- **高级（Advanced）**：需要深入理解和优化

## 知识点标签
使用一致的标签帮助分类和搜索：

### 基础类标签
- `basic-syntax` - 基础语法
- `data-types` - 数据类型
- `operators` - 运算符
- `control-flow` - 控制流
- `exception-handling` - 异常处理

### 面向对象标签
- `oop-concepts` - 面向对象概念
- `inheritance` - 继承
- `polymorphism` - 多态
- `encapsulation` - 封装
- `abstract-interface` - 抽象类和接口

### 集合框架标签
- `collections` - 集合框架
- `list` - List接口
- `set` - Set接口
- `map` - Map接口
- `iterator` - 迭代器

### 并发编程标签
- `concurrency` - 并发编程
- `threads` - 线程
- `synchronization` - 同步机制
- `thread-pool` - 线程池
- `locks` - 锁机制

### JVM标签
- `jvm-memory` - JVM内存
- `garbage-collection` - 垃圾回收
- `classloader` - 类加载
- `jvm-tuning` - JVM调优

### 框架标签
- `spring-core` - Spring核心
- `spring-boot` - Spring Boot
- `spring-mvc` - Spring MVC
- `mybatis` - MyBatis
- `microservices` - 微服务

## 题目类型
- `concept` - 概念解释题
- `coding` - 编程实现题
- `design` - 设计题
- `debugging` - 调试题
- `performance` - 性能优化题

## 组织建议
1. 每个目录按照从易到难的顺序组织题目
2. 相关题目可以创建主题系列
3. 定期回顾和更新题目内容
4. 添加实际面试中遇到的变种问题

## 示例目录结构
```
collections/
├── README.md
├── 01-arraylist-vs-linkedlist-basic.md
├── 02-hashmap-implementation-medium.md
├── 03-concurrenthashmap-advanced.md
└── series/
    ├── hashmap-series.md
    └── concurrent-collections.md
```