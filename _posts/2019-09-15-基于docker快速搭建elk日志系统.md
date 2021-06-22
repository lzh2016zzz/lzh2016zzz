---
title: 基于docker快速搭建elk日志系统
tags: docker,技术,elasticsearch
---


## 目的

因为公司业务要求,目前的架构是采用了`spring-cloud`微服务架构.

无论是开发,测试,还是生产环境,都需要通过日志来排查出现问题的原因.ssh上去查询日志太麻烦了,特别是一个服务有多个实例的情况下,根本不知道报错位置在哪个实例上.只能一个个上去查找.非常浪费时间.

因此需要搭建基于`elk`的日志系统,另外,采用docker搭建elk更加方便,所以采用docker来搭建elk.

## 搭建过程

1. **镜像版本**

   ​	kibana 6.8.2

   ​	elasticsearch 6.8.2

   ​	logstash 6.8.2
   

2. **脚本**

   创建elsearch,kibana ,logstash

   ```bash
   #创建子网用于服务内部通讯
   docker network create --subnet=192.119.0.0/16 multihost
   #设置文件句柄数量防止el-search启动失败
   sysctl -w vm.max_map_count=262144
   #安装elsearch
   # --net= 加入子网 multihost 并设定当前容器在子网内的ip地址为 192.119.2.51
   docker run -d --name elasticsearch  \
              --net=multihost --ip=192.119.2.51 \
              elasticsearch:6.8.2
   #安装kibana
   #--net=multihost 加入子网,地址由任意分配
   #-e ELASTICSEARCH_URL 配置el-search地址
   #-p 5601:5601 映射端口到宿主机的5601端口
   docker run -d --name mykibana \
         --net=multihost \
         -e ELASTICSEARCH_URL=http://192.119.2.51:9200 \
         -p 5601:5601 \
         kibana:6.8.2
   #安装logstash
   #指定当前路径为basedir
   basedir=`cd $(dirname $0); pwd -P`
   #-v /$basedir:/data/conf把当前路径映射到容器内的/data/conf
   docker run  -d \
               -v "$PWD":$basedir \
               -v /$basedir:/data/conf \
               --net multihost \
               logstash:6.8.2 \
               logstash -f /data/conf/logstash.conf
   ```

   **logstash.conf**,需要跟脚本放在同一目录下:

   ```
   #logstash.conf
   
   #接收数据的端口.
   input {
        tcp{
          port => 29820 #端口
          mode => "server"
          host => "0.0.0.0"
       }
   }
   
   #在将日志发送到logstash之前会将它转换成结构化的json格式.
   filter{
       json{
           source => "message"
       }
   }
   
   output {
     elasticsearch { 
           hosts => ["127.0.0.1"]
       index => "converge_log_%{+YYYY.MM.dd}"
       document_type => "verbose" 
     }
   }
   ```



## spring如何接入elk

使用LoggerAppender.

ElkLoggerConfiguration:

```java
@Configuration
@Slf4j
@EnableConfigurationProperties({ElkLoggerConfigureProperties.class, ElkLoggerLogstashProperties.class})
public class ElkLoggerConfiguration {



    @Value("${spring.application.name}")
    private String appName;

    @Value("${server.port}")
    private String serverPort;

    @Value("${eureka.instance.instanceId}")
    private String instanceId;

    @Value("${spring.profiles.active}")
    private String profiles;

    @Autowired
    private ElkLoggerConfigureProperties configProperties;

    @Autowired
    private ElkLoggerLogstashProperties logstashProperties;

    @Autowired(required = false)
    @Qualifier("loggerContext")
    private LoggerContext loggerContext;


    @Bean("loggerContext")
    public LoggerContext loggerContext() {
        return (LoggerContext) LoggerFactory.getILoggerFactory();
    }

    @Bean("logstashAppender")
    @Conditional(NotInTestCondition.class)
    @Lazy
    public AsyncAppender addLogstashAppender() {
        log.info("Initializing logstash elk logging");
        LoggerContext context = loggerContext;
        if (loggerContext != null && configProperties.getEnable()) {
            LogstashTcpSocketAppender logstashAppender = new LogstashTcpSocketAppender();
            logstashAppender.setName(ElkConstant.ELK_APPENDER);
            logstashAppender.setContext(context);
            // 编码及message自定义字段(用于指定log来源)
            LogstashEncoder logstashEncoder = new LogstashEncoder();

            String profile = Optional.ofNullable(this.profiles).orElse(ElkConstant.UNKNOWN);
            String customFields = "{\"app_name\":\"" + appName + "\",\"app_port\":\"" + serverPort + "\"," +
                    "\"instance_id\":\"" + instanceId + "\", \"profile\":\"" + profile + "\" }";

            logstashEncoder.setCustomFields(customFields);
            logstashAppender.setEncoder(logstashEncoder);

            // logstash地址
            String destination = logstashProperties.getHost() + ":" + logstashProperties.getPort();
            logstashAppender.addDestination(destination);

            // 连接策略
            DestinationConnectionStrategy destinationConnectionStrategy = new EnvDestinationConnectionStrategy();
            logstashAppender.setConnectionStrategy(destinationConnectionStrategy);

            // 传输的日志级别
            ThresholdFilter filter = new ThresholdFilter();
            filter.setLevel(configProperties.getLogLevel());
            filter.start();

            logstashAppender.addFilter(filter);

            logstashAppender.start();

            // Wrap the appender in an Async appender for performance
            AsyncAppender asyncLogstashAppender = new AsyncAppender();
            asyncLogstashAppender.setContext(context);
            asyncLogstashAppender.setName(configProperties.getAppenderName());
            asyncLogstashAppender.setQueueSize(logstashProperties.getQueueSize());
            asyncLogstashAppender.addAppender(logstashAppender);
            asyncLogstashAppender.start();

            context.getLogger(Logger.ROOT_LOGGER_NAME).addAppender(asyncLogstashAppender);

            log.info("初始化elk日志成功");
            return asyncLogstashAppender;
        }
        return null;
    }
}
```

EnvDestinationConnectionStrategy:

```java
@Slf4j
public class EnvDestinationConnectionStrategy extends DestinationConnectionStrategyWithTtl {

    private Map<Integer, Boolean> successMap = new TreeMap<>();

    /**
     * 顺序尝试连接，若爱上一个连接，即一往情深
     *
     * @param previousDestinationIndex
     * @param numDestinations
     * @return
     */
    @Override
    public int selectNextDestinationIndex(int previousDestinationIndex, int numDestinations) {
        int nextIndex = 0;
        if (!successMap.isEmpty()) {
            Set<Map.Entry<Integer, Boolean>> set = successMap.entrySet();
            for (Map.Entry<Integer, Boolean> entry : set) {
                nextIndex = entry.getKey();
                if (entry.getValue()) {
                    nextIndex--;
                    break;
                }
            }
            nextIndex++;
        }

        if (nextIndex >= numDestinations) {
            nextIndex %= numDestinations;
            successMap.clear();
        }
        log.warn("logstash节点选择：{}->{}", previousDestinationIndex, nextIndex);
        return nextIndex;
    }

    @Override
    public void connectSuccess(long connectionStartTimeInMillis, int connectedDestinationIndex, int numDestinations) {
        log.info("logstash节点：{}，连接成功！", connectedDestinationIndex);
        successMap.put(connectedDestinationIndex, Boolean.TRUE);
    }

    @Override
    public void connectFailed(long connectionStartTimeInMillis, int failedDestinationIndex, int numDestinations) {
        log.error("logstash节点：{}，连接失败！", failedDestinationIndex);
        if (!successMap.containsKey(failedDestinationIndex)) {
            successMap.put(failedDestinationIndex, Boolean.FALSE);
        }
    }
}
```

完整代码就不贴了,这类知识,重要的是思路

