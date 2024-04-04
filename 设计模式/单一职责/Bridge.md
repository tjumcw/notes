## Bridge 

### 一、简介

#### 1、动机

- 由于某些类型的固有的实现逻辑，使得它们具有两个变化乃至多个变化的维度

#### 2、模式定义

- 将抽象部分（业务功能）与实现部分（平台实现）分离，使它们都可以独立地变化



### 二、示例

#### 1、背景

-  实现通信的简单模块，定义基类Messager，需要处理
  - 用户的Login，SendMessage，SendPicture
  - 平台实现层面，可以播放声音，画图形，展示图象，网络连接

- 根据不同平台需要设计不同的类，如PC平台基类和移动平台基类
  - 平台实现层面需要重写播放声音，画图形，展示图象，网络连接四个虚函数
- 根据每个平台实现不同的版本，如轻量版，完美版等等
  - 需要继承PC基类或是移动基类，实现自己功能
  - 各个版本的功能实现不一样
  - 不同平台的相同版本实现流程一样，只是调用不同基类的函数



#### 2、结构化版本

```c++
class Messager{
public:
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;

    virtual void PlaySound()=0;
    virtual void DrawShape()=0;
    virtual void WriteText()=0;
    virtual void Connect()=0;
    
    virtual ~Messager(){}
};


//平台实现 n
class PCMessagerBase : public Messager{
public:
    
    virtual void PlaySound(){
        //**********
    }
    virtual void DrawShape(){
        //**********
    }
    virtual void WriteText(){
        //**********
    }
    virtual void Connect(){
        //**********
    }
};

class MobileMessagerBase : public Messager{
public:
    
    virtual void PlaySound(){
        //==========
    }
    virtual void DrawShape(){
        //==========
    }
    virtual void WriteText(){
        //==========
    }
    virtual void Connect(){
        //==========
    }
};



//业务抽象 m
class PCMessagerLite : public PCMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        PCMessagerBase::Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        PCMessagerBase::WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        PCMessagerBase::DrawShape();
        //........
    }
};



class PCMessagerPerfect : public PCMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        PCMessagerBase::PlaySound();
        //********
        PCMessagerBase::Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        PCMessagerBase::PlaySound();
        //********
        PCMessagerBase::WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        PCMessagerBase::PlaySound();
        //********
        PCMessagerBase::DrawShape();
        //........
    }
};


class MobileMessagerLite : public MobileMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        MobileMessagerBase::Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        MobileMessagerBase::WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        MobileMessagerBase::DrawShape();
        //........
    }
};


class MobileMessagerPerfect : public MobileMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        MobileMessagerBase::PlaySound();
        //********
        MobileMessagerBase::Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        MobileMessagerBase::PlaySound();
        //********
        MobileMessagerBase::WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        MobileMessagerBase::PlaySound();
        //********
        MobileMessagerBase::DrawShape();
        //........
    }
};


void Process(){
        //编译时装配
        Messager *m =
            new MobileMessagerPerfect();
}

```

- 若平台实现为n，业务抽象为m，需要有1 + n + m * n个类



#### 3、优化和重构

类似Decorator模式，找出重复代码，并化继承为组合,只展示Login，如下：

```c++
class PCMessagerPerfect : public PCMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        PCMessagerBase::PlaySound();
        //********
        PCMessagerBase::Connect();
        //........
    }

};

class MobileMessagerPerfect : public MobileMessagerBase {
public:
    
    virtual void Login(string username, string password){
        
        MobileMessagerBase::PlaySound();
        //********
        MobileMessagerBase::Connect();
        //........
    }

};

```

- 可见上述代码，流程完全是重复的，根据Decorator经验，一步到位转化为：

```c++
class PCMessagerPerfect  {
    Messager* messager;//= new PCMessagerBase()
public:
    
    virtual void Login(string username, string password){
        
       	messager->PlaySound();
        //********
        messager->Connect();
        //........
    }

};

class MobileMessagerPerfect {
    Messager* messager;//= new MobileMessagerBase()
public:
    
    virtual void Login(string username, string password){
        
		messager->PlaySound();
        //********
        messager->Connect();
        //........
    }

};
```

- 通过上述代码，编译时复用，运行时多态产生区别，可以去除除了名字不同其他都一致的类

```c++
class MessagerPerfect  {
    Messager* messager;//= new PCMessagerBase()
public:
    
    virtual void Login(string username, string password){
        
       	messager->PlaySound();
        //********
        messager->Connect();
        //........
    }

};
```

- 同理其他lite版本也可以只留下一个类，类似上述操作
- 但还存在问题，即在运行时，下面代码会有问题

```c++
Messager* messager;//= new PCMessagerBase()  因为PCMessagerBase是抽象类(纯虚基类)
```

- PCMessagerBase类只复写了Messager类的部分虚函数，没有全部重写

- 回头审视Messager类定义，发现其功能模块不一致，将平台实现与业务抽象塞到一个类中了
- 对Messager类进行拆分

```c++
class Messager{
public:
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;
    
    virtual ~Messager(){}
};

class MessagerImp{
public:
    virtual void PlaySound()=0;
    virtual void DrawShape()=0;
    virtual void WriteText()=0;
    virtual void Connect()=0;
    
    virtual ~MessagerImp(){}
};
```

- 需要对应修改其他类，如MessagerPerfect，只展示这一个类的一个函数，其他类和其他函数均类似

```c++
class MessagerPerfect : public Messager {
    MessagerImp* messagerImp;//= new PCMessagerBase()
public:
    
    virtual void Login(string username, string password){
        
       	messagerImp->PlaySound();
        //********
        messagerImp->Connect();
        //........
    }

};
```

- 其中，继承Messager表示使用基类定义的接口规范，类似Decorator模式
- 组合MessagerImp是为了具体的功能实现
- 再继续考虑MessagerPerfect和MessageLite两个子类都有MessagerImp对象，将其抽到上层基类

```c++
class Messager{
protected:
     MessagerImp* messagerImp;//...
public:
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;
    
    virtual ~Messager(){}
};

class MessagerPerfect : public Messager {
public:
    virtual void Login(string username, string password){
        
       	messagerImp->PlaySound();
        //********
        messagerImp->Connect();
        //........
    }
};

class MessagerLite :public Messager {
public:
    
    virtual void Login(string username, string password){
        
        messagerImp->Connect();
        //........
    }
};
```



#### 4、桥模式实现

```c++
class Messager{
protected:
     MessagerImp* messagerImp;//...
public:
    virtual void Login(string username, string password)=0;
    virtual void SendMessage(string message)=0;
    virtual void SendPicture(Image image)=0;
    
    virtual ~Messager(){}
};

class MessagerImp{
public:
    virtual void PlaySound()=0;
    virtual void DrawShape()=0;
    virtual void WriteText()=0;
    virtual void Connect()=0;
    
    virtual MessagerImp(){}
};


//平台实现 n
class PCMessagerImp : public MessagerImp{
public:
    
    virtual void PlaySound(){
        //**********
    }
    virtual void DrawShape(){
        //**********
    }
    virtual void WriteText(){
        //**********
    }
    virtual void Connect(){
        //**********
    }
};

class MobileMessagerImp : public MessagerImp{
public:
    
    virtual void PlaySound(){
        //==========
    }
    virtual void DrawShape(){
        //==========
    }
    virtual void WriteText(){
        //==========
    }
    virtual void Connect(){
        //==========
    }
};



//业务抽象 m

//类的数目：1+n+m

class MessagerLite :public Messager {
    
public:
    
    virtual void Login(string username, string password){
        
        messagerImp->Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        messagerImp->WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        messagerImp->DrawShape();
        //........
    }
};



class MessagerPerfect  :public Messager {  
   
public:
    
    virtual void Login(string username, string password){
        
        messagerImp->PlaySound();
        //********
        messagerImp->Connect();
        //........
    }
    virtual void SendMessage(string message){
        
        messagerImp->PlaySound();
        //********
        messagerImp->WriteText();
        //........
    }
    virtual void SendPicture(Image image){
        
        messagerImp->PlaySound();
        //********
        messagerImp->DrawShape();
        //........
    }
};


void Process(){
    //运行时装配
    MessagerImp* mImp=new PCMessagerImp();
    Messager *m =new Messager(mImp);
}
```

- 关键点：Messager类中有一个MessagerImp的指针
- 由上述代码和初始代码分析，类数量大大减少，但功能依靠组合和之前实现的功能数目是一样的
- 初始的Messager类放了不同的变化方向
  - 一个是平台实现，如PC，Mobile，Pad等平台
  - 一个是业务抽象，即Lite、Perfect等版本

- 两个不同变化方向带来的行为的多态实现也是不同方向，不应放在一个类中
- 每个变化方向都有继承自己的子类
  - 业务抽象方向：Messager有自己的子类MessagerLite和MessagerPerfect
  - 平台实现方向：MessagerImp有自己的子类PCMessagerImp和MobileMessagerImp



### 三、总结

- Bridge模式使用“对象间的组合关系”解耦了抽象和实现之间固有的绑定关系，使得抽象和实现可以沿着各自的维度来变化。所谓沿着各自维度的变化，即“子类化他们”
- Bridge有时候类似于多继承方案，但多继承方案往往违背单一职责原则（即一个类只有一个变化的原因），复用性较差。Bridge模式是比多继承方案更好的解决方法
- Bridge模式应用于类中存在多个变化维度