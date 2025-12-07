---
layout: post
title: logback
date: 2025-12-07 23:13 +0900
categories: [backend]
tags: [logging]
---

# Hello World

## What is logback?

Logback is intended as a sucessor to log4j project

## Requirements

`logback-classic` module requires the presence of `slf4j-api` and `logback-core`

## Hello World

```java
class HelloWorld {

    @Test
    void helloWorld() {
        // does not reference any logback classes.
        // only need to import SLF4J clases.
        var logger = LoggerFactory.getLogger(HelloWorld.class);

        logger.debug("Hello World");
    }
}
// 23:21:00.915 [main] DEBUG com.minjunk.logging.slf4j.HelloWorld -- Hello World
```

when no default configuration file is found, logback will add a `ConsoleAppender` to the root logger.

logback can report information about its internal state using a built-in status system.

important events occurring during logback's lifetime can be accessed through a component called `StatusManager`

```java
public class PrintInternalStatus {

    public static void main(String[] args) {
        var logger = LoggerFactory.getLogger(PrintInternalStatus.class);
        logger.debug("Hello World");

        var loggerContext = (LoggerContext) LoggerFactory.getILoggerFactory();
        StatusPrinter.print(loggerContext);
    }
}
```

```
23:26:15.293 [main] DEBUG com.minjunk.logging.slf4j.InternalStatus -- Hello World
23:26:15,241 |-INFO in ch.qos.logback.classic.LoggerContext[default] - This is logback-classic version 1.5.21
23:26:15,243 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - No custom configurators were discovered as a service.
23:26:15,243 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - Trying to configure with ch.qos.logback.classic.util.DefaultJoranConfigurator
23:26:15,244 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - Constructed configurator of type class ch.qos.logback.classic.util.DefaultJoranConfigurator
23:26:15,249 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
23:26:15,249 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.xml]
23:26:15,257 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - ch.qos.logback.classic.util.DefaultJoranConfigurator.configure() call lasted 6 milliseconds. ExecutionStatus=INVOKE_NEXT_IF_ANY
23:26:15,257 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - Trying to configure with ch.qos.logback.classic.BasicConfigurator
23:26:15,259 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - Constructed configurator of type class ch.qos.logback.classic.BasicConfigurator
23:26:15,259 |-INFO in ch.qos.logback.classic.BasicConfigurator@63e31ee - Setting up default configuration.
23:26:15,287 |-INFO in ch.qos.logback.core.ConsoleAppender[console] - NOTE: Writing to the console can be slow. Try to avoid logging to the 
23:26:15,287 |-INFO in ch.qos.logback.core.ConsoleAppender[console] - console in production environments, especially in high volume systems.
23:26:15,287 |-INFO in ch.qos.logback.core.ConsoleAppender[console] - See also https://logback.qos.ch/codes.html#slowConsole
23:26:15,289 |-INFO in ch.qos.logback.classic.util.ContextInitializer@3419866c - ch.qos.logback.classic.BasicConfigurator.configure() call lasted 30 milliseconds. ExecutionStatus=NEUTRAL
```

logback explains that having failed to find `logback-test.xml` and `logback.xml` configuration files

it configured itself using its default policy, which is a basic `ConsoleAppender`

an `Appender` is a class that can be seen as an **output destination**.

appenders exist for many different destinations including the console, files, Syslog, TCP Sockets, JMS and many more.

users can also easily create their own Appenders.

in case of errors, logback will automatically print its internal state on the console.

three required steps to enable logging in your application.
- configure the logback environment
- in every class where you wish ti perform logging, retrieve a `Logger` instance.
- use this logger instance by invoking its printing methods. this will produce logging output on the configured appenders.
