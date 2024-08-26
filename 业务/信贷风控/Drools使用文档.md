

# 1 .场景
## 1.1需求

```
商城系统消费赠送积分

100元以下, 不加分 
100元-500元 加100分 
500元-1000元 加500分 
1000元 以上 加1000分
......
```

## 1.2传统做法

### 1.2.1 if...else

```
if (order.getAmout() <= 100){
    order.setScore(0);
    addScore(order);
}else if(order.getAmout() > 100 && order.getAmout() <= 500){
    order.setScore(100);
    addScore(order);
}else if(order.getAmout() > 500 && order.getAmout() <= 1000){
    order.setScore(500);
    addScore(order);
}else{
    order.setScore(1000);
    addScore(order);
}
```

### 1.2.2 策略

```
interface Strategy {
    addScore(int num1,int num2);
}

class Strategy1 {
    addScore(int num1);
}
......................
interface StrategyN {
    addScore(int num1);
}

class Environment {
    private Strategy strategy;

    public Environment(Strategy strategy) {
        this.strategy = strategy;
    }

    public int addScore(int num1) {
        return strategy.addScore(num1);
    }
}
```

### 1.2.3 问题？

```
以上解决方法问题思考：
如果需求变更，积分层次结构增加，积分比例调整？
数据库？

遇到的问题瓶颈：
第一，我们要简化if else结构,让业务逻辑和数据分离！
第二，分离出的业务逻辑必须要易于编写，至少单独编写这些业务逻辑，要比写代码快！
第三，分离出的业务逻辑必须要比原来的代码更容易读懂！
第四，分离出的业务逻辑必须比原来的易于维护，至少改动这些逻辑，应用程序不用重启！
```

# 2.Drools基础概念
## 2.1概念

规则引擎由推理引擎发展而来，是一种嵌入在应用程序中的组件，实现了将业务决策从应用程序代码中分离出来，并使用预定义的语义模块编写业务决策。接受数据输入，解释业务规则，并根据业务规则做出业务决策

需要注意的是规则引擎并不是一个具体的技术框架，而是指的一类系统，即业务规则管理系统。目前市面上具体的规则引擎产品有：drools、VisualRules、iLog等

在很多企业的 IT 业务系统中，经常会有大量的业务规则配置，而且随着企业管理者的决策变化，这些业务规则也会随之发生更改。为了适应这样的需求，我们的 IT 业务系统应该能快速且低成本的更新。适应这样的需求，一般的作法是将业务规则的配置单独拿出来，使之与业务系统保持低耦合。目前，实现这样的功能的程序，已经被开发成为规则引擎。

## 2.2 起源
 	 	 		
 			![image.png](https://cdn.nlark.com/yuque/0/2022/png/8364057/1666689920151-debcdb10-e935-4d67-b3a7-393264959887.png#clientId=u6bf4c9a0-527a-4&from=paste&height=780&id=ufefe40cd&originHeight=780&originWidth=1592&originalType=binary&ratio=1&rotation=0&showTitle=false&size=116723&status=done&style=none&taskId=u0ec3442d-4f55-46c5-8b79-a8a91c5246f&title=&width=1592)
 				 			

## 2.3 原理--基于 rete 算法的规则引擎

### 2.3.1 原理

在 AI 领域，产生式系统是一个很重要的理论，产生式推理分为正向推理和逆向推理产生式，其规则的一般形式是：IF 条件 THEN  操作。rete 算法是实现产生式系统中正向推理的高效模式匹配算法，通过形成一个 rete  网络进行模式匹配，利用基于规则的系统的时间冗余性和结构相似性特征 ，提高系统模式匹配效率

正向推理（Forward-Chaining）和反向推理（Backward-Chaining）

（1）正向推理也叫演绎法，由事实驱动，从一个初始的事实出发，不断地从应用规则得出结论。首先在候选队列中选择一条规则作为启用规则进行推理，记录其结论作为下一步推理的证据。如此重复这个过程，直到再无可用规则可被选用或者求得了所要求的解为止。

（2）反向推理也叫归纳法，由目标驱动，首先提出某个假设，然后寻找支持该假设的证据，若所需的证据都能找到，说明原假设是正确的，若无论如何都找不到所需要的证据，则说明原假设不成立，此时需要另作新的假设。

### 2.3.2 rete算法

Rete 算法最初是由卡内基梅隆大学的 Charles L.Forgy 博士在 1974 年发表的论文中所阐述的算法 ,  该算法提供了专家系统的一个高效实现。自 Rete 算法提出以后 , 它就被用到一些大型的规则系统中 , 像 ILog、Jess、JBoss  Rules 等都是基于 RETE 算法的规则引擎 。

Rete 在拉丁语中译为”net”，即网络。Rete 匹配算法是一种进行大量模式集合和大量对象集合间比较的高效方法，通过网络筛选的方法找出所有匹配各个模式的对象和规则。

其核心思想是将分离的匹配项根据内容动态构造匹配树，以达到显著降低计算量的效果。Rete 算法可以被分为两个部分：规则编译和规则执行 。当 Rete 算法进行事实的断言时，包含三个阶段：匹配、选择和执行，称做 match-select-act cycle。

## 2.4 规则引擎应用场景
| 业务领域 | 示例 |
| --- | --- |
| 财务决策 | 贷款发放，征信系统 |
| 库存管理 | 及时的供应链路 |
| 票价计算 | 航空，传播，火车及其他公共汽车运输 |
| 生产采购系统 | 产品原材料采购管理 |
| 风控系统 | 风控规则计算 |
| 促销平台系统 | 满减，打折，加价购 |


## 2.5 Drools基本介绍

Drools 具有一个易于访问企业策略、易于调整以及易于管理的开源业务 规则引擎，符合业内标准，速度快、效率高。业务分析师或审核人员可以利用它轻松查看业务规则，从而检验已编码的规则是否执行了所需的业务规则。其前身是  Codehaus 的一个开源项目叫 Drools，后被纳入 JBoss 门下，更名为 JBoss Rules，成为了 JBoss  应用服务器的规则引擎。

Drools 被分为两个主要的部分：编译和运行时。编译是将规则描述文件按 ANTLR 3  语法进行解析，对语法进行正确性的检查，然后产生一种中间结构“descr”，descr 用 AST 来描述规则。目前，Drools  支持四种规则描述文件，分别是：drl 文件、 xls 文件、brl 文件和 dsl 文件，其中，常用的描述文件是 drl 文件和 xls  文件，而 xls 文件更易于维护，更直观，更为被业务人员所理解。运行时是将 AST传到 PackageBuilder，由  PackagBuilder来产生 RuleBase，它包含了一个或多个 Package 对象。

# 3 .KIE

**Kie全称为Knowledge Is Everything**，即"知识就是一切"的缩写，是Jboss一系列项目的总称。官网描述：这个名字渗透在GitHub账户和Maven POMs中。随着范围的扩大和新项目的展开，KIE（Knowledge Is Everything的缩写）被选为新的组名。KIE的名字也被用于系统的共享方面；如统一的构建、部署和使用。
### 3.1 常用类说明
#### KieServices
Kie构建和运行时的基础对象，都可以通过kieService来获取
```
KieServices kieServices = KieServices.Factory.get();
```
#### KieContainer
Java资源和Kie资源被编译部署到KieContainer，在运行时使用
```
KieContainer kieContainer = kieServices.getKieClasspathContainer();

KieContainer kieContainer = kieServices.newKieContainer(releaseId());
```
#### Kiebase
Kbase是应用所有的知识定义的仓储，包括规则、流程、方法等。不包括运行时数据。
Kiebase创建方式
通过kmodule.xml，该文件必须放在resource的META- INF下
#### KieSession
存储运行时数据
KieSession分为无状态和有状态，无状态例如一个函数执行完返回一个结果。有状态的生存期很长，允许随着时间的推移进行迭代更改。
#### KieModule
包含了，KieBase，KSession，可以通过kmodule.xml自动创建或者通过KieModuleModel手动创建。
#### KieFileSystem
虚拟的文件系统，可以添加不同resource到工程中
#### KieResources
提供了很多工厂方法把不同资源类型转换成字节流，供KieFileSystem使用，例如：URL、文件、String等。
#### KieBuilder
KieFileSystem build成功，Kmodule会被自动装配到**KieRepository**
#### KieRepository
Kmodule的目录
#### Commands and the CommandExecutor
ksession通过CommandFactory执行command
```
StatelessKieSession ksession = kbase.newStatelessKieSession(); 
ExecutionResults bresults =   ksession.execute( CommandFactory.newInsert( new Cheese( "stilton" ), "stilton_id" ) );
```
### 3.2 自动构建
#### 

#### kmodule.xml**为空**
```
<?xml version="1.0" encoding="UTF-8"?> 
<kmodule xmlns="http://www.drools.org/xsd/kmodule"/>

如上kmodule.xml什么都不配置，可以获取默认的Kiebase和KieSession
KieBase kieBase = kieContainer.getKieBase();
StatelessKieSession kieSession = kieContainer.newStatelessKieSession();
```
#### kmodule.xml**配置**
```
<kmodule xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xmlns="http://www.drools.org/xsd/kmodule">
    <configuration>
        <property key="drools.evaluator.supersetOf" value="definition.SupersetOfEvaluatorDefinition"/>
    </configuration>
    <kbase name="KBase1" default="true" eventProcessingMode="cloud" equalsBehavior="equality" declarativeAgenda="enabled" packages="org.mozzi.drools">
        <ksession name="KSession2_1" type="stateless" default="true" beliefSystem="jtms"/>
        <ksession name="KSession2_2" type="stateful" default="false"/>

    </kbase>
    <kbase name="KBase2" default="false" eventProcessingMode="stream" equalsBehavior="equality" declarativeAgenda="enabled" packages="org.mozzi.drools" includes="KBase1">
        <ksession name="KSession3_1" type="stateful" default="false" clockType="realtime">
            <fileLogger file="drools.log" threaded="true" interval="10"/>
            <workItemHandlers>
                <workItemHandler name="name" type="handler.WorkItemHandler"/>
            </workItemHandlers>
            <listeners>
                <ruleRuntimeEventListener type="listener.RuleRuntimeListener"/>
                <agendaEventListener type="listener.FirstAgendaListener"/>
                <agendaEventListener type="listener.SecondAgendaListener"/>
                <processEventListener type="listener.ProcessListener"/>
            </listeners>
        </ksession>
    </kbase>
</kmodule>

代码区域
        KieServices kieServices = KieServices.Factory.get();
        KieContainer kieContainer = kieServices.getKieClasspathContainer();
//      default
//      KieBase kBase1 = kContainer.getKieBase(); // returns KBase1
//      KieSession kieSession1 = kContainer.newKieSession(); // returns KSession2_1
        KieBase kieBase1 = kieContainer.getKieBase("KBase1");
        StatelessKieSession kieSession1 = kieContainer.newStatelessKieSession("KSession2_1");
        KieSession kieSession2 = kieContainer.newKieSession("KSession2_2");
```
### 3.3 手动构建
通过KieModuleModel手动创建KBase和KSession
```
KieServices kieServices = KieServices.Factory.get();
KieModuleModel kieModuleModel = kieServices.newKieModuleModel();

KieBaseModel kieBaseModel1 = kieModuleModel.newKieBaseModel( "KBase1 ")
        .setDefault( true )
        .setEqualsBehavior( EqualityBehaviorOption.EQUALITY )
        .setEventProcessingMode( EventProcessingOption.STREAM );

KieSessionModel ksessionModel1 = kieBaseModel1.newKieSessionModel( "KSession1" )
        .setDefault( true )
        .setType( KieSessionModel.KieSessionType.STATEFUL )
        .setClockType( ClockTypeOption.get("realtime") );

KieFileSystem kfs = kieServices.newKieFileSystem();
kfs.writeKModuleXML(kieModuleModel.toXML());

KieFileSystem kfs = ...
kfs.write( "src/main/resources/KBase1/ruleSet1.drl", stringContainingAValidDRL )
        .write( "src/main/resources/dtable.xls",
                kieServices.getResources().newInputStreamResource( dtableFileStream ) );
```

### 3.4 规则引擎运行时

drools规则引擎由以下几部分构成：

- Working Memory（工作内存）
- Rules（规则库）
- Facts
- **Production memory**
- **Working memory:**
- **Agenda**

如下图所示：
![image.png](https://cdn.nlark.com/yuque/0/2022/png/8364057/1667205747731-ff54d6cb-e02a-4ea5-8305-538e3ebafbac.png#clientId=u64c5809a-a2c0-4&from=paste&height=300&id=u46cd9f36&originHeight=300&originWidth=540&originalType=binary&ratio=1&rotation=0&showTitle=false&size=36447&status=done&style=none&taskId=ud3cfb6ee-b5cd-47d3-9758-09d83e76b50&title=&width=540)


#### 相关概念说明

**Working Memory**：工作内存，drools规则引擎会从Working Memory中获取数据并和规则文件中定义的规则进行模式匹配，所以我们开发的应用程序只需要将我们的数据插入到Working Memory中即可，例如本案例中我们调用kieSession.insert(order)就是将order对象插入到了工作内存中。

**Fact**：事实，是指在drools 规则应用当中，将一个**普通的JavaBean插入到Working Memory后的对象**就是Fact对象，例如本案例中的Order对象就属于Fact对象。Fact对象是我们的应用和规则引擎进行数据交互的桥梁或通道。

**Rules**：规则库，我们在规则文件中定义的规则都会被加载到规则库中。

**Pattern Matcher**：匹配器，将Rule Base中的所有规则与Working Memory中的Fact对象进行模式匹配，匹配成功的规则将被激活并放入Agenda中。

**Agenda**：议程，用于存放通过匹配器进行模式匹配后被激活的规则。



## [
](https://www.drools.org/#)
