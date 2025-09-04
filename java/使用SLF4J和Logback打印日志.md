### 使用SLF4J加Logback打印日志

---

#### 1、简介

##### SLF4J (Simple Logging Facade for Java)
- 是一个日志门面框架，提供了统一的日志API
- 不直接实现日志功能，而是作为各种日志框架的抽象层
- 允许在部署时插入所需的日志框架

##### Logback
- 是SLF4J的原生实现，由SLF4J创始人开发
- 性能优于log4j，是Spring Boot默认的日志实现
- 分为三个模块：`logback-core`、`logback-classic`、`logback-access`

#### 2、Maven依赖配置

```xml
<dependencies>
    <!-- SLF4J API -->
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
    </dependency>
    
    <!-- Logback Classic -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.2.12</version>
    </dependency>
    
    <!-- Logback Core -->
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-core</artifactId>
        <version>1.2.12</version>
    </dependency>
</dependencies>
```

#### 3、基本使用方法

##### 3.1 获取Logger实例

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogExample {
    // 方式1：根据类名获取Logger
    private static final Logger logger = LoggerFactory.getLogger(LogExample.class);
    
    // 方式2：根据字符串名称获取Logger
    private static final Logger namedLogger = LoggerFactory.getLogger("MyLogger");
    
    public void doSomething() {
        logger.info("这是一个信息日志");
        logger.debug("这是一个调试日志");
        logger.warn("这是一个警告日志");
        logger.error("这是一个错误日志");
    }
}
```

##### 3.2 日志级别

Logback支持5个日志级别（按优先级从低到高）：
- `TRACE`：最详细的日志信息
- `DEBUG`：调试信息
- `INFO`：一般信息
- `WARN`：警告信息
- `ERROR`：错误信息

```java
public class LogLevelExample {
    private static final Logger logger = LoggerFactory.getLogger(LogLevelExample.class);
    
    public void logAtAllLevels() {
        logger.trace("TRACE级别日志");
        logger.debug("DEBUG级别日志");
        logger.info("INFO级别日志");
        logger.warn("WARN级别日志");
        logger.error("ERROR级别日志");
    }
}
```

#### 4、参数化日志

##### 4.1 使用占位符

```java
public class ParameterizedLogging {
    private static final Logger logger = LoggerFactory.getLogger(ParameterizedLogging.class);
    
    public void logWithParameters() {
        String name = "张三";
        int age = 25;
        
        // 使用{}占位符
        logger.info("用户姓名：{}，年龄：{}", name, age);
        
        // 单个参数
        logger.debug("处理用户：{}", name);
        
        // 多个参数
        logger.warn("用户{}在{}岁时出现异常", name, age);
    }
}
```

##### 4.2 条件日志记录

```java
public class ConditionalLogging {
    private static final Logger logger = LoggerFactory.getLogger(ConditionalLogging.class);
    
    public void conditionalLog() {
        // 避免不必要的字符串拼接
        if (logger.isDebugEnabled()) {
            logger.debug("复杂计算结果：" + doExpensiveCalculation());
        }
        
        // 更好的方式：使用参数化日志
        logger.debug("复杂计算结果：{}", doExpensiveCalculation());
    }
    
    private String doExpensiveCalculation() {
        // 模拟耗时计算
        return "计算结果";
    }
}
```

#### 5、Logback配置文件

##### 5.1 logback.xml配置

在`src/main/resources`目录下创建`logback.xml`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- 定义日志文件存储路径 -->
    <property name="LOG_HOME" value="./logs" />
    
    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 文件输出 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_HOME}/app.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志文件名格式 -->
            <fileNamePattern>${LOG_HOME}/app.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <!-- 保留30天的日志 -->
            <maxHistory>30</maxHistory>
            <!-- 每个文件最大100MB -->
            <timeBasedFileNamingAndTriggeringPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedFNATP">
                <maxFileSize>100MB</maxFileSize>
            </timeBasedFileNamingAndTriggeringPolicy>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 异步日志配置 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <appender-ref ref="FILE" />
        <!-- 队列大小 -->
        <queueSize>512</queueSize>
        <!-- 丢弃TRACE, DEBUG, INFO级别的日志 -->
        <discardingThreshold>0</discardingThreshold>
        <!-- 关闭时是否等待队列清空 -->
        <neverBlock>true</neverBlock>
    </appender>
    
    <!-- 根日志级别和输出 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="ASYNC_FILE" />
    </root>
    
    <!-- 特定包的日志级别 -->
    <logger name="com.example.dao" level="DEBUG" />
    <logger name="org.springframework" level="WARN" />
</configuration>
```

##### 5.2 logback-spring.xml配置（Spring Boot）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <!-- Spring Boot提供的属性 -->
    <springProperty scope="context" name="appName" source="spring.application.name" defaultValue="app"/>
    
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId}] %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>
```

#### 6、高级特性

##### 6.1 MDC (Mapped Diagnostic Context)

```java
import org.slf4j.MDC;

public class MDCExample {
    private static final Logger logger = LoggerFactory.getLogger(MDCExample.class);
    
    public void logWithMDC() {
        // 添加MDC信息
        MDC.put("userId", "12345");
        MDC.put("requestId", "req-001");
        
        try {
            logger.info("开始处理用户请求");
            // 业务逻辑
            performBusinessLogic();
            logger.info("请求处理完成");
        } finally {
            // 清理MDC
            MDC.clear();
        }
    }
    
    private void performBusinessLogic() {
        logger.debug("执行业务逻辑");
    }
}
```

##### 6.2 异常日志记录

```java
public class ExceptionLogging {
    private static final Logger logger = LoggerFactory.getLogger(ExceptionLogging.class);
    
    public void logException() {
        try {
            // 可能抛出异常的代码
            int result = 10 / 0;
        } catch (Exception e) {
            // 记录异常堆栈
            logger.error("计算过程中发生异常", e);
            
            // 只记录异常信息（不包含堆栈）
            logger.error("计算过程中发生异常: {}", e.getMessage());
        }
    }
}
```

##### 6.3 自定义日志格式

```xml
<!-- 在logback.xml中定义自定义转换器 -->
<configuration>
    <conversionRule conversionWord="clr" converterClass="org.springframework.boot.logging.logback.ColorConverter" />
    
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%clr(%d{yyyy-MM-dd HH:mm:ss.SSS}){faint} %clr([%thread]){magenta} %clr(%-5level){highlight} %clr(%logger{36}){cyan} - %msg%n</pattern>
        </encoder>
    </appender>
</configuration>
```

#### 7、最佳实践

##### 7.1 Logger声明规范

```java
public class BestPracticeExample {
    // 推荐：使用类名作为Logger名称
    private static final Logger logger = LoggerFactory.getLogger(BestPracticeExample.class);
    
    // 不推荐：使用字符串字面量
    // private static final Logger logger = LoggerFactory.getLogger("MyLogger");
}
```

##### 7.2 日志级别使用建议

```java
public class LogLevelBestPractice {
    private static final Logger logger = LoggerFactory.getLogger(LogLevelBestPractice.class);
    
    public void demonstrateLogLevels() {
        // TRACE: 最详细的信息，仅在开发调试时使用
        logger.trace("进入方法：processData");
        
        // DEBUG: 调试信息，用于诊断问题
        logger.debug("处理了{}条数据记录", 100);
        
        // INFO: 一般信息，程序正常运行的关键信息
        logger.info("应用程序启动成功");
        
        // WARN: 警告信息，不影响程序运行但需要注意
        logger.warn("配置文件未找到，使用默认配置");
        
        // ERROR: 错误信息，需要立即关注的问题
        logger.error("数据库连接失败");
    }
}
```

##### 7.3 性能优化建议

```java
public class PerformanceOptimization {
    private static final Logger logger = LoggerFactory.getLogger(PerformanceOptimization.class);
    
    public void optimizedLogging() {
        // 好的做法：使用参数化日志
        logger.debug("用户{}执行了{}操作", "张三", "登录");
        
        // 避免的做法：字符串拼接
        // logger.debug("用户" + "张三" + "执行了" + "登录" + "操作");
        
        // 复杂对象的日志记录
        List<String> dataList = Arrays.asList("data1", "data2");
        if (logger.isDebugEnabled()) {
            logger.debug("处理数据列表：{}", dataList);
        }
    }
}
```

#### 8、常见问题解决

##### 8.1 多个SLF4J绑定

当出现以下警告时：
```
SLF4J: Class path contains multiple SLF4J bindings.
```

解决方法：
- 检查依赖，排除重复的SLF4J实现
- 使用Maven排除不需要的日志实现：

```xml
<dependency>
    <groupId>some-group</groupId>
    <artifactId>some-artifact</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

##### 8.2 配置文件位置

Logback会按以下顺序查找配置文件：
1. `logback-test.xml`
2. `logback-spring.xml`（Spring Boot项目）
3. `logback.xml`
4. 使用默认配置

通过系统属性指定配置文件：
```bash
-Dlogback.configurationFile=/path/to/config.xml
```