---
layout: post
title:  "Eureka 集群同步问题"
date:   2021-04-15 13:44:06 +0800
categories: Eureka Java
---
Eureka 版本 1.6.2

最近，我们团队在对Eureka集群进行切换的时候碰到这样一个问题，三个Eureka Server节点，Node A，Node B，Node C 三个节点的管理页面上看到的Eureka续约数量差距很大，从各个Eureka 节点的日志中经常看到类似于注册的应用被取消（Cancelled）然后又被注册（Register）的日志。对于正常的Eureka 集群而言，由于不同的节点之间会同步续约信息，一旦节点收到其他节点发送来的续约信息，那么日志里就会输出续约（Renew）的操作日志。如果在一定的时间内，该节点并未收到其他的续约信息，那么在此节点上注册的应用就会被取消，一旦后续再次收到续约的信息，那么这个续约的信息就会被转化为注册信息，并以新应用的方式注册到此节点上。一旦出现这种情况，那么最直接的表现就是Eureka集群的各个节点之间的续约数会有很大的差距，远远超出正常的数值范围。那么在集群同步过程中到底发生了什么呢？
针对应用向Eureka节点发起的注册，续约，取消等请求，Eureka节点接收到这些请求之后并不会直接进行处理，而是将这些请求封装成 InstanceReplicationTask，之后将此任务通过 TaskDispatcher 派发至某一个线程进行处理。以应用注册为例，具体代码如下：
```java
 public void register(final InstanceInfo info) throws Exception {
        long expiryTime = System.currentTimeMillis() + getLeaseRenewalOf(info);
        batchingDispatcher.process(
                taskId("register", info),
                new InstanceReplicationTask(targetHost, Action.Register, info, null, true) {
                    public EurekaHttpResponse<Void> execute() {
                        return replicationClient.register(info);
                    }
                },
                expiryTime
        );
    }
```
上述代码位于 com.netflix.eureka.cluster.PeerEurekaNode 这个类当中，可以自行参阅。
那么这里的 batchingDispatcher 是什么呢？ 具体代码如下：
```java
PeerEurekaNode(PeerAwareInstanceRegistry registry, String targetHost, String serviceUrl,
                                     HttpReplicationClient replicationClient, EurekaServerConfig config,
                                     int batchSize, long maxBatchingDelayMs,
                                     long retrySleepTimeMs, long serverUnavailableSleepTimeMs) {
        this.registry = registry;
        this.targetHost = targetHost;
        this.replicationClient = replicationClient;

        this.serviceUrl = serviceUrl;
        this.config = config;
        this.maxProcessingDelayMs = config.getMaxTimeForReplication();

        String batcherName = getBatcherName();
        ReplicationTaskProcessor taskProcessor = new ReplicationTaskProcessor(targetHost, replicationClient);
        this.batchingDispatcher = TaskDispatchers.createBatchingTaskDispatcher(
                batcherName,
                config.getMaxElementsInPeerReplicationPool(),
                batchSize,
                config.getMaxThreadsForPeerReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
        this.nonBatchingDispatcher = TaskDispatchers.createNonBatchingTaskDispatcher(
                targetHost,
                config.getMaxElementsInStatusReplicationPool(),
                config.getMaxThreadsForStatusReplication(),
                maxBatchingDelayMs,
                serverUnavailableSleepTimeMs,
                retrySleepTimeMs,
                taskProcessor
        );
    }
```
上述代码也位于 com.netflix.eureka.cluster.PeerEurekaNode 这个类当中，可以自行参阅。

```java
public static <ID, T> TaskDispatcher<ID, T> createBatchingTaskDispatcher(String id,
                                                                             int maxBufferSize,
                                                                             int workloadSize,
                                                                             int workerCount,
                                                                             long maxBatchingDelay,
                                                                             long congestionRetryDelayMs,
                                                                             long networkFailureRetryMs,
                                                                             TaskProcessor<T> taskProcessor) {
        final AcceptorExecutor<ID, T> acceptorExecutor = new AcceptorExecutor<>(
                id, maxBufferSize, workloadSize, maxBatchingDelay, congestionRetryDelayMs, networkFailureRetryMs
        );
        final TaskExecutors<ID, T> taskExecutor = TaskExecutors.batchExecutors(id, workerCount, taskProcessor, acceptorExecutor);
        return new TaskDispatcher<ID, T>() {
            @Override
            public void process(ID id, T task, long expiryTime) {
                acceptorExecutor.process(id, task, expiryTime);
            }

            @Override
            public void shutdown() {
                acceptorExecutor.shutdown();
                taskExecutor.shutdown();
            }
        };
    }
```
上述代码也位于 com.netflix.eureka.util.batcher.TaskDispatchers 这个类当中，可以自行参阅。
从代码中我们可以明确的看到真正进行处理的是 AcceptorExecutor ，那么这个类是怎么处理这些请求的呢？为了更好的理解这个处理过程，我把这个类的源码直接复制了过来，如下：
```java
class AcceptorExecutor<ID, T> {

    private static final Logger logger = LoggerFactory.getLogger(AcceptorExecutor.class);

    private final int maxBufferSize;
    private final int maxBatchingSize;
    private final long maxBatchingDelay;

    private final AtomicBoolean isShutdown = new AtomicBoolean(false);

    private final BlockingQueue<TaskHolder<ID, T>> acceptorQueue = new LinkedBlockingQueue<>();
    private final BlockingDeque<TaskHolder<ID, T>> reprocessQueue = new LinkedBlockingDeque<>();
    private final Thread acceptorThread;

    private final Map<ID, TaskHolder<ID, T>> pendingTasks = new HashMap<>();
    private final Deque<ID> processingOrder = new LinkedList<>();

    private final Semaphore singleItemWorkRequests = new Semaphore(0);
    private final BlockingQueue<TaskHolder<ID, T>> singleItemWorkQueue = new LinkedBlockingQueue<>();

    private final Semaphore batchWorkRequests = new Semaphore(0);
    private final BlockingQueue<List<TaskHolder<ID, T>>> batchWorkQueue = new LinkedBlockingQueue<>();

    private final TrafficShaper trafficShaper;

    /*
     * Metrics
     */
    @Monitor(name = METRIC_REPLICATION_PREFIX + "acceptedTasks", description = "Number of accepted tasks", type = DataSourceType.COUNTER)
    volatile long acceptedTasks;

    @Monitor(name = METRIC_REPLICATION_PREFIX + "replayedTasks", description = "Number of replayedTasks tasks", type = DataSourceType.COUNTER)
    volatile long replayedTasks;

    @Monitor(name = METRIC_REPLICATION_PREFIX + "expiredTasks", description = "Number of expired tasks", type = DataSourceType.COUNTER)
    volatile long expiredTasks;

    @Monitor(name = METRIC_REPLICATION_PREFIX + "overriddenTasks", description = "Number of overridden tasks", type = DataSourceType.COUNTER)
    volatile long overriddenTasks;

    @Monitor(name = METRIC_REPLICATION_PREFIX + "queueOverflows", description = "Number of queue overflows", type = DataSourceType.COUNTER)
    volatile long queueOverflows;

    private final Timer batchSizeMetric;

    AcceptorExecutor(String id,
                     int maxBufferSize,
                     int maxBatchingSize,
                     long maxBatchingDelay,
                     long congestionRetryDelayMs,
                     long networkFailureRetryMs) {
        this.maxBufferSize = maxBufferSize;
        this.maxBatchingSize = maxBatchingSize;
        this.maxBatchingDelay = maxBatchingDelay;
        this.trafficShaper = new TrafficShaper(congestionRetryDelayMs, networkFailureRetryMs);

        ThreadGroup threadGroup = new ThreadGroup("eurekaTaskExecutors");
        this.acceptorThread = new Thread(threadGroup, new AcceptorRunner(), "TaskAcceptor-" + id);
        this.acceptorThread.setDaemon(true);
        this.acceptorThread.start();

        final double[] percentiles = {50.0, 95.0, 99.0, 99.5};
        final StatsConfig statsConfig = new StatsConfig.Builder()
                .withSampleSize(1000)
                .withPercentiles(percentiles)
                .withPublishStdDev(true)
                .build();
        final MonitorConfig config = MonitorConfig.builder(METRIC_REPLICATION_PREFIX + "batchSize").build();
        this.batchSizeMetric = new StatsTimer(config, statsConfig);
        try {
            Monitors.registerObject(id, this);
        } catch (Throwable e) {
            logger.warn("Cannot register servo monitor for this object", e);
        }
    }

    void process(ID id, T task, long expiryTime) {
        acceptorQueue.add(new TaskHolder<ID, T>(id, task, expiryTime));
        acceptedTasks++;
    }

    void reprocess(List<TaskHolder<ID, T>> holders, ProcessingResult processingResult) {
        reprocessQueue.addAll(holders);
        replayedTasks += holders.size();
        trafficShaper.registerFailure(processingResult);
    }

    void reprocess(TaskHolder<ID, T> taskHolder, ProcessingResult processingResult) {
        reprocessQueue.add(taskHolder);
        replayedTasks++;
        trafficShaper.registerFailure(processingResult);
    }

    BlockingQueue<TaskHolder<ID, T>> requestWorkItem() {
        singleItemWorkRequests.release();
        return singleItemWorkQueue;
    }

    BlockingQueue<List<TaskHolder<ID, T>>> requestWorkItems() {
        batchWorkRequests.release();
        return batchWorkQueue;
    }

    void shutdown() {
        if (isShutdown.compareAndSet(false, true)) {
            acceptorThread.interrupt();
        }
    }

    @Monitor(name = METRIC_REPLICATION_PREFIX + "acceptorQueueSize", description = "Number of tasks waiting in the acceptor queue", type = DataSourceType.GAUGE)
    public long getAcceptorQueueSize() {
        return acceptorQueue.size();
    }

    @Monitor(name = METRIC_REPLICATION_PREFIX + "reprocessQueueSize", description = "Number of tasks waiting in the reprocess queue", type = DataSourceType.GAUGE)
    public long getReprocessQueueSize() {
        return reprocessQueue.size();
    }

    @Monitor(name = METRIC_REPLICATION_PREFIX + "queueSize", description = "Task queue size", type = DataSourceType.GAUGE)
    public long getQueueSize() {
        return pendingTasks.size();
    }

    @Monitor(name = METRIC_REPLICATION_PREFIX + "pendingJobRequests", description = "Number of worker threads awaiting job assignment", type = DataSourceType.GAUGE)
    public long getPendingJobRequests() {
        return singleItemWorkRequests.availablePermits() + batchWorkRequests.availablePermits();
    }

    @Monitor(name = METRIC_REPLICATION_PREFIX + "availableJobs", description = "Number of jobs ready to be taken by the workers", type = DataSourceType.GAUGE)
    public long workerTaskQueueSize() {
        return singleItemWorkQueue.size() + batchWorkQueue.size();
    }

    class AcceptorRunner implements Runnable {
        @Override
        public void run() {
            long scheduleTime = 0;
            while (!isShutdown.get()) {
                try {
                    drainInputQueues();

                    int totalItems = processingOrder.size();

                    long now = System.currentTimeMillis();
                    if (scheduleTime < now) {
                        scheduleTime = now + trafficShaper.transmissionDelay();
                    }
                    if (scheduleTime <= now) {
                        assignBatchWork();
                        assignSingleItemWork();
                    }

                    // If no worker is requesting data or there is a delay injected by the traffic shaper,
                    // sleep for some time to avoid tight loop.
                    if (totalItems == processingOrder.size()) {
                        Thread.sleep(10);
                    }
                } catch (InterruptedException ex) {
                    // Ignore
                } catch (Throwable e) {
                    // Safe-guard, so we never exit this loop in an uncontrolled way.
                    logger.warn("Discovery AcceptorThread error", e);
                }
            }
        }

        private boolean isFull() {
            return pendingTasks.size() >= maxBufferSize;
        }

        private void drainInputQueues() throws InterruptedException {
            do {
                drainReprocessQueue();
                drainAcceptorQueue();

                if (!isShutdown.get()) {
                    // If all queues are empty, block for a while on the acceptor queue
                    if (reprocessQueue.isEmpty() && acceptorQueue.isEmpty() && pendingTasks.isEmpty()) {
                        TaskHolder<ID, T> taskHolder = acceptorQueue.poll(10, TimeUnit.MILLISECONDS);
                        if (taskHolder != null) {
                            appendTaskHolder(taskHolder);
                        }
                    }
                }
            } while (!reprocessQueue.isEmpty() || !acceptorQueue.isEmpty() || pendingTasks.isEmpty());
        }

        private void drainAcceptorQueue() {
            while (!acceptorQueue.isEmpty()) {
                appendTaskHolder(acceptorQueue.poll());
            }
        }

        private void drainReprocessQueue() {
            long now = System.currentTimeMillis();
            while (!reprocessQueue.isEmpty() && !isFull()) {
                TaskHolder<ID, T> taskHolder = reprocessQueue.pollLast();
                ID id = taskHolder.getId();
                if (taskHolder.getExpiryTime() <= now) {
                    expiredTasks++;
                } else if (pendingTasks.containsKey(id)) {
                    overriddenTasks++;
                } else {
                    pendingTasks.put(id, taskHolder);
                    processingOrder.addFirst(id);
                }
            }
            if (isFull()) {
                queueOverflows += reprocessQueue.size();
                reprocessQueue.clear();
            }
        }

        private void appendTaskHolder(TaskHolder<ID, T> taskHolder) {
            if (isFull()) {
                pendingTasks.remove(processingOrder.poll());
                queueOverflows++;
            }
            TaskHolder<ID, T> previousTask = pendingTasks.put(taskHolder.getId(), taskHolder);
            if (previousTask == null) {
                processingOrder.add(taskHolder.getId());
            } else {
                overriddenTasks++;
            }
        }

        void assignSingleItemWork() {
            if (!processingOrder.isEmpty()) {
                if (singleItemWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    while (!processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            singleItemWorkQueue.add(holder);
                            return;
                        }
                        expiredTasks++;
                    }
                    singleItemWorkRequests.release();
                }
            }
        }

        void assignBatchWork() {
            if (hasEnoughTasksForNextBatch()) {
                if (batchWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    int len = Math.min(maxBatchingSize, processingOrder.size());
                    List<TaskHolder<ID, T>> holders = new ArrayList<>(len);
                    while (holders.size() < len && !processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            holders.add(holder);
                        } else {
                            expiredTasks++;
                        }
                    }
                    if (holders.isEmpty()) {
                        batchWorkRequests.release();
                    } else {
                        batchSizeMetric.record(holders.size(), TimeUnit.MILLISECONDS);
                        batchWorkQueue.add(holders);
                    }
                }
            }
        }

        private boolean hasEnoughTasksForNextBatch() {
            if (processingOrder.isEmpty()) {
                return false;
            }
            if (pendingTasks.size() >= maxBufferSize) {
                return true;
            }

            TaskHolder<ID, T> nextHolder = pendingTasks.get(processingOrder.peek());
            long delay = System.currentTimeMillis() - nextHolder.getSubmitTimestamp();
            return delay >= maxBatchingDelay;
        }
    }
}

```

这个类位于 com.netflix.eureka.util.batcher.AcceptorExecutor，可以自行参阅
在这个类当中，我们可以看到实际上这个类是通过队列对任务进行管理的，几个队列各自的含义就不一一解释了，有兴趣的同学可以自己去检索一下。现在我们回到集群同步的问题，我们知道集群各个节点之间会同步不同的事件，但是在我们的这个问题中，出现的问题确是有些续约事件并未同步导致其他节点认为此应用失效，将其从服务列表中剔除。一开始我们以为是网络不稳定导致的，但是在对网络质量检测之后，生产环境的网络很稳定，此原因排除。那么排除了通信载体的问题之后，剩下的就是发送者和接收者的问题了，后来我们通过日志发现，在同步事件的过程中，接收者接收是没有问题的，因为对于不同的事件接收者都可以正常的接收，那么问题只能出在发送者这边了。在 AcceptorExecutor 这个类当中，我们发现有三个地方会丢弃任务，也就是丢弃事件，具体代码如下：
```java
   private void appendTaskHolder(TaskHolder<ID, T> taskHolder) {
            if (isFull()) {
                pendingTasks.remove(processingOrder.poll());
                queueOverflows++;
            }
            TaskHolder<ID, T> previousTask = pendingTasks.put(taskHolder.getId(), taskHolder);
            if (previousTask == null) {
                processingOrder.add(taskHolder.getId());
            } else {
                overriddenTasks++;
            }
        }
   void assignSingleItemWork() {
            if (!processingOrder.isEmpty()) {
                if (singleItemWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    while (!processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            singleItemWorkQueue.add(holder);
                            return;
                        }
                        expiredTasks++;
                    }
                    singleItemWorkRequests.release();
                }
            }
        }
 void assignBatchWork() {
            if (hasEnoughTasksForNextBatch()) {
                if (batchWorkRequests.tryAcquire(1)) {
                    long now = System.currentTimeMillis();
                    int len = Math.min(maxBatchingSize, processingOrder.size());
                    List<TaskHolder<ID, T>> holders = new ArrayList<>(len);
                    while (holders.size() < len && !processingOrder.isEmpty()) {
                        ID id = processingOrder.poll();
                        TaskHolder<ID, T> holder = pendingTasks.remove(id);
                        if (holder.getExpiryTime() > now) {
                            holders.add(holder);
                        } else {
                            expiredTasks++;
                        }
                    }
                    if (holders.isEmpty()) {
                        batchWorkRequests.release();
                    } else {
                        batchSizeMetric.record(holders.size(), TimeUnit.MILLISECONDS);
                        batchWorkQueue.add(holders);
                    }
                }
            }
        }
```

注意一下  pendingTasks 这个队列，这个队列代表目前正在处理的任务，从上述代码中我们可以看到实际上只有两种情形会丢弃任务，一个是队列满了，另外一个是任务超时了，第一个条件很容易理解，第二个条件实际上是指TaskHolder 超时，从代码中我们可以看到 TaskHolder 的超时判断是根据 TaskHolder 的超时时间和当前系统时间比较滞后进行判定的，那么TaskHolder 的超时时间是指什么呢？ 请参见本文的第一段代码，TaskHolder 的超时时间实际上是当前系统时间加上续约间隔，参数名称：renewalIntervalInSecs，结果我们发现renewalIntervalInSecs 这个参数在大部分应用中设置的值为 1，换句话说每秒钟都会有续约请求产生，结果就导致部分续约事件因为超时而被丢弃掉，从而其他节点由于未收到续约事件，从而判定应用失效，导致应用从服务列表中被剔除掉，反映到集群上就是续约数量在不同节点之间差距很大，而且日志中也出现大量的Cancel事件，反映到应用上就是由于部分应用被剔除从而导致应用之间不能互相访问，日志中出现大量的No instance 异常。

解决办法也很简单，调整 renewalIntervalInSecs 参数为10S，集群恢复正常，应用之间也可以正常通信。
至于另外一个原因，队列满了的情况，这个在我们目前的应用规模下，默认的队列大小（10000）即可满足。有兴趣的同学可以通过调小 maxElementsInPeerReplicationPool 这个参数进行模拟，也会出现事件被丢弃的现象