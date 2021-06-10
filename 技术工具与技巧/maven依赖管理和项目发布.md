
项目中一直用maven，最近在重新搭建一个项目时遇到一些问题，趁现在有时间正好稍微做个总结。项目中maven主要用于依赖的管理和项目的部署。  
   
### maven依赖管理        
  
#### 基本的配置  

```  
<dependencies>   
	<dependency>
	  <groupId></groupId>   
	  <artifactId></artifactId>  
	  <version></version>  
	  <type></type>  
	  <scope></scope>  
	  <optional></optional>  
	  <exclusions>
       <exclusion></exclusion>  
     </exclusions>   
    </dependency>
</dependencies>    
```  
* groupId，artifactId，version唯一确定一个项目，groupId往往是公司域名的倒置，artifactId是具体的项目名称，version是项目的版本号。  
* type：依赖的类型，默认是jar。 pom也较方便。  
* scope：依赖范围，有Compile，Test，Provided，Runtime，System，Import。默认是Complie。六种范围的作用域如下。     
     
|依赖范围|编译有效|测试有效|运行有效|  
|----|----|----|----|  
|compile|是|是|是|  
|test|否|是|否|  
|provided|是|是|否|  
|runtime|否|是|是|  
|system|是|是|否|  
 
* optional，依赖是否可选，默认false。  
* exclusions，用于排除传递性依赖。  

#### 依赖的传递性     
  
如下图所示   
<img src="http://blogguo.b0.upaiyun.com/maven-dependency.jpg" width="400" height="150">    

* 如果A依赖B，B依赖C，那么A也依赖C。    
* 如果有配置scope，A依赖C也会有不同的情况。  
* 如果B依赖C，optional为true，则A不会依赖C。   
 
***根据上面的规则，当不同的项目之间有依赖时，公共的依赖可以放在最底层。***  
  
####  依赖相关的一些mvn命令。    

```      
查看所有的依赖
mvn dependency:list    
依赖关系的树图
mvn dependency:tree    
依赖分析
mvn dependency:analyze  
```
    
### maven的项目部署  
项目的部署有多种方式，现在tomcat和jetty都有比较成熟的插件。    
其中平时多采用，jetty的org.mortbay.jetty部署，如下配置。
  
```      
<build>
    <plugins>
      <plugin>
        <groupId>org.mortbay.jetty</groupId>
        <artifactId>maven-jetty-plugin</artifactId>
        <version>6.1.10</version>
        <configuration>
          <scanIntervalSeconds>10</scanIntervalSeconds>
          <connectors>
            <connector implementation="org.mortbay.jetty.nio.SelectChannelConnector">
              <port>8080</port>
              <maxIdleTime>60000</maxIdleTime>
            </connector>
          </connectors>
        </configuration>
      </plugin>
      ...
    </plugins>
</build>   
```     
运行：`mvn jetty:run`    
更详细的配置参考:[http://wiki.eclipse.org/Jetty/Feature/Jetty_Maven_Plugin](http://wiki.eclipse.org/Jetty/Feature/Jetty_Maven_Plugin)

      



              
  






