[TOC]
## 架构
#### （一）Actor和Component
从实体的组装方式来说，unity讲究的是全员皆是gameobject，就像是一个个node来组装实体，每一个node都可以取挂在component，UE4讲究的是使用Actor将实体封装起来，组装的坐标关系使用secenecomponent表示，并不使用node节点表示，归根结底就是Unity允许将逻辑细化到每一个node上面，而UE4提倡我们在actor中构建逻辑，并不提倡逻辑上的细分，避免了层级过深，层层转发带来的效率降低。  
关键的不同是在于你是怎么划分要操作的实体的粒度的

在unity的程序设计中我们都是依赖于mono来进行生命周期，GC的管理，而mono又需要依赖GameObject挂载在场景总才可以生效，好处就是在unity的代码逻辑中你会写的十分轻松便捷，有什么功能需要实现，就地加一个脚本，变量的引用也可以才场景中快速的绑定，但当在没有约束的情况下写了太多的代码，就会管理越来越困难，耦合越来越多，逻辑也会十分分散，一般在unity中开发时，需要我们自己管理限制自己的代码，避免造成上述情况，ue为了避免这种情况，采取了和unity不同的脚本管理逻辑，就是使用提供的框架，避免逻辑混乱和分散，提倡在actor中构建逻辑，其实总结下来，Actor就像一个Component的容器，这个实体需要的组件都放到Actor里，将scencecomponent单独抽离出来，组织层级关系，并不是想unity中同一个实体需要的component可能存在于不同层级下的GameObject中，避免了频繁转发和混乱，当然ue中也有actorcomponent这样类似GameObject的结构。



##### UObject

UObject是我们一个最基础的类，不管是我们各种资源，还是场景中的AActor，都是继承我们的UObject实现的，所以从Broswer中加载对象时，经常会看到用UObject来保存我们加载好的指针，然后再去实例。

![](https://pic3.zhimg.com/80/v2-750c05a282e8784c3af5815a481d549e_720w.png)

藉着UObject提供的元数据、反射生成、GC垃圾回收、序列化、编辑器可见，Class Default Object等，UE可以构建一个Object运行的世界。（后续会有一个大长篇深挖UObject）



##### AActor

![](https://pic3.zhimg.com/80/v2-9348f6bdadbe382a505aff9be7d5d99e_720w.png)

脱胎自Object的Actor也多了一些本事：Replication（网络复制）,Spawn（生生死死），Tick(有了心跳)。

之前提到Actor的Location是单独抽离到sceneComponent的，这是考虑到有很多Actor是不需要位置信息的，为了避免逆矩阵的内存占用，所以将sceneComponent封装了起来，如果spawn出一个AActor的话，会发现是没有sceneComponent的，提供给我们的GetActorLoaction的API也是通过先获取RootCommponent来Get  Location的  

同理，Actor能接收处理Input事件的能力，其实也是转发到内部的UInputComponent* InputComponent;同样也提供了便利方法。



##### Component

Component是类似unity组件模式的应用，UActorComponent也是基础于UObject的一个子类，这意味着其实Component也是有UObject的那些通用功能的。所以Component也是有反射，GC回收等功能的。

TSet<UActorComponent*> OwnedComponents 保存着这个Actor所拥有的所有Component,一般其中会有一个SceneComponent作为RootComponent。

以下是SceneComponent的周嵌套关系



![](https://pic1.zhimg.com/80/v2-91234c7d5bc32dd04c7221ac9dcc56d0_720w.jpg)

从上图可以看到，只有ScenceComponent可以进行嵌套，在UE的观念里，好像只有带Transform的SceneComponent才有资格被嵌套，好像Component的互相嵌套必须和3D里的transform父子对应起来。所以想AIComponent，inputComponent这样没有从sceneComponent继承，也不需要位置信息的类，都是无法嵌套的，这样设计的原因还是因为“不为你不需要的东西付出代价”的思维，因为他们不需要位置信息，所以不去使用一个逆矩阵存储Transform，因为没有Transform和位置的必要，所以也不必提供嵌套功能。



##### ChildActorComponent

再Actor中除了存储Component的两个数组之外还有一个TArray<AActor*> Children字段，就是保存actor的父子关系的，那么父子关系有什么用呢？

那就是提供组件之外的，空间内的坐标绑定关系，比如武器，我们可能持有多种武器，但不可能将每种都作为Charactor的组件加入进去，就需要使用childActorComponent，再我们设计好武器后，再将他绑定到需要的Charactor或者Enemy手上。

除了基本从SceneComponent继承的诸如StaticMesh的类之外，ChildActorComponent也需要我们的嵌套关系，他给我们提供了在Actor下再添加Actor的功能，




#### （二）Level和World
简单来说就是在常见的Level上，添加了一个World，World就像是一个世界的规则，在游戏世界里就是游戏的规则，在Level上添加了一个逻辑层面。

##### Level

Level继承UObject，所以也具有蓝图的功能，允许我们在关卡里编写脚本，可以对本关卡里的所有Actor通过名字呼之则来，关卡蓝图实际上就代表着该片大陆上的运行规则。  
同时我们需要对level的信息进行记录我，由此引进了AWorldSettings。AWorldSettings仍然是继承自Actor，但是因为没有SceneComponent，所以并不在场景中显示，虽然它叫做WorldSetting，但是和world没有太大的关心，是记录Level信息的。  
当一个level被添加进World后，这个Level的Setting如果是PersistentLevel，那它就会被当作整个World的WorldSettings。注意，Actors里也保存着AWorldSettings和ALevelScriptActor的指针，所以Actors实际上确实是保存了所有Actor。  
WorldSetting 是存在Actors[0]的位置，这是因为Actors们的排序依据是把那些“非网络”的Actor放在前面，而把“网络可复制”的Actor们放在后面，然后加一个起始索引标记iFirstNetRelevantActor，相当于为网络Actor划分了一个缓存，从而加速了网络复制时的检测速度。AWorldSettings因为都是静态的数据提供者，在游戏运行过程中也不会改变，不需要网络复制，所以也就可以一直放在前列，
如果一直放在第一个的话，就可以将AWorldSettings和其他的前列Actor们再度区分开，在需要的时候也能加速判断。  

##### world
一个world里可以有多个Level ，UE里每个World支持一个PersistentLevel和多个其他Level：Persistent的意思是一开始就加载进World，Streaming是后续动态加载的意思。Levels里保存有所有的当前已经加载的Level，StreamingLevels保存整个World的Levels配置列表。PersistentLevel和CurrentLevel只是个快速引用。在编辑器里编辑的时候，CurrentLevel可以指向其他Level，但运行时CurrentLevel只能是指向PersistentLevel。   

**思考：Levels们的Actors和World有直接关系吗？**
当别的Level被添加进当前World之后，我们能直接在WorldOutliner里看到其他Level的Actor们。
![image](https://pic3.zhimg.com/80/v2-2b8060bff22a403eb19bf1efb191b1da_1440w.png)  
但是并不代表World直接引用了Level里的Actor们，TActorIteratorBase（World的Actor迭代器）内部的实现也只是在遍历Levels来获得所有Actor。当然World为了更快速的操作Controllers和Pawn也都保存了引用。但Levels却共享着World的一个PhysicsScene，这也意味着Levels里的Actors的物理实体其实都是在World里的，这也好理解，毕竟物理的碰撞之类的当然要是全局的了。再说到导航，World在拼接Level的时候，也是会同时把两个Level的导航网格给“拼接”起来的。当然目前还不是深入细节的时候，现在只要从大局上明白World-Level-Actor的关系。


#### （三）WorldContext，GameInstance，Engine
##### WorldContext
当我们的游戏世界够庞大后，我们的就可以穿越到另一个世界，所以world并不是唯一的，而UE用来管理和跟踪这些World的工具就是WorldContext：  

FWorldContext保存着ThisCurrentWorld来指向当前的World。而当需要从一个World切换到另一个World的时候（比如说当点击播放时，就是从Preview切换到PIE），FWorldContext就用来保存切换过程信息和目标World上下文信息。所以一般在切换的时候，比如OpenLevel，也都会需要传FWorldContext的参数。一般就来说，对于独立运行的游戏，WorldContext只有唯一个。而对于编辑器模式，则是一个WorldContext给编辑器，一个WorldContext给PIE（Play In Editor）的World。一般来说我们不需要直接操作到这个类，引擎内部已经处理好各种World的协作。
不仅如此，同时FWorldContext还保存着World里Level切换的上下文：  
粗略的流程是UE在OpenLevel的时候， 先设置当前World的Context上的TravelURL，然后在UEngine::TickWorldTravel的时候判断TravelURL非空来真正执行Level的切换。具体的Level切换详细流程比较复杂，目前先从大局上理解整体结构。总而言之，WorldContext既负责World之间切换的上下文，也负责Level之间切换的操作信息。  

##### Tank项目中，之所以Level切换时，之所以数据会被丢失，是因为OpenLevel的逻辑是关闭当前word，重新生成一个world再载入Level，如果使用LoadStreamLevel的时候，就只是在当前的World中载入对象了，所以其实就没有这个限制了。

##### GameInstance
那么这些WorldContexts又是保存在哪里的呢？追根溯源：
![image](https://pic4.zhimg.com/80/v2-36b45a7b36ac77d978719bc6fe8db17b_720w.png)  
GameInstance里会保存着当前的WorldConext和其他整个游戏的信息。明白了GameInstance是比World更高的层次之后，我们也就能明白为何那些独立于Level的逻辑或数据要在GameInstance中存储了。
这一点其实也很好理解，大凡游戏引擎都会有一个Game的概念，不管是叫Application还是Director，它都是玩家能直接接触到的最根源的操作类。而UE的GameInstance因为继承于UObject，所以就拥有了动态创建的能力，所以我们可以通过指定GameInstanceClass来让UE创建使用我们自定义的GameInstance子类。所以不论是C++还是BP，我们通常会继承于GameInstance，然后在里面编写应用于整个游戏范围的逻辑。
因为经常有初学者会问到：我的Level切换了，变量数据就丟了，我应该把那些数据放在哪？再清晰直白一点，GameInstance就是你不管Level怎么切换，还是会一直存在的那个对象！



##### GamePlayStatics

既然我们在引擎内部C++层次已经有了访问World操作Level的能力，那么在暴露出的蓝图系统里，UE为了我们的使用方便，也在Engine层次为我们提供了便利操作蓝图函数库。

```text
UCLASS ()
class UGameplayStatics : public UBlueprintFunctionLibrary 
```

我们在蓝图里见到的GetPlayerController、SpawActor和OpenLevel等都是来至于这个类的接口。这个类比较简单，相当于一个C++的静态类，只为蓝图暴露提供了一些静态方法。在想借鉴或者是查询某个功能的实现时，此处往往会是一个入口。


#### （四）Pawn

#### （五）controller

Controller特别是PlayerController，跟网络，AI和Input的关系都非常的紧密，是GanePlay架构中是非常重要的一部分。

想要理解UE4的架构，首先要熟悉设计模式  

##### MVC
言归正传，设计模式的本质就是抽象变化。如果依照纯朴的"程序=数据+算法"的结构来看，再算上用于用户显示和输入的界面，那么就得到“程序=数据+算法+显示”。这三大基本块（数据，算法，显示）构成了程序的三大变化，而如何把这三者“+”到一起，用的就是我们的种种设计框架模式。
典型的，对于游戏：

“显示”指的是游戏的UI，是屏幕上显示的3D画面，或是手柄上的输入和震动，也可以是VR头盔的镜片和定位，是与玩家直接交互的载体；
“数据”指的是Mesh，Material，Actor，Level等各种元素组织起来的内存数据表示；
“算法”可以是各种渲染算法，物理模拟，AI寻路，本文咱们就先暂时特指游戏开发者们编写的游戏业务逻辑。  
抽象这三个变化，并归纳关系，就是典型的MVC模式了：  
![image](https://pic2.zhimg.com/80/v2-6bfd369c2f163d0fd730bb7db8c5a7f9_720w.png)  
**MVC并不是UI的专属，游戏的逻辑也需要使用设计模式来实现自己的架构，只不过之前接触的Unity引擎的mono脚本相当与将MC放到了一起，之所以这样设计是因为有时候一个简单的demo并不需要太多设计模式的限制，更快更顺手的开发才是王道，而复杂的大型游戏，需要的知识也并不是简单的MVC可以满足**  

##### AController
AController是继承自AActor的一个子类  

首先可以设想先我们对于一个玩家控制器有哪些要求
能够和Pawn对应起来，理想情况下，极端的灵活性应该是多对多。

1. 我希望我能同时控制多个Pawn，当然，一个Pawn也可以被多个我的兄弟姐妹们一起控制。想想那些RTS游戏和多人协作游戏，你应该能明白我有时候需要协调调度Pawn们走个方阵，有时候也得多人合作才能操纵得了一台机甲。当然越灵活也往往意味着越容易出错，但总之我们需要一个和Pawn关联的机制。
2. 多个控制实例，在需要的时候，我不介意可以克隆出多个我来，比如一段逻辑A，我们希望可以有多个实例在同时运行。就像行为树一样，可以有多个运行实例，彼此算法一样，但互不干扰。
3. 可挂载释放，我可以选择当前控制PawnA，也可以选择之后把它释放掉不再控制让她自生自灭，然后再另寻新欢控制PawnB，我必须拥有灵活的运行时增删控制Pawn的能力。
4. 能够脱离Pawn存在，我思故我在，就算当前没有任何Pawn控制，我也可以继续存在，这样我就可以延时动态的选择Pawn对象。有些Pawn值得我去等。
5. 操纵Pawn生死的能力，谁规定必须一定去控制世界当前存在的Pawn才行。当世界里没有Pawn可供我控制时，我希望可以自己造一个出来。你要说她是玩具、亦或傀儡也好，我不在乎。有时候我很羡慕暗黑里的沉沦魔巫师，身边总是围绕着一群沉沦魔，一个沉沦魔挂了，他可以紧接着再复活一个出来，这样永远都不会感动寂寞，你说多好？那索性再霸道一点吧，要是我这个控制实体不在了，我希望可以选择是否带Pawn们跟我一起走，没了我，她们都傻得让人心疼。当然如果有哪个Pawn能让我这个霸道总裁爱上，我也愿意陪她一起去死。
6. 根据配置自动生成，我（控制）虽然只是一段代码，但也不能无中生有，所以也得有个机制可以生成我这个控制实体，不过想来这应该是组织里更上层领导的事，但至少他应该知道怎么创建我出来。
7. 事件响应，游戏事件的一些控制关心的事件应该能够传到我这里，我可以酌情处理。同样，Pawn也可以向我汇报，我会好好研究决定的，嗯。
8. 持续的运行，没事的时候，我喜欢听世界大钟的每一次Tick，跟我的心跳同步起来，就仿佛真的活过来一样，可以自主的做一些我想做的事，这是我最自在的时候。
9. 自身有状态，你累了要休息，我也一样。我可以选择自身的状态，选择工作或者是休息，也可以选择今天是哪个Pawn和心情最配。
10. 拥有一定的扩展继承组合能力，一方面我希望我的家族开枝散叶繁荣昌盛，我的一身本领继承自我的父亲，而我也将有我的儿，大家各有天赋。另一方面，那些普通的Actor们都可以身背各个Component，更高贵的我当然也想有。
11. 保存数据状态，听说金鱼的记忆只有7秒，可是我却想记住你一辈子。所以我希望我能拥有一些记忆，人的过去成就了现在，也将指引着未来。以前有一个人跟我说过，当你不能再拥有的时候，唯一能做的就是令自己不要忘记。
12. 可在世界里移动，我可以选择帐中千里之外遥控Pawn，也可以选择附身在一个Pawn身上，这样我才能多角度无死角的观察我可爱的Pawn们，嘿嘿。
13. 可探查世界的对象，我要有眼睛，让我干活，基本的我得看得见知道当前世界里已经有哪些对象吧，否则不就抓瞎了嘛。
14. 可同步，这年头，要是不能适应网络环境，可真的没有竞争力。这个Object，Actor基本都有的能力，我当然也得有。位于服务器或客户端上的我也必须有能力在其他客户端上影分身，让他们都跟随我的步伐一致行动。

根据这些要求，我们选择从Actor继承构造了我们的controller  

![](https://pic3.zhimg.com/80/v2-b4e0dd15956ccb819fca93e73d1b8ed2_1440w.jpg)

![](https://pic1.zhimg.com/v2-c0cd2e5121f63c37615f78476e2a425c_r.jpg)

**思考：Controller和Pawn必须1:1吗？**  
**思考：为何Controller不能像Actor层级嵌套？**  
**思考：Controller的位置有什么意义？**  
**思考：哪些逻辑应该写在Controller中？**