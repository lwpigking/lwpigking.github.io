---
layout: post
title: Prometheus集成Flink
categories: [Prometheus, Flink]
description: Prometheus, Flink
keywords: Prometheus, Flink
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

拷贝新的flink目录，叫做flink-prometheus（在此不演示）

拷贝flink的jar包

```
cp /opt/module/flink-prometheus/plugins/metrics-prometheus/flink-metrics-prometheus-1.12.0.jar /opt/module/flink-prometheus/lib/
```

进入到flink的conf目录，修改flink-conf.yaml

添加如下配置

```
##### 与 Prometheus 集成配置 #####
metrics.reporter.promgateway.class: 
org.apache.flink.metrics.prometheus.PrometheusPushGatewayReporter
# PushGateway 的主机名与端口号
metrics.reporter.promgateway.host: hadoop202
metrics.reporter.promgateway.port: 9091
# Flink metric 在前端展示的标签（前缀）与随机后缀
metrics.reporter.promgateway.jobName: flink-metrics-ppg
metrics.reporter.promgateway.randomJobNameSuffix: true
metrics.reporter.promgateway.deleteOnShutdown: false
metrics.reporter.promgateway.interval: 30 SECONDS
```

测试一下，启动网猫！

```
nc -lk 9999
```

提交任务

```
[atguigu@hadoop202 flink-prometheus]$ bin/flink run -t yarn-per-job -c 
com.atguigu.flink.chapter02.Flink03_WordCount_UnboundStream ./flink-base-1.0-SNAPSHOT-jar-with-dependencies.jar
```

