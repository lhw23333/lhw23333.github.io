[TOC]
#### 反射
在程序运行过程中，动态的获取其他数据集的元数据，通过获取的类型信息，动态的创建对象，访问成员的过程   

有一种反射形式是可以在类初始化的时候返回变量的偏移地址和函数指针，提供给外界访问。



因为使用的是其他程序集的信息，所以并不能直接实例，也无法创建类型变量，通常通过Assmbly加载其他数据集，然后从中获得我们需要的type信息，根据反射提供的type信息，去使用其他程序集提供的类，函数，变量，在实例的时候也并不能直接通过构造函数实例，只能通过Activator传入Type信息实例，或者通过Type信息查找到构造函数的info信息后，invoke委托调用。

##### Type
类的信息类，，是反射的基础，，是访问元数据的主要方式，，使用Type的成员获取有关的的类型信息，，有关类型的成员（如构造函数，方法，字段，属性和类的事件）
###### 获取Type
```
//1. Object.GetType()可以获取对象的Type
Type type = a.GetType();

//2. 通过typeof关键字 传入类名 也可以得到对象的Type
Type type2 = typeof(int);

//3. 通过类名获取，必须包含命名空间
Type type3 = Type.GetType("System.Int32");

```
可以通过Type获得类型所在程序集的信息
###### 获取类中的所有公共成员

```
//首先得到Type
Type t = typeof(Test);
//然后得到所有公共成员
//需要引用命名空间 using System.Reflection
MemberInfo[] infos = t.GetMembers();
```

###### 获取所有构造函数

```
//1. 获取所有构造函数
ConstructInfo[] ctors = t.GetConstructors();
//2. 获取其中一个构造函数 并执行  
//获取构造函数需要传入Type数组  数组中内容按顺序是参数类型
//执行构造函数传入object数组 表示按顺序传入参数
//...得到无参构造(不需要传入任何类型信息)
ConstructInfo info = t.GetConstructor(new Type[0])
    //执行 无参构造
    info.Invoke(null) as Test;//执行完构造后，根据里氏转换获取构造出的实例
//...得到有参构造
ConstructInfo info2 = t.GetConstructor(new Type[] {typeof(int),typeof(string)});
    //执行 有参构造
    obj = info2.Invoke(new object[] {2,"asd"}) as Test;
```

###### 获取类的公共成员变量

```
//1. 得到所有成员变量
FieldInfo[] fieldInfos = t.GetFields();
//2. 得到指定名称的公共成员变量
FieldInfo infoj = t.GetField("j");
//3. 通过反射获取和设置对象的值
    //通过反射 获取指定对象的值
    print(infoj.GetValue(test));//持有实例对象时，获取指定值
    //通过反射 设置指定对象的某个值
    infoj.SetValue(tese,100);
    print(infoj.GetValue(test));
    
```
###### 获取类的公共成员方法

```
//通过Type类中的GetMethod方法 得到类中的方法
//MethodInfo 是方法的反射信息
Type strType = typeof(string);
//1. 如果存在方法重载 用Type数组表示参数类型
MethodInfo[] methods = strType.GetMethods();
MethodInfo[] subStr = strType.GetMethod("SubString",
    new Type[] {typeof(int),typedof(int) });
    //调用该方法
    
    string = "hello world";
    //第一个参数 相当与 是哪个对象要执行这个成员方法
    object result = subStr.Invoke(str,new object[] {7,5});

```
##### Acticvator
- 是一个可以快速实例化对象的类
- 将Type对象中元数据提取然后快速实例化为对象
- 有参构造

```
Type testType = typeof(Test);
Test testObj =  Activator.CreateInstance(testType) as Test;
```

- 无参构造

```
testObj = Activator.CreatInstance(testType,99,"asd"}) as Test;
```

##### Assmbly
- 程序集类，用于加载其他的程序集，加载后才能获取其他程序集的Type信息，使用dll文件(库文件)提供的变量，函数和类
- 三种加载程序集的函数

```
//加载同一文件下的程序集
Assembly asembly2 = Assembly.Load("程序集名称")；
//加载不同文件下的其他程序集
Assembly asembly = Assembly.LoadFrom("包含程序集清单的文件的名称和路径")；
Assembly.loadFile("要加载文件的完全限定路径")；
//先加载指定程序集，再加载程序集中的一个类对象，之后才可与使用反射

```



##### 特性
###### 特性是什么
- 特性是一种允许我们向程序的程序集添加元数据的语言结构，是用于保存程序结构信息的某种特殊类型的类  
- 特性通过方法将声明信息与C#代码关联。
- 特性与成勋实体关联后，即可在运行时使用反射查询特性信息
```
//[特性名(参数列表)]
//本质是调用特性类的构造函数
//写在类上方
class MyCustomAttribute : Attribute
{
    public string info;
    public MyCustomAttribute(string info)
    {
        this.info = info;
    }
    public void TestFun()
    {
        Console.WriteLine("特性的方法");
    }
}

[MyCustom("添加特性")]
class MyClass
{
    
}
//判断是由使用了某个特性
//参数一：特性的类型
//参数二：代表是否搜索继承链  属性和事件忽略此参数 
 if(t.IsDefined(typeof(MyCustomAttribute),false))//只查找和类关联的特性
        {
            Debug.Log("True Or False");
        }
//获取Type数据中的所有特性
object[] array = t.GetCustomAttributes(true);
        for(int i = 0; i < array.Length; i++)
        {
            if(array[i] is MyCustomAttribute)
            {
                Console.WriteLine((array[i] as MyCustomAttribute).info);
                (array[i] as MyCustomAttribute).TestFun();//通过特性类访问特性变量和特性方法
            }
        }

```

###### 特性限定使用范围

```
//通过给特性加一个系统特性，来限制其使用范围
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Struct,AllowMultiple = true,Inherited =true)]
//参数一：AttributeTargets.Class 特性能用在哪些地方（class 或 struct），通过枚举来限定
//参数二：AllowMultiple 是否允许多个特性实例在同一个目标上
//参数三：Inherited 特性是否能够被派生类和重写成员继承

```
###### 系统自带特性
- 过时特性  
```
//参数一：调用过时方法时 提示的内容
//参数二：true使用该方法时会报错 false使用该方法时会警告
[Obsolete("该方法过时",true)]

```
- 调用信息特性

```
public void SpeakCaller(string str , [CallerFilePath] string fileName = "",
[CallerLineNumber] int line = 0,
[CallerMemberName] string target  = "")
{
    print(str);
    print(fileName);
    print(LineNumber);
    print(target);
}
```
- 外部DLL包函数特性

```
//DLLImport

//用来标记非。Net（C#）的函数，表明该函数在一个外部的DLL中定义
//一般用来调用C或者C++的ＤＬＬ包写好的方法
//需要引入空间 using namespace System.Runtime.InteropService


```
##### 通过反射获取泛型类型

```
List<string>list = new List<string>();
Type listType = list.GetType();

Type[] types =  listType.GetGenericArguments()//得到泛型参数
```


#### 引用类型和值类型
##### 堆和栈的区别
①申请速度：

- 栈的申请速度较快，但却不受程序员的控制；
- 堆的申请速度较慢，容易产生内存碎片，但是用起来比较方便  

②申请大小：

- 栈申请的容量较小1-2M
- 堆申请大小虽受限于系统中有效的虚拟内存，但较大  

③存储内容

- 栈主要保存线程代码的执行位置和一些调用代码的数据
- 堆主要保存对象和数据  

④管理者

- 栈自我管理
- 堆，由项目脚本运行时的内存管理器（Mono或IL2CPP）管理。

如何确定分配到哪个内存区
- a.值类型和指针总分配在被声明的地方，声明在哪儿就分配在哪儿
- b.非空引用类型对象和所有装箱值类型对象总是分配在堆内存上

##### struct和class的区别 struct是值类型，class是引用类型，分配内存时
```
SomeType[] oneTypes = new SomeType[100];
//值类型一次分配，
//引用类型 刚开始需要100次分配，分配后数组的各元素值为null，
//然后再初始化100个元素
```
这将消耗更多的时间，造成更多的内存碎片。所以，如果类型的职责主要是存储数据，值类型比较合适。  
一般来说，值类型（不支持多态）适合存储供 C#应用程序操作的数据，而引用类型（支持多态）应该用于定义应用程序的行为。

#### unity协程
- 一个进程可以有多个线程、一个线程可以有多个协程。
- 协程是一边吃饭一边说话。线程是一边听音乐一边写作业。（线程并行，协程并发）  .
- 如果同一个协程方法被两个方法在相对短的时间内都进行调用，那么就会出现逻辑上的错误！这个时候需要用锁来锁定！

 ①协程是用户级的，拥有自己的寄存器上下文和栈，②完全由用户控制，协程切换开销比较小。③协程是串行的，在任何一个时刻，同属于一个组当中的协程，只有一个协程在跑。

①线程是内核级的，共享进程内寄存器上下文。②线程调度需要操作系统进行调度，线程切换需要切换内核态，开销耗时比较大。③线程是并行的，多个线程可以同时运行。因为是多线程共享资源，所有访问全局变量的时候要加锁

##### IEnumerator学习 

```
using System.Runtime.InteropServices;

namespace System.Collections
{
    [ComVisible(true)]
    [Guid("496B0ABF-CDEE-11d3-88E8-00902754C43A")]
    public interface IEnumerator
    {
        object Current { get; }

        bool MoveNext();
        void Reset();
    }
}
```
详细的Unity协程流程分析：  
- C#层调用StartCoroutine方法，将IEnumerator对象（或者是用于创建IEnumerator对象的方法名字符串）传入C++层。
- 通过mono的反射功能，找到IEnumerator上的moveNext、current两个方法，然后创建出一个对应的Coroutine对象，把两个方法传递给这个Coroutine对象。【创建好之后这个Coroutine对象会保存在MonoBehaviour一个成员变量List中，这样使得MonoBehaviour具备StopCoroutine功能，StopCoroutine能够找到对应Coroutine并停止。】
- 调用这个Coroutine对象的Run方法。
- 在Coroutine.Run中，然后调用一次MoveNext。①如果MoveNext返回false，表示Coroutine执行结束，进入清理流程；②如果返回true，表示Coroutine执行到了一句yield return处，这时就需要调用invocation(m_Current).Invoke取出yield return返回的对象monoWait，再根据monoWait的具体类型（null、WaitForSeconds、WaitForFixedUpdate等），将Coroutine对象保存到DelayedCallManager的callback列表m_CallObjects中。
- 至此，Coroutine在当前帧的执行即结束。
- 之后游戏运行过程中，游戏主循环的PlayerLoop方法会在每帧的不同时间点以不同的modeMask调用DelayedCallManager.Update方法，Update方法中会遍历callback列表中的Coroutine对象，如果某个Coroutine对象的monoWait的执行条件满足，则将其从callback列表中取出，执行这个Coroutine对象的Run方法，回到之前的执行流程中。 

总结:
- 协程只是看起来像多线程一样，其实还是在主线程上执行。
- 协程只是个伪异步，内部的死循环依旧会导致应用卡死。

##### unity协程时无栈协程  
函数调用的本质是压栈，协程的唤醒也一样，调用IEnumerator.MoveNext()时会把协程方法体压入当前的函数调用栈中执行，运行到yield return后再弹栈。这点和有些语言中的协程不大一样，有些语言的协程会维护一个自己的函数调用栈，在唤醒的时候会把整个函数调用栈给替换，这类协程被称为有栈协程，而像C#中这样直接在当前函数调用栈中压入栈帧的协程我们称之为无栈协程。
Unity中的协程是无栈协程，它不会维护整个函数调用栈，仅仅是保存一个栈帧。  

##### yield return
yield时c#的一个语法糖  
执行协程之后，unity的每一次GameLoop都会调用执行协程，遇到yield return会挂起，直到条件满足才会唤醒执行后面的代码。  
yield时什么   

```
public interface IEnumerator
{   
    object Current { get; } 
    bool MoveNext(); 
    void Reset(); 
}
```

C#从2.0开始提供了有yield组成的迭代器块，编译器会自动更具迭代器块创建了枚举器。当MoveNext返回true时，代表遇到了一个yield语句，这时候取出返回的monoWait对象,再根据具体类型将协程对象保存到DelayedCallManager的callback列表m_CallObjects中。

##### Lua协程  
Lua中的协程和Unity协程的区别，最大的就是Lua协程不是抢占式的执行，也就是说不会被主动执行类似MoveNext这样的操作，而是需要我们去主动激发执行，就像上一个例子一样，自己去tick这样的操作。

#### Mono和IL2CPP的区别
跨平台指一次编译，不需要任何代码修改，应用程序就可以运行在任意平台上跑。  
Unity就提供了跨平台发布功能  
![image](https://pic1.zhimg.com/80/v2-19eabdc00dc2c59c2239227716fb0df0_720w.jpg)  
unity的实现方式时通过Mono和IL2CPP两种脚本后处理的方式。  
Mono的目标是在尽可能多的平台上使.net标准的东西能正常运行的一套工具，核心在于“跨平台的让.net代码能运行起来“。  
Mono组成组件：C# 编译器，CLI虚拟机，以及核心类别程序库。  
Mono的编译器负责生成符合公共语言规范的映射代码，即公共中间语言CIL  
![image](https://pic4.zhimg.com/v2-822e8c5f5036ab4c5ac7650bf546ccaf_r.jpg)  
 优点：
1. 构建应用非常快
2. 由于Mono的JIT(Just In Time compilation ) 机制, 所以支持更多托管类库
3. 支持运行时代码执行
4. 必须将代码发布成托管程序集(.dll 文件 , 由mono或者.net 生成 )
5. Mono VM在各个平台移植异常麻烦，有几个平台就得移植几个VM（WebGL和UWP这两个平台只支持 IL2CPP）
6. Mono版本授权受限，C#很多新特性无法使用
7. iOS仍然支持Mono , 但是不再允许Mono(32位)应用提交到Apple Store  


#### IL2CPP【AOT编译】  
指运行前编译  

##### JIT和AOT编译的优缺点
JIT优点：  
1. 可以根据当前硬件情况实时编译生成最优机器指令（ps. AOT也可以做到，在用户使用是使用字节码根据机器情况在做一次编译）
2. 可以根据当前程序的运行情况生成最优的机器指令序列
3. 当程序需要支持动态链接时，只能使用JIT
4. 可以根据进程中内存的实际情况调整代码，使内存能够更充分的利用

JIT缺点：  
1. 编译需要占用运行时资源，会导致进程卡顿
2. 由于编译时间需要占用运行时间，对于某些代码的编译优化不能完全支持，需要在程序流畅和编译时间之间做权衡
3. 在编译准备和识别频繁使用的方法需要占用时间，使得初始编译不能达到最高性能

AOT优点：  
1. 在程序运行前编译，可以避免在运行时的编译性能消耗和内存消耗
2. 可以在程序运行初期就达到最高性能
3. 可以显著的加快程序的启动

IL2CPP分为两个部分
1. AOT（静态编译）编译器：把IL中间语言转换成CPP文件
2. 运行时库：例如垃圾回收、线程/文件获取（独立于平台，与平台无关）、内部调用直接修改托管数据结构的原生代码的服务与抽象  
![image](https://pic2.zhimg.com/v2-f2e9975835f3d2cc8e41b38dc94f6545_r.jpg)  
il2cpp.exe 是由C#编写的受托管的可执行程序，它接受我们在Unity中通过Mono编译器生成的托管程序集，并生成指定平台下的C++代码。  
优点：  
1. 运行效率快
2. Mono VM在各个平台移植，维护非常耗时，有时甚至不可能完成  
3. 可以利用现成的在各个平台的C++编译器对代码执行编译期优化，这样可以进一步减小最终游戏的尺寸并提高游戏运行速度。  
4. IL2CPP的虚拟机只是负责内存托管，分配线程等工作，不负责代码的解释与执行，和Mono的VM有很大区别