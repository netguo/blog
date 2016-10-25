---
layout: post
title: "drools-pmml学习"
date: 2015-06-14 16:46:24 +0800
comments: true
categories: Framework 
---    

<!--more-->

##### 标签：drools-pmml， drools， pmml 
  
---
* list element with functor item
{:toc}  

### 一 pmml  
全称预言模型标记语言（Predictive Model Markup Language），利用XML描述和存储数据挖掘模型，是一个已经被W3C所接受的标准。MML是一种基于XML的语言，用来定义预言模型。  

具体的语法可参考  
[http://www.dmg.org/v4-2-1/GeneralStructure.html](http://www.dmg.org/v4-2-1/GeneralStructure.html)
  
### 二 drools-pmml的安装使用
  
#### 1  下载drools-pmml源码####
[https://github.com/droolsjbpm/drools.git](https://github.com/droolsjbpm/drools.git) 。
    
  
#### 2  项目部署####
  
#####编译
    
```
➜  drools git:(master) ✗ cd drools-pmml & mvnieclipse
```    
  
#####导入eclipse。##### 
  
  坑:  
    
  + 1. 导入后，会报错，target/generated-sources/java下org.dmg.pmml.pmml_4_2.descr和target/generated-sources/下java.org.dmg.pmml.pmml_4_2.descr的java类，packet 处会显示错误。
  + 2. Google之，这两个包，Build Path “Remove from build path", 然后在项目文件中这个两个文件选中，Build Path－Use as Source Floder。  

分析：  
 
  +  1.初步以为是eclipse默认只识别src下的工程目录，所以需要重新Use as Source Floder。
  +  2.后续又重新测试，发现只需要把这两个包其中一个Remove from build path即可，仔细观察可以发现，这两个包中java类相同，断定是，这两个包命名冲突。    
  
  
### 三  drools-pmml代码分析 ###
drools-pmml代码安装以后，大体观察目录结构，可以看到主要有test,main,target三个目录，main和target是源码的实现，test便是提供给用户的使用测试目录，可方便的根据junit测试进行，对drools-pmml的使用进行学习。（可以看到，简单一个源码无需要提供api，根据test进行使用的学习）。接下来对test目录进行分析。  
  
#### test主要包括五个包，目录结构如下：####  
org.drools.pmml.pmml*_*4*_*2  （一些基本的测试）  
	－DroolsAbstractPMMLTest  
	－KieBaseTest  
	－PMMLErrorTest  
	－PMMLGenerationTest  
	－PMMLUsageDemoTest  
org.drools.pmml.pmml*_*4*_*2.global  （pmml中adapter，datadictory。。的测试）  
	－AdapterTest
	－ConstrainedDataDictionaryTest  
	－DataDictionaryTest  
	－HeaderTest  
org.drools.pmml.pmml*_*4*_*2.predictive  （pmml中model块的测试）   
	－AttributesTest  
	－MiningSchemaTest  
	－TargetsAndOutputsTest  
org.drools.pmml.pmml*_*4*_*2.predictive.models  （一些模型的功能测试）  
	－......（省略）  
org.drools.pmml.pmml*_*4*_*2.transformations   （pmml中含有transformations的测试）   
	－......（省略）

其中以上的测试类，可以望文生义的理解到其中各个模块的功能，drools和pmml功能实现核心都封装在DroolsAbstractPMMLTest中，就这个类进行分析。其它模块都是一些细节性的测试。  
核心代码分析：   

getModelSession方法返回一个KieSession，用户对pmml的操作，都通过KieSession来完成。

  
```java
protected KieSession getModelSession(String[] pmmlSources, boolean verbose) {
        // KieService是一个单例，用于创建KieRepository等。
        KieServices ks = KieServices.Factory.get();
        // KieRepository是一个单例，用于存储所有的KieModules
        KieRepository kr = ks.getRepository();
        // KieFileSystem用于为KieModules读取外部文件
        KieFileSystem kfs = ks.newKieFileSystem();
        // 根据路径读取外部文件
        for (int j = 0; j < pmmlSources.length; j++) {
            Resource res = ResourceFactory.newClassPathResource(pmmlSources[j]).setResourceType(ResourceType.PMML);
            kfs.write(res);
        }
        // 程序自定义的KieModule
        KieModuleModel model = ks.newKieModuleModel();
        KieBaseModel kbModel = model.newKieBaseModel(DEFAULT_KIEBASE).setDefault(true).addPackage(BASE_PACK).setEventProcessingMode(EventProcessingOption.STREAM);

        kfs.writeKModuleXML(model.toXML());

        KieBuilder kb = ks.newKieBuilder(kfs);

        kb.buildAll(); // 将KieModule中的资源进行编译，在此处PMML文件被编译为drools格式
        if (kb.getResults().hasMessages(Message.Level.ERROR)) {
            throw new RuntimeException("Build Errors:\n" + kb.getResults().toString());
        }

        // kieContain包含了KieModule的所以的kiebase
        KieContainer kContainer = ks.newKieContainer(kr.getDefaultReleaseId());

        // kiebase是应用knowledge definitions的存储地，不过只是静态的规则，函数等资源
        KieBase kieBase = kContainer.getKieBase();

        setKbase(kieBase);
        // 返回一个KieSession，进行真正运行时的操作
        return kieBase.newKieSession();
    }
```
    
 如代码所示，该方法会返回一个KieSession，KieSession会程序中动态的将数据注入，并编译成knowledge，进行执行。  
   
 kb.buildAll()将KieModule中的资源进行编译，在此处PMML文件被编译为drools格式。   
       
    
### 四  案例分析  

在drools-pmml中给出了Trees,Regression,Scorecard,NeuralNetwork等模型的测试方法，不过调用方式都大同小异，以下根据决策树模型做案例分析。
  
#### 代码分析 ####
  
```java
    /*
     * 该方法的类继承了DroolsAbstractPMMLTest类， DroolsAbstractPMMLTest对Ksession，Kbase做了一定的封装，
     * 该方法中的，setKSession，setKbase等方法都封装在DroolsAbstractPMMLTest中
     */
    @Test
    public void testSimpleTree() throws Exception {
        setKSession(getModelSession(source1, VERBOSE));
        setKbase(getKSession().getKieBase());
        KieSession kSession = getKSession();

        // kSession.addEventListener( new org.drools.event.rule.DebugAgendaEventListener() );

        kSession.fireAllRules(); // init model

        // 获取预测值字段
        FactType tgt = kSession.getKieBase().getFactType(packageName, "Fld5");
        // 获取输入参数字段并往里赋值
        kSession.getEntryPoint("in_Fld1").insert(30.0);
        kSession.getEntryPoint("in_Fld2").insert(60.0);
        kSession.getEntryPoint("in_Fld3").insert("false");
        kSession.getEntryPoint("in_Fld4").insert("optA");

        kSession.fireAllRules(); // 重新init model
		//调用DroolsAbstractPMMLTest中该方法，进行测试,检测模型的输出值是否正确。
        checkFirstDataFieldOfTypeStatus(tgt, true, false, "Missing", "tgtY");

        checkGeneratedRules();
    }
```  

#### pmml决策分析 ####
在以下的决策树模型中，输入：fld1＝30.0，fld2＝60.0，fld3＝false，fld4＝"optA"。  
决策分析： （节点用pmml注释中的值表示） 
 
  1.  第1节点满足fld4="optA"，继续递归。  第11节点是surrogate操作，fld1<30不满足，返回false，继续遍历第12节点，fld1<30 or fld2 >20，满足，继续递归121节点，fld4 notIn ["optA","optB"]，不满足，返回false。
  2. 继续递归第2节点，fld4="optA" or fld4="optB"，满足，继续递归。第21节点fld1 >10 and fld1 < 50 and fld4="optA" and fld2 < 100 and fld3=false，满足，停止递归。  
  3. 最终决策值为"tgtY"。
  
      
  
```
 <PMML version="4.2" xsi:schemaLocation="http://www.dmg.org/PMML-4_2 http://www.dmg.org/v4-1/pmml-4-2.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xmlns="http://www.dmg.org/PMML-4_2">
  <Header copyright="JBOSS"/>
  <DataDictionary numberOfFields="5">
    <DataField dataType="double" name="fld1" optype="continuous"/>
    <DataField dataType="double" name="fld2" optype="continuous"/>
    <DataField dataType="string" name="fld3" optype="categorical">
      <Value value="true"/>
      <Value value="false"/>
    </DataField>
    <DataField dataType="string" name="fld4" optype="categorical">
      <Value value="optA"/>
      <Value value="optB"/>
      <Value value="optC"/>
    </DataField>
    <DataField dataType="string" name="fld5" optype="categorical">
      <Value value="tgtX"/>
      <Value value="tgtY"/>
      <Value value="tgtZ"/>
    </DataField>
  </DataDictionary>
  <TreeModel functionName="classification" modelName="TreeTest">
    <MiningSchema>
      <MiningField name="fld1"/>
      <MiningField name="fld2"/>
      <MiningField name="fld3"/>
      <MiningField name="fld4"/>
      <MiningField name="fld5" usageType="predicted"/>
    </MiningSchema>
    <Node score="tgtX">//root
      <True/>
      <Node score="tgtX"> //1
        <SimplePredicate field="fld4" operator="equal" value="optA"/>
        <Node score="tgtX"> //11
          <CompoundPredicate booleanOperator="surrogate">
            <SimplePredicate field="fld1" operator="lessThan" value="30"/>
            <SimplePredicate field="fld2" operator="greaterThan" value="20"/>
          </CompoundPredicate>
          <Node score="tgtX"> //111
            <SimplePredicate field="fld2" operator="lessThan" value="40"/>
          </Node>
          <Node score="tgtZ"> //112
            <SimplePredicate field="fld2" operator="greaterOrEqual" value="10"/>
          </Node>
        </Node>
        <Node score="tgtZ">  //12
          <CompoundPredicate booleanOperator="or">
            <SimplePredicate field="fld1" operator="greaterOrEqual" value="60"/>
            <SimplePredicate field="fld1" operator="lessOrEqual" value="70"/>
          </CompoundPredicate>
          <Node score="tgtZ"> //121
            <SimpleSetPredicate booleanOperator="isNotIn" field="fld4">
              <Array type="string">optA optB</Array>
            </SimpleSetPredicate>
          </Node>
        </Node>
      </Node>
      <Node score="tgtY">  //2
        <CompoundPredicate booleanOperator="or">
          <SimplePredicate field="fld4" operator="equal" value="optA"/>
          <SimplePredicate field="fld4" operator="equal" value="optC"/>
        </CompoundPredicate>
        <Node score="tgtY"> //21
          <CompoundPredicate booleanOperator="and">
            <SimplePredicate field="fld1" operator="greaterThan" value="10"/>
            <SimplePredicate field="fld1" operator="lessThan" value="50"/>
            <SimplePredicate field="fld4" operator="equal" value="optA"/>
            <SimplePredicate field="fld2" operator="lessThan" value="100"/>
            <SimplePredicate field="fld3" operator="equal" value="false"/>
          </CompoundPredicate>
        </Node>
        <Node score="tgtZ">  //22
          <CompoundPredicate booleanOperator="and">
            <SimplePredicate field="fld4" operator="equal" value="optC"/>
            <SimplePredicate field="fld2" operator="lessThan" value="30"/>
          </CompoundPredicate>
        </Node>
      </Node>
    </Node>
  </TreeModel>
</PMML>
```  


