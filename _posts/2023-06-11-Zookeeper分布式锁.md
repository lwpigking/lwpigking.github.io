---
layout: post
title: Zookeeper分布式锁
categories: [Zookeeper]
description: Zookeeper
keywords: Zookeeper
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package case2;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/15
 * Created by fujiahao
 */

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.Collection;
import java.util.Collections;
import java.util.Currency;
import java.util.List;
import java.util.concurrent.CountDownLatch;

/**
 * 分布式锁案例
 * 分布式锁：所有节点可以访问同一个资源
 * 按照序列号大小依次访问，一个节点访问
 * 后枷锁，让其他节点无法访问，使用完资
 * 源后释放掉该锁，让下一个节点来进行访问
 */

public class DistributedLock {

    private final String connectionString = "master:2181,slave1:2181,slave2:2181";
    private final int sessionTimeout = 2000;
    private final ZooKeeper zk;

    // 连接
    private CountDownLatch connectLatch = new CountDownLatch(1);
    private CountDownLatch waitLatch = new CountDownLatch(1);

    private String waitPath;
    private String currentMode;

    public DistributedLock() throws IOException, InterruptedException, KeeperException {

        // 获取连接
        zk = new ZooKeeper(connectionString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                // connectLatch：如果连接上zk，可以释放
                if (watchedEvent.getState() == Event.KeeperState.SyncConnected) {
                    connectLatch.countDown();
                }

                // waitLatch：需要释放
                if (watchedEvent.getType() == Event.EventType.NodeDeleted && watchedEvent.getPath().equals(waitPath)) {
                    waitLatch.countDown();
                }
            }
        });

        // 等待zk正常连接后，往下走程序
        connectLatch.await();

        // 判断根节点/locks是否存在
        Stat stat = zk.exists("/locks", false);
        if (stat == null) {
            // 创建跟节点
            zk.create("/locks", "locks".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        }

    }

    // 对zk加锁
    public void zkLock() {
        // 创建对应的临时带序号的节点
        try {
            currentMode = zk.create("/locks/" + "seq-", null, ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

            // 判断创建的节点是否是最小的序号节点
            // 如果是，获取到锁
            // 如果不是，监听序号前一个节点
            List<String> children = zk.getChildren("/locks", false);

            // 如果children只有一个锁，那就直接获取锁；如果有多个节点，则判断谁最小；
            if (children.size() == 1) {
                return;
            } else {
                Collections.sort(children);

                // 获取节点名称 seq-0000000
                String thisNode = currentMode.substring("/locks/".length());
                // 通过seq-0000000获取该节点在children集合的位置
                int index = children.indexOf(thisNode);

                // 判断
                if (index == -1) {
                    System.out.println("数据异常");
                } else if (index == 0) {
                    // 就一个节点，可以获取锁
                    return;
                } else {
                    // 需要监听前一个节点变化
                    waitPath = "/locks/" + children.get(index - 1);
                    zk.getData(waitPath, true, null);

                    // 等待监听
                    waitLatch.await();
                }
            }

        } catch (KeeperException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }

    }

    // 对zk解锁
    public void unZkLock() {
        // 删除节点
        try {
            zk.delete(currentMode, -1);
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (KeeperException e) {
            e.printStackTrace();
        }
    }

}

```

