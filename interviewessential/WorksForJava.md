# Works For Java

# Exception

***

利用接口统一指定标准 比如状态码 和 消息

利用不同业务类型的枚举实现接口 从而快速实现相关逻辑

然后自定义异常来处理相关逻辑;

```java
public interface ErrorCode{
    String getCode();
    String getMessage();
}
```

```java
public enum TradeErrorCode implements ErrorCode{


    private String code;

    private String message;

    public TradeErrorCode(String code,String message){
        this.code = code;
        this.message = message;
    }

        @Override
        public String getCode() {
            return this.code;
        }
    
        @Override
        public String getMessage() {
            return this.message;
        }

}
```

然后接着: 自定义异常处理:

```java
public class BizException extends RuntimeException{
        //引入该接口
       private ErrorCode errorCode;


        public BizException(ErrorCode errorCode) {
        super(errorCode.getMessage());
        this.errorCode = errorCode;
    }


    public BizException(String message, ErrorCode errorCode) {
        super(message);
        this.errorCode = errorCode;
    }

}
```

***

# logger

***

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <include resource="org/springframework/boot/logging/logback/defaults.xml"/>

  <springProperty scope="context" name="app.name" source="spring.application.name"/>
  <!-- 基于 converter -->
  <conversionRule conversionWord="sensitive" converterClass="com.github.houbb.sensitive.logback.converter.SensitiveLogbackConverter" />

  <property name="APP_NAME" value="${app.name}"/>
  <property name="LOG_PATH" value="${user.home}/${APP_NAME}/logs"/>
  <property name="LOG_FILE" value="${LOG_PATH}/application.log"/>
  <property name="FILE_LOG_PATTERN" value="%d %-5level [%thread] %logger - %sensitive{%msg}%n"/>
  <property name="CONSOLE_LOG_PATTERN" value="%d %-5level [%thread] %logger - %sensitive{%msg}%n"/>

  <appender name="APPLICATION" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${LOG_FILE}</file>
    <encoder>
      <pattern>${FILE_LOG_PATTERN}</pattern>
    </encoder>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
      <fileNamePattern>${LOG_FILE}.%d{yyyy-MM-dd}.%i.log</fileNamePattern>
      <maxHistory>7</maxHistory>
      <maxFileSize>50MB</maxFileSize>
      <totalSizeCap>20GB</totalSizeCap>
    </rollingPolicy>
  </appender>

  <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
    <encoder>
      <pattern>${CONSOLE_LOG_PATTERN}</pattern>
      <charset>utf8</charset>
    </encoder>
  </appender>

  <root level="INFO">
    <!--   输出到日志文件    -->
    <appender-ref ref="APPLICATION"/>
    <!--   输出到控制台    -->
    <appender-ref ref="CONSOLE"/>
  </root>
</configuration>
```

***

# Enum

***

获取枚举实例集合: Enum.values()

获取枚举实例: Enum.valuesOf(String name);

获取枚举实例的名称(字符串) Enum.valuesOf(String name).name();

可以定义方法

***

Test

***

```java

docker run -d \
  --name nacos-server \
  -p 8848:8848 \
  -p 9848:9848 \
  -e MODE=standalone \
  -e SPRING_DATASOURCE_PLATFORM=mysql \
  -e MYSQL_SERVICE_HOST=192.168.1.100 \    # ← 替换为你的内网IP
  -e MYSQL_SERVICE_PORT=3306 \
  -e MYSQL_SERVICE_USER=root \
  -e MYSQL_SERVICE_PASSWORD=xiaoxu \
  -e MYSQL_SERVICE_DB_NAME=nacos_config \
  nacos/nacos-server:v2.4.1
```

配置中的变量的优先级:

1. 命令行参数 - 最高优先级
2. 测试类上的@ActiveProfiles注解 - 在测试环境中具有较高优先级
3. 系统属性（System.getProperties()）
4. 操作系统环境变量
5. application-{profile}.properties/yml文件
6. application.properties/yml文件中的配置 - 最低优先级

***

异常情况:

`NoSuchBeanDefinitionException` 一般出现的场景: 就是特定的bean没有被激活

在测试中学会的注解:

`@ActiveProfiles("prod")` 在测试类中启动prod级别的激活配置吧

`@TestPropertySource(properties={"xxxx"})` 相当于在开启一些写的配置环境测试喽

`@EnableConfigurationProperties(OssProperties.class)`

```plain
@ActiveProfiles("prod")
@TestPropertySource(properties = {"spring.oss.enabled=true"})
```

# JVM

对象初始化流程:

```java
对象创建过程：
分配内存空间
初始化对象属性为默认值（如null、0等）
执行构造方法
返回堆中的实例引用
```

XXL-job

标注在public 方法上面 void 返回类型 可以使用在@Controller 方法上

Function.identity()方法输出一个 恒等的函数

```plain
public class SpringExpressionUtils {

    /**
     * Spring EL 表达式解析器
     */
    private static final ExpressionParser EXPRESSION_PARSER = new SpelExpressionParser();
    /**
     * 参数名发现器
     */
    private static final ParameterNameDiscoverer PARAMETER_NAME_DISCOVERER = new DefaultParameterNameDiscoverer();

    private SpringExpressionUtils() {
    }

    /**
     * 从切面中，单个解析 EL 表达式的结果
     *
     * @param joinPoint        切面点
     * @param expressionString EL 表达式数组
     * @return 执行界面
     */
    public static Object parseExpression(JoinPoint joinPoint, String expressionString) {
        Map<String, Object> result = parseExpressions(joinPoint, Collections.singletonList(expressionString));
        return result.get(expressionString);
    }

    /**
     * 从切面中，批量解析 EL 表达式的结果
     *
     * @param joinPoint         切面点
     * @param expressionStrings EL 表达式数组
     * @return 结果，key 为表达式，value 为对应值
     */
    public static Map<String, Object> parseExpressions(JoinPoint joinPoint, List<String> expressionStrings) {
        // 如果为空，则不进行解析
        if (CollUtil.isEmpty(expressionStrings)) {
            return MapUtil.newHashMap();
        }

        // 第一步，构建解析的上下文 EvaluationContext
        // 通过 joinPoint 获取被注解方法
        MethodSignature methodSignature = (MethodSignature) joinPoint.getSignature();
        Method method = methodSignature.getMethod();
        // 使用 spring 的 ParameterNameDiscoverer 获取方法形参名数组
        String[] paramNames = PARAMETER_NAME_DISCOVERER.getParameterNames(method);
        // Spring 的表达式上下文对象
        EvaluationContext context = new StandardEvaluationContext();
        // 给上下文赋值
        if (ArrayUtil.isNotEmpty(paramNames)) {
            Object[] args = joinPoint.getArgs();
            for (int i = 0; i < paramNames.length; i++) {
                context.setVariable(paramNames[i], args[i]);
            }
        }

        // 第二步，逐个参数解析
        Map<String, Object> result = MapUtil.newHashMap(expressionStrings.size(), true);
        expressionStrings.forEach(key -> {
            Object value = EXPRESSION_PARSER.parseExpression(key).getValue(context);
            result.put(key, value);
        });
        return result;
    }

    /**
     * 从 Bean 工厂，解析 EL 表达式的结果
     *
     * @param expressionString EL 表达式
     * @return 执行界面
     */
    public static Object parseExpression(String expressionString) {
        return parseExpression(expressionString, null);
    }

    /**
     * 从 Bean 工厂，解析 EL 表达式的结果
     *
     * @param expressionString EL 表达式
     * @param variables        变量
     * @return 执行界面
     */
    public static Object parseExpression(String expressionString, Map<String, Object> variables) {
        if (StrUtil.isBlank(expressionString)) {
            return null;
        }
        Expression expression = EXPRESSION_PARSER.parseExpression(expressionString);
        StandardEvaluationContext context = new StandardEvaluationContext();
        context.setBeanResolver(new BeanFactoryResolver(SpringUtil.getApplicationContext()));
        if (MapUtil.isNotEmpty(variables)) {
            context.setVariables(variables);
        }
        return expression.getValue(context);
    }

}
```


> 更新: 2026-01-12 11:17:35  
> 原文: <https://www.yuque.com/alice-hv75k/mtczog/lyecoanucv4an9te>