# Spring Boot集成熔断、限流、降级

在引入resilience4j之前，我们先来讨论下服务稳定性的三大法宝。

- 降级：在有限资源情况下，为了应对超负荷流量，适当放弃一些功能，以保证服务的整体稳定性。例如：双十一大促时，关闭个性化推荐。

- 限流：为了应对突发流量，只允许一部分请求通过，放弃其余请求。例如：当前服务忙，请稍后再试。

- 熔断：这个概念最早源于物理学。
  
  - 在电路中，若电流过大，熔断器(保险丝 / 空气开关)会发生熔断，切断线路，以保证用电安全。
  
  - 在微服务架构中，若服务调用发生大量错误(超时)，可以直接将微服务降级，以保证服务的整体稳定性。

Resillence4j是一款轻量级、易用的"容错框架"，提供了保证稳定性所需的几大基础组件：

- Retry：重试

- Circuit Breaker：基于Ring Buffer的熔断器，根据失败率/次数，自动切换熔断器的开关状态。

- Rate Limiter：基于AtomicReference + 状态机 实现的限流器

- Time Limiter：基于限时Future / CompletationStage的时限器

- Bulk Head：基于信号量 / 线程池的壁仓隔离。

- Cache / Fallback：为上述组件提供降级时的包装函数

Resillence4j支持Java、注解等多种使用方法，我们这里选用最方便的Spring Boot注解方法。

## Circuit Breaker

首先来看一下熔断器，它内置了如下三种状态：

- CLOSE：初始状态，熔断器关闭，服务正常运行。

- OPEN：发生大量错误后，熔断器打开，直接返回降级结果，不再调用真实服务逻辑。

- HALF OPEN：OPEN一段时间后，小流量放开访问，看真实逻辑部分是否恢复正常。如果恢复，会切换到CLOSE状态。

老规矩，先添加依赖：

```groovy
implementation 'io.github.resilience4j:resilience4j-all:1.7.1'
implementation 'io.github.resilience4j:resilience4j-spring-boot2:1.7.1'
```

说明如下：

- 由于后续几个组件都会使用，我们这里直接使用了all，你可以根据实际情况，裁剪需要的组件。

- spring-boot：添加了对应的注解和自动配置。

熔断器的配置如下：

```yaml
resilience4j:
  circuitbreaker:
    instances:
      getUserById:
        registerHealthIndicator: true
        slidingWindowSize: 100
        failureRateThreshold: 50
```

说明如下：

- 熔断器名称是getUserById

- 滑动窗口大小100

- 失败(熔断)阀值是50%

代码用法如下：

```java
  @Override
    @CircuitBreaker(name = "getUser", fallbackMethod = "getUserByIdFallback")
    public Optional<User> getUserById(long id) {
        // Mock a failure
        if (ThreadLocalRandom.current().nextInt(100) < 90) {
            throw new RuntimeException("mock failure");
        }
        return userRepository.getUser(id);
    }

    public Optional<User> getUserByIdFallback(long id, Throwable e) {
        LOG.error("enter fallback for getUserById", e);
        return Optional.empty();
    }
```

在上述代码中，我们以90%的概率模拟了随机异常。

当熔断发生时，会使用getUserByIdFallback中的降级结果。

执行几次后，会出现类似如下的错误日志，熔断器已成功开启。

```shell
2021-10-09 01:34:32.156 ERROR 2214 --- [o-8080-exec-144] c.c.h.d.s.service.impl.UserServiceImpl   : enter fallback for getUserById

io.github.resilience4j.circuitbreaker.CallNotPermittedException: CircuitBreaker 'getUser' is OPEN and does not permit further calls
    at io.github.resilience4j.circuitbreaker.CallNotPermittedException.createCallNotPermittedException(CallNotPermittedException.java:48) ~[resilience4j-circuitbreaker-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine$OpenState.acquirePermission(CircuitBreakerStateMachine.java:696) ~[resilience4j-circuitbreaker-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.internal.CircuitBreakerStateMachine.acquirePermission(CircuitBreakerStateMachine.java:206) ~[resilience4j-circuitbreaker-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.CircuitBreaker.lambda$decorateCheckedSupplier$82a9021a$1(CircuitBreaker.java:70) ~[resilience4j-circuitbreaker-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.CircuitBreaker.executeCheckedSupplier(CircuitBreaker.java:834) ~[resilience4j-circuitbreaker-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.configure.CircuitBreakerAspect.defaultHandling(CircuitBreakerAspect.java:188) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.configure.CircuitBreakerAspect.proceed(CircuitBreakerAspect.java:135) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.configure.CircuitBreakerAspect.lambda$circuitBreakerAroundAdvice$6edadc33$1(CircuitBreakerAspect.java:118) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.fallback.DefaultFallbackDecorator.lambda$decorate$52452fd9$1(DefaultFallbackDecorator.java:36) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.circuitbreaker.configure.CircuitBreakerAspect.circuitBreakerAroundAdvice(CircuitBreakerAspect.java:118) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at sun.reflect.GeneratedMethodAccessor127.invoke(Unknown Source) ~[na:na]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_291]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_291]
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:634) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:624) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:72) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:175) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:97) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:692) [spring-aop-5.3.9.jar:5.3.9]
    at com.coder4.homs.demo.server.service.impl.UserServiceImpl$$EnhancerBySpringCGLIB$$19b58f1b.getUserById(<generated>) [main/:na]
    at com.coder4.homs.demo.server.web.logic.impl.UserLogicImpl.getUserById(UserLogicImpl.java:51) [main/:na]
    at com.coder4.homs.demo.server.web.ctrl.UserController.getById(UserController.java:36) [main/:na]
    at sun.reflect.GeneratedMethodAccessor113.invoke(Unknown Source) ~[na:na]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_291]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_291]
    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:808) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1064) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898) [spring-webmvc-5.3.9.jar:5.3.9]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:655) [tomcat-embed-core-9.0.50.jar:4.0.FR]
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) [spring-webmvc-5.3.9.jar:5.3.9]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:764) [tomcat-embed-core-9.0.50.jar:4.0.FR]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:228) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) [tomcat-embed-websocket-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:96) [spring-boot-actuator-2.5.3.jar:2.5.3]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:357) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:382) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:893) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1723) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_291]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_291]
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at java.lang.Thread.run(Thread.java:748) [na:1.8.0_291]
```

## Bulkhead & TimeLimiter

下面我们来看一下实线器，即限定必须在X时间内执行完毕，否则抛出异常。

Resillence4j的TimeLimiter设计中，并没有内置线程池，而是要业务代码自行处理。我们可以结合Bulkhead的线程池模式一同使用，首先配置如下：

```yaml
resilience4j:
  thread-pool-bulkhead:
    instances:
      getUserByName:
        maxThreadPoolSize: 100
        coreThreadPoolSize: 50
        queueCapacity: 200
  timelimiter:
    instances:
      getUserByName:
        timeoutDuration: 1s
        cancelRunningFuture: true
```

如上所述，我们配置了线程池，并设置时限为1秒。

接着看一下用法：

```java
@Override
@Bulkhead(name = "getUserByName", type = Type.THREADPOOL)
@TimeLimiter(name = "getUserByName", fallbackMethod = "getUserByNameWithCompletableFutureFallback")
public CompletableFuture<Optional<User>> getUserByNameWithCompletableFuture(String name) {
    // Mock timeout
    Try.run(() -> Thread.sleep(ThreadLocalRandom.current().nextInt(2000)));

    return CompletableFuture.completedFuture(userRepository.getUserByName(name));
}

public CompletableFuture<Optional<User>> getUserByNameWithCompletableFutureFallback(String name, Throwable e) {
    LOG.error("enter fallback for getUserByNameFallback", e);
    return CompletableFuture.completedFuture(Optional.empty());
}
```

我们模拟了随机超时时间，当超过1秒时，会自动抛出如下的降级异常，并走降级逻辑。

```shell
2021-10-09 01:53:32.637 ERROR 4890 --- [pool-7-thread-1] c.c.h.d.s.service.impl.UserServiceImpl   : enter fallback for getUserByNameFallback

java.util.concurrent.TimeoutException: TimeLimiter 'getUserByName' recorded a timeout exception.
    at io.github.resilience4j.timelimiter.TimeLimiter.createdTimeoutExceptionWithName(TimeLimiter.java:221) ~[resilience4j-timelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.timelimiter.internal.TimeLimiterImpl$Timeout.lambda$of$0(TimeLimiterImpl.java:185) ~[resilience4j-timelimiter-1.7.1.jar:1.7.1]
    at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) ~[na:1.8.0_291]
    at java.util.concurrent.FutureTask.run(FutureTask.java:266) [na:1.8.0_291]
    at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$201(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_291]
    at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:293) [na:1.8.0_291]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_291]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_291]
    at java.lang.Thread.run(Thread.java:748) [na:1.8.0_291]
```

## RateLimiter

最后我们来看一下限流器，配置如下：

```yaml
resilience4j:
  rateLimiter:
    instances:
      getUserByIdV2:
        limitForPeriod: 1
        limitRefreshPeriod: 500ms
        timeoutDuration: 0
```

设置了每0.5秒限1个请求，用法如下：

```java
@Override
@RateLimiter(name = "getUserByIdV2", fallbackMethod = "getUserByIdV2Fallback")
public Optional<User> getUserByIdV2(long id) {
    return Optional.ofNullable(userMapper.getUser(id)).map(UserDO::toUser);
}

public Optional<User> getUserByIdV2Fallback(long id, Throwable e) {
    LOG.error("getUserByIdV2 fallback exception", e);
    return Optional.empty();
}
```

当快速访问两次接口后，会抛出如下的异常，并返回降级结果。

```shell
2021-10-09 14:00:13.564 ERROR 5598 --- [nio-8080-exec-8] c.c.h.d.s.service.impl.UserServiceImpl   : getUserByIdV2 fallback exception

io.github.resilience4j.ratelimiter.RequestNotPermitted: RateLimiter 'getUserByIdV2' does not permit further calls
    at io.github.resilience4j.ratelimiter.RequestNotPermitted.createRequestNotPermitted(RequestNotPermitted.java:43) ~[resilience4j-ratelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.RateLimiter.waitForPermission(RateLimiter.java:591) ~[resilience4j-ratelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.RateLimiter.lambda$decorateCheckedSupplier$9076412b$1(RateLimiter.java:213) ~[resilience4j-ratelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.RateLimiter.executeCheckedSupplier(RateLimiter.java:898) ~[resilience4j-ratelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.RateLimiter.executeCheckedSupplier(RateLimiter.java:884) ~[resilience4j-ratelimiter-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.configure.RateLimiterAspect.handleJoinPoint(RateLimiterAspect.java:179) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.configure.RateLimiterAspect.proceed(RateLimiterAspect.java:142) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.configure.RateLimiterAspect.lambda$rateLimiterAroundAdvice$749d37c4$1(RateLimiterAspect.java:125) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.fallback.DefaultFallbackDecorator.lambda$decorate$52452fd9$1(DefaultFallbackDecorator.java:36) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at io.github.resilience4j.ratelimiter.configure.RateLimiterAspect.rateLimiterAroundAdvice(RateLimiterAspect.java:125) ~[resilience4j-spring-1.7.1.jar:1.7.1]
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_291]
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_291]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_291]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_291]
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethodWithGivenArgs(AbstractAspectJAdvice.java:634) ~[spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.aspectj.AbstractAspectJAdvice.invokeAdviceMethod(AbstractAspectJAdvice.java:624) ~[spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.aspectj.AspectJAroundAdvice.invoke(AspectJAroundAdvice.java:72) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:175) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.interceptor.ExposeInvocationInterceptor.invoke(ExposeInvocationInterceptor.java:97) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(ReflectiveMethodInvocation.java:186) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$CglibMethodInvocation.proceed(CglibAopProxy.java:750) [spring-aop-5.3.9.jar:5.3.9]
    at org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(CglibAopProxy.java:692) [spring-aop-5.3.9.jar:5.3.9]
    at com.coder4.homs.demo.server.service.impl.UserServiceImpl$$EnhancerBySpringCGLIB$$cba2db53.getUserByIdV2(<generated>) [main/:na]
    at com.coder4.homs.demo.server.web.logic.impl.UserLogicImpl.getUserByIdV2(UserLogicImpl.java:80) [main/:na]
    at com.coder4.homs.demo.server.web.ctrl.UserController.getByIdV2(UserController.java:51) [main/:na]
    at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method) ~[na:1.8.0_291]
    at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:62) ~[na:1.8.0_291]
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_291]
    at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_291]
    at org.springframework.web.method.support.InvocableHandlerMethod.doInvoke(InvocableHandlerMethod.java:197) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.method.support.InvocableHandlerMethod.invokeForRequest(InvocableHandlerMethod.java:141) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.ServletInvocableHandlerMethod.invokeAndHandle(ServletInvocableHandlerMethod.java:106) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.invokeHandlerMethod(RequestMappingHandlerAdapter.java:895) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter.handleInternal(RequestMappingHandlerAdapter.java:808) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.mvc.method.AbstractHandlerMethodAdapter.handle(AbstractHandlerMethodAdapter.java:87) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.DispatcherServlet.doDispatch(DispatcherServlet.java:1064) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.DispatcherServlet.doService(DispatcherServlet.java:963) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.FrameworkServlet.processRequest(FrameworkServlet.java:1006) [spring-webmvc-5.3.9.jar:5.3.9]
    at org.springframework.web.servlet.FrameworkServlet.doGet(FrameworkServlet.java:898) [spring-webmvc-5.3.9.jar:5.3.9]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:655) [tomcat-embed-core-9.0.50.jar:4.0.FR]
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:883) [spring-webmvc-5.3.9.jar:5.3.9]
    at javax.servlet.http.HttpServlet.service(HttpServlet.java:764) [tomcat-embed-core-9.0.50.jar:4.0.FR]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:228) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.websocket.server.WsFilter.doFilter(WsFilter.java:53) [tomcat-embed-websocket-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.RequestContextFilter.doFilterInternal(RequestContextFilter.java:100) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.FormContentFilter.doFilterInternal(FormContentFilter.java:93) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.boot.actuate.metrics.web.servlet.WebMvcMetricsFilter.doFilterInternal(WebMvcMetricsFilter.java:96) [spring-boot-actuator-2.5.3.jar:2.5.3]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.springframework.web.filter.CharacterEncodingFilter.doFilterInternal(CharacterEncodingFilter.java:201) [spring-web-5.3.9.jar:5.3.9]
    at org.springframework.web.filter.OncePerRequestFilter.doFilter(OncePerRequestFilter.java:119) [spring-web-5.3.9.jar:5.3.9]
    at org.apache.catalina.core.ApplicationFilterChain.internalDoFilter(ApplicationFilterChain.java:190) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.ApplicationFilterChain.doFilter(ApplicationFilterChain.java:163) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardWrapperValve.invoke(StandardWrapperValve.java:202) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardContextValve.invoke(StandardContextValve.java:97) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.authenticator.AuthenticatorBase.invoke(AuthenticatorBase.java:542) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardHostValve.invoke(StandardHostValve.java:143) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.valves.ErrorReportValve.invoke(ErrorReportValve.java:92) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.core.StandardEngineValve.invoke(StandardEngineValve.java:78) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.catalina.connector.CoyoteAdapter.service(CoyoteAdapter.java:357) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.http11.Http11Processor.service(Http11Processor.java:382) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.AbstractProcessorLight.process(AbstractProcessorLight.java:65) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.coyote.AbstractProtocol$ConnectionHandler.process(AbstractProtocol.java:893) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.util.net.NioEndpoint$SocketProcessor.doRun(NioEndpoint.java:1723) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at org.apache.tomcat.util.net.SocketProcessorBase.run(SocketProcessorBase.java:49) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_291]
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_291]
    at org.apache.tomcat.util.threads.TaskThread$WrappingRunnable.run(TaskThread.java:61) [tomcat-embed-core-9.0.50.jar:9.0.50]
    at java.lang.Thread.run(Thread.java:748) [na:1.8.0_291]
```

至此，我们已经熟悉了Resillence4j中的主要组件，并覆盖了yaml中的常见的配置。

更多配置选项，可以参考[这篇文档](https://resilience4j.readme.io/docs/getting-started-3)。

由于篇幅限制，本文并未涉及Retry、Cache两大组件，推荐你阅读[官方文档](https://resilience4j.readme.io/docs/retry)自行探索。
