# 集合框架

## 知识点概览
- Collection接口层次结构
- List、Set、Queue实现类
- Map接口及其实现
- 迭代器模式
- Collections工具类
- 性能比较和选择

## 核心集合类
### List
- ArrayList：动态数组实现
- LinkedList：双向链表实现
- Vector：线程安全的动态数组

### Set
- HashSet：哈希表实现
- LinkedHashSet：保持插入顺序
- TreeSet：红黑树实现，有序

### Map
- HashMap：哈希表实现
- LinkedHashMap：保持插入顺序
- TreeMap：红黑树实现，有序
- ConcurrentHashMap：线程安全

## 常见面试题
- ArrayList vs LinkedList性能比较
- HashMap的实现原理
- ConcurrentHashMap的分段锁机制
- 如何选择合适的集合类型？
- 集合的fail-fast机制