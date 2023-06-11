---
layout: post
title: Zookeeper服务器动态上下线
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

