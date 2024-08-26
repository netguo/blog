### 反射
提到aop之前，先了解一下Java中的反射，关于反射我们按照技术可以解决什么问题，设计，实现三个角度来考虑。
##### 反射可以解决什么问题
常规的java程序设计中，我们都是先new一个对象，根据这个对象进行方法的调用。而反射机制不需要先创建一个对象，直接可以class，方法，参数进行调用。当然这个只是我们常规的了解。看一下维基百科上的定义：
>在计算机学中，反射（英语：reflection）是指计算机程序在运行时（runtime）可以访问、检测和修改它本身状态或行为的一种能力。[1]用比喻来说，反射就是程序在运行的时候能够“观察”并且修改自己的行为。  

>要注意术语“反射”和“内省”（type introspection）的关系。内省（或称“自省”）机制仅指程序在运行时对自身信息（称为元数据）的检测；反射机制不仅包括要能在运行时对程序自身信息进行检测，还要求程序能进一步根据这些信息改变程序状态或结构。

>在类型检测严格的面向对象的编程语言如Java中，一般需要在编译期间对程序中需要调用的对象的具体类型、接口、字段和方法的合法性进行检查。反射技术则允许将对需要调用的对象的信息检查工作从编译期间推迟到运行期间再现场执行。这样一来，可以在编译期间先不明确目标对象的接口名称、字段（fields，即对象的成员变量）、可用方法，然后在运行根据目标对象自身的信息决定如何处理。它还允许根据判断结果进行实例化新对象和相关方法的调用。

##### java中反射的实现
oracle对于反射的定义  

> Reflection enables Java code to discover information about the fields, methods and constructors of loaded classes, and to use reflected fields, methods, and constructors to operate on their underlying counterparts, within security restrictions.
The API accommodates applications that need access to either the public members of a target object (based on its runtime class) or the members declared by a given class. It also allows programs to suppress default reflective access control.
  
> 简而言之，通过反射，我们可以在运行时获得程序或程序集中每一个类型的成员和成员的信息。程序中一般的对象的类型都是在编译期就确定下来的，而 Java 反射机制可以动态地创建对象并调用其属性，这样的对象的类型在编译期是未知的。所以我们可以通过反射机制直接创建对象，即使这个对象的类型在编译期是未知的。

反射的核心是 JVM 在运行时才动态加载类或调用方法/访问属性，它不需要事先（写代码的时候或编译期）知道运行对象是谁。

Java 反射主要提供以下功能：
* 在运行时判断任意一个对象所属的类；
* 在运行时构造任意一个类的对象；
* 在运行时判断任意一个类所具有的成员变量和方法（通过反射甚至可以调用private方法）；
* 在运行时调用任意一个对象的方法

了解Java内存模型的我们应该比较清楚，在编译过程中生成.class文件，会被加载在内存模型中的方法区。作为对象的元数据
### aop
##### 概念与历史
概念：面向切面的程序设计（Aspect-oriented programming，AOP，又译作面向方面的程序设计、剖面导向程序设计）是计算机科学中的一种程序设计思想，旨在将横切关注点与业务主体进行进一步分离，以提高程序代码的模块化程度。
历史：“面向切面的程序设计”这一术语出现的具体时间已经不可考证了，但该词是由施乐帕洛阿尔托研究中心的Chris Maeda首先提出的。术语“横切”是由Gregor Kiczales提出的。

#####