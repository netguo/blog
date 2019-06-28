#### 一、 activiti的命令模式
我们从一个具体的实例类看起，RepositoryServiceImpl的实例类
[RepositoryServiceImpl的类图](./activiti/respositoryService.png)
可以看到，RepositoryServiceImpl实现了接口RepositoryService，继承了ServiceImpl。
其中RepositoryService定义了自身域职责(流程部署)的所有方法，是域职责的定义接口。  

ServiceImpl，内部有两个依赖，commandExecutor、processEngineConfiguration，望文生义，可以理解到，ServiceImpl集成了工作流的配置信息，是命令模式的实现的根类。

我们在看，命令模式在具体实现类里怎么实现的。

```
public interface CommandExecutor {
  
  CommandConfig getDefaultConfig();

  <T> T execute(CommandConfig config, Command<T> command);

  <T> T execute(Command<T> command);
  
}

public class CommandExecutorImpl implements CommandExecutor {

  private final CommandConfig defaultConfig;
  private final CommandInterceptor first;
  
  public CommandExecutorImpl(CommandConfig defaultConfig, CommandInterceptor first) {
    this.defaultConfig = defaultConfig;
    this.first = first;
  }
  
  public CommandInterceptor getFirst() {
    return first;
  }

  @Override
  public CommandConfig getDefaultConfig() {
    return defaultConfig;
  }
  
  @Override
  public <T> T execute(Command<T> command) {
    return execute(defaultConfig, command);
  }

  @Override
  public <T> T execute(CommandConfig config, Command<T> command) {
    return first.execute(config, command);
  }
```
可以看到commandExecutor的实现类CommandExecutorImpl，其中引用了一个CommandInterceptor，命令拦截器，命令的执行是通过拦截器执行的。真正的执行者是实现了Command的类，这个拦截器的作用是什么呢。？
[CommandInterceptor的实现类](./activiti/CommandInterceptor.png)

可以看到CommandInterceptor的实现类有日志、分布式事务、事务、上下文、command。从下面CommandInterceptor，可以理解到拦截器，类似我们常见的filter，执行完，再执行next，在看CommandInvoker的逻辑，如果有getNext直接抛异常，从异常的定义可以理解到CommandInvoker是最后一个拦截，并且直接执行命令。

```
public interface CommandInterceptor {

  <T> T execute(CommandConfig config, Command<T> command);
 
  CommandInterceptor getNext();

  void setNext(CommandInterceptor next);

}

public class CommandInvoker extends AbstractCommandInterceptor {

  @Override
  public <T> T execute(CommandConfig config, Command<T> command) {
    return command.execute(Context.getCommandContext());
  }

  @Override
  public CommandInterceptor getNext() {
    return null;
  }

  @Override
  public void setNext(CommandInterceptor next) {
    throw new UnsupportedOperationException("CommandInvoker must be the last interceptor in the chain");
  }

}

```

从头梳理一下，举一个具体实例，RepositoryServiceImpl执行deploy，那么会先通过CommandExecutor传递DeployCmd，然后执行各个拦截器，最后命令拦截器执行DeployCmd的execute操作，该execute里面定义了deploy的具体业务操作。

小结：总结一下命令模式在activiti中使用，领域的service接口定义领域内的操作，每个操作都对应一个命令类。每一个命令执行都有拦截器，对应日志事务处理等。这样统一了activiti的编程风格，不过与我们日常的mvc下的编程风格有挺大的区别的。

