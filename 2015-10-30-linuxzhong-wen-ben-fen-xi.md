---
layout: post
title: "Linux中文本分析"
date: 2015-10-30 17:43:08 +0800
comments: true
categories: Linux
---

<!--more-->

##### 标签： sed, awk  

在线上分析应用日志时，常常需要统计分析，以便可以尽快的定位到问题。bash提供了awk和sed命令，可以方便的进行文本处理。

### awk  

awk是一个强大的文本分析工具，主要用于处理行处理，并且可以完成分析统计。 它有很多内建的功能，比如数组、函数等。灵活性很强。 适于把一行分为多个字段来处理。
##### 主要语法：  

```  
 awk [ -F fs ] [ -v var=value ] [ 'prog' | -f progfile ] [ file ...  ]  
 -F 分隔符，默认是空格。可以通过-F指定  
 可以指定file，也可以读取前个指令的standard output 

```    

##### awk的内建变量    

```  
NF：每一行 ($0) 拥有的字段总数
NR：目前 awk 所处理的是『第几行』数据
FS：目前的分隔字符,预设是空格键  
```  
示例：将/etc/hosts中host和行号输出  
```  
awk '{print "host:"$1 , "column:" NF}' /etc/hosts  
```  
##### awk计算统计   
 
利用awk的BEGIN和END，BEGIN、END的示例如下：    

```  
awk 'BEGIN { statments } {commands} END {end commands}'     
```  
* 执行BEGIN语句块中的语句  
* 从文件或者stdin中读取一行，然后执行statments,重复过程，直到文件全部被读取完毕。  
* 当读至输入流末尾时，执行END{end commands}
* statments中可以对变量初始化，commands中不同的command以分号分隔。  

示例：统计日志中出现demo的次数    

```    
cat log.txt|grep demo|awk 'BEGIN{i=0}{i=i+1}END{print i}'  
```  
当然根据变量的复制，可以做其他更复杂的统计，例如求平均，求和等等。    

实战：根据线上日志统计时间  
  
```  
统计ActivityConsumer时间：
cat app.log | grep "c.f.c.b.c.ActivityConsumer"|grep "Worker\[creditcloud\]"|\
awk 'BEGIN{begin=0;time=0;time10=0;time50=0;time100=0;time200=0;time500=0;timeM500=0;}\
{begin++;if($10<10){time10++} else if($10<50 && $10>=10){  time50++} \
else if($10 >=50 && $10 <100){time100++} else if($10 >=100 && $10 <200){time200++} \
else if($10 >=200 && $10 <500){ time500++}else{ timeM500++}; time+=$10}\
END{print "total:" begin " timetotal:"time;\
print " 10ms:" time10 " 50ms:" time50 " 100ms:" time100 "\
200ms:" time200 " 500ms:" time500 " more500ms:" timeM500;\
print " 10msP:" time10*100/time"%" " 50msP:" time50*100/begin "%"\
" 100msP:" time100*100/begin"%" " 200msP:" time200*100/begin"%"\
" 500msP:" time500*100/begin"%" " more500ms:" timeM500*100/begin"%" }'  

结果：
total:49734 timetotal:4.23502e+06

10ms:10 50ms:22448 100ms:18872 200ms:6895 500ms:1314 more500ms:195

10msP:0.000236126% 50msP:45.1361% 100msP:37.9459% 200msP:13.8638% 500msP:2.64206% more500ms:0.392086%    
```

### sed   
 
sed是常用的文本替换工具，它能完美的匹配正则表达式  
先看sed命令的语法      

```    
sed [-nefr] [n1[,n2]]function  
-n 安静模式，经过sed特殊处理的那一行会被输出到屏幕上   
-e 在指令列模式上进行sed的动作编辑  
-f 将sed的动作写在一个档案内，-f filename可以执行filename内的sed动作  
-r sed的动作支持延伸型正规表示的语法。（默认是基础的正规表示语法）  

动作说明，n1,n2指需要操作的行数。
function的话主要由以下功能：  
a :新增，a后面可以接字符串。而这些字符串会在新的一行出现。  
c :取代，c后面可以接字符串，这些字符串可以取代n1,n2之间的行。  
d：删除  
i：插入，i后面可以接字符串，这些字符串可以在新的一行出现。（目前行的上一行）  
p：打印，亦即将某个选择的数据打印出。通常p会与参数sed -n 一块使用。   
s：取代，可以直接进行取代的工作。通常这个s的动作可以搭配正规表示法。  
```   
示例：    

* 写了一个shell脚本，本来路径用的直接路径，因为非root用户，把脚本中直接路径/home/user 改为~

```  
# 本地文件执行
sed  's/\home\/admin/~/g' shell.sh       

# 修改放入新的文件  
sed  's/\home\/admin/~/g' shell.sh > shell1.sh  
```  

* 删除一个文件中几行，这个比较简单    

```    
# 可以看到输出结果中，2，3行被删除
 cat -n demo|sed '2,3d'
     1  demo
     4  test    
# 移除空白行 
sed '/^$/d' shell.sh      
```  

* 添加一行   
```
cat demo | sed '1a aaa'    
```    



