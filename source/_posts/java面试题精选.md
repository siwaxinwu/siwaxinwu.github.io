---
\---

title:  java面试题精选
\---

随着业务的发展，访问量的极速增长，上述的方案很快不能满足性能需求。每次请求的响应时间越来越长，比如用户在H5页面上不断刷新商品，响应时间从最初的500毫秒增加到了2秒以上。业务高峰期，系统甚至出现过宕机。在这生死存亡的关键时刻，通过监控，我们发现高峰期MySQL CPU使用率已接近80%，磁盘IO使用率接近90%，slow  query（慢查询）从每天1百条上升到1万条，而且一天比一天严重。数据库俨然已成为瓶颈，我们必须得快速做架构升级。

采取了MySQL主从同步和应用服务端读写分离的方案。

MySQL支持主从同步，实时将主库的数据增量复制到从库，而且一个主库可以连接多个从库同步。利用此特性，我们在应用服务端对每次请求做读写判断，若是写请求，则把这次请求内的所有DB操作发向主库；若是读请求，则把这次请求内的所有DB操作发向从库，如下图所示。

实现读写分离后，数据库的压力减少了许多，CPU使用率和IO使用率都降到了5%以内，Slow Query（慢查询）也趋近于0。主从同步、读写分离给我们主要带来如下两个好处：

- 减轻了主库（写）压力：商城业务主要来源于读操作，做读写分离后，读压力转移到了从库，主库的压力减小了数十倍。
- 从库（读）可水平扩展（加从库机器）：因系统压力主要是读请求，而从库又可水平扩展，当从库压力太时，可直接添加从库机器，缓解读请求压力。

当然，没有一个方案是万能的。读写分离，暂时解决了MySQL压力问题，同时也带来了新的挑战。业务高峰期，用户提交完订单，在我的订单列表中却看不到自己提交的订单信息（典型的read after  write问题）；系统内部偶尔也会出现一些查询不到数据的异常。通过监控，我们发现，业务高峰期MySQL可能会出现主从复制延迟，极端情况，主从延迟高达数秒。这极大的影响了用户体验。

那如何监控主从同步状态？在从库机器上，执行show slave  status，查看Seconds_Behind_Master值，代表主从同步从库落后主库的时间，单位为秒，若主从同步无延迟，这个值为0。MySQL主从延迟一个重要的原因之一是主从复制是单线程串行执行（高版本MySQL支持并行复制）。

那如何避免或解决主从延迟？我们做了如下一些优化：

- 优化MySQL参数，比如增大innodb_buffer_pool_size，让更多操作在MySQL内存中完成，减少磁盘操作。
- 使用高性能CPU主机。
- 数据库使用物理主机，避免使用虚拟云主机，提升IO性能。
- 使用SSD磁盘，提升IO性能。SSD的随机IO性能约是SATA硬盘的10倍甚至更高。
- 业务代码优化，将实时性要求高的某些操作，强制使用主库做读操作。
- 升级高版本MySQL，支持并行主从复制。

读写分离很好的解决了读压力问题，每次读压力增加，可以通过加从库的方式水平扩展。但是写操作的压力随着业务爆发式的增长没有得到有效的缓解，比如用户提交订单越来越慢。通过监控MySQL数据库，我们发现，数据库写操作越来越慢，一次普通的insert操作，甚至可能会执行1秒以上。

另一方面，业务越来越复杂，多个应用系统使用同一个数据库，其中一个很小的非核心功能出现延迟，常常影响主库上的其它核心业务功能。这时，主库成为了性能瓶颈，我们意识到，必须得再一次做架构升级，将主库做拆分，一方面以提升性能，另一方面减少系统间的相互影响，以提升系统稳定性。这一次，我们将系统按业务进行了垂直拆分。如下图所示，将最初庞大的数据库按业务拆分成不同的业务数据库，每个系统仅访问对应业务的数据库，尽量避免或减少跨库访问。

垂直分库过程，我们也遇到不少挑战，最大的挑战是：不能跨库join，同时需要对现有代码重构。单库时，可以简单的使用join关联表查询；拆库后，拆分后的数据库在不同的实例上，就不能跨库使用join了。

经过近十天加班加点的底层架构调整，以及业务代码重构，终于完成了数据库的垂直拆分。拆分之后，每个应用程序只访问对应的数据库，一方面将单点数据库拆分成了多个，分摊了主库写压力；另一方面，拆分后的数据库各自独立，实现了业务隔离，不再互相影响。

## ID生成器

ID生成器是整个水平分库的核心，它决定了如何拆分数据，以及查询存储-检索数据。ID需要跨库全局唯一，否则会引发业务层的冲突。此外，ID必须是数字且升序，这主要是考虑到升序的ID能保证MySQL的性能（若是UUID等随机字符串，在高并发和大数据量情况下，性能极差）。同时，ID生成器必须非常稳定，因为任何故障都会影响所有的数据库操作。

![image-20201111203319041](C:\Users\siwaxinwu5\AppData\Roaming\Typora\typora-user-images\image-20201111203319041.png)

在mysql命令加上选项-U后，当发出没有WHERE或LIMIT关键字的UPDATE或DELETE时，MySQL程序拒绝执行。

## 二、根据上面的程序来理解下面的执行原理。

下面分几步来解释：

1.`DispatcherServlet`：前端控制器，作为整个SpringMVC的控制中心。用户发出请求，DispatcherServlet接收请求并拦截请求。

2.`HandlerMapping`：处理器映射器，DispatcherServlet调用HandlerMapping,HandlerMapping根据请求url去查找对应的处理。

3.`HandlerExecution`：具体的handler(处理)，将解析后的url传递给DispatcherServlet。

4.`HandlerAdapter`：处理器适配器，将DispatcherServlet传递的信息去执行相应的controller。

5.Controller层中调用service层，获得数据放在ModelAndView对象中，并给ModelAndView设置页面信息。

6.HandlerAdapter将视图名传递给DispatcherServlet。

7.DispatcherServlet调用视图解析器来解析HandlerAdapter传递的视图名。

8.视图解析器将解析的视图名传给DispatcherServlet。

9.DispatcherServlet根据视图解析器返回的视图名调用具体的视图。

10.用户获得视图。

实现微服务网关的技术有很多，

- nginx  Nginx (engine x) 是一个高性能的HTTP和反向代理web服务器，同时也提供了IMAP/POP3/SMTP服务
- zuul ,Zuul 是 Netflix 出品的一个基于 JVM 路由和服务端的负载均衡器。
- spring-cloud-gateway, 是spring 出品的 基于spring 的网关项目，集成断路器，路径重写，性能比Zuul好。





## 1.3 微服务为什么要使用网关呢？

不同的微服务一般会有不同的网络地址，而外部客户端可能需要调用多个服务的接口才能完成一个业务需求，如果让客户端直接与各个微服务通信，会有以下的问题：

1. 客户端会多次请求不同的微服务，增加了客户端的复杂性
2. 存在跨域请求，在一定场景下处理相对复杂
3. 认证复杂，每个服务都需要独立认证
4. 难以重构，随着项目的迭代，可能需要重新划分微服务。例如，可能将多个服务合并成一个或者将一个服务拆分成多个。如果客户端直接与微服务通信，那么重构将会很难实施
5. 某些微服务可能使用了防火墙 / 浏览器不友好的协议，直接访问会有一定的困难

以上这些问题可以借助网关解决。



## 1.4 微服务网关优点

- 安全 ，只有网关系统对外进行暴露，微服务可以隐藏在内网，通过防火墙保护。
- 易于监控。可以在网关收集监控数据并将其推送到外部系统进行分析。
- 易于认证。可以在网关上进行认证，然后再将请求转发到后端的微服务，而无须在每个微服务中进行认证。
- 减少了客户端与各个微服务之间的交互次数
- 易于统一授权。

CORS（CORS，Cross-origin resource sharing）跨域源资源共享

所谓同源是指协议、域名以及端口要相同。同源策略是基于安全方面的考虑提出来的，

@CrossOrigin(value = "http://localhost:8081")

在Spring Boot中，还可以通过全局配置一次性解决这个问题，全局配置只需要在配置类中重写addCorsMappings方法即可，如

1. @Configuration`
2. `public class WebMvcConfig implements WebMvcConfigurer {`
3. `    @Override`
4. `    public void addCorsMappings(CorsRegistry registry) {`
5. `        registry.addMapping("/**")`
6. `        .allowedOrigins("http://localhost:8081")`
7. `        .allowedMethods("*")`
8. `        .allowedHeaders("*");`
9. `    }`
10. `}`

了解了整个CORS的工作过程之后，我们通过Ajax发送跨域请求，虽然用户体验提高了，但是也有潜在的威胁存在，常见的就是CSRF（Cross-site request forgery）跨站请求伪造。跨站请求伪造也被称为one-click attack 或者 session riding，通常缩写为CSRF或者XSRF，是一种挟制用户在当前已登录的Web应用程序上执行非本意的操作的攻击方法，举个例子：

基于此，浏览器在实际操作中，会对请求进行分类，分为简单请求，预先请求，带凭证的请求等，预先请求会首先发送一个options探测请求，和浏览器进行协商是否接受请求。默认情况下跨域请求是不需要凭证的，但是服务端可以配置要求客户端提供凭证，这样就可以有效避免csrf攻击。



## SQL 查找"存在"，别再用 count 了，很耗费时间的！

SELECT 1 FROM table  WHERE a = 1 AND b = 2 LIMIT 1

Integer exist = xxDao.existXxxxByXxx(params);
if ( exist != NULL ) {
 *//当存在时，执行这里的代码*
} else {
 *//当不存在时，执行这里的代码*
}



```
public class FutureTest {

  public static void main(String[] args) throws ExecutionException, InterruptedException {
    ExecutorService executorService = Executors.newSingleThreadExecutor();
    Future<String> future = executorService.submit(new Callable<String>() {
      @Override
      public String call() throws Exception {
        return "测试Future获取异步结果";
      }
    });
    System.out.println(future.get());
    executorService.shutdown();
  }
}
```

```
public class FutureTaskTest {
    public static void main(String[] args)throws ExecutionException, InterruptedException{
        FutureTask<String> futureTask = new FutureTask<>(new Callable<String>() {
            @Override
            public String call() throws Exception {
                return "测试FutureTask获取异步结果";
            }
        });
        new Thread(futureTask).start();
        System.out.println(futureTask.get());
    }
}
```

cancel(boolean)

取消任务的执行，接收一个boolean类型的参数，成功取消任务，则返回true，否则返回false。

当任务已经完成，已经结束或者因其他原因不能取消时，方法会返回false，表示任务取消失败。

当任务未启动调用了此方法，并且结果返回true（取消成功），则当前任务不再运行。

如果任务已经启动，会根据当前传递的boolean类型的参数来决定是否中断当前运行的线程来取消当前运行的任务。



一旦将SimpleDateFormat类定义成全局的静态变量，那么SimpleDateFormat类在多个线程间是共享的，这就导致ParsePosition类在多个线程间共享。****在高并发场景下，一个线程对ParsePosition类中的索引进行修改，势必会影响到其他线程对ParsePosition类中索引的读操作。这就造成了线程的安全问题。**

至于在高并发场景下使用局部变量为何能解决线程的安全问题，会在【JVM专题】的JVM内存模式相关内容中深入剖析，这里不做过多的介绍了。

当然，这种方式在高并发下会创建大量的SimpleDateFormat类对象，影响程序的性能，所以，这种方式在实际生产环境不太被推荐。

综上所示：在解决SimpleDateFormat类的线程安全问题的几种方案中，局部变量法由于线程每次执行格式化时间时，都会创建SimpleDateFormat类的对象，这会导致创建大量的SimpleDateFormat对象，浪费运行空间和消耗服务器的性能，因为JVM创建和销毁对象是要耗费性能的。所以，**不推荐在高并发要求的生产环境使用。**

synchronized锁方式和Lock锁方式在处理问题的本质上是一致的，通过加锁的方式，使同一时刻只能有一个线程执行格式化日期和时间的操作。这种方式虽然减少了SimpleDateFormat对象的创建，但是由于同步锁的存在，导致性能下降，所以，**不推荐在高并发要求的生产环境使用。**

ThreadLocal通过保存各个线程的SimpleDateFormat类对象的副本，使每个线程在运行时，各自使用自身绑定的SimpleDateFormat对象，互不干扰，执行性能比较高，**推荐在高并发的生产环境使用。**

DateTimeFormatter是Java 8中提供的处理日期和时间的类，DateTimeFormatter类本身就是线程安全的，经压测，DateTimeFormatter类处理日期和时间的性能效果还不错（后文单独写一篇关于高并发下性能压测的文章）。所以，**推荐在高并发场景下的生产环境使用。**

joda-time是第三方处理日期和时间的类库，线程安全，性能经过高并发的考验，**推荐在高并发场景下的生产环境使用。**

- Executors.newCachedThreadPool：创建一个可缓存的线程池，如果线程池的大小超过了需要，可以灵活回收空闲线程，如果没有可回收线程，则新建线程
- Executors.newFixedThreadPool：创建一个定长的线程池，可以控制线程的最大并发数，超出的线程会在队列中等待
- Executors.newScheduledThreadPool：创建一个定长的线程池，支持定时、周期性的任务执行
- Executors.newSingleThreadExecutor: 创建一个单线程化的线程池，使用一个唯一的工作线程执行任务，保证所有任务按照指定顺序（先入先出或者优先级）执行
- Executors.newSingleThreadScheduledExecutor:创建一个单线程化的线程池，支持定时、周期性的任务执行
- Executors.newWorkStealingPool：创建一个具有并行级别的work-stealing线程池

#### 4.合理配置线程的一些建议

（1）CPU密集型任务，就需要尽量压榨CPU，参考值可以设置为NCPU+1(CPU的数量加1)。
（2）IO密集型任务，参考值可以设置为2*NCPU（CPU数量乘以2）

### 1.什么是队列

队列是一种特殊的线性表，特殊之处在于它只允许在表的前端（front）进行删除操作，而在表的后端（rear）进行插入操作，和栈一样，队列是一种操作受限制的线性表。进行插入操作的端称为队尾，进行删除操作的端称为队头。



### 2.什么是堵塞队列?

当队列为空时，消费者挂起，队列已满时，生产者挂起，这就是生产-消费者模型，堵塞其实就是将线程挂起。
因为生产者的生产速度和消费者的消费速度之间的不匹配，就可以通过堵塞队列让速度快的暂时堵塞,
如生产者每秒生产两个数据，而消费者每秒消费一个数据，当队列已满时，生产者就会堵塞（挂起），等待消费者消费后，再进行唤醒。

堵塞队列会通过挂起的方式来实现生产者和消费者之间的平衡，这是和普通队列最大的区别。



### 3.如何实现堵塞队列？

jdk其实已经帮我们提供了实现方案，java5增加了concurrent包，concurrent包中的BlockingQueue就是堵塞队列，我们不需要关心BlockingQueue如何实现堵塞，一切都帮我们封装好了，只需要做一个没有感情的API调用者就行



### 4.BlockingQueue如何使用？

BlockingQueue本身只是一个接口，规定了堵塞队列的方法，主要依靠几个实现类实现。

#### 4.1BlockingQueue主要方法

![img](https://mmbiz.qpic.cn/mmbiz_png/8KKrHK5ic6XA6FicoqgVtUOicRcTDaYLp6mU7VbubjXOicaoEke7LB8ofLwG6F5aut8Z2WOjc8m8PpYqWEbV7qWA8g/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**1.插入数据**

（1）offer(E e)：如果队列没满，返回true，如果队列已满，返回false（不堵塞）
（2）offer(E e, long timeout, TimeUnit unit)：可以设置等待时间，如果队列已满，则进行等待。超过等待时间，则返回false
（3）put(E e)：无返回值，一直等待，直至队列空出位置

**2.获取数据**

（1）poll()：如果有数据，出队，如果没有数据，返回null
（2）poll(long timeout, TimeUnit unit)：可以设置等待时间，如果没有数据，则等待，超过等待时间，则返回null
（3）take()：如果有数据，出队。如果没有数据，一直等待（堵塞）

#### 4.2BlockingQueue主要实现类

1.ArrayBlockingQueue：ArrayBlockingQueue是基于数组实现的，通过初始化时设置数组长度，是一个有界队列，而且ArrayBlockingQueue和LinkedBlockingQueue不同的是，ArrayBlockingQueue只有一个锁对象，而LinkedBlockingQueue是两个锁对象，一个锁对象会造成要么是生产者获得锁，要么是消费者获得锁，两者竞争锁，无法并行。

2.LinkedBlockingQueue：LinkedBlockingQueue是基于链表实现的，和ArrayBlockingQueue不同的是，大小可以初始化设置，如果不设置，默认设置大小为Integer.MAX_VALUE，LinkedBlockingQueue有两个锁对象，可以并行处理。

3.DelayQueue：DelayQueue是基于优先级的一个无界队列，队列元素必须实现Delayed接口，支持延迟获取，元素按照时间排序，只有元素到期后，消费者才能从队列中取出。

4.PriorityBlockingQueue：PriorityBlockingQueue是基于优先级的一个无界队列，底层是基于数组存储元素的，元素按照优选级顺序存储，优先级是通过Comparable的compareTo方法来实现的（自然排序），和其他堵塞队列不同的是，其只会堵塞消费者，不会堵塞生产者，数组会不断扩容，这就是一个彩蛋，使用时要谨慎。

5.SynchronousQueue：SynchronousQueue是一个特殊的队列，其内部是没有容器的，所以生产者生产一个数据，就堵塞了，必须等消费者消费后，生产者才能再次生产，称其为队列有点不合适，现实生活中，多个人才能称为队，一个人称为队有些说不过去。