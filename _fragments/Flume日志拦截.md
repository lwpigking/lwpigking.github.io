---
layout: fragment
title: Flume日志拦截
tags: [Flume]
description: some word here
keywords: Flume
mermaid: false
sequence: false
flow: false
mathjax: false
mindmap: false
mindmap2: false
---

```java
package interceptor;

import org.apache.flume.Event;
import org.apache.flume.interceptor.Interceptor;

import java.nio.charset.Charset;
import java.nio.charset.StandardCharsets;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

/**
 * Project:  BigDataCode
 * Create date:  2023/5/24
 * Created by fujiahao
 */

/**
 * 日志拦截
 */

public class LogTypeInterceptor implements Interceptor {
    @Override
    public void initialize() {

    }

    @Override
    public Event intercept(Event event) {
        // 区分日志类型 body header
        // 1.获取body数据
        byte[] body = event.getBody();
        String log = new String(body, StandardCharsets.UTF_8);

        // 2.获取header
        Map<String, String> headers = event.getHeaders();

        // 3.判断数据类型并向Header中赋值
        if (log.contains("start")) {
            headers.put("topic", "topic_start");
        } else {
            headers.put("topic", "topic_event");
        }

        return event;
    }

    @Override
    public List<Event> intercept(List<Event> list) {
        ArrayList<Event> interceptors = new ArrayList<>();

        for (Event event : list) {
            Event intercept1 = intercept(event);
            interceptors.add(intercept1);
        }

        return interceptors;
    }

    @Override
    public void close() {

    }
}

```

