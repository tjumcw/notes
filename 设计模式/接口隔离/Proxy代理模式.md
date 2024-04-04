## Proxy代理模式

### 一、简介

#### 1、动机

- 在面向对象系统中，有些对象由于某种原因，直接访问会给使用者、或者系统结构带来很多麻烦
  - 比如对象创建的开销很大，或者某些操作需要安全控制，或者需要进程外的访问等

- 如何在不失去透明操作（一致性，不改变本来访问对象的方式）的同时管理/控制这些对象特有的复杂性
  - 背后的麻烦事（对象开销大、安全控制、分布式访问）都不知道，实现对使用者隔离

#### 2、定义

- 为其他对象提供一种代理以控制（隔离，使用接口）对这个对象的访问

![image](https://user-images.githubusercontent.com/106053649/176859225-0ae7452e-dd71-4db5-8136-109ba8bfad9d.png)

- 接口是Subject（抽象基类）
- 真正对象是RealSubject
- 由于某种原因，可能由于性能优化、安全控制或分布式访问等，无法直接访问真正对象
- 此时通过代理类对象Proxy来实现对真正对象的控制/管理（实际代理类的实现很负责）



### 二、示例

#### 1、背景

- 按照对上述结构的分析为背景

#### 2、基础代码

```c++
//接口类
class ISubject(){
public:
    virtual void process() = 0;
};

//实现接口的具体类
class RealSubject : public ISubject{
public:
    virtual void process(){
        //...
    }
};

class ClientApp{
    ISubjetc* subject;
public:
    
    ClientApp(){
        subject = new RealSubject();	//或者通过工厂模式等方式进行包装对象
    }
    
    void DoTask(){
        //...
        subject->process();
        //...
    }
};
```

- 但这种创建对象的方式可能由于性能原因、安全控制原因或分布式环境，压根拿不到这个RealSubject对象



#### 3、重构和优化（针对特殊场景）

- 针对上述拿不到对象的场景，考虑增加一个代理类

```c++
//接口类
class ISubject(){
public:
    virtual void process() = 0;
};

//实现接口的代理类
class SubjectProxy : public ISubject{
public:
    virtual void process(){
        //...对RealSubject的一种间接访问
    }
};

class ClientApp{
    ISubjetc* subject;
public:
    
    ClientApp(){
        subject = new SubjectProxy();	//或者通过工厂模式等方式进行包装
    }
    
    void DoTask(){
        //...
        subject->process();
        //...
    }
};
```



### 三、总结

- “增加一层间接层”是软件系统中对许多复杂问题的一种常见解决方案。
- 在面向对象系统中，直接使用某些对象会带来很多问题
  - 作为间接层的proxy对象便是解决这一问题的常用手段
- 具体proxy设计模式的实现方法、实现粒度都相差很大
