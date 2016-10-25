---
layout: post
title: "Restful API验证框架的设计与实现"
date: 2015-11-15 11:34:05 +0800
comments: true
categories: design
---   

<!--more-->   

##### 标签：dubbox ,restful ,api验证   
  
### dubbox中api统一验证的设计与实现  

API层提供restful接口，接口可能对接的不仅仅是前端平台，还可能是开放式的平台。不过无论是那种业务，提供restful的接口层都需要做数据的强校验。统一校验的优点很简单，将校验与API接口解耦，简化代码逻辑。
校验数据大体分为两种情况：   

*  数据本身格式不正确。例如为空，不符合业务规定格式。  
*  数据不满足于数据库中数据逻辑。例如唯一性字段在数据库中已存在。

针对以上的两种情况分别做了设计与实现。  

#### 数据格式校验  

dubbox在接收参数时，可以通过一个Bean进行映射。我们可以通过JSR303规范中的Bean Validation对字段进行验证。dubbox中也提供了对bean validation的支持。只需要在api层的bean中配置`validation="true"`即可， 针对不同的业务逻辑可以分为以下情况。  

##### 基本数据格式验证       

bean validation提供了一些常规的注解供我们调用，如下：    

约束注解名|约束注解说明      
---     |    ---     

@Null   |验证对象是否为空   

@NotNull    |验证对象是否为非空   

@AssertTrue |验证 Boolean 对象是否为 true   

@AssertFalse    |验证 Boolean 对象是否为 false 
@Min    |验证 Number 和 String 对象是否大等于指定的值 
@Max    |验证 Number 和 String 对象是否小等于指定的值
@DecimalMin|    验证 Number 和 String 对象是否大等于指定的值，小数存在精度
@DecimalMax |验证 Number 和 String 对象是否小等于指定的值，小数存在精度
@Size|  验证对象（Array,Collection,Map,String）长度是否在给定的范围之内
@Digits |验证 Number 和 String 的构成是否合法
@Past   |验证 Date 和 Calendar 对象是否在当前时间之前
@Future |验证 Date 和 Calendar 对象是否在当前时间之后
@Pattern    |验证 String 对象是否符合正则表达式的规则 
  
**示例代码：**  

```    
@Pattern(regexp = "^\\d{0,16}\\.\\d{0,6}|^\\d{0,16}|^\\s*$", message = "贷款金额输入非法")
private String            amount;  

@NotBlank(message = "姓名不能为空")
private String            name;  
```  

##### 业务数据格式验证    

上述的基本约束可能并不能满足于项目中业务的需求，项目中可能会有一些特定的数据格式的要求，例如身份证格式，邮箱格式等等。此时，可以利用bean validation中自定义接口的功能。    

**示例代码：**   

```  
 @Target({ METHOD, FIELD, ANNOTATION_TYPE, CONSTRUCTOR, PARAMETER }) //应用范围
 @Retention(RUNTIME) //影响范围
 @Documented 
 @Constraint(validatedBy = {IdNumberValidator.class}) //通过该类进行实际验证
 public @interface IdNumber { 
    String message() default "this string may be empty"; 
    Class<?>[] groups() default { }; 
    Class<? extends Payload>[] payload() default {}; 
 }     
 
 //验证类
public class IdNumberValidator implements ConstraintValidator<NotEmpty, String>{ 
    public void initialize(NotEmpty parameters) { 
    } 
    public boolean isValid(String string, 
        ConstraintValidatorContext constraintValidatorContext) { 
        //此处写验证身份证逻辑，如果符合则返回true。输入参数为string
    } 
 }  
```

#### 复杂逻辑验证     

在bean validation中验证一个属性中加入多个标签的话，可能会统一验证，把标签中所有标签都验证完成，输出验证结果。有的时候我需要有业务逻辑的验证，例如判断一个字段，不为空，如果为空的话，就不进行下一步的验证，如果不为空继续验证格式是否正确。  可以利用自定义Group的功能，分层次对一条字段进行验证。
```  
 //定义顺序接口
 public interface Order1 { 
 } 
 public interface Order2 { 
 } 
 @GroupSequence({Default.class, Order1.class, Order1.class}) 
 public interface Group { 
 }    
 
 //接口使用  
 @NotEmpty(message = "amount may be empty", groups = Default.class) 
 @Pattern(regexp = "^\\d{0,16}\\.\\d{0,6}|^\\d{0,16}|^\\s*$", message = "金额长度非法")
 private String amount;   
```

#### 与数据库交互的验证    

与数据库的交互验证，最多场景就是先判断一个字段在数据库中是否存在，如果存在则返回错误，不存在则业务作价继续进行。本想抽取公共方法，抽象类形式进行验证，但是同样也是要维护大量的验证代码。后来想到通过简单的sql进行拼装验证，直接走jdbc而不是通过mybatis。

##### 通过jdbctemplate连接数据库。  
```    
<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource">
            <ref local="templateDataSource" />
        </property>
    </bean>
    <bean id="templateDataSource" class="com.alibaba.druid.pool.DruidDataSource">
        <property name="driverClassName">
            <value>${database.driver}</value>
        </property>
        <property name="url">
            <value>${database.url}</value>
        </property>
        <property name="username">
            <value>${database.username}</value>
        </property>
        <property name="password">
            <value>${database.password}</value>
        </property>
        <!-- Connection Pooling Info -->
        <property name="initialSize" value="${jdbc.template.initialSize}" />
        <property name="minIdle" value="1" />
        <property name="maxActive" value="${jdbc.template.maxActive}" />
        
                <!-- 配置获取连接等待超时的时间 -->
        <property name="maxWait" value="60000" />

        <!-- 配置间隔多久才进行一次检测，检测需要关闭的空闲连接，单位是毫秒 -->
        <property name="timeBetweenEvictionRunsMillis" value="60000" />

        <!-- 配置一个连接在池中最小生存的时间，单位是毫秒 -->
        <property name="minEvictableIdleTimeMillis" value="300000" />

        <!--<property name="validationQuery" value="SELECT 'x'" />-->
        <property name="testWhileIdle" value="true" />
        <property name="testOnBorrow" value="false" />
        <property name="testOnReturn" value="false" />

        <!-- 打开PSCache，并且指定每个连接上PSCache的大小 -->
        <property name="poolPreparedStatements" value="true" />
        <property name="maxPoolPreparedStatementPerConnectionSize" value="20" />

        <!-- 配置监控统计拦截的filters -->
        <property name="filters" value="stat" />
    </bean>

```  
  
  
##### 通过PreparedStatement方式执行sql。    

PreparedStatement执行sql，是先对sql进行预编译，可以有效的防止sql注入。
示例：  
```  
 public int selectCount(String tableName,List<FieldValue> fieldValues){

        final String table = tableName;
        final List<FieldValue> fieldValueList= fieldValues;
        int count = jdbcTemplate.execute(new PreparedStatementCreator() {
            @Override
            public PreparedStatement createPreparedStatement(Connection conn)
                    throws SQLException {
                StringBuffer sql = new StringBuffer("select count(*) from ");
                sql.append(table);
                sql.append(" where ");
                int count = 1;
                for(FieldValue fieldValue:fieldValueList){
                    if(count > 1){
                        sql.append(" and ");
                    }
                    sql.append(fieldValue.getField());
                    sql.append("=");
                    sql.append("?");
                    count ++;
                }
                PreparedStatement preparedStatement = conn.prepareStatement(sql.toString());
                count = 1;
                for(FieldValue fieldValue:fieldValueList) {
                    preparedStatement.setString(count, fieldValue.getValue());
                    count++;
                }
                return preparedStatement;
            }}, new PreparedStatementCallback<Integer>() {
            @Override
            public Integer doInPreparedStatement(PreparedStatement pstmt)
                    throws SQLException, DataAccessException {
                pstmt.execute();
                ResultSet rs = pstmt.getResultSet();
                int resultCount = 0;
                while (rs.next()){
                    resultCount = rs.getInt(1);
                }
                return resultCount;
            }});

        return count;
    }    
```    

##### 调用方式     

```  
    String tableName = "tablename";
    //sql注入测试
    FieldValue fieldValue = new FieldValue();
    fieldValue.setField("field");
    fieldValue.setValue("value");
    List<FieldValue> fieldValues = new LinkedList<>();
    fieldValues.add(fieldValue);
    int count = dbTemplate.selectCount(tableName,fieldValues);  
```         




