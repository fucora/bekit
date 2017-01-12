# bekit工具箱
1. 简介

> 本来想叫框架，但现在还上升不到那样的高度，就暂时称之为工具箱。本工具箱致力于解决在应用开发中公共性的问题，目前只有流程引擎一个功能，后续会陆续增加其他功能。

2. 环境要求：

> * jdk1.8
> * Spring 4.2及以上


> 注意：本工具已经上传到maven中央库

## 流程引擎
流程引擎比较适合订单类业务场景，功能包含流程编排和事务控制。
> 诞生原因：在订单类业务场景中，一条订单从创建到最后完成期间，往往会经历很多个状态，状态我将之称为节点，状态和状态的流转我将它称之为流程。在开发的时候，我们往往是将流程嵌入到处理过程代码中。这样做的缺点就是会将流程和处理过程耦合在一起，即不方便查看整个流程，也不方便维护处理过程。当一种新流程需要相同的处理过程时，那么这个处理过程里面就嵌入了两种流程，这样就更
不好维护了。

> 为了解决这种问题，我们势必要将流程从处理过程中剥离出来。但是流程通过什么形式来定义，以及每个节点和它对应的处理过程怎么进行关联，还有整个过程中什么时候应该新启事务，什么时候应该提交事务。为了解决这些问题，流程引擎就有了存在的价值。

### 1. 将流程引擎引入进你的系统：
1. 引入流程引擎依赖

        <dependency>
            <groupId>top.bekit</groupId>
            <artifactId>flow</artifactId>
            <version>1.0.0.RELEASE</version>
        </dependency>

2. 如果是spring-boot项目则只需要在application.properties中加入配置：

        bekit.flow.enable=true
        
    如果是非spring-boot项目则需要手动引入流程引擎配置类FlowEngineConfiguration，比如：
    
        @Configuration
        @Import(FlowEngineConfiguration.class)
        public class MyImport {
        }

3. 在需要使用的地方注入流程引擎FlowEngine：

        @Autowired
        private FlowEngine flowEngine;


    然后就可以调用执行流程方法：
    
        // demoFlow是需要执行的流程名称，trade是这个流程需要执行的目标对象（可以是订单、交易记录等等）
        flowEngine.start("demoFlow", trade);
    
### 2. 一个简单的流程定义样例：
一个完整的流程应该具备：流程编排（@Flow）、处理器（@Processor）、流程监听器（@FlowListener）、流程事务（@FlowTx）

1. #### 流程编排

    流程编排的职责就是定义一个流程中所有节点，并把这些节点之间的流转通过节点选择方法表示出来。


        @Flow  // 流程注解（流程名称默认就是类名首字母小写，也可以自己在@Flow注解里面自己指定。本流程的名称为：demoFlow）
        public class DemoFlow {
        
            @StartNode      // 开始节点，一个流程必须得有一个开始节点
            public String start()
                // 返回下一个节点名称。节点名称默认情况下就是对应的方法名，不过也可以在节点注解上指定name属性，自己指定名称
                return "node1";
            }
        
            @ProcessNode(processor = "node1Processor")      // 处理节点（一种节点类型），属性processor指定这个节点对应的处理器
            public String node1(String processResult) {// 入参processResult是本节点对应的处理器处理后的返回结果，它的类型需要满足能被处理器返回值进行赋值就可以
                // 本方法叫做“下个节点选择方法”，意思是本节点处理器已经处理完了，现在需要选择下一个节点进行执行，返回参数就是下一个节点名称。
                // 一般下一个节点选择是根据处理器返回结果来选择。
                // 比如：处理器返回转账成功了就表示这笔业务成功了，则应该进入成功节点；如果转账失败了就表示这笔业务失败了，则应该进入失败节点；如果转账被挂起了，那么就只能停留在当前节点直到得到明确结果为止，这个时候就可以返回null，停止流程。
                // 这里为简单演示，直接返回下一个节点
                return "node2";
            }
        
            @StateNode(processor = "node2Processor")        // 状态节点
            public String node2(String processResult) {
                return "node3";
            }
        
            @WaitNode(processor = "node3Processor")         // 等待节点（下面会细说处理节点、状态节点、等待节点的区别）
            public String node3(){
                // 也可以没有processResult入参
                return "node4";
            }
        
            @ProcessNode(processor = "node4Processor")
            public String node4(String processResult) {
                return "end";
            }
        
            @EndNode        // 结束节点（一个流程必须有一个结束节点）
            public void end() {
            }
            
            @TargetMapping      // 目标对象映射
            public String targetMapping(Object target){ // 入参target是你传给流程引擎的那个目标对象
                // 将目标对象（比如订单）映射到需要执行的节点
                
                // 流程引擎并不知道当前这个目标对象应该从哪个节点开始执行（是从开始节点，还是从node1，还是从node2等等）。
                // 因为流程引擎并不知道这个目标对象是新建的，还是半路从数据库捡起来扔给流程引擎来执行的，
                // 所以需要你来告诉流程引擎现在应该从哪个节点开始执行，一般你要做的就是根据目标对象当前状态判断当前应该执行哪个节点。
                // 返回值就是需要执行的节点名称。
                // 下面的“XXX”表示逻辑判断
                if (XXX){
                    return "start";
                }else if (XXX){
                    return "node3";
                }else if (XXX){
                    return "node4";
                }else {
                    return "end";
                }
            }
        }
 
    流程通过@Flow注解标识，流程中的节点分为开始节点（@StartNode）、处理节点（@ProcessNode）、状态节点（@StateNode）、等待节点（@WaitNode）、结束节点（@EndNode）这几种类型。
    
    流程在开始前会自动的开启一个新事务并调用流程事务锁住目标对象（下面会介绍），流程正常执行结束后（无异常抛出），则会提交事务；否则如果有异常抛出，则会回滚事务（如果前面有状态节点，则已经提交的哪些事务是回滚不了的。要的就是这个效果）。
    
    每个流程都必须有一个开始节点（@StartNode）和一个结束节点（@EndNode）；处理节点（@ProcessNode）执行完成后不会提交事务；状态节点处理完后会提交事务，然后会再开启一个新事务并锁住目标对象；当自动执行到需要执行等待节点时（还未执行）流程会自动停止，如果需要执行等待节点，则需要手动的调用流程引擎的start方法（这符合等待异步通知这类场景）。
 
    除了开始节点和结束节点，其他类型的节点都可以通过processor属性指定一个处理器（真正干活的，就是前面说的处理过程）。
    
    每个流程还必须有一个目标对象映射，它是用来让流程引擎知道应该从哪个节点开始执行。
    
> @Flow注解有个属性enableFlowTx，它是用来控制是否开启流程事务，默认是开启。如果设置为false，则流程引擎不会在事务上做任何控制（既不会主动开启事务，也不会主动提交事务，@StateNode节点效果也会跟@ProcessNode节点效果一样），当然下面要介绍的流程事务（@FlowTx）你也不需要使用了。
 
2. #### 处理器

    处理器是功能单一的执行某个节点的任务，一个处理器可以被多个节点共同使用。

        @Processor // 处理器注解（处理器名称默认就是类型首字母小写，也可以在@Processor注解里面自己指定。本处理器的名称：node1Processor）
        public class Node1Processor {
            
            @Before  // 业务前置处理，可选    
            public void before(TargetContext targetContext) { // TargetContext叫做目标上下文，可以通过他获取你传给流程引擎的目标对象（targetContext.getTarget()）
                // 最先执行
                // 可以做一些检查之类的操作
            }
        
            @Execute  // 业务处理，必选
            public String execute(TargetContext targetContext) {
                // 在@before类型方法之后执行
                // 这个是处理器的主体方法，一般就是真正干活的方法，它的返回值会成为整个处理器的返回值，然后传给流程节点中的下个节点选择方法
                return "success";
            }
        
            @After  // 业务后置处理，可选
            public void after(TargetContext targetContext) {
                // 在@Execute类型方法之后执行
                // 可以做一些结果校验之类的
            }
        
            @End    // 业务结束处理，可选
            public void end(TargetContext targetContext) {
                // 最后执行（即使发生异常也会执行）
            }
        
            @Error   // 异常处理，可选
            public void error(TargetContext targetContext) {
                // 当@before、@execute、@after类型方法任何一个发生异常后会执行
            }
        }

    处理器通过@Processor进行注解，根据可能存在的需求将处理器方法分成了5种类型，只有@Execute类型方法是必须有的，其他都是可选的。
    
    处理器方法可以没有入参，也可以有入参。有入参的话只能有一个入参且类型必须是TargetContext。TargetContext是目标上下文，可以通过它获取你传给流程引擎的目标对象（targetContext.getTarget()）
    
    处理器执行过程中发生任何异常都会往外抛。一个处理器可以同时被多个流程使用。

3. #### 流程监听器

        流程监听器的职责就是监听某一个流程内发生的事件，目前只有一种类型事件————节点选择事件（这种事件一般的作用就是用来更新目标对象的状态）

        @FlowListener(flow = "demoFlow")  // 流程监听器注解，属性flow指定被监听的流程名称
        public class DenoFlowListener {
        
            @ListenNodeDecide(nodeExpression = "node1")     // 监听节点选择事件，nodeExpression是需要被监听节点的正则表达式
            public void listenNode1(String node, TargetContext targetContext) { // node是被选择的节点名称，targetContext是目标上下文
                // 监听流程节点中的节点选择方法被执行后的返回值
                // 一般监听节点选择事件的目的是用来更新目标对象的状态，
                // 因为当节点选择事件发生时，就表明已经执行完当前节点，即将进入到下一个节点，
                // 所以目标对象的状态应该修改为下一个状态
                
                // 本监听方法只有在被选择节点是node1时才会执行
            }
        
            @ListenNodeDecide(nodeExpression = ".*")     // 监听节点选择事件，正则表达式“.*”表示监听所有的节点选择事件
            public void listenAllNode(String node, TargetContext targetContext) {
                // 本监听方法在所有节点被选择是都会执行
            }
        }

    流程监听器可以监听对应流程发生的事件。目前只有节点被选择事件，此事件的作用是可以是用来更新目标对象状态，实现更新状态和流程分离；也可以干其他事，就看你需求是怎么的了。
    
    监听方法的要求是要么没入参，要么入参是（String, TargetConstext）

4. #### 流程事务

    流程事务的职责就是对某一个流程事务上的操作定义，目前只有两种：锁目标对象、插入目标对象

        @FlowTx(flow = "demoFlow") // 流程事务注解，属性flow指定本流程事务所对应的流程
        public class DemoFlowTx {
        
            @LockTarget     // 锁目标对象，必选
            public Trade lockTarget(TargetContext targetContext) { // 目标上下文
                // 因为在并发情况下需要锁住目标对象下才能保证不会出错，
                // 流程引擎知道什么时候应该锁目标对象，但是流程引擎并不知道怎么锁住目标对象（不知道数据库表等信息）
                // 所以需要你在这儿指定锁目标对象的具体代码
                
                // 锁住的目标对象后必须返回被锁后的目标对象，流程引擎需要将它更新到目标上下文，
                // 因为当你锁住目标对象后，里面的内容可能已经发生变化了
            }
        
            @InsertTarget   // 插入目标对象，可选
            public Trade insertTarget(TargetContext targetContext) {
                // 插入目标对象到数据库具体代码
                // 必须返回插入后的目标对象
            }
        }

    流程事务必须包括锁目标对象类型方法（@LockTarget），流程在开始执行前会先开启一个新事务同时立马调用@LockTarget方法锁住目标对象，当遇到状态节点执行完时会提交事务，接着会再开启新事务并再调用@LockTarget方法锁住目标对象。目的就是为了目标对象在被执行时是被锁住状态

    插入目标对象方法（@InsertTarget）的作用是开启一个新事务用来插入目标对象到数据库并提交。它存在的原因：如果在调用流程引擎前自行将目标对象插入到数据库但未提交，而流程引擎是新启事务后再锁对象，这样会导致流程引擎锁不到目标对象。所以流程引擎留一个口子专门新启一个事务让用户来插入目标对象。（需要调用流程引擎的flowEngine.insertTargetAndStart方法）

>特别注意：如果你自己先把目标对象锁了，再调用流程引擎执行流程，那么流程引擎在开启新事务后锁目标对象时就会出现死锁，所以你在调用流程引擎前不能把目标对象锁了，这个千万要注意！！！


