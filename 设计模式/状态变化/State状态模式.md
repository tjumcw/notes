## State状态模式

### 一、简介

#### 1、动机

- 在软件构建过程中，某些对象的状态如果改变，其行为也会随之而发生变化。
  - 比如文档处于只读状态，其支持的行为和读写状态支持的行为就可能完全不同
- 如何在运行时根据对象的状态来透明地更改对象的行为，而不会为对象操作和状态转化之间引入紧耦合

#### 2、定义

- 允许一个对象在其内部状态改变时改变它的行为。从而使对象看起来似乎修改了其行为。



### 二、示例

#### 1、背景

- 设计一个网络应用，该应用要根据网络的状态做出行为调整

#### 2、基础代码

```c++
enum NetworkState
{
    Network_Open,
    Network_Close,
    Network_Connect,
};

class NetworkProcessor{
    
    NetworkState state;

public:
    
    void Operation1(){
        if (state == Network_Open){

            //**********
            state = Network_Close;
        }
        else if (state == Network_Close){

            //..........
            state = Network_Connect;
        }
        else if (state == Network_Connect){

            //$$$$$$$$$$
            state = Network_Open;
        }
    }

    public void Operation2(){

        if (state == Network_Open){
            
            //**********
            state = Network_Connect;
        }
        else if (state == Network_Close){

            //.....
            state = Network_Open;
        }
        else if (state == Network_Connect){

            //$$$$$$$$$$
            state = Network_Close;
        }
    
    }

    public void Operation3(){

    }
};
```

- 针对每个操作，其根据当前状态不同所执行的具体行为也不同
- 执行完具体行为后，状态会发生转移，到一个新状态
- 不同的操作的具体行为和状态转移方式不同



#### 3、重构和优化

- 根据上述代码可知，对象的状态发生改变，其对应的行为也会发生变化
- 考虑到不易于增加新状态，如增加一个Network_Wait，要修改：
  - 枚举类型
  - 增加新的else if分支
  - 状态转移的逻辑要发生很大变化

- 对代码按照State模式进行重构和优化，首先将枚举类型改为类，并设计一个作为接口的抽象基类

```c++
class NetworkState{

public:
    NetworkState* pNext;
    virtual void Operation1() = 0;
    virtual void Operation2() = 0;
    virtual void Operation3() = 0;

    virtual ~NetworkState(){}
};
```



- 设计具体类，每个不同的状态继承接口并实现自己的虚函数

```c++
class OpenState :public NetworkState{
    
    static NetworkState* m_instance;
public:
    static NetworkState* getInstance(){
        if (m_instance == nullptr) {
            m_instance = new OpenState();
        }
        return m_instance;
    }

    void Operation1(){
        
        //**********
        pNext = CloseState::getInstance();
    }
    
    void Operation2(){
        
        //..........
        pNext = ConnectState::getInstance();
    }
    
    void Operation3(){
        
        //$$$$$$$$$$
        pNext = OpenState::getInstance();
    }
    
    
};
```

- static字段和方法体现了Singleton设计模式的思想，因为某一个特定的状态对象只需要一个就可以了
- 这样修改后的Operation1()方法与原来有差异，可能需要上下文context作为参数传入，但核心逻辑没变
  - 因为之前是在if-else的流程控制语句中，满足条件执行特定语句
- pNext是定义在接口基类中的NetworkState类型指针，调用operation1()会指向CloseState类的单例对象
  - 对比最上面的基础代码，相当于就是在OpenState时调用完operation1()后状态转移至CloseState
  - 这样的设计把所有与OpenState相关的内容都抽出来整合在一起
  - 类似，可以把与其他具体状态相关的所有内容也抽出来，整合在一起形成一个继承接口并实现自己功能的具体状态类
  - 这样未来若有新的状态，只需扩展这样的子类并实现其正确行为即可，无需修改，接口是稳定的



- 当进行网络应用时，不断执行操作并修改状态，类设计如下:

```c++
class NetworkProcessor{
    
    NetworkState* pState;
    
public:
    
    NetworkProcessor(NetworkState* pState){
        
        this->pState = pState;
    }
    
    void Operation1(){
        //...
        pState->Operation1();
        pState = pState->pNext;
        //...
    }
    
    void Operation2(){
        //...
        pState->Operation2();
        pState = pState->pNext;
        //...
    }
    
    void Operation3(){
        //...
        pState->Operation3();
        pState = pState->pNext;
        //...
    }

};
```

- 根据构造函数传入的具体指针，实现了NetworkState类型的父类指针pState指向子类对象
- 调用某个Operation即在函数体内部，如传入构造函数为OpenState的指针，若调用Operation1，其通过
  - pState->Operation()多态调用OpenState重写的虚函数Operation1()
  - 执行完Operation1()内的其他操作后，pNext会通过CloseState::getInstance()方法变更状态为唯一的CloseState单例
    - 静态函数通过类名来调用
  - 利用pState = pState->pNext进行状态转移



### 三、总结

- State模式将所有于一个特定状态相关的行为都放入一个State的子类对象中，在对象状态切换时，切换相应的对象
- 同时维持State的接口（State虚基类是稳定的），这样实现了具体操作与状态转换之间的解耦
- 为不同的状态引入不同的对象使得状态转换变得更加明确
  - 即实现了继承接口的不同具体子类，如OpenState和CloseState
- 而且可以保证不会出现状态不一致的情况，因为转换是原子的
  - 即要么彻底转换过来，要么不转换
- 如果State对象没有实例对象，那么上下文可以共享同一个Satte对象（单例模式），从而节省对象开销