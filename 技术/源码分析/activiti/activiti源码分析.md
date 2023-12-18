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

### 二、PVM的实现
pvm，流程虚拟机，bpmn5的执行的核心，在activiti中明确一下几个类的定义以及职责：
PvmProcessDefinition：流程的定义，形象点说就是用户画的那个图。静态含义。
PvmProcessInstance：流程实例，用户发起的某个PvmProcessDefinition的一个实例，动态含义。
PvmActivity：流程中的一个节点
PvmTransition：衔接各个节点之间的路径，形象点说就是图中各个节点之间的连接线。
PvmEvent：流程执行过程中触发的事件

从startProcessInstanceByKey这个方法开始看，最终执行者是StartProcessInstanceCmd的execute，这个方法，主要做了三流程个操作，通过DeploymentManager创建一个ProcessDefinitionEntity，然后通过ProcessDefinitionEntity创建一个ExecutionEntity，然后ExecutionEntity执行start。

我们先从start看起，从下面代码可以看到，AtomicOperation是控制执行顺序的逻辑，ActivityBehavior是真正的执行者。
```
  public void start() {
    if(startingExecution == null && isProcessInstanceType()) {
      startingExecution = new StartingExecution(processDefinition.getInitial());
    }
    performOperation(AtomicOperation.PROCESS_START);
  }

    public void performOperation(AtomicOperation executionOperation) {
    if (executionOperation.isAsync(this)) {
      scheduleAtomicOperationAsync(executionOperation);
    } else {
      performOperationSync(executionOperation);
    }    
  }
  
  protected void performOperationSync(AtomicOperation executionOperation) {
    Context
      .getCommandContext()
      .performOperation(executionOperation, this);
  }

  public void performOperation(AtomicOperation executionOperation, InterpretableExecution execution) {
    nextOperations.add(executionOperation);
    if (nextOperations.size()==1) {
      try {
        Context.setExecutionContext(execution);
        while (!nextOperations.isEmpty()) {
          AtomicOperation currentOperation = nextOperations.removeFirst();
          if (log.isTraceEnabled()) {
            log.trace("AtomicOperation: {} on {}", currentOperation, this);
          }
          if (execution.getReplacedBy() == null) {
            currentOperation.execute(execution);
          } else {
            currentOperation.execute(execution.getReplacedBy());
          }
        }
      } finally {
        Context.removeExecutionContext();
      }
    }
  }

public void execute(InterpretableExecution execution) {
    ActivityImpl activity = (ActivityImpl) execution.getActivity();
    
    ActivityBehavior activityBehavior = activity.getActivityBehavior();
    if (activityBehavior==null) {
      throw new PvmException("no behavior specified in "+activity);
    }

    log.debug("{} executes {}: {}", execution, activity, activityBehavior.getClass().getName());
    
    try {
      if(Context.getProcessEngineConfiguration() != null && Context.getProcessEngineConfiguration().getEventDispatcher().isEnabled()) {
        Context.getProcessEngineConfiguration().getEventDispatcher().dispatchEvent(
            ActivitiEventBuilder.createActivityEvent(ActivitiEventType.ACTIVITY_STARTED, 
                execution.getActivity().getId(),
                (String) execution.getActivity().getProperty("name"),
                execution.getId(), 
                execution.getProcessInstanceId(), 
                execution.getProcessDefinitionId(),
                (String) activity.getProperties().get("type"),
                activity.getActivityBehavior().getClass().getCanonicalName()));
      }
      
      activityBehavior.execute(execution);
    } catch (RuntimeException e) {
      throw e;
    } catch (Exception e) {
      LogMDC.putMDCExecution(execution);
      throw new PvmException("couldn't execute activity <"+activity.getProperty("type")+" id=\""+activity.getId()+"\" ...>: "+e.getMessage(), e);
    }
  }
  ```