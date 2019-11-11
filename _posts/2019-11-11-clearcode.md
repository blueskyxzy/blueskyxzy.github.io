---
layout: post
title: 代码规范总结
date: 2019-11-11
tags: 总结    
---

不废话，直接总结

### Java Examples
#### 1.命名驼峰  
类名，方法名、参数名、成员变量、局部变量 驼峰命令规则，但以下情形例外:DO / BO / DTO / VO等  

#### 2.常量  
常量写在常量类中，不能代码写死即不允许出现未经定义的常量 并且不要只使用一个常量类，要归类成多个常量类方便维护

常量命名全部大写，单词间用下划线隔开

long或Long赋值时，使用大写的 L，小写l易跟数字 1 混淆

#### 3.POJO 类  
POJO类中布尔类型的变量，不要加 is，部分框架解析会引起序列化错误

#### 4.格式  
任何运算符左右必须加一个空格

方法参数在定义和传入时，多个参数逗号后边加个空格

#### 5.equals
equals 方法容易抛空指针异常,对象放后，常量放前，如String重写了equals方法

所有的相同类型的包装类对象之间值的比较，全部使用 equals 方法比较  
因为Integer 在-128至127之间的赋值，有缓存，就算是多个new 的也是同一个对象，

#### 6.基本数据类型与包装数据类型的使用标准
POJO类属性必须使用包装数据类型。  

RPC方法的返回值和参数使用包装数据类型。  

所有的局部变量使用基本数据类型。  

#### 7.序列化
序列化类新增属性时，请不要修改 serialVersionUID 字段，避免反序列失败;

如完全不兼容升级，那么请修改 serialVersionUID 值 来避免反序列化混乱，。

#### 8.多个字符串拼接
StringBuilder 的 append方法。

#### 9.clone拷贝对象
慎用Object 的 clone 方法来拷贝对象。  
对象的clone 方法默认是浅拷贝，若想实现深拷贝需要重写clone方法实现属性对象的拷贝

#### 10.hashCode 和 equals 
只要重写equals，就必须重写hashCode。  

Set存储的是不重复的对象，依据hashCode和equals进行判断，所以Set存储的对象必须重写这两个方法。  

如果自定义对象做为Map的键，那么必须重写hashCode和equals。 

#### 11.循环体中增加和删除元素
对原集合元素个数的修改，如增加、删除会产生ConcurrentModificationException 异常

#### 12.toArray方法
集合转数组的方法，必须使用集合的toArray(T[] array)。如果使用toArray无参方法，返回值只能是Object[]类，若强转其它类型数组将出现 ClassCastException 错误

#### 13.Arrays.asList()
Arrays.asList()把数组转换成集合时，不能使用add/remove/clear方法，不然会抛出UnsupportedOperationException 异常。

#### 14.split
String的split方法用"|"， "."等字符分割字符串时有问题， 需要转义

#### 15.BigDecimal精度丢失问题
new BigDecimal(包装类型)，由于数是二进制存的，转的时候回精度丢失。建议用BigDecimal.valueOf(包装类型)或者new BigDecimal(String), BigDecimal.valueOf底层代码用的就是new BigDecimal(String)

#### 16.泛型
泛型通配符<? extends T>来接收返回的数据，此写法的泛型集合不能使用add方法，而<? super T>不能使用get方法，做为接口调用赋值时易出错。

#### 17.线程安全
单例对象，资源驱动类、工具类、单例工厂类需要保证线程安全，其中的方法也要保证线程安全。

线程不安全的类一般不要定义为static变量，如果定义为static，必须加锁的线程安全类

Instant 代替 Date，LocalDateTime 代替 Calendar， DateTimeFormatter 代替 Simpledateformatter，

 Random 实例被多线程使用，虽然共享该实例是线程安全的，但会因竞争同一 seed 导致的性能下降  
在 JDK7 之后，可以直接使用 API ThreadLocalRandom，在 JDK7 之前，可以做到每个 线程一个实例。

#### 18.Executors 和 ThreadPoolExecutor 
线程池不允许使用 Executors 去创建，建议用ThreadPoolExecutor 的方式，因为Executors的四个线程池有资料的浪费  
1)FixedThreadPool 和 SingleThreadPool:  
允许的请求队列长度为 Integer.MAX_VALUE，可能会堆积大量的请求，从而导致 OOM。  
2)CachedThreadPool 和 ScheduledThreadPool:  
允许的创建线程数量为 Integer.MAX_VALUE，可能会创建大量的线程，从而导致 OOM。

#### 19.高并发锁
高并发时，同步调用应该去考量锁的性能损耗。能用无锁数据结构，就不要用锁;能 锁区块，就不要锁整个方法体;能用对象锁，就不要用类锁

对多个资源、数据库表、对象同时加锁时，需要保持一致的加锁顺序，否则可能会造 成死锁。

并发修改同一记录时，避免更新丢失，需要加锁。要么在应用层加锁，要么在缓存加 锁，要么在数据库层使用乐观锁，使用 version 作为更新依据。  
说明:如果每次访问冲突概率小于 20%，推荐使用乐观锁，否则使用悲观锁。乐观锁的重试次 数不得小于 3 次。

#### 20.并行处理定时任务
多线程并行处理定时任务时，Timer 运行多个 TimeTask 时，只要其中之一没有捕获 抛出的异常，其它任务便会自动终止运行，使用 ScheduledExecutorService 则没有这个问题。

#### 21.CountDownLatch 
CountDownLatch 进行异步转同步操作，每个线程退出前必须调用 countDown方法，  
线程执行代码注意 catch 异常，确保 countDown 方法可以执行，避免主线程无法执行 至 await 方法，直到超时才返回结果。

#### 22.volatile线程安全问题
volatile 解决多线程内存不可见问题。对于一写多读，是可以解决变量同步问题， 但是如果多写，同样无法解决线程安全问题  
如果是 count++操作，使用如下类实现: AtomicInteger count = new AtomicInteger(); count.addAndGet(1); 如果是 JDK8，推 荐使用 LongAdder 对象，比 AtomicLong 性能更好(减少乐观锁的重试次数)

#### 23.ThreadLocal
ThreadLocal 无法解决共享对象的更新问题，ThreadLocal 对象建议使用 static 修饰。  
这个变量是针对一个线程内所有操作共有的，所以设置为静态变量，所有此类实例共享 此静态变量 ，也就是说在类第一次被使用时装载，只分配一块存储空间，所有此类的对象(只 要是这个线程内定义的)都可以操控这个变量。

#### 24.注释
类、类属性、类方法的注释使用/**内容*/格式，不建议使用 //xxx 方式

#### 25.异常处理
RuntimeException 比如:IndexOutOfBoundsException，NullPointerException可以通过预先检查进行规避，而不应该 通过catch 来处理

#### 26.日志
生产环境禁输出debug日志;注意不要打印过多info,warn日志，不然服务器磁盘可能会撑爆，并定期检查服务器，删除过时日志

#### 27.MYSQL建表
varchar 是可变长字符串，不预先分配存储空间，长度不要超过 5000，如果存储长 度大于此值，定义字段类型为 text，独立出来一张表，用主键来对应。

单表行数超过500万行或者单表容量超过 2GB，才推荐进行分库分表。

#### 28.数据订正
数据订正时，要先select并备注修改的数据确认无误才能执行更新语句。

数据库不建议delete数据

#### 29.sql
查询中，不要使用 * 作为查询的字段列表

count(*)会统计值为 NULL 的行，而 count(列名)不会统计此列为 NULL 值的行
