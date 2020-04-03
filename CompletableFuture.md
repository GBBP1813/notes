#  五分钟带你了解CompletableFuture

## 前言

面试官：JAVA8特性你知道哪些

我：Lambda表达式，函数式接口，Stream流

正当我准备向面试官大展身手时，面试官突然了一句：还有别的嘛，CompletableFuture了解嘛

我：顿时脑袋一懵，啥玩意呀，只听说过Future呀，内心一万句mmp。

别慌，小编今天带带大家了解下CompletableFuture。



## 正文

​        在正式了解CompleteFuture之前，小编有必要先带大家了解下Future。Future接口是在JKD5引入的，他设计的初衷是对将来某个时候会发生的结果进行建模，它建模了一种异步计算，返回一个执行运算结果的引用，当运算结束后，调用方可以拿到这个引用。这样通过Future就可以把那些触发耗时操作的线程解放出来，让它去处理别的有价值的工作，不需要在等待。小编之前在工作中就使用过，下面是一个例子。

```java
List<CommissionDetailVo> bigList = new ArrayList<>();
List<CommissionDetailVo> onePage = new ArrayList<>();
List<Future> futureList = new ArrayList<>();
try {
    for ( int i = 0; i < THREADS; i++) {

        final int k = i;
        Future list = threadPool.submit(() -> {
            return esSearch(k,settlementDetailSearchVo);
        });
        futureList.add(list);
    }
}catch (Exception e) {
    e.printStackTrace();
    logger.error("Es查询失败", e.getMessage());
}
for ( Future list: futureList) {
        onePage =(List<CommissionDetailVo>) list.get();
        bigList.addAll(onePage);
}
```

那么就有同学会问，既然有了Future,为啥还要学CompletableFuture ? 不急不急，请听小编我娓娓道来。

在使用Future的过程中，大家会发现Future有两种方式返回结果的方式

1. 阻塞方式get()
2. 轮询方法isDone() 

阻塞的方式获取结果在某种程度上与异步编程的初衷相违背，而且轮询的方式又会浪费CPU。因此，导致Future在获取结果上总显得那么不尽如人意。

​        如果我们期望异步任务结束后 能自动获取结果，这时Future就不能满足我们的需求了。于是乎，JDK8就引入了今天我们要说的CompletableFuture，它实现了Future接口，弥补了Future模式的缺点，还提供了非常强大的Future扩展功能，帮助我们简化异步编程的复杂性，提供了函数编程的能力，可以通过回调的方式处理计算结果，并且提供了转换，组合，串行化CompletableFuture的功能。

1.异步任务结束后，可以自动回调某个对象的方法；

2.异步任务出错后，可以自动回调某个对象的方法；

3.主线程设置好回调后，可以不再关心异步任务的执行；

可以看下面的例子：

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync( () -> feeItem.getAmount());
future.thenAccept(amount -> System.out.println(amount));
future.exceptionally(throwable -> {System.out.println("发生异常"+throwable.toString());return 111;});
```

不论是异步任务正常执行或是出现了异常，我们都可以设置好回调函数，异步任务结束后都会自行调用。

### 基本API

**runAsync与supplyAsync** 

创建一个异步操作

```java
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

如果调用时没有指定线程池，则会使用ForkJoinPool.commonPool()作为默认的线程池执行异步的代码。两个方法的区别:runAsync 不支持返回值，supplyAsync支持返回值

实例：

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> feeItem.getAmount());
```



**计算结果完成时的回调方法**

当异步计算完成时，或者出现异常的时候可以执行指定的Action.

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

那whenComplete 与 whenCompleteAsync有啥区别呢

whenComplete：执行当前任务的线程会继续执行 whenComplete 中传入的任务。
whenCompleteAsync： 会把whenCompleteAsync 中的提交的任务给其他线程池来执行。

实例：

```java
CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> feeItem.getAmount());
future3.whenComplete( (amount, throwable)-> System.out.println(amount));
```



**handle**

handle方法虽然返回的也是CompleatableFuture对象，但是对象的值和原来的CompleatableFuture计算的值不同。当原先的CompletableFuture的值计算完成或者抛出异常的时候，会触发这个CompletableFuture对象的计算，结果由BiFunction参数计算而得。因此这组方法兼有whenComplete和转换的两个功能。

```java
public <U> CompletableFuture<U>  handle(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U>  handleAsync(BiFunction<? super T,Throwable,? extends U> fn)
public <U> CompletableFuture<U>  handleAsync(BiFunction<? super T,Throwable,? extends U> fn, Executor executor)
```

实例：

```java
CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> feeItem.getAmount());
CompletableFuture<Integer> future4 = future3.handleAsync((amount,throwable) -> amount*3);
System.out.println(future4.get());
```

**转换**

thenApply会把原来CompletableFuture计算的结果传给函数fn，将fn的结果作为新的CompletableFuture的计算结果，即CompletableFuture\<T> 转化成了CompletableFuture\<U>  

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
```

实例：

```java
CompletableFuture<Integer> a =CompletableFuture.supplyAsync(() -> 1).thenApply(i -> i+1).thenApply(i -> i*i).whenComplete((r,throwable) -> System.out.println(r));
System.out.println(a.get());
```



**纯消费执行Action**

thenAccpet 只对计算结果执行Action,不会返回新的计算的值，返回类型为Void

```java
public CompletableFuture<Void>  thenAccept(Consumer<? super T> action)
public CompletableFuture<Void>  thenAcceptAsync(Consumer<? super T> action)
public CompletableFuture<Void>  thenAcceptAsync(Consumer<? super T> action, Executor executor)
```

**thenCombine**

可以将两个CompletableFuture对象结果整合起来

```java
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

**thenCompose**

对两个异步操作进行串联，第一个操作完成时，对第一个**CompletableFuture**对象调用thenCompose，并向其传递一个函数。当第一个**CompletableFuture**执行完毕后，它的结果将作为该函数的参数，这个函数的返回值是以第一个**CompletableFuture**的返回做输入计算出第二个**CompletableFuture**对象。

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, ? extends CompletionStage<U>> fn)
```

实例：

```java
List<CompletableFuture<Integer>> list = feeItemList.stream()
        .map(feeItems ->
        CompletableFuture.supplyAsync(() -> feeItems.getAmount()))
        .map(future -> future.thenApply(i -> i * 2))
        .map(future -> future.thenCompose(amount -> CompletableFuture.supplyAsync( () -> amount*3 )))
        .collect(Collectors.toList());     System.out.println(list.stream().map(CompletableFuture::join).collect(Collectors.toList()));

```

这里的Compfuture的join方法是为了获取CompletableFuture计算后的值



**辅助方法allOf和anyOf**

allOf: 当所有的CompletableFuture都执行完才计算

在所有CompletableFuturer执行完之前，主线程会阻塞

anyOf: 当任意一个ComplteableFuture执行完就执行计算

```java
public static CompletableFuture<Void>       allOf(CompletableFuture<?>... cfs)
public static CompletableFuture<Object>     anyOf(CompletableFuture<?>... cfs)
```

实例：

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> feeItem.getAmount());
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> feeItem1.getAmount());
CompletableFuture<Object> f1 = CompletableFuture.anyOf(future2, future1);
CompletableFuture<Void> f2= CompletableFuture.allOf(future2, future1);
System.out.println(f1.get()+" "+f2.get());
```



CompletableFuture的方法还有很多,大概有60多个，这里就不一一赘述了。



### 并行流 or CompletableFuture

对集合进行并行运算可以有两种方式，一个是使用并行流，即parallerStream(等待结果会阻塞), 一个是 使用CompletableFuture.CompletableFuture提供了更大的灵活性，可以自己传入线程池，也就可以调整线程池大小，这能帮助我们确保整体的计算不会因为线程在等待I/O而发生阻塞。所以整体的建议如下：

1.如果你进行的的计算密集型操作，并且没有I/O操作，那么推荐使用并行流，因为实现简单，同事效率也可能是最高的。(如果所有的线程都是计算密集型的，那就没有必要创建比处理器合数更多的线程)。

2.如果你并行的工作单元还涉及等到I/O或者网络操作,那么使用CompletableFuture的灵活性更好。



### 使用CompletableFuture优化你的代码性能

我们可以利用CompletableFuture对系统性能以及吞吐量上进行优化。我相信很多的同学系统一般会这样设计：

![优化前](https://github.com/GBBP1813/notes/blob/master/picture/优化前.jpg)

这样会有什么问题呢

1.系统的吞吐量不高

2.接口调用次数频繁，时间长

我们就可以进行如下改进：

![性能优化](https://github.com/GBBP1813/notes/blob/master/picture/性能优化.jpg)

可把大量的请求先放进一个阻塞丢列，而不是来一个请求就去调用查询接口，而后用定时任务每隔一定时间去队列

获取请求，批量去查询。当批量查询的结果返回后，再根据批次信息，将结果返回给对应的线程。

下面是小编写的一个例子：

```java
public class AsyncServie {

    @Autowired
    private OrderService orderService;

    public AtomicInteger atomicInteger = new AtomicInteger(0);

    class Request {
        String orderCode;
        String serialNo;
        CompletableFuture<Map<String,Object>> future;
    }

    LinkedBlockingDeque<Request> blockingDeque = new LinkedBlockingDeque<>();

    public Map<String, Object> queryOrderInfo(String orderCode) throws InterruptedException, ExecutionException {
        Request request = new Request();
        request.orderCode = orderCode;
        request.serialNo = UUID.randomUUID().toString();
        CompletableFuture<Map<String, Object>> future = new CompletableFuture<>();
        request.future = future;
        blockingDeque.add(request);

        atomicInteger.getAndIncrement();
        System.out.println(atomicInteger);
        //监听有没有返回值 一直阻塞
         return future.get();
    }

    @PostConstruct
    public void init() {
        //每隔10ms去队列获取请求，批量发起
        ScheduledExecutorService executorService = Executors.newScheduledThreadPool(2);
        executorService.scheduleAtFixedRate( () -> {
            int size = blockingDeque.size();
            if (0 == size) {
                return;
            }
            List<Map<String, String>> params = Lists.newArrayList();
            List<Request> requestList = Lists.newArrayList();
            for (int i=0; i< size; i++){
                Request request = blockingDeque.poll();
                Map<String ,String> map = Maps.newHashMap();
                map.put("orderCode", request.orderCode);
                map.put("serialNo", request.serialNo);
                params.add(map);
                requestList.add(request);
            }
            System.out.println("批量处理的size"+size);
            System.out.println(Thread.currentThread().getName());
            List<Map<String, Object>>  orderInfo =  orderService.getOrderInfo(params);

            //匹配对应的serialNo
             Optional.ofNullable(requestList).ifPresent(requests -> requests.forEach(request -> {
                String serialNo = request.serialNo;
                Optional.ofNullable(orderInfo).ifPresent(orderInfos -> orderInfos.forEach(response -> {
                    String serial = response.get("serialNo").toString();
                    if (Objects.equals(serialNo, serial)) {
                        request.future.complete(response);
                    }
                }));
            }));


        }, 0, 10, TimeUnit.MILLISECONDS);

    }
}
```



好了今天就介绍到这了。
