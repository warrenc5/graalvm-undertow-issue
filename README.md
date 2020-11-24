# GraalVM Undertow Issue

This repository reproduce an issue with GraalVM and Undertow that makes native image failing

## build native image

```bash
$ mvn clean package -Pnative
```

## run (leading to error)

```bash
$ mvn clean package -Pnative
$ ./target/graalvm-undertow-issue-native

Nov 24, 2020 11:48:10 AM io.undertow.Undertow start
INFO: starting server: Undertow - 2.2.2.Final
Nov 24, 2020 11:48:10 AM org.xnio.Xnio <clinit>
INFO: XNIO version 3.8.0.Final
Nov 24, 2020 11:48:10 AM org.xnio.nio.NioXnio <clinit>
INFO: XNIO NIO Implementation Version 3.8.0.Final
Nov 24, 2020 11:48:10 AM org.jboss.threads.Version <clinit>
INFO: JBoss Threads version 3.1.0.Final
Exception in thread "main" com.oracle.svm.core.jdk.UnsupportedFeatureError: Unsupported method of Unsafe
	at com.oracle.svm.core.util.VMError.unsupportedFeature(VMError.java:87)
	at jdk.internal.misc.Unsafe.staticFieldBase(Unsafe.java:236)
	at sun.misc.Unsafe.staticFieldBase(Unsafe.java:677)
	at org.jboss.threads.EnhancedQueueExecutor.<clinit>(EnhancedQueueExecutor.java:295)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.invokeClassInitializer(ClassInitializationInfo.java:351)
	at com.oracle.svm.core.classinitialization.ClassInitializationInfo.initialize(ClassInitializationInfo.java:271)
	at org.xnio.XnioWorker.<init>(XnioWorker.java:139)
	at org.xnio.nio.NioXnioWorker.<init>(NioXnioWorker.java:84)
	at org.xnio.nio.NioXnio.build(NioXnio.java:233)
	at org.xnio.XnioWorker$Builder.build(XnioWorker.java:1191)
	at org.xnio.Xnio.createWorker(Xnio.java:481)
	at org.xnio.Xnio.createWorker(Xnio.java:463)
	at org.xnio.Xnio.createWorker(Xnio.java:450)
	at io.undertow.Undertow.start(Undertow.java:122)
	at com.softinstigate.App.main(App.java:25)
```

## Analysis of error

The error comes from [jboss-threads](https://github.com/jbossas/jboss-threads).

Here the static class initializer of [`EnhancedQueueExecutor`](https://github.com/jbossas/jboss-threads/blob/master/src/main/java/org/jboss/threads/EnhancedQueueExecutor.java) fails.

The code is:

```java
static {
        try {
            terminationWaitersOffset = unsafe.objectFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("terminationWaiters"));

            queueSizeOffset = unsafe.objectFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("queueSize"));

            peakThreadCountOffset = unsafe.objectFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("peakThreadCount"));
            activeCountOffset = unsafe.objectFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("activeCount"));
            peakQueueSizeOffset = unsafe.objectFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("peakQueueSize"));

            sequenceBase = unsafe.staticFieldBase(EnhancedQueueExecutor.class.getDeclaredField("sequence"));
            sequenceOffset = unsafe.staticFieldOffset(EnhancedQueueExecutor.class.getDeclaredField("sequence"));
        } catch (NoSuchFieldException e) {
            throw new NoSuchFieldError(e.getMessage());
        }
    }
```

The offending code line is:

```java
static volatile int sequence = 1;

sequenceBase = unsafe.staticFieldBase(EnhancedQueueExecutor.class.getDeclaredField("sequence"));
```

The `unsafe` class is from [`JBossExecutors`](https://github.com/jbossas/jboss-threads/blob/master/src/main/java/org/jboss/threads/JBossExecutors.java)