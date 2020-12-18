## AOP注解实现方法重试

## 1 注解
#### 1.1 @Retryable注解
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Retryable {
    /**
     * 抛出什么异常需要重试
     */
    Class<? extends Throwable>[] value() default {};

    /**
     * 最大重试次数
     */
    int maxAttempts () default 3;

    Backoff backoff() default @Backoff();
}
```
#### 1.2 Backoff回避
```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Backoff {
    long value() default 1000;

    long maxDelay() default 0;

    double multiplier() default 0;
}
```

#### 1.3 回避策略
```java

public class BackoffPolicy {
    // 延迟时间，毫秒
    private long delay;
    // 最大延迟，毫秒
    private long maxDelay;
    // 倍增器
    private final double multiplier;

    public BackoffPolicy(long delay, long maxDelay, double multiplier) {
        this.delay = delay;
        this.maxDelay = maxDelay;
        this.multiplier = multiplier;
    }

    public synchronized Long getSleepAndIncrement() {
        long sleep = this.delay;
        if (sleep > this.maxDelay) {
            sleep = maxDelay;
        } else {
            this.delay = (long) (delay * multiplier);
        }
        return sleep;
    }
}
```

#### 1.4 Recover兜底
```java
@Target({ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Recover {
}
```

## 2 RetryAop
```java

@Component
@Aspect
@Slf4j
public class RetryAop {
    @Pointcut("@annotation(org.dmqk.cloud.service.pay.util.retry.Retryable)")
    public void pointcut(){}

    @Around(value="pointcut()")
    public Object around(ProceedingJoinPoint joinPoint) {

        Method recoverMethod = recover(joinPoint);

        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        Retryable retryable = method.getAnnotation(Retryable.class);

        // 最大重试次数
        int maxAttempts = retryable.maxAttempts();
        // 重试异常
        Class<? extends Throwable>[] exceptions = retryable.value();

        //获取方法参数值数组
        Object[] args = joinPoint.getArgs();
        Object result = null;

        Backoff backoff = retryable.backoff();
        BackoffPolicy backoffPolicy = new BackoffPolicy(backoff.value(), backoff.maxDelay(), backoff.multiplier());

        int count = 0;
        boolean retry = false;
        do {
            log.info("重试次数：{}", count);
            try {
                result = joinPoint.proceed(args);
                count++;
                retry = false;
            } catch (Throwable e) {
                log.error("方法异常", e);

                for (Class<? extends Throwable> value : exceptions) {
                    if (value.isAssignableFrom(e.getClass())) {
                        System.out.println("匹配");
                        count++;
                        retry = true;

                        // 退避策略
                        try {
                            TimeUnit.MILLISECONDS.sleep(backoffPolicy.getSleepAndIncrement());
                        } catch (InterruptedException interruptedException) {
                            interruptedException.printStackTrace();
                        }
                        break;
                    }
                }
            }
        } while (retry && count <= maxAttempts);

        if (count >= maxAttempts && recoverMethod != null) {
            try {
                recoverMethod.invoke(joinPoint.getTarget());
            } catch (IllegalAccessException | InvocationTargetException e) {
                e.printStackTrace();
            }
        }
        return result;
    }

    private Method recover(ProceedingJoinPoint joinPoint) {
        Method result = null;
        Method[] methods = joinPoint.getTarget().getClass().getMethods();
        for (Method method : methods) {
            Recover annotation = AnnotationUtils.findAnnotation(method, Recover.class);
            if (annotation != null) {
                result = method;
                break;
            }
        }
        return result;
    }
}
```

## 3 测试
```java
    @GetMapping("test2")
    @Retryable(value = Exception.class, maxAttempts = 3, backoff = @Backoff(value = 1000, maxDelay = 100000, multiplier = 1.2))
    public ResultJson test2() {
        int i = 1/0;
        return ResultJson.SUCCESS();
    }

    @Recover
    public void recover() {
        log.info("执行兜底方法。。。");
    }
```


## 4 参考：
[Spring-Retry重试实现原理](https://mp.weixin.qq.com/s?__biz=MzUyNDkzNzczNQ==&mid=2247499588&idx=3&sn=0db7c1436d680e494b9bf5b0a65460db&chksm=fa27002ccd50893a9e47c76cc293f280a7fd68fcb2b5e7ad5b553fe1b7736a565509f14aae70&scene=126&sessionid=1608088039&key=e335634ccac5ee250badf1149b06490b472fa5e20bc1ac58ca30bc56e0fefd5d165f5f50f7d97b2faa45a7b5b77e1961506cec0cdd97b4fb09366083eb67cddd478d43c0e86eda4a37225457aad8951523a14428190f1d0297bd8644f7007ec9e7b82e899b1e1a77d7452d045949624b9123dff44cc813d7d60de1305302066d&ascene=1&uin=NjA1NTM5NjU%3D&devicetype=Windows+10+x64&version=6300002f&lang=zh_CN&exportkey=AV7jsBft%2FupVzF2dvp96nwY%3D&pass_ticket=r6JzYQZmb87Dg9S%2BrxHGhn1GyIsAQcOv8lJVxjDHrwspKXFtzQKKQvExCKoixRA5&wx_header=0)