---
layout: post
title: spring boot日志配置logback-spring.xml
categories: Java
description: spring boot日志配置logback-spring.xml
keywords: spring boot,java
---

# logback-spring.xml
以下内容复制到logback-spring.xml即可。
```xml
<?xml version="1.0" encoding="utf-8" ?>
<configuration>
    <property name="log_path" value="./log" />

    <appender name="console" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS,Asia/Shanghai} [%thread] %-5level %logger{35} - %msg %n</pattern>
        </encoder>
    </appender>

    <appender name="file" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover -->
            <fileNamePattern>${log_path}/%d{yyyyMMdd}.log</fileNamePattern>

            <!-- keep 30 days' worth of history capped at 3GB total size -->
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>

        </rollingPolicy>

        <encoder>
            <pattern>%d{HH:mm:ss.SSS,Asia/Shanghai} [%thread] %-5level %logger{35} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="errorFile" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- daily rollover -->
            <fileNamePattern>${log_path}/%d{yyyyMMdd}.error.log</fileNamePattern>

            <!-- keep 30 days' worth of history capped at 3GB total size -->
            <maxHistory>30</maxHistory>
            <totalSizeCap>3GB</totalSizeCap>

        </rollingPolicy>

        <encoder>
            <pattern>%d{HH:mm:ss.SSS,Asia/Shanghai} [%thread] %-5level %logger{35} - %msg%n</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <filter class="ch.qos.logback.classic.filter.ThresholdFilter"><!-- 只打印错误以上日志 -->
            <level>ERROR</level>
        </filter>
    </appender>

    <root level="info">
        <appender-ref ref="console"/>
        <appender-ref ref="file"/>
        <appender-ref ref="errorFile"/>
    </root>
</configuration>
```
