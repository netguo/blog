### 一、PVM的实现
pvm，流程虚拟机，bpmn5的执行的核心，在activiti中明确一下几个类的定义以及职责：
PvmProcessDefinition：流程的定义，形象点说就是用户画的那个图。静态含义。
PvmProcessInstance：流程实例，用户发起的某个PvmProcessDefinition的一个实例，动态含义。
PvmActivity：流程中的一个节点
PvmTransition：衔接各个节点之间的路径，形象点说就是图中各个节点之间的连接线。
PvmEvent：流程执行过程中触发的事件

从startProcessInstanceByKey这个方法开始看，最终执行者是StartProcessInstanceCmd的execute，这个方法，主要做了三流程个操作，通过DeploymentManager创建一个ProcessDefinitionEntity，然后通过ProcessDefinitionEntity创建一个ExecutionEntity，然后ExecutionEntity执行start。
从类的命名，以及这个流程定义可以看到

```
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
  ```