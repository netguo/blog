一、引言
规则引擎从早期的产生式推理系统演化而来，产生式推理系统是根据用户定义的规则(其一般的形式是IF- THEN)和当前系统获取的事实 (Facts)进行规则匹配并进行冲突消解 ，结果供用户或运行系统使用的一种推理机制 。
产生式系统最大的缺点是其匹配效率低下，Forgy证明了产生式系统90%以上的时间被用于匹配过程，1979年Forgy提出Rete算法，通过规则的模式共享以及缓存匹配状态来加速规则匹配。
二、产生式系统
2.1  事实（fact）：
事实：对象之间及对象属性之间的多元关系。为简单起见，事实用一个三元组来表示：（identifier ^attribute value），例如如下事实：
```
w1:(B1  ^ on B2)    w6:(B2  ^color blue)
w2:(B1  ^ on B3)    w7:(B3  ^left-of B4)
w3:(B1  ^ color red)   w8:(B3 ^on table)
w4:(B2  ^on table)   w9:(B3  ^color red)
w5:(B2  ^left-of B3)
```
2.2  规则（rule）:
由条件和结论构成的推理语句，当存在事实满足条件时，相应结论被激活。一条规则的一般形式如下：
```
(name-of-this-production
LHS /*one or more conditions*/
-->
RHS /*one or more actions*/
)
其中LHS为条件部分，RHS为结论部分。
下面为一条规则的例子：
(find-stack-of-two-blocks-to-the-left-of-a-red-block
(^on)
(^left-of)
(^color red)
-->
...RHS...
)
```
2.3  模式（patten）：
模式：规则的IF部分，已知事实的泛化形式，未实例化的多元关系。
```
(^on)
(^left-of)
(^color red)
```
2.4 匹配的普通算法
规则主要由两部分组成：条件和结论，条件部分也称为左端（记为LHS, left-hand side），结论部分也称为右端（记为RHS, right-hand side）。为分析方便，假设系统中有N条规则，每个规则的条件部分平均有P个模式，工作内存中有M个事实，事实可以理解为需要处理的数据对象。
规则匹配，就是对每一个规则r, 判断当前的事实o是否使LHS(r)=True，如果是，就把规则r的实例r(o)加到冲突集当中。所谓规则r的实例就是用数据对象o的值代替规则r的相应参数，即绑定了数据对象o的规则r。
规则匹配的一般算法：
1) 从N条规则中取出一条r；
2) 从M个事实中取出P个事实的一个组合c；
3) 用c测试LHS(r)，如果LHS(r（c）)=True，将RHS(r（c）)加入冲突集中；
4) 取出下一个组合c，goto 3；
5) 取出下一条规则r，goto 2；
三、Rete算法
如上传统算法效率低下，Rete算法初衷是：利用规则之间各个域的公用部分减少规则存储，同时保存匹配过程的临时结果以加快匹配速度。为了达到这种效果，算法讲规则拆分，其中每个条件单元作为基本单位(节点)连接成一个数据辨别网络 ，然后将事实经过网络筛选并传播 ，最终所有条件都有事实匹配的规则被激活。
3.1 节点
网络中共五类节点类型：Root节点、Type节点、Alpha（单输入节点）、Beta（双输入结点）、Terminal节点

- Root节点代表整个Rete网络的入口。
- ObectType节点用于选择事实的类型，将符合本节点类型的事实向后继的Alpha节点传播。
- Alpha节点(也称1-inputNode)主要进行同对象类型内属性的约束(Intra-conditions)或常量测试(LiteralRestriction)，比如“name==zhang”，“age>15”等等。
- Beta节点(也称2-inputNode)主要根据不同对象之间的约束(Inter-conditions)如“P.name一一C.friend”，“P.age>Cat.age”等进行连接操作。Beta节点又分为Join节点、Not节点等。
- Terminal节点是规则的末尾节点，它代表一个规则匹配结束。

3.2 算法实例
```
R1:
if
  C:Cofee(type== ”instant”)
  p:Person(money>Coffe.price) 
then
  p.buy(c)
R2:
if
  C:Cofe(type== ”instant”)
  P:Person(money<Coffe.price) 
then
p.return(c)
```
![image.png](https://cdn.nlark.com/yuque/0/2022/png/8364057/1665396514718-feb5a8e1-3848-47c8-a611-584ee6b808ec.png#averageHue=%23efeeee&clientId=u07536e81-c70b-4&from=paste&height=468&id=u0c99b9d0&originHeight=862&originWidth=1044&originalType=binary&ratio=1&rotation=0&showTitle=false&size=253392&status=done&style=none&taskId=u64286df2-7a3c-45db-babe-585412c4dc1&title=&width=567)

3.2 算法步骤
Rete的算法步骤主要分两个部分：规则网络的建立和事实的匹配
 规则网络的建立
(1)创建根。
(2)取出一个规则r。
a.取出一个模式P，检查参数类型，如果是新类型，则加入一个类型节点;
b.检查模式P的条件约束，对于单类型约束，检查对应的Ct节点是否已存在，如果存在则记录下节点位置，否则将该约束作为一个
节点加入链的后继，所有a处理完后连接a内存;
c.检查模式P的域，若为多类型约束，则创建相应的B节点，其左输入为前一p节点(第一个p节点左部输入为空)，右输入为当前链
的d内存;
d重复b—c，直到模式p的所有约束处理完毕;
e.重复a—d，直到所有的模式处理完毕，创建Terminal节点，每个
模式链的末尾连到Terminal节点;
f.将动作(Then部分)封装成叶节点(Action节点)作为输出节点。
(3)重复(2)直到所有规则处理完毕。

运行时的匹配过程如下:
* (1)从工作内存中取一WME放人根节点进行匹配。
* (2)遍历每个a节点(含ObjectType节点)，如果a节点约束条件与该WME一致，则将该WME存在该节点的匹配内存中，并向其后
继节点传播。
* (3)对a节点的后继结点继续(2)的过程，直到a内存所有通过匹配的
事实保存在a内存中。
* (4)对每个8节点进行匹配，如果单个事实进入B节点左部，则转换成一个元素的元组存在节点左侧内存中。如果是一个元组进入左部，则将其存在左内存中。如果一个事实进入右侧，则将其与左内存中的元组按照节点约束进行匹配，符合条件则将该事实对象与左部元组合并，并传递到下一节点。
* (5)重复(4)直到所有口处理完毕，元组对象进入到Terminal节点。对应的规则被触活，将规则后件加入议程(Agenda)。
* (6)对Agenda里的规则进行冲突消解，选择合适的规则执行。
![image.png](https://cdn.nlark.com/yuque/0/2022/png/8364057/1665386946888-595698a2-f0e0-4d4c-8f3c-c2941db12e74.png#averageHue=%23f3f2f1&clientId=u07536e81-c70b-4&from=paste&height=372&id=u21cdfb79&originHeight=372&originWidth=572&originalType=binary&ratio=1&rotation=0&showTitle=false&size=98434&status=done&style=none&taskId=ube3aad65-2c5e-4406-98be-0269830cca3&title=&width=572)
引用
> [开源规则引擎实践](https://www.yelcat.cc/index.php/archives/1373/)

> R ete 算 法 :研 究 现 状 与 挑 战

