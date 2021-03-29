## 面试流程
1. ⾃自我介绍(2分钟)
2. 聊简历上的项目(5分钟) 
3. 基础知识(15分钟)
4. 算法题(20分钟)
5. 提问环节(3分钟)

## 基础知识
### 操作系统
1. [进程 & 线程 & 协程](https://www.cnblogs.com/lxmhhy/p/6041001.html)
	- 进程是具有一定功能的程序关于某个数据集合上的⼀一次运⾏行行活动，进程是系统 进⾏行行资源调度和分配的⼀一个独⽴立单位
	- 线程是进程的实体，是CPU调度和分派的基本单位，它是⽐比进程更更⼩小的能独⽴立 运⾏行行的基本单位
	- 一个进程可以有多个线程，多个线程也可以并发执⾏行行

2. [进程间通信](https://www.jianshu.com/p/c1015f5ffa74)
	- 管道 (普通管道PIPE 、流管道(s_pipe)、命名管道(name_pipe)) 
	- 系统IPC(包括消息队列列、信号量、共享存储)
	- SOCKET

3. Linux系统命令
	- 查看日志命令，压测用到命令

4. Linux从开机到登陆页面，进行了什么操作 [...](https://blog.csdn.net/changexhao/article/details/80913699)

5. top/ps (查看系统进程信息/cpu使用率)[...](https://www.cnblogs.com/zhoug2020/p/6336503.html) 

6. 查看大日志文件（grep, head, tail , awk）
7. 文件权限，rm/shell
8. 软连接/硬链接 [...](https://xzchsia.github.io/2020/03/05/linux-hard-soft-link/)inode[...](https://www.ruanyifeng.com/blog/2011/12/inode.html)
9. 孤儿进程和僵尸进程（僵尸进程过多）
10. python协程，阻塞io/非阻塞io[...](https://cloud.tencent.com/developer/article/1684951)
11. 

### 计算机网络
1. tcp/ip 4层模型 [...](https://www.cnblogs.com/BlueTzar/articles/811160.html)--> tcp数据格式中的ack,rst,syn,fin的含义
2. 请简单说一下你了解的端口及对应的服务
3. hosts文件作用。[...](https://blog.csdn.net/qq_35246620/article/details/66970211)
1. TCP/UDP区别  ——> http,ftp.telnet属于tcp还是udp --> snmp发送信息属于tcp,udp
2. [TCP3次握手/4次挥手](https://juejin.im/post/5d9c284b518825095879e7a5)
	- TCP第四次挥⼿手为什什么要等待2MSL?
5. ip类型 
	- A类:10.0.0.0 - 10.255.255.255
	- B类:172.16.0.0 - 172.31.255.255
	- C类:192.168.0.0 - 192.168.255.255
	- -->子网掩码
	- 假设两个IP地址分别是172.20.0.18和172.20.1.16，子网掩码都是255.255.255.0，主机能否直接通信
6. 常见http状态码
	- 1XX 信息性状态码(Informational) 服务器器正在处理理请求
	- 2XX 成功状态码(Success) 请求已正常处理理完毕
	- 3XX 重定向状态码(Redirection) 需要进⾏行行额外操作以完成请求
	- 4XX 客户端错误状态码(Client Error) 客户端原因导致服务器器⽆无法处理理请求 
	- 5XX 服务器器错误状态码(Server Error) 服务器器原因导致处理理请求出错
	
7. 在浏览器中输入www.163.com，经历哪些过程？[...](https://segmentfault.com/a/1190000006879700)

8. tcpdump抓包
9. 滑动窗口
8. ipv4不够用了，扩充ip地址？

### 数据库
1. [数据库隔离级别](https://zhuanlan.zhihu.com/p/25419593)
	- read uncommitted(读取未提交数据)
	- read committed(可以读取其他事务提交的数据)
	- repeatable read(可重读，两次都是读取到10)---MySQL默认的隔离级别). 
	- serializable(串行化)
	
2. mvcc(DATA_TRX_ID, DATA_ROLL_PTR,链表)[...](https://cloud.tencent.com/developer/article/1454636)
	- SELECT 操作可以不加锁而是通过 MVCC 机制读取指定的版本历史记录，并通过一些手段保证保证读取的记录值符合事务所处的隔离级别[...](https://zhuanlan.zhihu.com/p/64576887)
	- RR 下的 ReadView 生成：每个事务 touch first read 时（本质上就是执行第一个 SELECT 语句时，后续所有的 SELECT 都是复用这个 ReadView
	- RC 下的 ReadView 生成：生成 ReadView 的时间点不同，一个是事务之后第一个 SELECT 语句开始、一个是事务中每条 SELECT 语句开始
2. 长连接/端链接
	show processfull，最大链接，空闲链接
	
3. select for update [...](https://www.jianshu.com/p/1dae2393270d)

3. innodb锁级别
 	- 表级锁：开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高,并发度最低。
	- 行级锁：开销大，加锁慢；会出现死锁；锁定粒度最小，发生锁冲突的概率最低,并发度也最高。
	- 页面锁：开销和加锁时间界于表锁和行锁之间；会出现死锁；锁定粒度界于表锁和行锁之间，并发度一般
	
	什么情况下会造成死锁，怎么解决？
		查出的进程杀死 kill
	
	乐观锁/悲观锁
	
	```
	悲观锁和乐观锁是数据库用来保证数据并发安全防止更新丢失的两种方法，例子在select ... for update前加个事务就可以防止更新丢失。悲观锁和乐观锁大部分场景下差异不大，一些独特场景下有一些差别，一般我们可以从如下几个方面来判断。
	
	响应速度： 如果需要非常高的响应速度，建议采用乐观锁方案，成功就执行，不成功就失败，不需要等待其他并发去释放锁。'
	冲突频率： 如果冲突频率非常高，建议采用悲观锁，保证成功率，如果冲突频率大，乐观锁会需要多次重试才能成功，代价比较大。
	重试代价： 如果重试代价大，建议采用悲观锁。
	```	
	
	[共享锁/排他锁]	(https://zhuanlan.zhihu.com/p/48127815)
	
	[记录锁/间隙锁](https://zhuanlan.zhihu.com/p/48269420)
	
	
4. 平时开发的时候对数据库或者SQL语句的优化
5. 优化sql语句 -explain, groupby,orderby
6. redis跳表？
### 软件工程（设计模式）
1. 几种模式/详细描述单例模式[...](https://juejin.im/post/5cf09270f265da1bd260d239)
2. 当你有一个db操作的接口，但是在具体对象查询到sql前需要额外做一些操作，或做一些限制等，你觉得可以用哪种设计模式 （代理）

### 编程语言
1. java
	- 针对Spring，问一下 IoC、 AOP
	- 有项目经验的， 可以问一下： 如何对自己的代码做测试的
	- Object类有哪些方法，请全部列出
		```
		getClass⽅方法 
		hashCode⽅方法 
		equals⽅方法
		clone⽅方法
		toString⽅方法
		notify⽅方法
		notifyAll⽅方法
		wait(long timeout) throws InterruptedException⽅方法
		wait(long timeout, int nanos) throws InterruptedException⽅方法
		wait() throws InterruptedException⽅方法 !!. finalize⽅方法
		```
	- HashMap、Hashtable、ConcurrentHashMap的区别？[戳](https://www.jianshu.com/p/0e3c7bfa6064)
	- [线程同步方法](https://www.cnblogs.com/xhjt/p/3897440.html)
		synchronized, volatile, ReentrantLock
	- [equals⽅方法 vs ==](https://zhuanlan.zhihu.com/p/77616029)
		 - equals调⽤用对象的equals⽅方法，没有覆盖则⽐比较内存地址
      -  ==⽐比较对象内存地址
2. python
	- 装饰器运用


### 其他
1. docker/k8s
2. vim/git/ci

## 算法与数据结构
1. 动态规划硬币问题

给定【1，3，5】面额的硬币(coins)和一个总金额(amount)。写一个函数来计算可以凑成总金额所需的最少的硬币个数 https://zhuanlan.zhihu.com/p/61277271

2. [排序（归并，快排，冒泡）思想](https://www.runoob.com/w3cnote/ten-sorting-algorithm.html)

3. 给定一个链表，判断链表中是否有环。 (https://leetcode-cn.com/problems/linked-list-cycle/solution/huan-xing-lian-biao-by-leetcode-solution/)
结构体 ListNode{val, next} 

3.1 找到链表的入口 （https://leetcode-cn.com/problems/linked-list-cycle-ii/solution/huan-xing-lian-biao-ii-by-leetcode-solution/）

4. hash的存储结构 [...](https://jishuin.proginn.com/p/763bfbd242b8)
	- 容量，动态因子
	- 开放地址法，链表法，[区别](https://blog.csdn.net/weixin_41563161/article/details/105104239)

## 项目经验
1. (如果对方有web开发经验)怎么理解微服务？
2. (如果对方有web开发经验)怎么理解RESTful?
3. 项目里面，用户的(接口、数据)访问权限是如何控制的？
4. 一个功能的实现，除了基本功能要求，还会考虑什么（性能、测试与后期维护、部署复杂度、异常和故障考虑）？如果性能有要求，该怎么达到？
5. 有没有考虑过项目如何做到解耦合
6. (如果对方有web开发经验) 想要记录用户的访问行为日志并做一些统计分析，
该如何实现？应该针对哪些指标做统计呢？
7. ajax跨域，同源策略
8. 技术难点



