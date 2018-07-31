# Hystrix源码剖析

hystrix里面有很多有意思的地方，比如缓存、回退方法、滑动窗口...

本人目前理解不够透彻，暂打印回退方法栈来展示整个回退方法的流程，其他章节后续会补充进来。

> 代码基于注解方式，需引入包com.netflix.hystrix.hystrix-javanica

## 入口

注解依赖aspectj，com.netflix.hystrix.contrib.javanica.aop.aspectj.HystrixCommandAspect切面处理@HystrixCommand的方法注解，方法案例：

```text
    /**
     * 故意抛出dubbo的超时异常
     * @param messageReq
     * @return
     */
    // HystrixCommandAspect
    // https://github.com/Netflix/Hystrix/wiki/Configuration
    @HystrixCommand(
            // HystrixCommandGroupKey
            groupKey = "MessageProxy",
            // HystrixCommandKey
            commandKey = "sendTimeout",
            commandProperties = {
                    //HystrixCommandProperties
                    @HystrixProperty(name = "execution.timeout.enabled", value = "false"),    //执行超时时间|default:1000
//                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "5000"),    //执行超时时间|default:1000|RPC本身支持超时选项则才优先RPC方式
                    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "20"), //触发断路最低请求数|default:20
                    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000"),    //断路器恢复时间|default:5000
                    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "50"),   //|触发短路错误率,单位%|default:50
                    @HystrixProperty(name = "fallback.isolation.semaphore.maxConcurrentRequests", value = "100"),   // 设置调用线程产生的HystrixCommand.getFallback()方法的允许最大请求数目。|default: 10
            },
            threadPoolProperties = {
                    //HystrixThreadPoolProperties
                    @HystrixProperty(name = "coreSize", value = "15"),  //线程池核心数|default:10 超过队列fallback
                    @HystrixProperty(name = "maxQueueSize", value = "-1"),  //队列长度|default:-1(SynchronousQueue)
                    @HystrixProperty(name = "keepAliveTimeMinutes", value = "1"),   // 设置存活时间，单位分钟| 默认值：1|如果coreSize小于maximumSize，那么该属性控制一个线程从实用完成到被释放的时间。
//                    @HystrixProperty(name = "queueSizeRejectionThreshold", value = "15"),   //队满拒绝服务阈值|default:5|此值生效优先于队满, maxQueueSize设置为-1，该属性不可用。
                    // Metrics统计属性
                    //@HystrixProperty(name = "metrics.rollingStats.numBuckets", value = "12"),   // 设置滑动统计的桶数量。默认10。metrics.rollingStats.timeInMilliseconds必须能被这个值整除。
                    //@HystrixProperty(name = "metrics.rollingStats.timeInMilliseconds", value = "1440")  //窗口维持时间|default:10000
            },
            // HystrixThreadPoolKey|threadPoolKey =
            fallbackMethod = "fallback")
    public BaseResp sendTimeout(MessageReq messageReq){
        return iMessageFacade.sendTimeout(messageReq, 5000L);
    }
```

## 上下文数据装载器

通过阅读HystrixCommandAspect执行逻辑，com.netflix.hystrix.contrib.javanica.command.MetaHolder装载了切面的上下文元数据

```text
/**
 * Simple immutable holder to keep all necessary information about current method to build Hystrix command.
 */
// todo: replace fallback related flags with FallbackMethod class
@Immutable
public final class MetaHolder {
```

## Fallback

在装载上下文元数据，创建CommandAction组装执行器

```text
/**
 * Simple action to encapsulate some logic to process it in a Hystrix command.
 */
public interface CommandAction {
```

创建回退执行器com.netflix.hystrix.contrib.javanica.command::createFallbackAction

```text
 private CommandAction createFallbackAction(MetaHolder metaHolder) {

        FallbackMethod fallbackMethod = MethodProvider.getInstance().getFallbackMethod(metaHolder.getObj().getClass(),
                metaHolder.getMethod(), metaHolder.isExtendedFallback());
        fallbackMethod.validateReturnType(metaHolder.getMethod());
        CommandAction fallbackAction = null;
        if (fallbackMethod.isPresent()) {

            Method fMethod = fallbackMethod.getMethod();
            Object[] args = fallbackMethod.isDefault() ? new Object[0] : metaHolder.getArgs();
            if (fallbackMethod.isCommand()) {
                fMethod.setAccessible(true);
                HystrixCommand hystrixCommand = fMethod.getAnnotation(HystrixCommand.class);
                MetaHolder fmMetaHolder = MetaHolder.builder()
                        .obj(metaHolder.getObj())
                        .method(fMethod)
                        .ajcMethod(getAjcMethod(metaHolder.getObj(), fMethod))
                        .args(args)
                        .fallback(true)
                        .defaultFallback(fallbackMethod.isDefault())
                        .defaultCollapserKey(metaHolder.getDefaultCollapserKey())
                        .fallbackMethod(fMethod)
                        .extendedFallback(fallbackMethod.isExtended())
                        .fallbackExecutionType(fallbackMethod.getExecutionType())
                        .extendedParentFallback(metaHolder.isExtendedFallback())
                        .observable(ExecutionType.OBSERVABLE == fallbackMethod.getExecutionType())
                        .defaultCommandKey(fMethod.getName())
                        .defaultGroupKey(metaHolder.getDefaultGroupKey())
                        .defaultThreadPoolKey(metaHolder.getDefaultThreadPoolKey())
                        .defaultProperties(metaHolder.getDefaultProperties().orNull())
                        .hystrixCollapser(metaHolder.getHystrixCollapser())
                        .observableExecutionMode(hystrixCommand.observableExecutionMode())
                        .hystrixCommand(hystrixCommand).build();
                fallbackAction = new LazyCommandExecutionAction(fmMetaHolder);
            } else {
                MetaHolder fmMetaHolder = MetaHolder.builder()
                        .obj(metaHolder.getObj())
                        .defaultFallback(fallbackMethod.isDefault())
                        .method(fMethod)
                        .fallbackExecutionType(ExecutionType.SYNCHRONOUS)
                        .extendedFallback(fallbackMethod.isExtended())
                        .extendedParentFallback(metaHolder.isExtendedFallback())
                        .ajcMethod(null) // if fallback method isn't annotated with command annotation then we don't need to get ajc method for this
                        .args(args).build();

                fallbackAction = new MethodExecutionAction(fmMetaHolder.getObj(), fMethod, fmMetaHolder.getArgs(), fmMetaHolder);
            }

        }
        return fallbackAction;
    }
```

最后组装成命令执行器集com.netflix.hystrix.contrib.javanica.command.CommandActions

```text
/**
 * Wrapper for command actions combines different actions together.
 *
 * @author dmgcodevil
 */
public class CommandActions {
    private final CommandAction commandAction;
    private final CommandAction fallbackAction;
```

执行Command逻辑使用rxjava，串联事件链\(缓存、熔断、\)，这里不做详述

关键逻辑定义了观察者异常，错误传播执行回退方法com.netflix.hystrix.AbstractCommand::executeCommandAndObserve.handleFallback

```text
/**
 * This decorates "Hystrix" functionality around the run() Observable.
 *
 * @return R
 */
private Observable<R> executeCommandAndObserve(final AbstractCommand<R> _cmd) {
        ...
        final Func1<Throwable, Observable<R>> handleFallback = new Func1<Throwable, Observable<R>>() {
            @Override
            public Observable<R> call(Throwable t) {
                circuitBreaker.markNonSuccess();
                Exception e = getExceptionFromThrowable(t);
                executionResult = executionResult.setExecutionException(e);
                if (e instanceof RejectedExecutionException) {
                    return handleThreadPoolRejectionViaFallback(e);
                } else if (t instanceof HystrixTimeoutException) {
                    return handleTimeoutViaFallback();
                } else if (t instanceof HystrixBadRequestException) {
                    return handleBadRequestByEmittingError(e);
                } else {
                    /*
                     * Treat HystrixBadRequestException from ExecutionHook like a plain HystrixBadRequestException.
                     */
                    if (e instanceof HystrixBadRequestException) {
                        eventNotifier.markEvent(HystrixEventType.BAD_REQUEST, commandKey);
                        return Observable.error(e);
                    }
                    // 程序RuntimeException, by zzqfsy 2018-07-31
                    return handleFailureViaFallback(e);
                }
            }
        };
        ...
        return execution.doOnNext(markEmits)
        .doOnCompleted(markOnCompleted)
        .onErrorResumeNext(handleFallback)
        .doOnEach(setRequestContext);
}
```

通过回退来处理故障com.netflix.hystrix.AbstractCommand::handleFailureViaFallback

```text
private Observable<R> handleFailureViaFallback(Exception underlying) {
    /**
     * All other error handling
     */
    logger.debug("Error executing HystrixCommand.run(). Proceeding to fallback logic ...", underlying);

    // report failure
    eventNotifier.markEvent(HystrixEventType.FAILURE, commandKey);

    // record the exception
    executionResult = executionResult.setException(underlying);
    return getFallbackOrThrowException(this, HystrixEventType.FAILURE, FailureType.COMMAND_EXCEPTION, "failed", underlying);
}
```

获取回退处理方法或直接抛出异常com.netflix.hystrix.AbstractCommand::getFallbackOrThrowException

```text
/**
 * Execute <code>getFallback()</code> within protection of a semaphore that limits number of concurrent executions.
 * <p>
 * Fallback implementations shouldn't perform anything that can be blocking, but we protect against it anyways in case someone doesn't abide by the contract.
 * <p>
 * If something in the <code>getFallback()</code> implementation is latent (such as a network call) then the semaphore will cause us to start rejecting requests rather than allowing potentially
 * all threads to pile up and block.
 *
 * @return K
 * @throws UnsupportedOperationException
 *             if getFallback() not implemented
 * @throws HystrixRuntimeException
 *             if getFallback() fails (throws an Exception) or is rejected by the semaphore
 */
private Observable<R> getFallbackOrThrowException(final AbstractCommand<R> _cmd, final HystrixEventType eventType, final FailureType failureType, final String message, final Exception originalException) {
            ...
            Observable<R> fallbackExecutionChain;

            // acquire a permit
            if (fallbackSemaphore.tryAcquire()) {
                try {
                    if (isFallbackUserDefined()) {
                        executionHook.onFallbackStart(this);
                        fallbackExecutionChain = getFallbackObservable();
                    } else {
                        //same logic as above without the hook invocation
                        fallbackExecutionChain = getFallbackObservable();
                    }
                } catch (Throwable ex) {
                    //If hook or user-fallback throws, then use that as the result of the fallback lookup
                    fallbackExecutionChain = Observable.error(ex);
                }

                return fallbackExecutionChain
                        .doOnEach(setRequestContext)
                        .lift(new FallbackHookApplication(_cmd))
                        .lift(new DeprecatedOnFallbackHookApplication(_cmd))
                        .doOnNext(markFallbackEmit)
                        .doOnCompleted(markFallbackCompleted)
                        .onErrorResumeNext(handleFallbackError)
                        .doOnTerminate(singleSemaphoreRelease)
                        .doOnUnsubscribe(singleSemaphoreRelease);
            } else {
               return handleFallbackRejectionByEmittingError();
            }
        } else {
            return handleFallbackDisabledByEmittingError(originalException, failureType, message);
        }
    }
}
```

获取回退方法观察者com.netflix.hystrix.HystrixCommand::getFallbackObservable

```text
final protected Observable<R> getFallbackObservable() {
    return Observable.defer(new Func0<Observable<R>>() {
        @Override
        public Observable<R> call() {
            try {
                return Observable.just(getFallback());
            } catch (Throwable ex) {
                return Observable.error(ex);
            }
        }
    });
}
```

最终执行的函数com.netflix.hystrix.GenericCommand::getFallback

```text
/**
 * The fallback is performed whenever a command execution fails.
 * Also a fallback method will be invoked within separate command in the case if fallback method was annotated with
 * HystrixCommand annotation, otherwise current implementation throws RuntimeException and leaves the caller to deal with it
 * (see {@link super#getFallback()}).
 * The getFallback() is always processed synchronously.
 * Since getFallback() can throw only runtime exceptions thus any exceptions are thrown within getFallback() method
 * are wrapped in {@link FallbackInvocationException}.
 * A caller gets {@link com.netflix.hystrix.exception.HystrixRuntimeException}
 * and should call getCause to get original exception that was thrown in getFallback().
 *
 * @return result of invocation of fallback method or RuntimeException
 */
@Override
protected Object getFallback() {
    final CommandAction commandAction = getFallbackAction();
    if (commandAction != null) {
        try {
            return process(new Action() {
                @Override
                Object execute() {
                    MetaHolder metaHolder = commandAction.getMetaHolder();
                    Object[] args = createArgsForFallback(metaHolder, getExecutionException());
                    return commandAction.executeWithArgs(metaHolder.getFallbackExecutionType(), args);
                }
            });
        } catch (Throwable e) {
            LOGGER.error(FallbackErrorMessageBuilder.create()
                    .append(commandAction, e).build());
            throw new FallbackInvocationException(unwrapCause(e));
        }
    } else {
        return super.getFallback();
    }
}
```

这个回退方法的栈非常长，其他细节的暂时对源码剖析告一段落...

> 代码面前，了无秘密。



