###  Task 任务调度算法 DAG 实现

##### 1 有向图的构建 
```java
    DAG dag = new DAG();
    dag.addVertex("A");
    dag.addVertex("B");
    dag.addVertex("C");
    dag.addVertex("D");
    dag.addEdge("A", "B");
    dag.addEdge("A", "C");
    System.out.println(dag);

```
##### 2 拓扑排序检测图中是否有环
```java
 public boolean isCircularity() {
    Set<Object> set = inDegree.keySet();
    //入度表
    Map<Object, AtomicInteger> inDegree = set.stream().collect(Collectors
        .toMap(k -> k, k -> new AtomicInteger(this.inDegree.get(k).size())));
    //入度为0的节点
    Set sources = getSources();
    LinkedList<Object> queue = new LinkedList();
    queue.addAll(sources);
    while (!queue.isEmpty()) {
      Object o = queue.removeFirst();
      outDegree.get(o)
          .forEach(so -> {
            if (inDegree.get(so).decrementAndGet() == 0) {
              queue.add(so);
            }
          });
    }
    return inDegree.values().stream().filter(x -> x.intValue() > 0).count() > 0;
  }
``` 

##### 4 stage优化
```text
  eg   
  如果任务存在如下的关系 ， task1 执行完后执行 task2 ,task2 执行完后执行task3 ...     
  Task1 -> Task2 -> Task3 -> Task4 
  这些task 本来就要串行执行的 可以把这些task 打包在一块 减少线程上下文的切换  
  
  eg : 复杂一点的DAG:
   /**
     *  H
     *    \
     *      G
     *        \
     *     A -> B
     *            \
     *  C- D  -E  - F-> J
     *
     *
     *
     *    优化后得  ==>
     *
     *     (H,G)
     *         \
     *     A -> B
     *            \
     *  (C,D,E)  - (F,J)
     *
     */
     
      详见chain方法： 关键代码如下
     
     private void chain_(Set sources, final LinkedHashSetMultimap foutChain, final LinkedHashSetMultimap finChain) {
         sources.forEach(sourceNode -> {
     
           ArrayList<Object> maxStage = Lists.newArrayList();
           findMaxStage(sourceNode, maxStage);
           if (maxStage.size() > 1) { //存在需要合并的stage
             addVertex(foutChain, finChain, maxStage);//添加一个新节点
             Object o = maxStage.get(maxStage.size() - 1); //最后一个节点
             reChain_(foutChain, finChain, maxStage, o);
           }
           if (maxStage.size() == 1) {
             //不存在需要合并的stage
             addVertex(foutChain, finChain, sourceNode);//添加一个新节点
             Set subNodes = outDegree.get(sourceNode);
             addSubNodeage(foutChain, finChain, sourceNode, subNodes);
           }
         });
       }
     
     
     
      
     
4 测试DAG 执行
  
  测试程序: 详见 DAGExecTest
  1 新建一个task  只打印一句话
  public static class Task implements Runnable {
 
     private String taskName;
 
     public Task(final String taskName) {
       this.taskName = taskName;
     }
 
     @Override public void run() {
       try {
         Thread.sleep(2000);
       } catch (InterruptedException e) {
         e.printStackTrace();
       }
       System.out.println("i am running  my name is " + taskName + "  finish ThreadID: " + Thread.currentThread().getId());
     }
 
     public String getTaskName() {
       return taskName;
     }
 
     @Override public String toString() {
       return  taskName;
     }
   }
   2 构建DAG 
       DAG dag = DAG.create();
       Task a = new Task("a");
       Task b = new Task("b");
       Task c = new Task("c");
       Task d = new Task("d");
       Task e = new Task("e");
       Task f = new Task("f");
       Task g = new Task("g");
       Task h = new Task("h");
       Task j = new Task("j");
       dag.addVertex(a);
       dag.addVertex(b);
       dag.addVertex(c);
       dag.addVertex(d);
       dag.addVertex(e);
       dag.addVertex(f);
       dag.addVertex(g);
       dag.addVertex(h);
       dag.addVertex(j);
       dag.addEdge(h, g);
       dag.addEdge(g, b);
       dag.addEdge(a, b);
       dag.addEdge(b, f);
       dag.addEdge(c, d);
       dag.addEdge(d, e);
       dag.addEdge(e, f);
       dag.addEdge(f, j);
     构建完成后如图
          *   H
          *    \
          *      G
          *        \
          *     A -> B
          *            \
          *  C- D  -E  - F-> J
          
    3 stage 切分  
    DAG chain = dag.chain();
    执行完图入下：
         *     (H,G)
         *         \
         *     A -> B
         *            \
         *  (C,D,E)  - (F,J)
         
    4 执行 DAG  DAGExecTest   最终结果打印如下如下：
     
     可以发现有3个Stage   stage1  包含3个task  task分别在不同的线程里面执行  
     其中c-d-e   g-c  f-j是经过优化的在同一个线程里面执行，减少了不必要的上下文切换 
    
      i am running  my name is a  finish ThreadID: 10
      i am running  my name is c  finish ThreadID: 11
      i am running  my name is h  finish ThreadID: 12
      i am running  my name is d  finish ThreadID: 11
      i am running  my name is g  finish ThreadID: 12
      i am running  my name is e  finish ThreadID: 11
      stage 结束 ：  task detached:a, task chain c-d-e task chain h-g
      -----------------------------------------------
      i am running  my name is b  finish ThreadID: 14
      stage 结束 ：  task detached:b,
      -----------------------------------------------
      i am running  my name is f  finish ThreadID: 11
      i am running  my name is j  finish ThreadID: 11
      stage 结束 ：  task chain f-j
      测试执行关键代码如下：
      chain.execute(col -> {
            Set set = (Set) col;
            List<CompletableFuture> completableFutures = Lists.newArrayList();
            StringBuilder sb = new StringBuilder();
            set.stream().forEach(x -> {
              if (x instanceof Task) {
                CompletableFuture<Void> future = CompletableFuture.runAsync((Task) x, executorService);
                completableFutures.add(future);
                sb.append(" task detached:" + ((Task) x).getTaskName()).append(",");
              }
              if (x instanceof List) {
                List<Task> taskList = (List) x;
                CompletableFuture<Void> future = CompletableFuture.runAsync(()->
                  taskList.forEach(Task::run));
                completableFutures.add(future);
                sb.append(
                    " task chain " + Joiner.on("-").join(taskList.stream().map(Task::getTaskName).collect(Collectors.toList())));
              }
            });
            CompletableFuture.allOf(completableFutures.toArray(new CompletableFuture[completableFutures.size()])).join();
            System.out.println("stage 结束 ： " + sb.toString());
            System.out.println("-----------------------------------------------");
          });
      
      mail:787069354@qq.com
```



