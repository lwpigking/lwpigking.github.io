---
layout: fragment
title: Zookeeper的JavaApi
tags: [tag1, tag2]
description: some word here
keywords: keyword1, keyword2
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package zkClient;
/**
 * Project:  BigDataCode
 * Create date:  2023/5/15
 * Created by fujiahao
 */

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;
import org.junit.Before;
import org.junit.Test;

import java.io.IOException;
import java.util.List;

/**
 * zookeeper-java-api
 */
public class zkClient {

    // master:2181、slave1:2181、slave2:2181三个逗号之间不能有空格
    private String connectString = "master:2181,slave1:2181,slave2:2181";
    private int sessionTimeout = 2000;
    // ctrl + alt + f：全局变量zkClient
    private ZooKeeper zkClient;

    // 连接Zookeeper集群
    @Before
    public void init() throws IOException {

        zkClient = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                System.out.println("=========================");
                List<String> children = null;
                try {
                    children = zkClient.getChildren("/", true);
                    for (String child : children) {
                        System.out.println(child);
                    }
                    System.out.println("=========================");
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
    }

    // 创建节点
    @Test
    public void create() throws KeeperException, InterruptedException {
        String nodeCreated = zkClient.create("/lwPigKing", "lwPigKing.avi".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
    }

    // 监听节点
    @Test
    public void getChildren() throws KeeperException, InterruptedException {
        List<String> children = zkClient.getChildren("/", true);
        for (String child : children) {
            System.out.println(child);
        }
        // 延时
        Thread.sleep(Long.MAX_VALUE);
    }

    // 判断节点是否存在
    @Test
    public void exist() throws KeeperException, InterruptedException {
        Stat stat = zkClient.exists("/lwPigKing", false);
        System.out.println(stat == null? "not exist" : "exist");
    }
}

```

