[TOC]
http://manistein.club/post/program/let-us-build-a-lua-interpreter/%E6%9E%84%E5%BB%BAlua%E8%A7%A3%E9%87%8A%E5%99%A8part1/
## lua

#### lua面向对象
1. 多个返回值多个变量接收  
2. 函数类型就是function
3. 不支持重载，默认调用最后声明的函数
4. 变长参数，通过遍历进行处理
5. 函数嵌套，函数声明内部声明函数
6. 函数闭包
```
function F9(x)
    return function(y)
        return x + y
    end
end
```
7. lua的类，像是一个类中有很多静态变量和函数
8. 成员函数内部调用成员对象，要么表名限定，要么将自己作为一个参数传进去
9. .调用需要传参。:调用默认把调用者作为第一个参数传进去
10. Student:fun()的声明方式相当与有一个默认参数，可以用self调用
11. 使用require加载的脚本执行过一遍，就不会在去执行，可是使用package.loaded["脚本名"]检测是否加载成功
12. _G表是一个总表，我们的所有全局变量都是储存在_G表中，相当于给我们提供编程环境，加了local的不会加入到_G表中
13. 元表 当我们子表进行一些特定操作时，会执行元表的操作  
设置元表 setmetatable()  
_ToString 当子表要被当作字符串使用时，会默认调用这个元表中的String方法  
_call 当子表被当作函数使用是，默认调用  
运算符重载 _add _sub _mul _div _mod _pow  
_index当子表找不到某个属性时，去元表中查找  
_newindex 当子表赋值一个不存在的索引时，会赋值到newindex指向的元表中，不会修改自己
14. 继承，在_G表中声明一个空表，然后再设置元表
15. 实现多态，通过添加base属性来保留父类方法,不过不能使用：调用，需要手动调用传入自己self 
-```
object.base = self
```
function Object:subClass(classname)
    _G[classname] = {}
    local obj = _G[classname]
    setmetatable(obj,self)
end
```


#### c#调用lua
- 全局变量获取
- 函数获取，多返回值用out接收
- list dic映射table
- 类映射table
- 接口映射（需要标签生产代码，且是引用拷贝）
- luatable 映射table，引用拷贝

使用标签的方式官方并不建议，所有我们可以使用动态列表和静态列表的方式  

c#执行lua代码将分三个步骤:  
- 加载lua代码到vm中，对应api - luaL_loadbuffer
- luaL_loadbuffer会同时在栈上压入代码块的指针
- 执行lua代码，对应api - lua_pcall
- lua_pcall会从栈上依次弹出{nargs}个数据作为函数参数，再弹出函数进行执行，并将结果压入栈
- 如果lua代码有返回值，那么通过lua_toXXX相关api从栈上获取结果

c#调用lua函数
- 将全局函数压入栈中, 对应api - lua_getglobal
- 将函数所需的参数依次压入栈中，对应api - lua_pushnumber
- 执行栈中函数,对应api - lua_pcall
- 获取函数返回结果，对应api - lua_tonumber

```
//从全局表里读取addSub函数，并压入栈
Lua.lua_getglobal(L,"addSub"); 
//压入参数a
Lua.lua_pushnumber(L,101); 
//压入参数b
Lua.lua_pushnumber(L,202); 
//2个参数,2个返回值
Lua.lua_pcall(L,2,2,0); 
//pcall会让参数和函数指针都出栈
//pcall执行完毕后，会将结果压入栈
Debug.Log(Lua.lua_tonumber(L,-2));
Debug.Log(Lua.lua_tonumber(L,-1));
Lua.lua_pop(L,2);
```




#### lua调用C#
https://zhuanlan.zhihu.com/p/352910479  
https://zhuanlan.zhihu.com/p/395361399  
lua没有办法直接调用C#，一定是c#先调用lua脚本后，才把逻辑核心交给lua来编写，在我们实现核心逻辑的时候，肯定会涉及到对c#的调用，有两种方式，一个是通过反射的方式进行调用，但是效率较低，二是通过标签的形式，生产warp代码，进行注册来使用c#代码  
- CS.命名空间.类名    
- 方便使用定义全局变量储存
- 使用成员方法transform：Transform（）
- 继承了Mono的类不能直接new，Mono依附在GameObj身上，通过AddComponent创建，luo不支持无参泛型函数，所以AddCompoment（typeof(CS.LuaCallCSharper)）
- lua使用C#的list和dic，使用方式不变，因为lua不支持无参泛型构造，所以  CS.System.Array.CreatInstance(typeof(CS.System.Int32),10)
- 这里的数组依旧是按照C#的规则，下标从0开始
- lua创建list和dic，

```
local List_String = CS.System.Collection.Generic.List(CS.System.String)
local list1 = List_String()

```
- lua中使用拓展方法，一定要在工具类前加标签 LuaCallCSharp   建议lua中要使用的类都加上可以提高性能，如果不加使用反射机制调用，效率降低 
- lua调用c#ref和out ref参数会以多返回值的形式返回给lua如果函数存在返回值 那么第一个值就是该返回值，之后的ref的结果从左到右一一对应
- ref需要占位，out不需要占位
- lua使用C#的重载函数，lua不支持重载，但可以使用C#的重载  但是因为只有Number类型，所以避免多精度重载  可以通过反射机制调用GetMethod  
- tofunction可以将C#函数转成lua函数重复使用，成员方法第一个参数传参数
- lua使用委托区别和C#不大，使用事件时使用冒号事件名

```
obj:eventAction("+",function())
//清除事件不能直接设nil
调用C#中的清楚函数
```
- lua使用C#二维数组用getValue访问，不能[]访问
- nil和null不能==比较

```
function IsNull(obj)
    if obj == nil or obj:Equals(nil) then
        return true;
    end
    return false
end
```
- 可以通过list对无法直接添加标签的系统类或者第三方代码添加标签

```
[CSharpCallLua]
public static List<Type> csharpCallLuaList = new List<Type>(){
  typeof(UnityAction<float>),
};
public static List<Type> luaCallCSharpList = new List<Type>{
  typeof(GameObject)  
};
//一定在静态类中，一定重新生产代码
```
- lua使用c#协程,lua都是通过继承了Mono的类来启动携程，lua没有yield return，使用lua自己的协程返回，并且不能直接将function传入协程中，需要使用xlua.util

```
util = require("xlua.util")
mono.StartCoroutine(util.cs_generator(fun))
```
- lua 不支持无约束泛型，不支持非class约束
- 可以通过反射得到通用函数，设置泛型类型后再调用

```
local testfun = xlua.get_generic_method(CS.Lesson12,"Testfun2")
local testfun_R = testfun(CS.System.Int32)
//调用
//成员函数：第一个参数 传调用的对象
//静态方法：不用传
//有一定限制
//使用Mono打包正常使用
//使用cpp打包，泛型参数只能使用引用类型，如果是值类型，除非C#中调用过了lua中才可以使用
testfun_R(obj,1)
```

#### GC管理
在c#类注册到lua中，需要将C#的构造函数也注册到lua中，但是我们没有办法实际将c#对象返回到lua，因此这里使用了userdata类型的lua对象作为c#对象在lua中的**替身**  
userdata是lua中的一种类型，其代表了在宿主语言中分配出来的一块内存区域，但生命周期却是交给lua的gc来管理的。我们同样可以为userdata变量设置metatable,以此为其增加各种方法、属性.  

在C#中我们使用objectCache缓存了userdata和实际C#对象，并且将内存的生命周期控制权限交给了luaVM，当lua的userdata被回收后，  在c#端objectCache中缓存的userdata就会成为传说中的野指针，同时造成内存泄露。  
为了解决这个问题，需要在c#端监听lua中对象的gc情况,当userdata被lua vm gc回收时，我们同步将其从objectCache中移除.  
或者提供gc函数，在回收的时候触发objectCache的回收  

c#引用lua中的临时函数  
在使用lua处理核心逻辑是，需要和宿主语言的消息系统通信，所以涉及到在C#端EventManager中注册lua的回调函数。在C#端需要对其进行引用，并在合适的时机回调，并且我们需要在合适的时机释放这个临时变量，否则lua端会发生引用泄露。  

```
[MonoPInvokeCallback(typeof(LuaCSFunction))]
private static int EventManager_Register(System.IntPtr L){
    //即LuaRegistery[reference] = luaCallback
    //luaL_ref这个api，它会将栈顶元素添加到lua的注册表中(这样就不会被luavm gc回收)。
    //luaL_ref会返回一个int类型的reference，用于后续去注册表中重新获取该元素.
    var reference = Lua.luaL_ref(L,(int)LUA_REGISTRY.Index);
    var luaFunc = new LuaFunction(_globalL,reference);
    EventManager.Register((value)=>{
        luaFunc.PCall(value);
    });
    return 0;
}
```
设计管理类  

```
public class LuaFunction{

    private int _reference;
    private System.IntPtr _L;
    public LuaFunction(System.IntPtr L, int reference){
        _reference = reference;
        _L = L;
    }

    public void PCall(int value){
        //根据reference从registery中取到lua callback，放到栈顶
        Lua.lua_rawgeti(_L,(int)LUA_REGISTRY.Index,_reference);
        //压入参数
        Lua.lua_pushinteger(_L,value);
        //执行lua callback
        Lua.lua_pcall(_L,1,0,0);
    }

    ~LuaFunction(){
        Lua.luaL_unref(_L,(int)LUA_REGISTRY.Index,_reference);
        Debug.Log("LuaFunction gc in c#:" + _reference);
    }
}
```
可以看到，LuaFunction实现了析构函数。即当LuaFunction这个对象在c#端被GC回收时，我们同步释放其所维护的lua reference.  

在lua实际的使用中会遇到一个问题，就是循环引用的问题  

```
local csObj = CSObject()
csObj:AddCallback(function()
    csObj:DoSomething()
end)
```

lua端依赖c#这边的gc释放LuaFunction，从而对luaCallback进行解引用，才能触发csObj(userData)的gc

但c#端又依赖lua这边对csObj(userData)进行gc回收，才能从objectCache中移除csObj.




#### table实现  
https://zhuanlan.zhihu.com/p/295524176
https://www.cnblogs.com/zblade/p/8819609.html  
table的底层是使用hashtable乱序存储的，会破坏ECS带来的效率提升  

每个 Table 结构，最多会由三块连续内存构成。
- 一个 table 结构，
- 一块存放了连续整数索引的数组，
- 和一块大小为 2 的整数次幂的哈希表。
![image](https://pic2.zhimg.com/v2-1649fc6bd83e7e61f9cc1432ad3a0229_r.jpg)

TValue

```
typedef struct lua_TValue {
  TValuefields;
} TValue;

#define TValuefields    Value value; int tt;

typedef union {
  GCObject *gc;
  void *p;
  lua_Number n;
  int b;
} Value;
```
Node

```
typedef union TKey {
  struct {
    TValuefields;
    struct Node *next;  /* for chaining */
  } nk;
  TValue tvk;
} TKey;

typedef struct Node {
  TValue i_val;
  TKey i_key;
} Node;
```

value是一个共用体，可以存储我们的指针，数字等，用来存放我门的键值对，TKey中的tvk存储的是key的具体值，nk存储下一个的冲突节点  

##### table通过ipairs和pairs遍历数组
ipairs遍历数组部分，pairs遍历整个table   
ipairs遍历顺序就是从1开始一次加1往后遍历table的数组部分,即table[i]。pairs的遍历实际上是调用luaH_next：先遍历数组，再遍历hashtable   
ipairs只能返回0，遇到nil结束遍历 ,pairs能返回nil  

##### lua table存储和ECS缓冲命中的影响
和c++中的多种容器相比lua并没有提供太多数据结构，而是都使用lua进行存储，并且在处理不同类型的数据时，lua并没有使用模板去进行二次编译，而是在底层存贮结构中使用了许多共用体，来实现同一块内存空间，存储不同类型。  
![image](https://img-blog.csdn.net/20150330193313656?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTWF4aW11c1pob3U=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)  
在table中不管key还是value都是通过value存储的，造成实际开辟内存可能比实际用的内存大，可能会占据更多的缓存空间，导致缓存命中下降。  

##### tbale的插入
创建key的流程如下所示：
- 计要新建的key值为k，并计算k的hash值，记为k_hash
- 计算key应该落在hash表的哪个位置，计算方式为index = k_hash & (2^lsizenode-1)
- 如果hash[index]这个node的value值为nil，将node的key值设置为k的值，并返回value_对象指针，供调用者设置
- 如果hash[index]这个node的value值不为nil，需要分两种情况处理
- 计算node key的hash值，如果经过定位运算后，index的值不在自己所处的位置上，那么lastfree不断左移，直至找到一个空闲的节点，将其移动到这里，修改链表关系，令其上一个与自己计算得到相同index值的节点的next域指向自己（如果存在的话）。新插入的key和value设置到hash[index]节点上
- 计算node key的hash值，如果经过定位运算后，index的值在自己所处的位置上，那么lastfree不断左移，直至找到一个空闲的节点，将自己的key和value值设置到这个节点上，并调整链表关系，将与自己计算得到相同index值的上一个节点的key的next指向自己的位置。 

**总结** ：  
如果节点位置被占据，保存的是没有冲突的节点，则lastfree后移找到空节点存放自己。
如果保存的是冲突后开链的节点，则lastfree后移找到空节点存放该节点，hash后的位置存放自己



## tolua源码解析
### 1. wrap文件是如何生成，为什么要生成wrap？  
每一个wrap文件都是对一个C#类的包装  

![image](https://img2018.cnblogs.com/blog/1362861/201809/1362861-20180919010454611-1438294246.png)
- ①CustomSetting.cs添加需要生成wrap的c#类
- ②通过GenerateClassWraps()生成wrap文件
- ③通过GenLuaBinder()生成绑定类，用于开启lua虚拟机时导入注册类
- ④在开启lua虚拟机时，将需要用的类绑定到lua虚拟机中

#### 一个类通过wrap文件注册进lua虚拟机后是什么样子的
![image](https://img2018.cnblogs.com/blog/1362861/201809/1362861-20180919010548055-1175628480.png)

#### 每个函数实际的调用过程
假如说在lua中有这么一个调用：
```
local tempGameObject = UnityEngine.GameObject("temp")
local transform = tempGameObject.GetComponent("Transform")
```
第二行代码对应的实际调用过程是
1. 先去tempGameObject的元表GameObject元表中尝试去取GetComponent函数，取到了。
2. 调用取到的GetComponent函数，调用时会将tempGameObject,"Transform"作为参数先压栈，然后调用GetComponent函数。
3. 接下来就进入GetComponent函数内部进行操作，因为生成了新的ci，所以此时栈中只有tempGameOjbect,"Transfrom"两个元素。
4. 根据参数的数量和类型判断需要使用的重载。
5. 通过tempGameObject代表的c#实例的索引，在objects表中找到对应的实例。同时取出"Transform"这个参数，准备进行真正的函数调用。
6. 执行obj.GetComponent(arg0)，将结果包装成一个fulluserdata后压栈，结束调用。
7. lua中的transfrom变量赋值为这个压栈的fulluserdata。
8. 结束。
其中3-7的操作都在c#中进行，也就是wrap文件中的GetComponent函数。

#### lua中c#实例的真正存储位置
前面说了每一个c#实例在lua中是一个内容为整数索引的fulluserdata，在进行函数调用时，通过这个整数索引查找和调用这个索引代表的实例的函数和变量。    
生成或使用一个代表c#实例的lua变量的过程大概是这样的。
还用这个例子来说明：  

```
local tempGameObject = UnityEngine.GameObject("temp")
local transform = tempGameObject.GetComponent("Transform")
```
![image](https://img2018.cnblogs.com/blog/1362861/201809/1362861-20180919010621743-317240348.png)  

所以说lua中调用和创建的c#实例实际都是存在c#中的objects表中，lua中的变量只是一个持有该c#实例索引位置的fulluserdata，并没有直接对c#实例进行引用。
对c#实例进行函数的调用和变量的修改都是通过元表调用操作wrap文件中的函数进行的。

### 2. lua是怎么获取、调用c#的静态方法、成员方法？c#对象在lua栈里是以什么形式存在的？


### 3. c#如何调用到lua的方法的？tolua是怎么把lua的table、function转成c#的table、function实例的？

### 4. tolua把对象存在objects里，而值类型的Struct如果存在objects了，会发生封箱、拆箱的操作，tolua是如何避免的？
方法一：申请一块userdata（size=12），将Vector3拆成3个float，Pack到userdata，将常量下标push到lua  
方法二： 不与C#交互，使用table保存三个float值，和c#互相引用时都需要将三个float值单独取出转化
### 5. objects里的对象是什么时候会被移除？lua怎样才算正确释放了c#对象？  
通过dictionary将lua的userdata和c# object关联起来，只要lua中的userdata没回收，c# object也就会被这个dictionary拿着引用，导致无法回收。  
最常见的就是gameobject和component，如果lua里头引用了他们，即使你进行了Destroy，也会发现他们还残留在mono堆里。  

所以只有解除了lua对c#对象的引用，才可以正常回收对象，这就涉及到lua的GC系统。  
tolua在Update里调用了luaState.Collect(); 然后调用了ObjectTranslator.Collect();用了脏标记模式，gcList长度大于0的时候进行回收，调用DestroyUnityObject()，对于C#对象，GC会通知C#中的dictionary清理资源，这样就避免内存泄露  
**情况一**：Lua对象是全局变量，直接放在_G中。因为全局变量不会被GC回收，需要手动nil，所以尽量避免使用全局。  
禁止定义全局变量，给现有的全局变量前加载local声明。可以使用一些Lua静态语法检查的手段，如Luacheck来检查。  
**情况二**：Lua对象被一些全局的Table引用。  
将持有C#对象的变量，定义在会赋值为空的对象中  

**情况三**：Lua对象的function字段被赋值给了C#的事件/委托。   
比如，lua对象引用了C# button对象，又将lua function作为回调委托传给button   
**应对方法：**  
- 对于每一个提供给Lua注册事件/委托的C#类，都继承一个IClear接口，该接口内实现清理事件/委托。用来清理C#中的lua委托
- 在MonoBehavior的OnDestroy函数内，调用IClear的接口。但要注意的是，这并不能保证所有的组件都是清理完毕，因为deactvie状态的组件，是不会触发OnDestroy的。因此需要手动的调用清理。
- 提供一个清理GameObject Lua事件/委托的接口，该接口会找到GameObject上所有继承于IClear接口的类，执行清理操作。需要手动清理的GameObject都需要调用该函数。

```
void ClearGameObject(UnityEngine.GameObject target)
{
    if(target == null) return;
    var list = target.GetComponentsInChildren<IClear>(true);
    foreach(var component in list)
    {
        component.Clear();
    }
}
```


6. 利用tolua如何实现热更？
热更是通过服务器帮助客户端实现增量更新的一种方式。
Unity通过将资源和代码打包成AB包的形式，放到服务器上实现热更。  
Unity3D的热更新会涉及3个目录：游戏资源目录、数据目录、网络资源地址。  
大体热更步骤如下图：
2.1、导出热更资源
- 打包热更资源的对应的md5信息（涉及到增量打包）
- 上传热更ab到热更服务器
- 上传版本信息到版本服务器
2.2、游戏流程热更
- 启动游戏
- 根据当前版本号，和平台号去版本服务器上检查是否有热更
- 从热更服务器上下载md5文件，比对需要热更的具体文件列表
- 从热更服务器上下载需要热更的资源，解压到热更资源目录
- 游戏运行加载资源，优先到热更目录中加载，再到母包资源目录加载

7. 针对lua和c#的交互有什么优化手段？



##  热更新
热更新的作用是可以在用户不重新下载客户端的情况下，实现动态更新游戏内容。  
在unity中，主要的热更新方式有如下三种：
- 使用Lua编写游戏逻辑；
- C#Light
- C#反射技术  
可以将部分逻辑提取至一个单独的代码库工程中，打包为DLL，将DLL打包为AssetBundle，Unity程序动态加载AssetBundle中的DLL文件，使用反射机制来调用代码。用C#反射加载程序集的方式可以动态的从assetBundle资源包或其他资源包里加载脚本到工程中。

### AssetBundle
https://zhuanlan.zhihu.com/p/38122862
#### AB包在热更新中的开发流程
1. 创建Asset bundle，开发者在unity编辑器中通过脚本将所需要的资源打包成AssetBundle文件。
2. 上传服务器。开发者将打包好的AssetBundle文件上传至服务器中。使得游戏客户端能够获取当前的资源，进行游戏的更新。
3. 下载AssetBundle，首先将其下载到本地设备中，然后再通过AsstBudle的加载模块将资源加到游戏之中。
4. 加载，通过Unity提供的API可以加载资源里面包含的模型、纹理图、音频、动画、场景等来更新游戏客户端。
5. 卸载AssetBundle，卸载之后可以节省内存资源，并且要保证资源的正常更新。

#### Res和AB包的区别
Resources:
1. 构建工程时文件消失，压缩为二进制文件
2. 运行时加载
3. 只读文件，无法更新
4. 导致内存颗粒化  

AssetBundle:
1. 一组对象组合成的压缩文件。
2. 本身是压缩包，运行时加载，自身保存依赖关系，可以下载。
3. 在硬盘上本质是文件夹，里有两类文件，序列化文件，源文件。 

### Hotfix
对于需要热更新的C#类，我们可以再lua中打热补丁，对代码进行重定向。   

1. 在需要热更的代码类上打上HotFix标签，然后生成代码。会生成需要热更方法的委托方法
2. 在Lua层调用xlua.hotfix(cs, field, func)
3. xlua.hotfix会把第三个参数，也就是这个闭包，以委托的形式注入到IL层的代码
4. 然后在代码走到需要热更的代码的时候，判断委托是否为空，委托不为空的话走热更逻辑，委托空的话走原逻辑。这样就实现了C#层的热更


- 热补丁的固定写法 


```
//C#代码
public class HotfixMain : MonoBehavior
{
    void Start()
    {
        LuaMgr.GetInstance().Init();
        LuaMgr.GetInstance().DoLuaFile("Main");
    }
    void Update()
    {}
    public int Add(int a,int b)
    {
        return 0;
    }
    public static void Speak(string str)
    {
        Debug.Log("哈哈哈")；
    }
}
```

```
//Lua代码
xlua.hotfix(CS.HotfixMain, "Add" , function(self , a , b)
    return a + b
)

xlua.hotfix(CS.HotfixMain, "Speak" , function(a)
    print(a)
)
```
- 热补丁的固定步骤  
1. 加特性[Hotfix]，声明该类可以热更新
2. 在build setting中scripting Define中加宏HOTFIX_ENBLE
3. 生成代码
4. hotfix注入，需要引入Tools，热补丁类修改后，需要重新注入

- 多函数替换和构造，析构函数替换  

```
xlua.hotfix(CS.HotfixMain,{
    //构造函数和析构函数的规则是先执行C#的构造析构，再执行lua的构造析构逻辑
    //构造函数的固定写法 [".ctor"]
    [".ctor"] = function()
        print("Lua热补丁构造函数")
    end,
    Speak = function(self,a)
        print("唐老师say" + a)
    end,
    //析构函数的固定写法
    Finalize = function()
    end
})
```
- 协程函数替换 通过util返回lua协程

```
util = require("xlua.util")
xlua.hotfix(CS.HotfixMain,{
    TestCoroutine = function(self)
        //转化为lua协程后返回
        return util.cs_generator(function ()
            while true do
                coroutine.yield(CS.UnityEngine.WaitForSecond(1))
                print("Lua热补丁后的协程函数")
            end
        end)
    end
})
```
- 索引器和属性替换

```
xlua.hotfix(CS.HotfixMain,{
    //属性写法
    //set_属性名  
    //get_属性名
    set_Age = function(self,v)
        print("Lua的重定向属性"..v)
    end,
    get_Age = function(self)
        return 10
    end
    //索引器固定写法
    set_Item = function(self,index,v)
        print("Lua重定向索引器")
    end
    get_Item = function(self,index)
        print("Lua重定向索引器")
        return 0
    end
})
```
- 事件加减替换

```
xlua.hotfix(CS.HotfixMain,{
    //add_事件名
    //remode_事件名
    //不能再去调用c#的方法，不然回出现循环调用
    add_myEvent = function(self,del)
        print(del)
        print("添加事件")
    end,
    remove_myEvent = function(self,del)
        print(del)
        print("移除事件函数")
    end
    
})

```
- 泛型类替换，需要一个类型一个类型的替换

```
xlua.hotfix(CS.Hotfixtest2(CS.System.String)，{
    Test = function(self,str)
        print("Lua中打的补丁"..str)
    end
}）

xlua.hotfix(CS.Hotfixtest2(CS.System.Int32),{
    Test = function(self,str)
        print("lua中打的补丁"..str)
    end
})
```








https://ashan.org/archives/956
## UI



## Age

ActionBaseEvent

ActionBase

Age

Action

Event



## ECS
Entity-Component-System（实体-组件-系统）

![](https://pic1.zhimg.com/v2-04e15b14964c9b61bffdfad42e907ffc_r.jpg)



类似于Unity的挂载脚本，Entity相当于场景的GameObject，实体就是一个ID，这个ID对应了组件的集合

Com用来存储游戏状态并且没有任何行为，

sys拥有处理实体的行为但是没有状态和数据

### Entity

代码层面用ID进行表示，所有组成实体的组件都会被这个ID标记，可以动态的增加删除组件  
**组合优于继承**，既然是组合关系了，那么热插拔和复用的特性也能用上了，方便拓展（不是升级），几乎不会影响到其他功能，降低耦合，因为数据和状态都在com中，便于进行预测和回滚，记录关键帧的数据和状态（便于提高打击感和流畅度，打击冻结感）

### Com

是一堆数据的集合，对应PGame的DurationBaseEvent，我们还可以用Com对实体进行标记，比如 `EnemyComponent`这个组件可以不含有任何数据，拥有该组件的实体被标记为“敌人”。

```C#
CopyDataImpl(ActionBaseEvent src)//从XML中读取对应的数据
```
## ECS

Entity-Component-System（实体-组件-系统）

![](https://pic1.zhimg.com/v2-04e15b14964c9b61bffdfad42e907ffc_r.jpg)



类似于Unity的挂载脚本，Entity相当于场景的GameObject，实体就是一个ID，这个ID对应了组件的集合

Com用来存储游戏状态并且没有任何行为，

sys拥有处理实体的行为但是没有状态和数据

### Entity

代码层面用ID进行表示，所有组成实体的组件都会被这个ID标记，可以动态的增加删除组件

### Com

是一堆数据的集合，对应PGame的DurationBaseEvent，我们还可以用Com对实体进行标记，比如 `EnemyComponent`这个组件可以不含有任何数据，拥有该组件的实体被标记为“敌人”。

```C#
CopyDataImpl(ActionBaseEvent src)//从XML中读取对应的数据
```


### Sys

系统就是用来处理游戏逻辑的部分，一个系统就是对拥有一个或多个相同组件的实体集合进行操作的工具，项目中，系统驱动时会获取到所有拥有对应组件的实体和旗下的组件，对数据进行逻辑处理

通过单例Mgr，我们可以快速的获取我们的对象旗下的组件，或者多个系统依赖同一个组件，比如说  `GetSingletonInput()`  方法来引用获得该组件的引用  

这样最大的好处就是我们的系统之间解耦十分彻底，但同样的，简单的继承并不能满足我们代码复用的要求，所以我们可以设计一个 **UtilityFunction**（实用函数），它将被多个系统调用的方法单独提取出来，放到统一的地方，各个系统通过**UtilityFunction**调用想执行的方法，同系统一样，**UtilityFunction** 中不能存放状态，它应该是拥有各个方法的纯净集合。


Component 只有数据，行为是 System 的事。这样的模式，避免了上一节提到的 Unity3D 中容易出现的问题。Component 没有逻辑上的互相引用，Component 的耦合和依赖由 System 处理。此外，由 System 进行统一的状态修改，也有利于定位和隔离问题。    
在Unity中的com将数据和逻辑抽象，实现过程中不仅造成逻辑耦合，还会造成数据耦合，并不能高效的管理数据。而真正的ECS系统，由于将数据单独抽离，在实体数据十分多的时候，会有显著的性能优势，提高了缓存命中率。

### 浅谈CPU缓存命中
ECS相比OOP的优点就是对于数据密集型，有着更好的效率，对于缓存有更好的命中率。

CPU由运算器，控制器和寄存器组成，控制器取指令，交给运算器计算，之后把结果存入寄存器。
![image](https://pic1.zhimg.com/v2-5b77642349421b47cddfee1954a8d110_r.jpg)

CPU被设计用来进行高速的运算，所以寄存器的存储容量是非常小的，所以我们需要从主存甚至磁盘去读取数据，但是从主存读取的效率是非常满的，根本和CPU的处理效率匹配不了，缓存就是为了解决这个问题。    

CPU在取指令时，会先去多级的缓存中查找，如果在最后一级都没有查找到才会去主存查找，称为缓存未命中（cache miss），如果每次都miss的话，我们的执行效率就会特别的低。   

所以CPU在执行某个程序片段的时候，它会安排缓存帮他预测下次要查找的数据，然后逐级上报，如果下次查找的数据正好在缓存里，那么就叫缓存命中，如果不在就叫miss，如果Miss了，CPU就不得不亲自去内存里找，那么速度可想而知。  

预知的策略一般分为  
时间局部性：当前用到的一个存储器位置，在不久的将来还会被用到。   
![image](https://pic2.zhimg.com/80/v2-495251dc41c85d458a8b421deb2cebb5_720w.jpg)
空间局部性：当前用到的一个存储器位置，它临近的几个位置也会被用到。  
![image](https://pic4.zhimg.com/80/v2-869e6d456709eb539d798b8bbe084dc3_720w.jpg)
根据二位数组的排列方式我们知道，第二种的写法全都会cache miss

### 逻辑与表现分离
https://zhuanlan.zhihu.com/p/79454315  
#### 逻辑和表现分离的收益
- 解耦后的逻辑和表现分离，由逻辑层掌控一切，表现层受逻辑层驱动，在不影响逻辑的表现下自主表现，逻辑层一定要能完全脱表现层独立存在。如果我们把这个逻辑层放在服务器，那就是Client-Server模式，如果放在客户端那就是帧同步的制作方式（王者荣耀就是这么干的）。
- 安全既然能解耦了，那么就可以让业务更加安全。安全来自两个部分，一个是CS模式下对数据和外挂的安全，一个是帧同步模式下，表现层的BUG影响到逻辑的安全。
- 倍速如果只操作逻辑层，不考虑表现那么就可以通过加快或者减慢逻辑帧率，快速实现0.5倍，2倍，4倍等变速运算，也可以进行秒算验证结果。
- 回放 数据是独立存在的，运算和输入也是固定的，那么只要保证逻辑计算一致，那么得到结果必然一致。所以只需要保存很少的初始变量，和中间输入就可以完成整体回放，数据量还贼小。（想想WAR3 一场战斗40分钟，录像文件只有200K。怀念上学的时候，去网吧下载录像回家看的日子。）不过这里要提一个缺点，那就是版本和数据必须一致，否则计算就会不一致。
- 移植表现层可以根据自己使用的开发引擎做快速移植，而不需要修改整体逻辑。

#### PVP和PVE架构  
**PVP**
![image](https://pic2.zhimg.com/80/v2-43dab961c02d3068204172f463b2a359_720w.jpg)  
**PVE**  
![image](https://pic2.zhimg.com/80/v2-5fbcfa16059933a0b7f8e27e1c61b359_720w.jpg)  

#### 逻辑帧和表现帧
逻辑帧往往是用不了这么高的，士兵攻击频率1秒1次，是不用每16ms（60帧）去计算一次的，我的项目设置为15帧已经可以满足了。那么表现层其实是需要对某些表现做插值处理，最明显的就是移动。

移动速度假如是60m/s，逻辑15帧每帧跨度4m，如果不补帧看起来就像是卡顿。所以表现层是要根据自己的帧率对移动进行插值，保证平滑。

逻辑帧是独立驱动的，所以它有自己的核心逻辑。
![image](https://pic3.zhimg.com/v2-a303c373ca8a5ca9eb9a507ac6d98702_r.jpg)    
TotalPassTime是当前已经过去的总时间，下面接着是一个While循环，循环的判定条件就是当前pass的总时间只要大于下一帧的时间就执行逻辑帧。这这样的目的是就是为了解决不会因为某些帧的间隔过大而导致逻辑帧的波动。简单的来说就是追帧。战斗到现在已经过去10秒了，理论上有下一帧是151帧，而这个时候实际因为某些原因才计算到100帧，那么接下来会在while里循环直到追平当前帧。这也是服务器能够秒算（给一个初始非常大的pass值），以及客户端能够实现倍速战斗的原因。看下下面的代码：  
![image](https://pic4.zhimg.com/80/v2-fcb4c40210cb68d40af6d5c618a4bcf7_720w.jpg)  
UpdateLogic的客户端逻辑是由deltaTime和ClientFrameDelta两个部分来控制的，deltaTime*控制倍率能控制每次补偿的时间差（实现倍率播放），ClientFrameDelta则是初始化帧的进度值（实现秒算）。




### 使用中遇到的具体问题

#### 为什么不使用OOP模型？

Connection组件用来管理服务器上的玩家网络连接，是挂在代表玩家的实体上的。它可以是正在进行比赛的玩家、观战者或者其他玩家控制的角色。

然而不同的System对于Connection又意味着不同的行为，使用OOP模型我们就会纠结于具体哪个行为应该放在组件的Update中调用？其余部分应该又放在那里。

而传统的OOP模型，一个类既是行为又是数据，这并不符合Connection的设计

#### System真的不能保存状态吗？

为了实现Killcam，我们会有两个不同的、并行的游戏环境，一个用来进行实时游戏过程渲染，一个用来专门做Killcam。

首先，也很直接，我会添加第二个全新的ECS World，现在就有两个World了，一个是liveGame(正常游戏)，一个是replayGame用来实现回放（Replay）。

回放(Replay)的工作方式是这样的，服务器会下发大概8到12秒左右的网络游戏数据，接着客户端翻转World，开始渲染replayAdmin这个World的信息到玩家屏幕上。然后转发网络游戏数据给replayAdmin，假装这些数据真的是来自网络的。此时，所有的System，所有的组件，所有的行为都不知道它们并没有被预测(predict，译注：后面才讲到的同步技术)，它们以为客户端就是实时运行在网络上的，像正常游戏过程一样。

这样需要全局访问System的调用点会突然出错，猜测原因就是因为我们的全局访问点是放在System之中，所以overwatch的开发团队的方案依赖于这样一个事实：开发一个只有唯一实例的组件其实没什么不对！根据这个原则，我们实现了一个单例（Singleton）组件。

这些组件属于单一的匿名实体，可以通过EntityAdmin直接访问。我们把System中的大部分状态都移到了单例中。

这里我要提一句，只需要被一个System访问的状态其实是很罕见的。后来在开发一个新System的过程中我们保持了这个习惯，如果发现这个系统需要依赖一些状态。就做一个单例来存储，几乎每一次都会发现其他一些System也同样需要这些状态，所以这里其实已经提前解决了前面架构里的耦合问题。

同样的，对于System之间的调用，我们用一个共享的Utility函数替换System间的函数调用，同样的，我们在使用时，需要尽可能的减少调用点，减少代码的复杂性，因为所有的副作用都限定到函数调用的地方了。

#### 使用单例进行推迟（Deferment）避免多次调用的副作用

这里我要提一句，只需要被一个System访问的状态其实是很罕见的。后来在开发一个新System的过程中我们保持了这个习惯，如果发现这个系统需要依赖一些状态。就做一个单例来存储，几乎每一次都会发现其他一些System也同样需要这些状态，所以这里其实已经提前解决了前面架构里的耦合问题。

包括hitscan(译注：直射，没有飞行时间)子弹；带飞行时间的可爆炸抛射物；查里娅的粒子光束，光束长得就像墙壁裂缝，而且在开火时需要保持接触目标；另外还有喷涂。

创建碰撞特效的副作用很大，因为你需要在屏幕上创建一个新的实体，这个实体可能间接地影响到生命周期、线程、场景管理和资源管理。

碰撞特效的生命周期，需要在屏幕渲染之前就开始，这意味着它们不需要在游戏模拟的中途显现，在不同的调用点都是如此。

这些代码确保了像弹孔、焦痕持久特效不会很奇怪的叠在一起。例如，你用猎空的枪去射击一面墙，留下了一堆麻点，然后法老之鹰发出一枚火箭弹，在麻点上面造成了一个大面积焦痕。你肯定想删了那些麻点，要不然看起来会很丑，像是那种深度冲突（Z-Fighting）引起的闪烁。我可不想在到处去执行那个删除操作，最好能在一处搞定。

于是我们有了Contact单例。

它包含了一个未决的碰撞记录的数组，每个记录都有足够的信息，来在本帧的晚些时候创建那个特效。如果你想要生成一个特效的时候，只需要添加一条新记录并填充数据就可以了。等运行到帧的后期，进行场景更新和准备渲染的时候，ResolveContactSystem会遍历数组，根据LOD规则生成特效并互相叠加。这样的话，即使有严重的副作用，在每一帧也只是发生在一个调用点而已。

除了降低复杂度以外，“推迟”方案还有很多其他优点。数据和指令都缓存在本地，可以带来性能提升；你可以针对特效做性能预算了，例如你有12个D.VA同时在射墙，她们会带来数百个特效，你不用立即创建全部这些特效，你可以仅仅创建自己操纵的D.VA的特效就可以了，其他特效可以在后面的运算过程中分摊开来，平滑性能毛刺。这样做有很多好处，真的，你现在可以实现一些复杂的逻辑了。即使ResolveContactSystem需要执行多线程协作，来确定单个粒子效果的朝向， 现在也很容易做。“推迟”技术真的很酷。

Utility函数，单例，推迟，这些都只是我们过去3年时间建立ECS架构的一小部分模式。除了限制System中不能有状态，组件里不能有行为以外，这些技术也规定了我们在Overwatch中如何解决问题。

#### 网络同步（netcode）

要开发快速响应的网络对战游戏，为了实现快速响应，就必须针对玩家的操作做预测。如果每个操作都要等回包的话，就不可能又高响应性了，尽管这样会导致作弊行为的泛滥，但仍然时有必要的。

快速响应的操作包括移动，技能，命中判定

这些都有统一的原则，玩家按下按键后必须立即能够看到响应，即使网络延迟很高也必须如此

然而呢，带预测的客户端，服务器的验证和网络延迟就会带来副作用：预测错误（misprediction，或者说预测失败）了。预测错误的主要症状就一点，会使得你没能成功执行“你认为你已经做出的”操作。

所以我们需要确保行为的**确定性**来减少预测错误的发生概率

##### 确定性

确定性模拟技术依赖于时钟的同步，固定的更新周期和量化。服务器和客户端都运行在这个保持同步的时钟和量化值之上。时间被量化成command frame，我们称之为“命令帧”。每个命令帧都是固定的16毫秒，不过在电竞比赛时是7毫秒。

在我们的ECS框架内，任何需要进行预表现、或者基于玩家的输入模拟结果的System，都不会使用Update，而是用UpdateFixed。UpdateFixed会在每个固定的命令帧调用。

假定输出流是稳定的，那么客户端的始终总是会超前于服务器的，超前了大概半个RTT加上一个缓存帧的时长。这里的RTT是PING值加上逻辑处理的时间。全加起来就是客户端相对于服务器的提前量。

![](https://ashan.org/2017/07/5281474879213769955.png)

之所以这样是因为客户端是一股脑的尽快接受玩家输入，尽可能地贴近现在时刻，如果还需要等待服务器回包才能响应的话，那看起来就太慢了，会让游戏比较卡顿

如上图所示，我们在接受到操作杆的输入后，就根据其输入预测猎空的状态，让猎空向前走，经过完整的RTT之后加上缓冲事件，最终猎空会从服务器上回到客户端，这是服务器带回来的是确定正确的状态，但是副作用就是需要等待半个RTT时间才能回到客户端。

客户端使用一个环形缓存是为了和服务器的返回结果进行对比，如果当前的预测结果和服务器结果一样，那么说明我们的预测结果就是正确的，如果不一致，那就是一个“预测错误”，这时候就需要一个“和解”的策略。

或者说简单一点，直接用服务器结果覆盖结果呢？但是这个结果已经是旧的了。

![](https://ashan.org/2017/07/6398039739754029418.png)

上图是我们预测出错的情况，我们在移动中状态发生改变，被眩晕了，这时候出现了“预测错误”，我们的解决办法就是一旦从服务器回包发现预测失败，我们把你的全部输入都重播一遍直至追上当前时刻。这就是一个追帧的过程，我们之前的输入操作都存放在一个环形缓冲之中，遇到“预测错误时”，从中取出来重新执行一遍就可以了。

当客户端收到描述角色状态的数据包时，我们基本上就得把移动状态及时恢复到最近一次经过服务器验证过状态上去，而且必须重新计算之后所有的输入操作，直至追上当前时刻（第25帧）。

所以当我们在27帧时，收到了服务器上第17帧的回包。一旦重新同步（译注：注意下图41中客户端猎空的状态全都更正为“晕”了）以后，就相当于回退到了“帧同步”（lockstep）算法了。

到了第33帧以后，客户端就知道已经不再是晕住的状态了，而服务器上也正在模拟相同的情况。不再有奇怪的同步追赶问题了。一旦进入这个移动状态，就可以重发玩笑当前时刻的操作输入了。

然而我们的客户端网络并不保证如此稳定，时有丢包发生，我们游戏的输入都是通过定制化的可靠UDP实现的。所以为了解决丢包问题，服务器会试图保持一个小小的，保存未模拟输入的缓冲区，但是让他尽量的小，以保证游戏操作的流畅。

一旦这个缓冲区是空的，服务器只能根据你最后一次输入去“猜测”。等到真正的输入到达时，它会试着“缓和”，确保不会弄丢你的任何操作，但是也会有预测错误。

![](https://ashan.org/2017/07/1703108041608610231.png)

上图就可以看到，我们已经丢失了一些来自客户端的包，服务器意识到之后，就会复制先前的输入操作来就行预测

，一边祈祷希望预测正确，一边发包告诉客户端，发生了丢包的问题，这时候会发生奇怪的事件，客户端会进行时间膨胀，比约定的帧率更快地进行模拟。

![](https://ashan.org/2017/07/5580941640827314230.jpg)

这个例子里，约定好的帧速是16毫秒，客户端就会假装现在帧速是15.2毫秒，它想要更加提前。结果就是，这些输入来的越来越快。服务器上缓冲区也会跟着变大，这就是为了在尽量不浪费的情况下，度过（丢包的）难关。这样我们的输入就会变成一个连续的，不需要通过上一次输入来进行不可控的预测。

![](https://ashan.org/2017/07/5960516899105394224.jpg)

从上图可以看出，在经过事件膨胀后，我们的帧率变高，帧速变快，发包数量变多，你可以看见图中右边的坡越来越平坦了。它比以前更加快速地上报输入。同时服务器上的缓冲也越来越大了，可以容忍更多地丢包，如果真的发生丢包也有可能在缓冲期间补上。

同时，当我们网络恢复健康的时候，客户端会缩小事件刻度，以较慢的速度去进行发包，服务器也会减小缓冲池的尺寸。

注意，此处的缓冲区功能和TCP中的滑动窗口是不一样的，滑动窗口是在网速好的时候可以发更多的包，所以增大缓冲区，网速堵塞的时候减少发包，减小缓冲区，所以两缓冲区的作用刚好是相反的。

这项技术的目的就是为了解决服务器饥饿的问题，避免忽略掉我们丢失的包。

并且我们还可以通过给包装入更多的帧来优化这个问题，我们当前帧的包还会包括前面每一帧的输入，这是因为玩家顶多每1/60秒才会有一次操作，所以压缩后数据量其实不大。一般你按住“向前”按钮之前，很可能是已经在“前进”了。

结果就是，即使发生丢包，下一个数据包到达时依然会有全部的输入操作，这会在你真正模拟以前，就填充上所有因为丢包而出现的空洞。

## 反射实现状态机
普通状态机方案是通过枚举状态进行状态的转变，这需要我们手动添加枚举状态，并且代码不容易管理，我们可以将每一个状态封装为一个类，然后通过一个Mgr用反射的方式进行控制。  
- GameState的控制类：
![image](https://pic3.zhimg.com/80/v2-e2efcb2582a2c87bf2cafb38e458e0a6_720w.jpg)
- StateMachine的注册函数 RegisterStateByAttributes，这个类调用了一个ClassEnumerator,通过传入的特性类型，以及IState是状态接口，来查找程序集里所有定义了特性标签和继承自这个接口的类。关键部分用红框标记出来了。
![image](https://pic1.zhimg.com/80/v2-dccb2fea556043aa77459194e5cf4b9c_720w.jpg)
- ClassEnumerator 是怎么运作的。遍历程序集里的所有Type，然后找出从指定接口继承的，并且带有指定特性标签的类型，加入到结果。结合第三步的创建实例，就能把所有注册好的状态机找出来，并且创建好实例。  
![image](https://pic1.zhimg.com/80/v2-4e27ead60b8c3f9146bfeaffa7066540_720w.jpg)


## TextMeshPro
































































































































































































































































































































































































































































