Edit
jmeter接口压测

本文只是讲述在一个一次线上压测中遇到的一些重要点，并没有详细介绍jmeter

一、https接口访问
调用https接口的话，会需要注入ssl证书，菜单栏—选项— SSL管理— 选择自己的证书即可。

二、入参参数化

- 配置：     

```
可以从CSV中取数据。右键，添加— 配置原件 — CSV Data Set Config，配置参数如下：
Filename: CSV文件地址
File encoding : 文件的编码
Variable Names (comma-delimited) : CSV中列对应关系，例如：user,password
Delimiter ： 分隔符 一般是(,)
Allow quoted data?:
Recycle on EOF?: 数据是否循环使用，压测一般会用TRUE
Stop thread on EOF?: 在结束时，是否停止线程。（具体的效果不知为何，我在Recycle 配置为true时，这个没有起到作用）
Sharing mode: 作用域  
```  

- CSV文件
test,123456
test1,123456  
……  


- 取参
`${user},${password}`

三、配置压测方案
默认的会有线程组，向一个有经验的同事取经，借用jmeter-plugins会更方便一些。https://jmeter-plugins.org/downloads/all/ 最新的版本，是同一个manager来管理，感觉不太好用。采用的旧的版本，JMeterPlugins-Standard-1.3.1.zip。把CMDRunner.jar,JMeterPlugins-Standard.jar拷贝到/lib/ext目录下。

压测策略：
添加plugins会多两种策略选择：
stepping thread group
望文生义，一步步，提升线程的数量，到最大压测线程数：
包含较多的参数，也比较好理解：this group will start XX threads,、First,wait for XX thread ….. 可以提供最大线程数，一次stepping的个数，一次stepping的时间，load 时间等等。

ultimate thread group
可以自己配置时间点，在多少秒时多少个线程。不做详细解析。

结果监听：
可以配置  
```
Response Times vs Threads ，可以观测到不同并发的响应时间
Response Times Distribution，响应时间的分布
聚合报告，会生成一个报告，报告中会展示中位数，90%line，95%line，max，eror%等等。
Response Time Graph，响应时间表格  
```
