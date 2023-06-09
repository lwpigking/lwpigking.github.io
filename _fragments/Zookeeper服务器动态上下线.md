---
layout: fragment
title: Zookeeper服务器动态上下线
tags: [Zookeeper]
description: some word here
keywords: Zookeeper
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package case1;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/15
 * Created by fujiahao
 */

import org.apache.zookeeper.KeeperException;
import org.apache.zookeeper.WatchedEvent;
import org.apache.zookeeper.Watcher;
import org.apache.zookeeper.ZooKeeper;

import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

/**
 * 服务器动态上下线
 * 客户端监听
 */
public class DistributeClient {

    private String connectString = "master:2181,slave1:2181,slave2:2181";
    private int sessionTimeout = 2000;
    private ZooKeeper zk;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {

        DistributeClient client = new DistributeClient();

        // 1.获取zk连接
        client.getConnect();

        // 2.监听lwPigKing下面子节点的增加和删除
        client.getlwPigKingList();

        // 3.业务逻辑（睡觉）
        client.business();

    }

    private void business() throws InterruptedException {
        Thread.sleep(Long.MAX_VALUE);
    }

    private void getlwPigKingList() throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren("/lwPigKing", true);

        ArrayList<String> lwPigKing = new ArrayList<>();
        for (String child : children) {
            byte[] data = zk.getData("/lwPigKing/" + child, false, null);
            lwPigKing.add(new String(data));
        }

        System.out.println(lwPigKing);

    }

    private void getConnect() throws IOException {
        zk = new ZooKeeper(connectString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {

            }
        });
    }
}

```

```java
package case1;

import org.apache.zookeeper.*;

import java.io.IOException;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/15
 * Created by fujiahao
 */

/**
 * 服务器动态上下线
 * 服务器注册（创建临时的带序号的节点）（内容为主机名称）
 */

public class DistributeServer {

    private String connectionString = "master:2181,slave1:2181,slave2:2181";
    private int sessionTimeout = 2000;
    private ZooKeeper zk;

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {

        DistributeServer server = new DistributeServer();

        // 1.获取zk连接
        server.getConnect();

        // 2.注册服务器到zk集群
        server.regist(args[0]);

        // 3.启动业务逻辑（睡觉）
        server.business();
    }

    private void business() throws InterruptedException {
        Thread.sleep(Long.MAX_VALUE);
    }

    // 创建节点
    private void regist(String hostname) throws KeeperException, InterruptedException {
        String s = zk.create("/lwPigKing/" + hostname, hostname.getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.EPHEMERAL_SEQUENTIAL);

        System.out.println(hostname + "is online");
    }

    // 获取连接
    private void getConnect() throws IOException {

        zk = new ZooKeeper(connectionString, sessionTimeout, new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {

            }
        });
    }
}

```

