之前有学习过mysql的事务，可以理解到事务是在mysql层或者innodb层实现的。Spring中的事务，主要是聚焦于工程中的事务管理。

#### mybatis中的事务配置
```
    <!-- ======= 事务定义开始 ======= -->
    <!-- Ibatis事务管理器 -->
    <bean id="xxxTransactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="xxxxxDataSource"/>
    </bean>

    <!-- 事务属性，方法以此开头的需要进行事务控制 -->
    <bean id="xxxTxAttributeSource"
          class="org.springframework.transaction.interceptor.NameMatchTransactionAttributeSource">
        <property name="properties">
            <props>
                <prop key="update*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                <prop key="modify*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                <prop key="insert*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                <prop key="save*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                <prop key="create*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                <prop key="delete*">PROPAGATION_REQUIRED,-RollbackableException</prop>
                ......
                <prop key="newTransactionWrapper">PROPAGATION_REQUIRES_NEW,-RollbackableException
                </prop>
            </props>
        </property>
    </bean>

    <!-- 事务拦截器 -->
    <bean id="xxxTxInterceptor" class="org.springframework.transaction.interceptor.TransactionInterceptor">
        <property name="transactionManager" ref="xxxTransactionManager"/>
        <property name="transactionAttributeSource" ref="xxxTxAttributeSource"/>
    </bean>

    <!-- 为匹配的Bean自动创建代理 -->
    <bean class="org.springframework.aop.framework.autoproxy.BeanNameAutoProxyCreator">
        <property name="beanNames">
            <list>
                <value>*Service</value>
            </list>
        </property>
        <property name="includePackages">
            <list>
                <value>cn.xxx.xxx.xxx.biz.service.impl</value>
            </list>
        </property>
        <property name="interceptorNames">
            <list>
                <value>xxxxxTxInterceptor</value>
            </list>
        </property>
    </bean>
```    
我们先从上述配置中看
BeanNameAutoProxyCreator 定义了扫描路径，扫描的类
    TransactionInterceptor
        DataSourceTransactionManager  数据源事务管理
        NameMatchTransactionAttributeSource 定义扫描的方法，以及事务传播特性
大概可以猜测Spring的事务管理，通过aop做的，BeanNameAutoProxyCreator定义了扫描路径，NameMatchTransactionAttributeSource定义了需要aop切面的方法，以及事务传播行为，DataSourceTransactionManager中是对于事务的实现。

#### 源码分析
##### BeanNameAutoProxyCreator
##### DataSourceTransactionManager
![1](../../picture/DataSourceTransactionManager.png)