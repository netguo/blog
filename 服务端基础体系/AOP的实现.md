AOP的实现
AOP(Aspect-OrientedProgramming，面向切面编程)，是对面向对象编程的补充。
AOP的相关概念：
连接点(Joinpoint):程序执行过程中明确的点，如方法的调用或特定的异常被抛出
通知(Advice)：在特定的连接点，AOP框架执行的动作。各种类型的通知包括"around"、"before"、"throws"的通知，维护一个"围绕"连接点的拦截器链。Spring中定义了四个advice：BeforeAdvice，AfterAdvice，ThrowAdvice和DynamicIntroductionAdvice
切入点(Pointcut)：指定一个通知将被引发的一系列连接点的集合。
引用（Introduction）:添加方法或字段到被通知的类。
目标对象（Target Object）:包含连接点的对象。也被称做被通知或被代理对象。