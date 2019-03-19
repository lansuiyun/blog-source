---
title: Zookeeper原生java客户端
date: 2019-03-19 16:11:48
tags: [Zookeeper]
categories: 分布式
---
- maven依赖
```
<dependency>
  <groupId>org.apache.zookeeper</groupId>
  <artifactId>zookeeper</artifactId>
  <version>3.4.13</version>
</dependency>
```
<!-- more -->
- 创建`Zookeeper`对象
```
ZooKeeper zooKeeper = new ZooKeeper("localhost:2187", 5000, watch);
//集群
ZooKeeper zooKeeper = new ZooKeeper("localhost:2187,localhost:2188,localhost:2189", 5000,
         watch);
//指定跟路径，操作/test时，其真实路径为/root/test
ZooKeeper zooKeeper = new ZooKeeper("localhost:2187,localhost:2188,localhost:2189/root", 5000,
            totalWatch);
```
- Watcher
实现`Watcher`接口
- 检查节点是否存在
```
//使用默认watch，即构建Zookeeper中的watch
Stat exists = zooKeeper.exists(TOTAL_PATH, true);
//使用新的watch
Stat exists = zooKeeper.exists(TOTAL_PATH, new XxxWatch());
```
- 创建节点
```
String createdPath = zooKeeper.create(PATH, "TEST".getBytes(Charsets.UTF_8), Ids.CREATOR_ALL_ACL,
                    CreateMode.EPHEMERAL);
```
- 获取节点数据
```
Stat stat = new Stat();
byte[] dataBytes = zooKeeper.getData(PATH, true, stat);
//byte[] dataBytes = zooKeeper.getData(PATH, new XxxWatch(), stat);
String data = new String(dataBytes, Charsets.UTF_8);
```
- 删除节点
```
zooKeeper.delete(PATH, stat.getVersion());
```
- 修改节点数据
```
zooKeeper.setData(PATH, "UPDATE".getBytes(Charsets.UTF_8), stat.getVersion());
```
- 获取子节点
```
List<String> children = zooKeeper.getChildren("/", false);
```
- 完整实例
```
package com.fei.curator;

import com.google.common.base.Charsets;
import java.util.ArrayList;
import java.util.List;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooDefs.Ids;
import org.apache.zookeeper.ZooDefs.Perms;
import org.apache.zookeeper.ZooKeeper;
import org.apache.zookeeper.data.ACL;
import org.apache.zookeeper.data.Id;
import org.apache.zookeeper.data.Stat;
import org.apache.zookeeper.server.auth.DigestAuthenticationProvider;

/**
 * ZookeeperTest
 *
 * @author fei
 */
public class ZookeeperTest implements Watcher {

    private static final String PATH = "/test";
    private static final String PATH2 = "/test2";

    public static void main(String[] args) throws Exception {
        ZookeeperTest watch = new ZookeeperTest();
        //构建Zookeeper，以/root为根路径
        ZooKeeper zooKeeper = new ZooKeeper("localhost:2187,localhost:2188,localhost:2189/root",
            5000, watch);
        //ACL验证信息
        zooKeeper.addAuthInfo("digest", "root:123".getBytes(Charsets.UTF_8));
        //检查是否存在
        Stat exists = zooKeeper.exists(PATH, true);
        System.out.println(PATH + " 是否存在：" + (exists != null));
        if (exists == null) {
            String createdPath = zooKeeper.create(PATH, "TEST".getBytes(Charsets.UTF_8),
                Ids.CREATOR_ALL_ACL, CreateMode.PERSISTENT);
            System.out.println("创建节点：" + createdPath);
        }
        //获取节点数据
        Stat stat = new Stat();
        byte[] dataBytes = zooKeeper.getData(PATH, true, stat);
        String data = new String(dataBytes, Charsets.UTF_8);
        System.out.println("节点数据为：" + data);
        System.out.println("节点版本号：" + stat.getVersion() + " ,状态为：" + stat);
        //更新节点数据
        zooKeeper.setData(PATH, "UPDATE".getBytes(Charsets.UTF_8), stat.getVersion());
        dataBytes = zooKeeper.getData(PATH, true, stat);
        data = new String(dataBytes, Charsets.UTF_8);
        System.out.println("更新后节点数据为：" + data);
        System.out.println("更新后节点版本号：" + stat.getVersion() + " ,状态为：" + stat);

        // 自定义权限
        List<ACL> acls = new ArrayList<>(1);
        Id digest = new Id("digest", DigestAuthenticationProvider.generateDigest("root:123"));
        acls.add(new ACL(Perms.READ | Perms.WRITE, digest));
        //创建/test2为顺序节点，对root可读可写.
        String node2 = zooKeeper.create(PATH2, "TEST2".getBytes(Charsets.UTF_8), acls,
            CreateMode.PERSISTENT_SEQUENTIAL);
        //获取子节点
        List<String> children = zooKeeper.getChildren("/", false);
        System.out.println("children: " + children);
        //删除节点
        zooKeeper.delete(PATH, stat.getVersion());
        zooKeeper.getData(node2, false, stat);
        zooKeeper.delete(node2, stat.getVersion());

        //验证是否删除成功
        exists = zooKeeper.exists(PATH, false);
        System.out.println(PATH + " 是否删除成功：" + (exists == null));
        exists = zooKeeper.exists(node2, false);
        System.out.println(node2 + " 是否删除成功：" + (exists == null));
    }

    @Override
    public void process(WatchedEvent event) {
        System.out.println(event.toString());
    }
}
```



