---
title: Zookeeper客户端Curator
date: 2019-03-19 16:12:53
tags: [Zookeeper]
categories: 分布式
---
Curator 是一个提供了高级API框架的java语音Zookeeper客户端库，使用其开发不需要在关系网络连接管理等细节，使开发变得更容易、可靠。Curator还提供了锁，leader选举等高级特性。Curator分为以下几个部分:
<!-- more -->
- curator-client ZooKeeper类替代品
- curator-framework 提供了高级API
- curator-recipes 提供了利用Zookeeper开发的高级特性。

### 添加依赖
```
<dependency>
  <groupId>org.apache.curator</groupId>
  <artifactId>curator-recipes</artifactId>
  <version>4.1.0</version>
</dependency>
```
### 主要API
#### 构建连接类
Curator的连接类为`CuratorFramework`，通过工厂类`CuratorFrameworkFactory`来实例化。对于一个Zookeeper集群，只需要一个`CuratorFramework`对象。开始使用`CuratorFramework`前，需要调用`start()`方法。程序结束前，需要调用`close()`方法。
有两种构建方式：
```
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3)
CuratorFramework client = CuratorFrameworkFactory.newClient(zookeeperConnectionString, retryPolicy);
client.start();
```
还可以使用流式风格构建：
```
RetryPolicy retryPolicy = new ExponentialBackoffRetry(1000, 3);
CuratorFramework client =
CuratorFrameworkFactory.builder()
        .connectString(connectionInfo)
        .sessionTimeoutMs(5000)
        .connectionTimeoutMs(5000)
        .retryPolicy(retryPolicy)
        .build();
client.start();
```
构建时还可以设置命名空间
```
CuratorFramework    client = CuratorFrameworkFactory.builder().namespace("MyApp") ... build();
…
client.create().forPath("/test", data);
// node was actually written to: "/MyApp/test"
```
#### 节点操作
##### 创建
```
String createdPath = client.create().withMode(CreateMode.PERSISTENT)
           .forPath(PATH, "TEST".getBytes(Charsets.UTF_8));
```
注意：`client.create().storingStatIn(stat)` stat并未设置值。
还调用`creatingParentsIfNeeded()`直接创建不存在的父节点
```
createdPath = client.create().creatingParentsIfNeeded().withMode(CreateMode.PERSISTENT)
                .forPath("/a/b/c", "TEST".getBytes(Charsets.UTF_8));
```
##### 是否存在
```
Stat stat = client.checkExists().usingWatcher(curatorTest).forPath(PATH);
```
`stat == null`说明节点不存在。
##### 获取数据
```
byte[] dataBytes = client.getData().storingStatIn(stat).usingWatcher(watcher).forPath(PATH);
String data = new String(dataBytes, Charsets.UTF_8);
```
##### 更新数据
```
Stat newStat = client.setData().withVersion(stat.getVersion())
    .forPath(PATH, "update".getBytes(Charsets.UTF_8));
```
##### 获取子节点
```
List<String> children = client.getChildren().usingWatcher(watcher).forPath("/");
```
##### 删除节点
```
client.delete().guaranteed().deletingChildrenIfNeeded().withVersion(100).forPath(multiPath);
```
`guaranteed()`：保证删除。当由于网络问题导致删除失败，只有`CuratorFramework`是开启的，则会在后台持续尝试删除，直至成功。当要注意，仍然会收到删除失败的异常。
##### 事物
`transaction()`方法可以开启Zookeeper事物，可以组合create、setData、check、delete与commit操作，将其当做一个单元。
```
CuratorOp createOp = client.transactionOp().create()
    .forPath("/transaction", "transaction".getBytes(Charsets.UTF_8));
CuratorOp setDataOp = client.transactionOp().setData().withVersion(0)
    .forPath("/transaction", "NEW DATA".getBytes(Charsets.UTF_8));
CuratorOp checkOp = client.transactionOp().check().withVersion(1).forPath("/transaction");
CuratorOp deleteOp = client.transactionOp().delete().forPath("/transaction");

try {
    List<CuratorTransactionResult> curatorTransactionResults = client.transaction()
        .forOperations(createOp, setDataOp, checkOp, deleteOp);
    for (CuratorTransactionResult result : curatorTransactionResults) {
        System.out.println(result.getForPath()+"  "+result.getType()+"  "+ result.getError());
    }
} catch (Exception e) {
    e.printStackTrace();
}
```
##### 例子
```
package com.fei.curator;

import com.google.common.base.Charsets;
import java.util.List;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.api.CuratorWatcher;
import org.apache.curator.framework.api.transaction.CuratorOp;
import org.apache.curator.framework.api.transaction.CuratorTransactionResult;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.data.Stat;

/**
 * CuratorTest
 *
 * @author fei
 */
public class CuratorTest implements CuratorWatcher {

    private static final String PATH = "/CuratorTest";
    private static final String PATH2 = "/CuratorTest2";
    private static final String WITH_PARENT_PATH = "/a/b/c";

    public static void main(String[] args) throws Exception {
        CuratorTest watcher = new CuratorTest();
        CuratorFramework client = CuratorClient.getFramework();
        //检查节点是否存在
        Stat stat = client.checkExists().usingWatcher(watcher).forPath(PATH);
        System.out.println(PATH + " 是否存在：" + (stat != null));
        if (stat == null) {
            stat = new Stat();
            stat.setAversion(9999);
            String createdPath = client.create().withMode(CreateMode.PERSISTENT)
                .forPath(PATH, "TEST".getBytes(Charsets.UTF_8));
            System.out.println("创建节点：" + createdPath);
        }
        //同时创建父节点
        String createdPath = client.create().creatingParentsIfNeeded()
            .withMode(CreateMode.PERSISTENT)
            .forPath(WITH_PARENT_PATH, "TEST".getBytes(Charsets.UTF_8));
        System.out.println("创建节点：" + createdPath);
        // 获取节点数据、状态
        byte[] dataBytes = client.getData().storingStatIn(stat).usingWatcher(watcher).forPath(PATH);
        String data = new String(dataBytes, Charsets.UTF_8);
        System.out.println("节点数据为：" + data);
        System.out.println("节点版本号：" + stat.getVersion() + " ,状态为：" + stat);
        //更新节点数据
        Stat newStat = client.setData().withVersion(stat.getVersion())
            .forPath(PATH, "update".getBytes(Charsets.UTF_8));
        System.out.println("stat after updated " + newStat);
        dataBytes = client.getData().storingStatIn(stat).usingWatcher(watcher).forPath(PATH);
        data = new String(dataBytes, Charsets.UTF_8);
        System.out.println("节点数据为：" + data);

        //创建/test2为顺序节点，对root可读可写.
        String node2 = client.create().withMode(CreateMode.PERSISTENT_SEQUENTIAL).forPath(PATH2);

        // 事物
        CuratorOp createOp = client.transactionOp().create()
            .forPath("/transaction", "transaction".getBytes(Charsets.UTF_8));
        CuratorOp setDataOp = client.transactionOp().setData().withVersion(0)
            .forPath("/transaction", "NEW DATA".getBytes(Charsets.UTF_8));
        CuratorOp checkOp = client.transactionOp().check().withVersion(1).forPath("/transaction");
        CuratorOp deleteOp = client.transactionOp().delete().forPath("/transaction");
        List<CuratorTransactionResult> curatorTransactionResults = null;
        try {
            curatorTransactionResults = client.transaction()
                .forOperations(createOp, setDataOp, checkOp, deleteOp);
            for (CuratorTransactionResult result : curatorTransactionResults) {
                System.out.println(
                    result.getForPath() + "  " + result.getType() + "  " + result.getError());
            }
        } catch (Exception e) {
            e.printStackTrace();
        }

        System.out.println();
        //获取子节点
        List<String> children = client.getChildren().usingWatcher(watcher).forPath("/");
        System.out.println("children: " + children);
        //删除节点
        client.delete().forPath(PATH);
        Stat node2Stat = client.checkExists().forPath(node2);
        client.delete().withVersion(node2Stat.getVersion()).forPath(node2);
        client.delete().guaranteed().deletingChildrenIfNeeded().forPath(WITH_PARENT_PATH);

        //验证是否删除成功
        Stat exists = client.checkExists().forPath(PATH);
        System.out.println(PATH + " 是否删除成功：" + (exists == null));
        exists = client.checkExists().forPath(node2);
        System.out.println(node2 + " 是否删除成功：" + (exists == null));
        exists = client.checkExists().forPath(WITH_PARENT_PATH);
        System.out.println(WITH_PARENT_PATH + " 是否删除成功：" + (exists == null));
    }

    /**
     * Same as {@link Watcher#process(WatchedEvent)}. If an exception is thrown, Curator will log
     * it
     *
     * @param event the event
     * @throws Exception any exceptions to log
     */
    @Override
    public void process(WatchedEvent event) throws Exception {
        System.out.println(event.toString());
    }
}
```
### cache
Zookeeper原生客户端注册的watch只能生效一次，想要监听节点变化需要反复注册watch。Curator中的cache可以自动注册监听，方便开发者获取节点相关信息。
#### NodeCache
`NodeCache` 用来缓存一个节点的视图。当节点被创建、修改、删除时，`NodeCache`会自动的修改其内容。调用`getCurrentData()`可以获取节点当前的数据、状态，返回值为null则说明节点不存在。还可以调用`nodeCache.getListenable().addListener(new NodeCacheListener())`添加监听，当节点有变化时，触发监听。
要开启需要调用`start()`或者`start(buildInitial)`，`buildInitial`参数表示开启时否是获取数据。使用完毕后需要调用`close()`。

需要注意的是，数据完全同步是不可能的，所以修改数据时要使用版本号来修改，以免覆盖了其他用户的修改。
> IMPORTANT - it's not possible to stay transactionally in sync. Users of this class must be prepared for false-positives and false-negatives. Additionally, always use the version number when updating data to avoid overwriting another process' change.

```
package com.fei.curator;

import com.google.common.base.Charsets;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.cache.ChildData;
import org.apache.curator.framework.recipes.cache.NodeCache;
import org.apache.curator.framework.recipes.cache.NodeCacheListener;

/**
 * NodeCacheTest
 *
 * @author fei
 */
public class NodeCacheTest {

    private static final String NODE_CACHE_PATH = "/nodeCachePath";

    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorClient.getFramework();
        NodeCache nodeCache = new NodeCache(client, NODE_CACHE_PATH);
        nodeCache.getListenable().addListener(new NodeCacheListener() {
            @Override
            public void nodeChanged() throws Exception {
                System.out.println("node changed ");
                printCurrentNodeInfo(nodeCache);
            }
        });
        nodeCache.start(true);
        printCurrentNodeInfo(nodeCache);
        ChildData currentData = nodeCache.getCurrentData();
        if (currentData == null) {
            client.create().forPath(NODE_CACHE_PATH);
        }
        client.setData().forPath(NODE_CACHE_PATH, "update".getBytes(Charsets.UTF_8));
        client.delete().forPath(NODE_CACHE_PATH);
        Thread.sleep(4000);
        nodeCache.close();
        client.close();
    }

    private static void printCurrentNodeInfo(NodeCache nodeCache) {
        ChildData currentData = nodeCache.getCurrentData();
        if (currentData != null) {
            String data = new String(currentData.getData(), Charsets.UTF_8);
            System.out.println(nodeCache.getPath() + " " + data + " " + currentData.getStat());
        }else {
            System.out.println("node not exist");
        }
    }
}

```
#### PathChildrenCache
`PathChildrenCache`用来监听一个节点的子节点的变化，包括update/create/delete，并会在本地缓存所有子节点的数据、状态。可以注册一个监听器，当改变发生的时候会收到通知。
##### 相关类：
- PathChildrenCache
- PathChildrenCacheMode
- PathChildrenCacheListener
- ChildData

##### 启动 关闭
调用`start()`开启cache，使用完毕后调用`close()`.`start()`可以传入启动模式`StartMode`参数，`StartMode`有三种：
1. `NORMAL` 默认模式。cahe在后台初始化数据，每一个已经存在的节点都会收到`CHILD_ADDED`事件.调用`start()`后立即调用`getCurrentData()`返回为空。
2. `BUILD_INITIAL_CACHE` 调用`start(StartMode.BUILD_INITIAL_CACHE)`前会自动初始化数据，之后立即调用`getCurrentData()`会返回所有子节点数据。
3. `POST_INITIALIZED_EVENT` 与`NORMAL`相同，但在初始化数据完成后会抛出`INITIALIZED`事件。

##### 实例
```
package com.fei.curator;

import com.google.common.base.Charsets;
import java.util.List;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.cache.ChildData;
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.cache.PathChildrenCache.StartMode;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent.Type;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
import org.apache.zookeeper.CreateMode;

/**
 * PathCacheTest
 *
 * @author fei
 */
public class PathCacheTest {

    private static void printChildData(Type type, ChildData data) {
        System.out.println(
            type + " path: " + data.getPath() + " content:" + new String(data.getData(),
                Charsets.UTF_8));
    }

    private static final String PATH = "/pathCachePath";

    public static void main(String[] args) throws Exception {

        CuratorFramework client = CuratorClient.getFramework();

        PathChildrenCache cache = new PathChildrenCache(client, PATH, true);
        PathChildrenCacheListener listener = new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event)
                throws Exception {
                System.out.println("childEvent:   " + event.toString());
                Type type = event.getType();
                ChildData data = event.getData();
                switch (type) {
                    case CHILD_ADDED:
                    case CHILD_UPDATED:
                    case CHILD_REMOVED: {
                        printChildData(type, data);
                        break;
                    }
                    case INITIALIZED: {
                        List<ChildData> initialData = event.getInitialData();
                        initialData.forEach(d -> printChildData(Type.INITIALIZED, d));
                        break;
                    }
                    default:
                        System.out.println("PathChildrenCacheEvent:" + type);
                }
                System.out.println();
            }
        };
        cache.getListenable().addListener(listener);
        cache.start(StartMode.BUILD_INITIAL_CACHE);
        List<ChildData> currentData = cache.getCurrentData();
        System.out.println("currentData:" + currentData);

        //修改节点数据
        client.setData().forPath(PATH, "UPDATE".getBytes(Charsets.UTF_8));

        String newChild = PATH + "/n4";
        client.create().withMode(CreateMode.EPHEMERAL).forPath(newChild);

        Thread.sleep(2000);
        client.setData().forPath(newChild, "UPDATE".getBytes(Charsets.UTF_8));

        Thread.sleep(2000);
        client.delete().forPath(newChild);
        Thread.sleep(2000);

        System.out.println("currentData:" + cache.getCurrentData());
        cache.close();
        Thread.sleep(15000);
        cache.close();
        client.close();
    }
}
```

需要注意的是，数据完全同步是不可能的，所以修改数据时要使用版本号来修改，以免覆盖了其他用户的修改。
> IMPORTANT - it's not possible to stay transactionally in sync. Users of this class must be prepared for false-positives and false-negatives. Additionally, always use the version number when updating data to avoid overwriting another process' change.

实际测试中发现，如果`PathChildrenCache`处于`start()`状态，视图删除cache观察的目录无效。调用`close()`后再删除，则生效。


#### TreeCache
`TreeCache`用来监听一个节点为起始的整个节点树的变化，包括update/create/delete，并会在本地缓存树的所有子节点的数据、状态。可以注册一个监听器，当改变发生的时候会收到通知。
##### 相关类：
- TreeCache
- TreeCacheListener

##### 启动 关闭
调用`start()`开启cache，使用完毕后调用`close()`。
注意`start()`后，调用`start()`后立即调用`getCurrentChildren(xxx)`或`getCurrentData(xxx)`可能会返回空数据，因为还未初始化完成。

##### 实例
```
package com.fei.curator;

import com.google.common.base.Charsets;
import java.util.Map;
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.api.UnhandledErrorListener;
import org.apache.curator.framework.recipes.cache.ChildData;
import org.apache.curator.framework.recipes.cache.TreeCache;
import org.apache.curator.framework.recipes.cache.TreeCacheEvent;
import org.apache.curator.framework.recipes.cache.TreeCacheListener;
import org.apache.zookeeper.CreateMode;

/**
 * TreeCacheTest
 *
 * @author fei
 */
public class TreeCacheTest {


    public static void main(String[] args) throws Exception {
        CuratorFramework client = CuratorClient.getFramework();
        TreeCache cache = new TreeCache(client, "/");
        cache.getListenable().addListener(new TreeCacheListener() {
            /**
             * Called when a change has occurred
             *
             * @param client the client
             * @param event describes the change
             * @throws Exception errors
             */
            @Override
            public void childEvent(CuratorFramework client, TreeCacheEvent event) throws Exception {
                System.out.println("changed: " + event);
            }
        });
        cache.getUnhandledErrorListenable().addListener(new UnhandledErrorListener() {
            @Override
            public void unhandledError(String message, Throwable e) {
                System.out.println("UnhandledErrorListener:" + message + " error:");
                e.printStackTrace();
            }
        });

        cache.start();
        String sonPath = "/pathCachePath";
        Map<String, ChildData> currentChildren = cache.getCurrentChildren(sonPath);
        System.out.println("=======currentChildren==========");
        System.out.println(currentChildren);
        System.out.println("========/pathCachePath data=============");
        ChildData currentData = cache.getCurrentData(sonPath);
        System.out.println(currentData);

        Thread.sleep(2000);
        String newChild = "/pathCachePath/" + System.currentTimeMillis();
        client.create().withMode(CreateMode.PERSISTENT).forPath(newChild);

        Thread.sleep(2000);
        String newChildSon = newChild + "/" + System.currentTimeMillis();
        client.create().withMode(CreateMode.PERSISTENT).forPath(newChildSon);

        Thread.sleep(2000);
        client.setData().forPath(newChild, String.valueOf(System.currentTimeMillis()).getBytes(
            Charsets.UTF_8));

        Thread.sleep(2000);
        client.delete().forPath(newChildSon);

        client.setData().forPath("/", "root".getBytes(Charsets.UTF_8));
        Thread.sleep(2000);

        currentChildren = cache.getCurrentChildren(sonPath);
        System.out.println("=======currentChildren==========");
        System.out.println(currentChildren);
        Thread.sleep(15000);
        cache.close();
        client.close();
    }
}
```

### lock
#### Shared Reentrant Lock
分布式可重入互斥锁，该锁时公平锁，按照请求的顺序获得锁。
- 相关类：`InterProcessMutex`
- 创建： `public InterProcessMutex(CuratorFramework client,String path)`
- 获取锁：`public void acquire()`，`public boolean acquire(long time,TimeUnit unit)`
- 释放锁：`public void release()`

- 实例

```
public class FakeLimitedResource
{
    private final AtomicBoolean     inUse = new AtomicBoolean(false);

    public void     use() throws InterruptedException
    {
        // in a real application this would be accessing/manipulating a shared resource

        if ( !inUse.compareAndSet(false, true) )
        {
            throw new IllegalStateException("Needs to be used by one client at a time");
        }

        try
        {
            Thread.sleep((long)(3 * Math.random()));
        }
        finally
        {
            inUse.set(false);
        }
    }
}
```
```
public class ExampleClientThatLocks
{
    private final InterProcessMutex lock;
    private final FakeLimitedResource resource;
    private final String clientName;

    public ExampleClientThatLocks(CuratorFramework client, String lockPath, FakeLimitedResource resource, String clientName)
    {
        this.resource = resource;
        this.clientName = clientName;
        lock = new InterProcessMutex(client, lockPath);
    }

    public void     doWork(long time, TimeUnit unit) throws Exception
    {
        if ( !lock.acquire(time, unit) )
        {
            throw new IllegalStateException(clientName + " could not acquire the lock");
        }
        try
        {
            System.out.println(clientName + " has the lock");
            resource.use();
        }
        finally
        {
            System.out.println(clientName + " releasing the lock");
            lock.release(); // always release the lock in a finally block
        }
    }
}
```
```
public class LockingExample {

    private static final int QTY = 5;
    private static final int REPETITIONS = QTY * 10;

    private static final String PATH = "/examples/locks";
    private static final String CONNECT_STRING = "127.0.0.1:2187,127.0.0.1:2188,127.0.0.1:2189";

    public static void main(String[] args) throws Exception {
        // all of the useful sample code is in ExampleClientThatLocks.java

        // FakeLimitedResource simulates some external resource that can only be access by one process at a time
        final FakeLimitedResource resource = new FakeLimitedResource();

        ExecutorService service = Executors.newFixedThreadPool(QTY);
        for (int i = 0; i < QTY; ++i) {
            final int index = i;
            Callable<Void> task = new Callable<Void>() {
                @Override
                public Void call() throws Exception {
                    CuratorFramework client = CuratorFrameworkFactory
                        .newClient(CONNECT_STRING, new ExponentialBackoffRetry(1000, 3));
                    try {
                        client.start();

                        ExampleClientThatLocks example = new ExampleClientThatLocks(client, PATH,
                            resource, "Client " + index);
                        for (int j = 0; j < REPETITIONS; ++j) {
                            example.doWork(10, TimeUnit.SECONDS);
                        }
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } catch (Exception e) {
                        e.printStackTrace();
                        // log or do something
                    } finally {
                        CloseableUtils.closeQuietly(client);
                    }
                    return null;
                }
            };
            service.submit(task);
        }

        service.shutdown();
        service.awaitTermination(10, TimeUnit.MINUTES);
    }
}
```
- 错误处理
强烈建议增加一个`ConnectionStateListener`监听类，并关注`SUSPENDED `、`LOST`状态。`SUSPENDED`状态下不能确定客户端是否还持有锁，除非随后收到`RECONNECTED`状态。`LOST`状态下不在持有锁。

#### Shared Lock
全局同步不可重入互斥锁。
- 相关类：`InterProcessSemaphoreMutex`
- 创建：`public InterProcessSemaphoreMutex(CuratorFramework client,String path)`
- 获取锁：`public void acquire()`，`public boolean acquire(long time,TimeUnit unit)`
- 释放锁：`public void release()`
- 错误处理
同`Shared Reentrant Lock`

#### Shared Reentrant Read Write Lock
分布式读写锁。读锁是共享的，可以同时被多个线程持有，写锁则只能被一个线程持有。一个写锁可以同时请求读锁，但反之不行。一个读锁视图去获取写锁，则永远不会成功。
- 相关类：`InterProcessReadWriteLock`、`InterProcessLock`
- 创建：
```
public InterProcessReadWriteLock(CuratorFramework client,String basePath)
public InterProcessLock readLock()
public InterProcessLock writeLock()
```
- 获取锁：`public void acquire()`，`public boolean acquire(long time,TimeUnit unit)`
- 释放锁：`public void release()`
- 错误处理
同`Shared Reentrant Lock`

#### Multi Shared Lock
管理多个锁的容器。当`acquire()`被调用后，会请求获取所有锁，如果失败，会双方所有已经请求的锁。同样的，`release()`被调用后，所有的锁都会被释放。
- 创建：`public InterProcessMultiLock(List<InterProcessLock> locks)`或`public InterProcessMultiLock(CuratorFramework client, List<String> paths)`
- 获取、释放、错误处理：同其他锁一致。

### Leader选举
分布式计算中，leader选举是指从多个计算机节点中选举出一个作为组织者的过程。Curator中有两个leader选举recipes。
#### LeaderSelector
公平选举，内部采用`InterProcessMutex`实现，按照原始的请求顺序，在前一个leader放弃后依次成为leader。
##### 相关类：
- LeaderSelector
- LeaderSelectorListener
- LeaderSelectorListenerAdapter
##### API
- 创建：
```
/**
 * @param client     the client
 * @param leaderPath the path for this leadership group
 * @param listener   listener
 */
public LeaderSelector(CuratorFramework client, String leaderPath, LeaderSelectorListener listener)
```

- `leaderSelector.start();` 开启选举
- `takeLeadership()` start 后，当成为leader后，listener的`takeLeadership()`方法会被调用。`takeLeadership()`应该只有在需要放弃leader时才退出。
- `leaderSelector.close();` 关闭
-  `leaderSelector.autoRequeue()` 放弃leader后继续参与选举

##### 错误处理
使用`LeaderSelector`时需要关注连接状态变化。如果已经成为leader，需要对`SUSPENDED`、`LOST`做出响应。
如果收到`SUSPENDED`，应该假定自己已经不再是leader，知道收到`RECONNECTED`。如果收到`LOST`，则自己已经不再是leader，`takeLeadership` 方法应该退出。
##### 实例
```
/**
 * An example leader selector client. Note that {@link LeaderSelectorListenerAdapter} which has the
 * recommended handling for connection state issues
 */
public class ExampleClient extends LeaderSelectorListenerAdapter implements Closeable {

    private final String name;
    private final LeaderSelector leaderSelector;
    private final AtomicInteger leaderCount = new AtomicInteger();

    public ExampleClient(CuratorFramework client, String path, String name) {
        this.name = name;

        // create a leader selector using the given path for management
        // all participants in a given leader selection must use the same path
        // ExampleClient here is also a LeaderSelectorListener but this isn't required
        leaderSelector = new LeaderSelector(client, path, this);

        // for most cases you will want your instance to requeue when it relinquishes leadership
        leaderSelector.autoRequeue();
    }

    public void start() throws IOException {
        // the selection for this instance doesn't start until the leader selector is started
        // leader selection is done in the background so this call to leaderSelector.start() returns immediately
        leaderSelector.start();
    }

    @Override
    public void close() throws IOException {
        leaderSelector.close();
    }

    @Override
    public void takeLeadership(CuratorFramework client) throws Exception {
        // we are now the leader. This method should not return until we want to relinquish leadership
        final int waitSeconds = (int) (5 * Math.random()) + 1;

        System.out.println(name + " is now the leader. Waiting " + waitSeconds + " seconds...");
        System.out.println(
            name + " has been leader " + leaderCount.getAndIncrement() + " time(s) before.");
        try {
            Thread.sleep(TimeUnit.SECONDS.toMillis(waitSeconds));
        } catch (InterruptedException e) {
            System.err.println(name + " was interrupted.");
            Thread.currentThread().interrupt();
        } finally {
            System.out.println(name + " relinquishing leadership.\n");
        }
    }
}
```
```
public class LeaderSelectorExample
{
    private static final int        CLIENT_QTY = 10;

    private static final String CONNECT_STRING = "127.0.0.1:2187,127.0.0.1:2188,127.0.0.1:2189";

    private static final String     PATH = "/examples/leader";

    public static void main(String[] args) throws Exception
    {
        // all of the useful sample code is in ExampleClient.java

        System.out.println("Create " + CLIENT_QTY + " clients, have each negotiate for leadership and then wait a random number of seconds before letting another leader election occur.");
        System.out.println("Notice that leader election is fair: all clients will become leader and will do so the same number of times.");

        List<CuratorFramework>  clients = Lists.newArrayList();
        List<ExampleClient>     examples = Lists.newArrayList();
        try
        {
            for ( int i = 0; i < CLIENT_QTY; ++i )
            {
                CuratorFramework client = CuratorFrameworkFactory
                    .newClient(CONNECT_STRING, new ExponentialBackoffRetry(1000, 3));
                clients.add(client);

                ExampleClient       example = new ExampleClient(client, PATH, "Client #" + i);
                examples.add(example);

                client.start();
                example.start();

            }

            System.out.println("Press enter/return to quit\n");
            new BufferedReader(new InputStreamReader(System.in)).readLine();
        }
        finally
        {
            System.out.println("Shutting down...");

            for ( ExampleClient exampleClient : examples )
            {
                CloseableUtils.closeQuietly(exampleClient);
            }
            for ( CuratorFramework client : clients )
            {
                CloseableUtils.closeQuietly(client);
            }
        }
    }
}
```
#### LeaderLatch

##### 相关类
- LeaderLatch
- LeaderLatchListener

##### API
- 创建：
```
/**
 * @param client    the client
 * @param latchPath the path for this leadership group
 */
public LeaderLatch(CuratorFramework client, String latchPath)
/**
 * @param client    the client
 * @param latchPath the path for this leadership group
 * @param id        participant ID
 */
public LeaderLatch(CuratorFramework client, String latchPath, String id)
```
- 开启 `leaderLatch.start();`
- 是否成为leader `leaderLatch.hasLeadership();`
- 阻塞直至成为leader `leaderLatch.await();`
- 关闭 `leaderLatch.close();`
退出选举，如果本身时领导，则会放弃领导，是唯一放弃leader的方法。
- leader状态改变通知
当成为leader后`LeaderLatchListener.isLeader()`会被调用，放弃leader后，`LeaderLatchListener.notLeader()`会被调用。

### barrier
#### DistributedBarrier
分布式系统中用来阻止一系列节点的运行，直接某一刻满足条件后，所有节点继续运行。
`DistributedBarrier`原理新简单，调用`waitOnBarrier()`方法后，线程会检查特定节点是否存在，如果不存在则代表条件满足，否则继续等待(Object.wait())。
`DistributedBarrier`提供了`setBarrier()`、`removeBarrier()`两个工具方法，代表开始等待、条件满足，继续执行。
```
/**
 * BarrierTest
 *
 * @author fei
 */
public class BarrierTest {

    private static final int CLIENT_QTY = 5;
    private static final String CONNECT_STRING = "127.0.0.1:2187,127.0.0.1:2188,127.0.0.1:2189";
    private static final String PATH = "/examples/barrier";

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(CLIENT_QTY);
        List<CuratorFramework> clients = new ArrayList<>(CLIENT_QTY);
        List<DistributedBarrier> barriers = new ArrayList<>(CLIENT_QTY);
        try {
            CuratorFramework c = CuratorFrameworkFactory
                .newClient(CONNECT_STRING, new ExponentialBackoffRetry(1000, 3));
            c.start();
            DistributedBarrier b = new DistributedBarrier(c, PATH);
            try {
                //设置栅栏
                b.setBarrier();
            } catch (Exception e) {
                e.printStackTrace();
            }
            clients.add(c);
            barriers.add(b);
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client = CuratorFrameworkFactory
                    .newClient(CONNECT_STRING, new ExponentialBackoffRetry(1000, 3));
                clients.add(client);
                client.start();
                DistributedBarrier barrier = new DistributedBarrier(client, PATH);
                barriers.add(barrier);
                int index = i;
                long sleepTime = index * 1000;
                executorService.submit(() -> {
                    try {
                        Thread.sleep(sleepTime);
                        System.out.println(index + "start wait");
                        barrier.waitOnBarrier();
                        System.out.println(index + "start continue");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                });

            }
            Thread.sleep(6000);
            try {
                b.removeBarrier();
            } catch (Exception e) {
                e.printStackTrace();
            }
            Thread.sleep(2000);
        } finally {
            clients.forEach(CloseableUtils::closeQuietly);
            executorService.shutdownNow();
        }
    }
}
```
#### DistributedDoubleBarrier
`DistributedDoubleBarrier`可以使多个客户端同步计算的开始与结束。当足够数量的处理加入栏栅时，开始记性计算，并在全部结束后离开计算。
- 构造
```
/**
 * Creates the barrier abstraction. <code>memberQty</code> is the number of members in the
 * barrier. When {@link #enter()} is called, it blocks until all members have entered. When
 * {@link #leave()} is called, it blocks until all members have left.
 *
 * @param client the client
 * @param barrierPath path to use
 * @param memberQty the number of members in the barrier. NOTE: more than <code>memberQty</code>
 *                  can enter the barrier. <code>memberQty</code> is a threshold, not a limit
 */
public DistributedDoubleBarrier(CuratorFramework client, String barrierPath, int memberQty)
```
- 进入 `enter();`
- 退出 `leave();`

调用`enter()`、`leave()`方法后会阻塞，知道调用相应方法的客户端数量超过`memberQty`

```
/**
 * DoubleBarrierTest
 *
 * @author fei
 */
public class DoubleBarrierTest {

    private static final int CLIENT_QTY = 5;
    private static final String CONNECT_STRING = "127.0.0.1:2187,127.0.0.1:2188,127.0.0.1:2189";
    private static final String PATH = "/examples/doubleBarrier";

    public static void main(String[] args) throws InterruptedException {
        ExecutorService executorService = Executors.newFixedThreadPool(CLIENT_QTY);
        List<CuratorFramework> clients = new ArrayList<>(CLIENT_QTY);
        List<DistributedDoubleBarrier> barriers = new ArrayList<>(CLIENT_QTY);
        try {
            for (int i = 0; i < CLIENT_QTY; i++) {
                CuratorFramework client = CuratorFrameworkFactory
                    .newClient(CONNECT_STRING, new ExponentialBackoffRetry(1000, 3));
                clients.add(client);
                client.start();
                DistributedDoubleBarrier barrier = new DistributedDoubleBarrier(client, PATH,
                    CLIENT_QTY);
                barriers.add(barrier);
                int index = i;
                long sleepTime = index * 1000;
                executorService.submit(() -> {
                    try {
                        Thread.sleep(sleepTime);
                        System.out.println(index + " wait to enter");
                        barrier.enter();
                        System.out.println(index + " enter");
                        Thread.sleep(sleepTime);
                        System.out.println(index + " wait to leave");
                        barrier.leave();
                        System.out.println(index + " leave");
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    } catch (Exception e) {
                        e.printStackTrace();
                    }

                });

            }
            Thread.sleep(20000);
        } finally {
            clients.forEach(CloseableUtils::closeQuietly);
            executorService.shutdownNow();
        }
    }
}
```

### 注意事项
#### ZooKeeper watches 使用单线程
所有的ZooKeeper watches处理时串行的，当一个watcher在执行时，其他的watcher都不能执行。所以，watcher处理应该尽可能快的返回。
```
...
InterProcessMutex   lock = ...

public void process(WatchedEvent event)
{
    lock.acquire();
       ...
}
```
上述代码不能工作。`InterProcessMutex`依赖watcher获得通知。可以使用另外的线程请求锁：
```
...
InterProcessMutex   lock = ...
ExecutorService    service = ...

public void process(WatchedEvent event)
{
    service.submit(new Callable<Void>(){
        Void call() {
            lock.acquire();
              ...
        }
    });
}
```
#### 如何在`InterProcessMutex`获取锁失败后立即返回
```
InterProcessMutex lock = ...
boolean didLock = lock.acquire(0, TimeUnit.any);
if ( !didLock )
{
    // comes back immediately
}
```

#### 处理session failure
Curator处理session failure的默认策略月处理网络连接失败一样：检查当前重试策略，如果运行重试，则重试。
但是，如果一系列操作都与session相关，例如，临时节点创建后作为一种标记，然后执行其他操作。如果在任何地方session过期，则整个操作过期。如果需要这种行为，应该使用`SessionFailRetryLoop`。

#### Zookeeper不适合用作实现分布式队列

