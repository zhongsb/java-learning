这个包主要介绍并实践《JAVA并发编程实战》中的多线程开发中的例子。


## 获取线程执行结果的几种方式

### 1、Callable 线程

    public class FetchAdTask implements Callable<Ad> {
    
        @Override
        public Ad call() throws Exception {
            System.out.println("fetch task");
            sleep(1000L);
            return null;
        }
    }

### 2、使用Future，包括 FutureTask、CompletableFuture

    CompletableFuture.get();
    
Future 的优点：可以对任务设置时限，如果超时了，可以取消，然后返回一个默认结果，防止
某一个任务出现问题，导致系统出现问题。

    f.get(timeLeft, TimeUnit.NANOSECONDS);
    
或者通过 invokeAll() 返回限定时间范围内的所有任务的结果。     

    executor.invokeAll(tasks, time, unit);   

CompletableFuture, 使用 supplyAsync 方法提交线程，使用 get 方法获取结果。

    CompletableFuture<Integer> task3 = CompletableFuture.supplyAsync(() -> {
            System.out.println("任务3, 线程名字" + Thread.currentThread().getName());
            try {
                sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            return 3;
        });

        CompletableFuture.allOf(task1, task2, task3, task4);
        System.out.println("end: " + new Date());
        task1.get();

### 3、使用 CompletionService，

    CompletionService.take();
    
例子

    private static final long TIME_BUDGET = 100L;
    private static final Ad DEFAULT_AD = new Ad();

    private final ExecutorService executor = new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors() + 1,
            Runtime.getRuntime().availableProcessors() + 1, 0L,
            TimeUnit.MILLISECONDS, new LinkedBlockingQueue(1000));
    

    public static void main(String[] args) {
        try {
            Test616LimitedTimeTask task = new Test616LimitedTimeTask();
            task.test();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    /**
     * 测试超时取消
     *
     * @throws InterruptedException
     */
    void test() throws InterruptedException {
        Ad ad = null;
        long endNanos = System.nanoTime() + TIME_BUDGET;
        Future<Ad> f = executor.submit(new FetchAdTask());
        try {
            long timeLeft = endNanos - System.nanoTime();
            // 增加参数 超时时间和超时时间的单位
            ad = f.get(timeLeft, TimeUnit.NANOSECONDS);
        } catch (ExecutionException e) {
            ad = DEFAULT_AD;
        } catch (TimeoutException e) {
            // 超时，取消任务
            ad = DEFAULT_AD;
            System.out.println("超时取消");
            f.cancel(true);
        }
    }    
    
优点：多个 CompletionService 可以共享一个 Executor，因此可以创建一个对于特定计算私有，
又能共享一个公共 Executor 的 ExecutorCompletionService。

## 取消任务

### Future 正确姿势

通过 Future.cancel 来取消任务。

当Future.get抛出 InterruptedException或者TimeoutException时，如果你知道不再需要结果，
那么就可以调用Future.cancel来取消任务

### cancel标志取消线程

```
@Override
public void run() {
    System.out.println("执行线程");
    BigInteger p = BigInteger.ONE;
    while (!cancelled) {
        p = p.nextProbablePrime();
        synchronized (this) {
            primes.add(p);
        }
    }
}
```

问题：如果阻塞了，导致不走cancel标志，那么就会导致停止不了。

### 通过interrupt中断线程

```
@Override
public void run() {
    try {
        BigInteger p = BigInteger.ONE;
        while (!Thread.currentThread().isInterrupted()) {
            queue.put(p = p.nextProbablePrime());
        }
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```

问题：存在join的不足，不知道执行控制是因为线程正常退出而返回还是因为join超时而返回

### 毒丸取消线程

只有在生产者和消费者的数量都已知的情况下，才可以使用"毒丸"对象。

**只有在无界队列中，"毒丸"对象次啊能可靠地工作**。

### ExecutorService 取消

ExecutorService 线程池取消，可以调用线程池的 shutdown() 方法或者 shutdownNow()

建议使用 shutdown() 方法。

```
exec.shutdown();
// TODO 问：为什么停止线程，需要调用以下语句
// 这个方法就是调用shutdown() 之后等待任务执行完毕的方法，可以查看源码的注释
// Blocks until all tasks have completed execution after a shutdown
// request, or the timeout occurs, or the current thread is
// interrupted, whichever happens first.
exec.awaitTermination(TIMEOUT, UNIT);
```

## 守护线程

在JVM启东市创建的所有线程中，除了主线程以外，其他的线程都是守护线程。

1、我们应该尽量少地使用守护线程。



源码：https://github.com/zhongsb/Java-learning.git