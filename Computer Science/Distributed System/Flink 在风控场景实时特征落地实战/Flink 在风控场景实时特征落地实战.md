#flink #distributed #software-engineering #java 
### 背景介绍

#### 风控简介

二十一世纪，信息化时代到来，互联网行业的发展速度远快于其他行业。一旦商业模式跑通，有利可图，资本立刻蜂拥而至，助推更多企业不断的入场进行快速的复制迭代，企图成为下一个“行业领头羊”。  
​

带着资本入场的玩家因为不会有资金的压力，只会更多的关注业务发展，却忽略了业务上的风险点。强大如拼多多也被“薅羊毛”大军光顾损失千万。  
​

风控，即风险管理（risk management），是一个管理过程，包括对风险的定义、测量、评估和应对风险的策略。目的是将可避免的风险、成本及损失极小化[1]。

#### 特征平台简介

互联网企业每时每刻都面临着黑灰产的各种攻击。业务安全团队需要事先评估业务流程中有风险的地方，再设置卡点，用来采集相关业务信息，识别当前请求是否有风险。专家经验（防控策略）就是在长期以往的对抗中产生的。  
​

策略的部署需要一个个特征来支持，那什么是特征？  
特征分为基础型特征、衍生型特征、统计型特征等，举例如下：

- 基础型特征：可以直接从业务获取的，如订单的金额、买家的手机号码、买家地址、卖家地址等
- 衍生特征：需要二次计算，如买家到买家的距离、手机号前3位等
- 统计型特征：需要实时统计的，如5分钟内某手机号下购买订单数、10分钟内购买金额大于2w元订单数等

​

随着业务的迅猛发展，单纯的专家经验已不能满足风险识别需求，算法团队的加入使得拦截效果变得更加精准。算法部门人员通过统一算法工程框架，解决了模型和特征迭代的系统性问题，极大地提升了迭代效率。  
​

根据功能不同，算法平台可划分为三部分：模型服务、模型训练和特征平台。其中，模型服务用于提供在线模型预估，模型训练用于提供模型的训练产出，特征平台则提供特征和样本的数据支撑。本文将重点阐述特征平台在建设过程中实时计算遇到的挑战以及优化思路。  
​

### 挑战与方案

#### 面临的挑战

业务发展的初期，我们可以通过硬编码的方式满足策略人员提出的特征需求，协同也比较好。但随着业务发展越来越快，业务线越来越多，营销玩法越来越复杂，用户数和请求量成几何倍上升。适用于早期的硬编码方式出现了策略分散无法管理、逻辑同业务强耦合、策略更新迭代率受限于开发、对接成本高等多种问题。此时，我们急需一套线上可配置、可热更新、可快速试错的特征管理平台。

#### 旧框架的不足

##### 实时框架1.0：基于 Flink DataStream API构建

如果你熟悉 Flink DataStream API，那你肯定会发现 Flink 的设计天然满足风控实时特征计算场景，我们只需要简单的几步即可统计指标，如下图所示：
![[Pasted image 20241224191713.png]]
Flink DataStream 流图  
​

实时特征统计样例代码如下：

```java
// 数据流，如topic
DataStream<ObjectNode> dataStream = ...

SingleOutputStreamOperator<AllDecisionAnalyze> windowOperator = dataStream
                // 过滤
                .filter(this::filterStrategy)
                // 数据转换
                .flatMap(this::convertData)
                // 配置watermark
                .assignTimestampsAndWatermarks(timestampAndWatermarkAssigner(config))
                // 分组
                .keyBy(this::keyByStrategy)
                // 5分钟滚动窗口
                .window(TumblingEventTimeWindows.of(Time.seconds(300)))
                // 自定义聚合函数，内部逻辑自定义
                .aggregate(AllDecisionAnalyzeCountAgg.create(), AllDecisionAnalyzeWindowFunction.create());
```
1.0框架不足：

- 特征强依赖开发人员编码，简单的统计特征可以抽象，稍微复杂点就需要定制
- 迭代效率低，策略提需求、产品排期、研发介入、测试保障、一套流程走完交付最少也是两周
- 特征强耦合，任务拆分难，一个 JOB 包含太多逻辑，可能新上的特征逻辑会影响之前稳定的指标

总的来说，1.0在业务初期很适合，但随着业务发展，研发速度逐渐成为瓶颈，不符合可持续、可管理的实时特征清洗架构。  
​

##### 实时框架2.0：基于 Flink SQL 构建

1.0架构的弊端在于需求到研发采用不同的语言体系，如何高效的转化需求，甚至是直接让策略人员配置特征清洗逻辑直接上线？如果按照两周一迭代的速度，可能线上早被黑灰产薅的“面目全非”了。  
​

此时我们研发团队注意到 Flink SQL，SQL 是最通用的数据分析语言，数分、策略、运营基本必备技能，可以说 SQL 是转换需求代价最小的实现方式之一。  
​

看一个 Flink SQL 实现示例：
```java
-- error 日志监控
-- kafka source
CREATE TABLE rcp_server_log (
    thread varchar,
    level varchar,
    loggerName varchar,
    message varchar,
    endOfBatch varchar,
    loggerFqcn varchar,
    instant varchar,
    threadId varchar,
    threadPriority varchar,
    appName varchar,
    triggerTime as LOCALTIMESTAMP,
    proctime as PROCTIME(),

    WATERMARK FOR triggerTime AS triggerTime - INTERVAL '5' SECOND
) WITH (
    'connector.type' = 'kafka',
    'connector.version' = '0.11',
    'connector.topic' = '${sinkTopic}',
    'connector.startup-mode' = 'latest-offset',
    'connector.properties.group.id' = 'streaming-metric',
    'connector.properties.bootstrap.servers' = '${sinkBootstrapServers}',
    'connector.properties.zookeeper.connect' = '${sinkZookeeperConnect}}',
    'update-mode' = 'append',
    'format.type' = 'json'
);

-- 此处省略 sink_feature_indicator 创建，参考 source table
-- 按天 按城市 各业务线决策分布
INSERT INTO sink_feature_indicator
SELECT
    level,
    loggerName,
    COUNT(*)
FROM rcp_server_log
WHERE
    (level <> 'INFO' AND `appName` <> 'AppTestService')
    OR loggerName <> 'com.test'
GROUP BY
    TUMBLE(triggerTime, INTERVAL '5' SECOND),
    level,
    loggerName;
```

我们在开发 Flink SQL 支持平台过程中，遇到如下问题：

- 一个 SQL 如果清洗一个指标，那么数据源将极大浪费
- SQL merge，即一个检测如果同源 SQL 则进行合并，此时将极大增加作业复杂度，且无法定义边界
- SQL 上线需要停机重启，此时如果任务中包含大量稳定指标，会不会是临界点
### 技术实现

#### 痛点总结
![[Pasted image 20241224191805.png]]
#### 实时计算架构

策略/算法人员每天需要观测实时和离线数据分析线上是否存在风险，针对有风险的场景，会设计防控策略，透传到研发侧其实就是一个个实时特征的开发。所以实时特征的上线速度、质量交付、易用性完全决定了线上风险场景能否及时堵漏的关键。  
​

在统一实时特征计算平台构建之前，实时特征的产出上主要有以下问题：

- 交付速度慢，迭代开发：策略提出到产品，再到研发，提测，在上线观测是否稳定，速度奇慢
- 强耦合，牵一发动全身：怪兽任务，包含很多业务特征，各业务混在一起，没有优先级保证
- 重复性开发：由于没有统一的实时特征管理平台，很多特征其实已经存在，只是名字不一样，造成极大浪费

平台话建设，最重要的是“**整个流程的抽象”**，平台话的目标应该是能用、易用、好用。基于如上思想，我们尝提取实时特征研发痛点：**模板化 + 配置化**，即平台提供一个实时特征的创建模板，用户基于该模板，可以通过简单的配置即可生成自己需要的实时特征。

![[Pasted image 20241224191816.png]]

Flink 实时计算架构图

##### 计算层

**数据源清洗：**不同数据源抽象 Flink Connector，标准输出供下游使用  
**数据拆分：**1拆N，一条实时消息可能包含多种消息，此时需要数据裂变  
**动态配置：**允许在不停机 JOB 情况下，动态更新或新增清洗逻辑，涉及特征的清洗逻辑下发  
**脚本加载：**Groovy 支持，热更新  
**RTC:** 即 Real-Time Calculate，实时特征计算，高度抽象的封装模块  
**任务感知：**基于特征业务域、优先级、稳定性，隔离任务，业务解耦

##### 服务层

**统一查询SDK:** 实时特征统一查询SDK，屏蔽底层实现逻辑
基于统一的 Flink 实时计算架构，我们重新设计了实时特征清洗架构

![[Pasted image 20241224191831.png]]

Flink 实时计算数据流图

##### 特征配置化 & 存储/读取

特征底层的存储应该是“原子性”的，即最小不可分割单位。为何如此设计？实时统计特征是和窗口大小挂钩的，不同策略人员防控对特征窗口大小有不同的要求，举例如下：

- 可信设备判定场景：其中当前手机号登录时长窗口应适中，不宜过短，防扰动
- 提现欺诈判定场景：其中当前手机号登录时长窗口应尽量短，短途快速提现的，结合其它维度，快速定位风险

​

基于上述，急需一套通用的实时特征读取模块，满足策略人员任意窗口需求，同时满足研发人员快速的配置清洗需求。我们重构后特征配置模块如下：
![[Pasted image 20241224191842.png]]

实时特征模块：

- 特征唯一标识
- 特征名称
- 是否支持窗口：滑动、滚动、固定大小窗口
- 事件切片单位：分钟、小时、天、周
- 主属性：即分组列，可以多个
- 从属性：聚合函数使用，如去重所需输入基础特征

​

业务留给风控的时间不多，大多数场景在 100 ms 以内，实时特征获取就更短了，从以往的研发经验看，RT 需要控制在 10 ms 以内，以确保策略执行不会超时。所以我们的存储使用 Redis，确保性能不是瓶颈。
![[Pasted image 20241224191851.png]]
如上述，实时特征计算模块强依赖于上游消息内传递的“主属性” 和 “从属性”，此阶段也是研发需要介入的地方，如果消息内主属性字段不存在，则需要研发补全，此时不得不加入代码的发版，那又会回到原始阶段面临的问题：Flink Job 需要不停的重启，这显然是不能接受的。  
此时我们想到了 Groovy，能否让 Flink + Groovy，直接热部署代码？答案是肯定的！  
​

由于我们抽象了整个 Flink Job 的计算流图，算子本身是不需要变更的，即 DAG 是固定不变的，变得是算子内部关联事件的清洗逻辑。所以，只要关联清洗逻辑和清洗代码本身变更，即不需要重启 Flink Job 完成热部署。

![[Pasted image 20241224191904.png]]
研发或策略人员在管理后台（Operating System）添加清洗脚本，并存入数据库。Flink Job 脚本缓存模块此时会感知脚本的新增或修改（如何感知看下文整体流程详解）

- warm up：脚本首次运行较耗时，首次启动或者缓存更新时提前预热执行，保证真实流量进入脚本快速执行
- cache：缓存已经在好的 Groovy 脚本
- Push/Poll：缓存更新采用推拉两种模式，确保信息不回丢失
- router：脚本路由，确保消息能寻找到对应脚本并执行
脚本加载核心代码：
```java
    // 缓存，否则无限加载下去会 metaspace outOfMemory
    private final static Map<String, GroovyObject> groovyObjectCache = new ConcurrentHashMap<>();

    /**
     * 加载脚本
     * @param script
     * @return
     */
    public static GroovyObject buildScript(String script) {
        if (StringUtils.isEmpty(script)) {
            throw new RuntimeException("script is empty");
        }

        String cacheKey = DigestUtils.md5DigestAsHex(script.getBytes());
        if (groovyObjectCache.containsKey(cacheKey)) {
            log.debug("groovyObjectCache hit");
            return groovyObjectCache.get(cacheKey);
        }

        GroovyClassLoader classLoader = new GroovyClassLoader();
        try {
            Class<?> groovyClass = classLoader.parseClass(script);
            GroovyObject groovyObject = (GroovyObject) groovyClass.newInstance();
            classLoader.clearCache();

            groovyObjectCache.put(cacheKey, groovyObject);
            log.info("groovy buildScript success: {}", groovyObject);
            return groovyObject;
        } catch (Exception e) {
            throw new RuntimeException("buildScript error", e);
        } finally {
            try {
                classLoader.close();
            } catch (IOException e) {
                log.error("close GroovyClassLoader error", e);
            }
        }
    }
```

##### 标准消息 & 清洗流程

策略需要统计的消息维度很杂，涉及多个业务，研发本身也有监控用到的实时特征需求。所以实时特征对应的数据源是多种多样的。所幸 Flink 是支持多种数据源接入的，对于一些特定的数据源，我们只需要继承实现 Flink Connector 即可满足需求，我将拿 Kafka举例，整体流程是如何清洗实时统计特征的。  
​

首先介绍风控整体数据流，多个业务场景对接风控中台，风控内部核心链路是：决策引擎、规则引擎、特征服务。  
一次业务请求决策，我们会异步记录下来，并发送Kafka消息，用于实时特征计算 & 离线埋点。

![[Pasted image 20241224191930.png]]

###### 标准化消息模板

Flink 实时计算 Job 在接收到 MQ 消息后，首先是消息模板标准化解析，不同的 Topic 对应消息格式不一致，JSON、CSV、异构（如错误日志类消息，空格隔断，对象内包含 JSON 对象）等。  
​

为方便下游算子统一处理，标准化后消息结构如下 JSON 结构：

```java
public class RcpPreProcessData {

    /**
     * 渠道，可以直接写topic即可
     */
    private String channel;

    /**
     * 消息分类 channel + eventCode 应唯一确定一类消息
     */
    private String eventCode;

    /**
     * 所有主从属性
     */
    private Map<String, Object> featureParamMap;

    /**
     * 原始消息
     */
    private ObjectNode node;
    
}

```

###### 消息裂变

一条“富消息”可能包含大量的业务信息，某些实时特征可能需要分别统计。举例，一条业务请求风控的上下文消息，包含本次消息是否拒绝，即命中了多少策略规则，命中的规则是数组，可能包含多条命中规则。此时如果想基于一条命中的规则去关联其它属性统计，就需要用到消息的裂变，由1变N。  
​

消息裂变的逻辑由运营后台通过 Groovy 脚本编写，定位清洗脚本逻辑则是 channel（父） + eventCode（子），此处寻找逻辑分“父子”，“父”逻辑对当前 channel 下所有逻辑适用，避免单独配置 N 个 eventCode 的繁琐，“子”逻辑则对特定的eventCode适用。  
​

###### 消息清洗 & 剪枝

消息的清洗就是我们需要知道特征需要哪些主从属性，带着目的清洗更清晰，定位清洗的脚本同上，依然依据 channel + eventCode 实现。清洗出的主从属性存在 featureParamMap 中，供下游实时计算使用。  
​

此处需要注意的是，我们一直是带着原始消息向下传递的，但如果已经确认了清洗的主从属性，那么原始消息就没有存在的必要了，此时我们需要“剪枝”，节省 RPC 调用过程 I/O 流量的消耗。  
​

至此，一条原始消息已经加工成只包含 channel（渠道）、eventCode（事件类型）、featureParamMap（所有主从属性），下游算子只需要且仅需要这些信息即可计算。  
​

###### 实时计算

依然同上面两个算子，实时计算算子依赖 channel + eventCode 查找到对应实时特征元数据，一个事件可能存在多个实时特征配置，运营平台填写好实时特征配置后，依据缓存更新机制，快速分发到任务中，依据 Key构造器 生成对应的 Key，传递下游直接 Sink 到 Redis中。  
​

### 任务问题排查&调优思路

任务的排查是基于完善的监控上实现的，Flink 提供了很多有用的 Metric 供我们排查问题，如下是我罗列的常见的任务异常，希望对你有所帮助。

#### TaskManager Full GC 问题排查

出现上面这个异常的可能原因是因为：

- 大窗口：90% TM 内存爆表，都是大窗口导致的
- 内存泄漏：如果是自定义节点，且涉及到缓存等很容易导致内存膨胀

解决办法：

- 合理制定窗口导线，合理分配 TM 内存（1.10默认是1G），聚合数据应交由 Back State 管理，不建议自己写对象存储
- 可 attach heap 快照排查异常，分析工具如 MAT，需要一定的调优经验，也能快速定位问题

​

#### Flink Job 反压

出现上面这个异常的可能原因是因为：

- 数据倾斜：90%的反压，一定是数据倾斜导致的
- 并行度并未设置好，错误估计数据流量或单个算子计算性能

​

解决办法：

- 数据清洗参考下文
- 对于并行度，可以在消息传递过程中埋点，看各个节点cost

​

#### 数据倾斜

核心思路：

- key 加随机数，然后执行 keyby 时会根据新 key 进行分区，此时会打散 key 的分布，不会造成数据倾斜问题
- 二次 keyby 进行结果统计

​

打散逻辑核心代码：
```java
public class KeyByRouter {

    private final static String SPLIT_CHAR = "#";

    /**
     * 不能太散，否则二次聚合还是会有数据倾斜
     *
     * @param sourceKey
     * @return
     */
    public static String randomKey(String sourceKey) {
        int endExclusive = (int) Math.pow(2, 7);
        return sourceKey + SPLIT_CHAR + (RandomUtils.nextInt(0, endExclusive) + 1);
    }

    public static String restoreKey(String randomKey) {
        if (StringUtils.isEmpty(randomKey)) {
            return null;
        }

        return randomKey.split(SPLIT_CHAR)[0];
    }
}

```

#### 作业暂停并保留状态失败

出现上面这个异常的可能原因是因为：

- 作业本身处于反压的情况，做 Checkpoint 可能失败了，所以暂停保留状态的时候做 Savepoint 肯定也会失败
- 作业的状态很大，做 Savepoint 超时了
- 作业设置的 Checkpoint 超时时间较短，导致 SavePoint 还没有做完，作业就丢弃了这次 Savepoint 的状态

解决办法：

- 代码设置 Checkpoint 的超时时间尽量的长一些，比如 10min，对于状态很大的作业，可以设置更大
- 如果作业不需要保留状态，那么直接暂停作业，然后重启就行

### 总结与展望

这篇文章分别从实时特征清洗框架演进，特征可配置，特征清洗逻辑热部署等方面介绍了目前较稳定的实时计算可行架构。经过近两年的迭代，目前这套架构在稳定性、资源利用率、性能开销上有最优的表现，给业务策略人员及业务算法人员提供了有力的支撑。  
​

未来，我们期望特征的配置还是回归 SQL 化，虽然目前配置已经足够简单，但是毕竟属于我们自己打造的“领域设计语言”，对新来的的策略人员 & 产品人员有一定的学习成本，我们期望的是能够通过像 SQL 这种全域通过用语言来配置化，类似 Hive 离线查询一样，屏蔽了底层复杂的计算逻辑，助力业务更好的发展。