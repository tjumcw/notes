## Visitor模式

### 一、简介

#### 1、动机

- 在软件构建过程中，由于需求的改变，某些类层次结构中常常需要增加新的行为（方法）
- 如果直接在基类中做这样的更改，将会给子类带来很繁重的变更负担，甚至破坏原有设计
- 如何在不更改层次结构的前提下，在运行时根据需要透明地为类层次结构上的各个类动态添加新的操作，避免上述问题

#### 2、定义

- 表示一个作用于某对象结构中的各元素的操作。使得可以在不改变（稳定）各元素的前提下定义（扩展）作用于这些元素的新操作（变化）



### 二、示例

#### 1、基础代码

```c++
#include <iostream>
using namespace std;

class Visitor;


class Element
{
public:
    virtual void Func1() = 0;
    virtual ~Element(){}
};

class ElementA : public Element
{
public:
    void Func1() override{
        //...
    }
};

class ElementB : public Element
{
public:
    void Func1() override{
        //***
    }
};
```

- 由以上代码，有一个接口基类Element，还有两个实现了接口的具体类ElementA，ElementB



- 假设需求变更，若需要对Element添加新的行为如增加Func2()，则需要修改接口以及override虚函数，若

```c++
#include <iostream>
using namespace std;

class Visitor;


class Element
{
public:
    virtual void Func1() = 0;
    
    virtual void Func2(int data)=0;
    virtual void Func3(int data)=0;
    //...
    
    virtual ~Element(){}
};

class ElementA : public Element
{
public:
    void Func1() override{
        //...
    }
    
    void Func2(int data) override{
        //...
    }
    
};

class ElementB : public Element
{
public:
    void Func1() override{
        //***
    }
    
    void Func2(int data) override {
        //***
    }
    
};
```

- 开发完毕并部署的情况下，这样的改变方式代价太高了（修改接口基类及所有子类的方法），违背开闭原则



#### 2、重构和优化

- 预料到未来会为整个类层次结构增加新的操作，但增加多少操作，什么操作现在是未知的
- 修改Element类，做出预先设计，即定义了accept()的虚函数，参数为Visitor& visitor

```c++
#include <iostream>
using namespace std;

class Visitor;


class Element
{
public:
    virtual void accept(Visitor& visitor) = 0; //第一次多态辨析

    virtual ~Element(){}
};

class ElementA : public Element
{
public:
    void accept(Visitor &visitor) override {
        visitor.visitElementA(*this);
    }
};

class ElementB : public Element
{
public:
    void accept(Visitor &visitor) override {
        visitor.visitElementB(*this); //第二次多态辨析
    }
};

//针对每个Element的具体子类定义虚函数，故需要直到有多少子类
class Visitor{
public:
    virtual void visitElementA(ElementA& element) = 0;
    virtual void visitElementB(ElementB& element) = 0;
    
    virtual ~Visitor(){}
};

//==================================

//扩展1
class Visitor1 : public Visitor{
public:
    void visitElementA(ElementA& element) override{
        cout << "Visitor1 is processing ElementA" << endl;
    }
        
    void visitElementB(ElementB& element) override{
        cout << "Visitor1 is processing ElementB" << endl;
    }
};
     
//扩展2
class Visitor2 : public Visitor{
public:
    void visitElementA(ElementA& element) override{
        cout << "Visitor2 is processing ElementA" << endl;
    }
    
    void visitElementB(ElementB& element) override{
        cout << "Visitor2 is processing ElementB" << endl;
    }
};
        
        
int main()
{
    Visitor2 visitor;
    ElementB elementB;
    elementB.accept(visitor);// double dispatch
    
    ElementA elementA;
    elementA.accept(visitor);

    
    return 0;
}
```

- 看main函数部分，其调用流程如下：
  - 声明一个具体的子类Visitor2对象以及ElementB对象
  - elementB调用override的虚函数accept(visitor)，接受的是Visitor2类型的对象，定义的参数为Visitor&
    - 即实现了父类的引用指向子类对象，多态调用Visitor2重写的visitElementB()函数
  - 下面关于ElementA的部分类似

- 与第一种示例代码的对比及理解

  - Element基类中的任一个Func，如Func1()，就是指所有Element具体类的一种操作

  - 对应到优化后的版本，就是一个Visitor类的具体实现，如Visitor1，原来增加一个Func就对应到增加一个新的Visitor子类

  - 初始代码中每个Element具体类的Func都不一致，每个具体子类都需要重写虚函数

  - 对应到优化后的版本，就是一个Visitor类的具体实现，即需要对所有Element的具体类都对应创建一个虚函数（预先知道数量）

    - ```c++
      class Visitor{
      public:
          virtual void visitElementA(ElementA& element) = 0;
          virtual void visitElementB(ElementB& element) = 0;
          
          virtual ~Visitor(){}
      };
      ```

    - 这也引出该设计模式的一大缺点：必须知道有多少个这样需要增加操作的具体子类，即Element子类的数量

    - 其优点在于，若需增加操作，只需要扩展一个Visitor具体子类，并重写针对每个Element子类的虚函数（扩展）

    - 而初始代码若增加一个Func，需要直接修改Element基类，并在所有子类中重写虚函数（更改）

  - Visitor基类稳定的前提就是Element的所有子类都确定（需要对每个子类定义一个接口虚函数，交给实现类重写）
    - 每个具体Visitor实现类都重写针对每个Element具体子类的虚函数，对应到初始代码每个Element具体子类重写Func
      - 保证针同一种操作在不同Element子类上有各自的实现
    - 即将一个Func对应到一个Visitor，原先新增Func需要修改Element接口基类，现在通过二次分发只需扩展Visitor子类

- Element有几个具体子类，Visitor基类就要有几个对应的虚函数



### 三、总结

#### 1、结构

![image](https://user-images.githubusercontent.com/106053649/176994343-6785269f-0f65-464d-b5b2-83df3210bd50.png)

- 红色稳定，蓝色变化

- 若Element子类数量不能缺点，不能使用Visitor模式

#### 2、总结

- Visitor模式通过所谓双重分发（double dispatch）来实现在不更改（不添加新的操作-编译时）Element类层次结构的前提下，在运行时透明地为类层次结构上的各个类动态添加新的操作（支持变化）
  - 类层次结构稳定即需要直到Element具体子类的数量
- 所谓双重分发即Visitor模式中间包括了两个多态分发（注意其中的多态机制）
  - 第一个为accept方法的多态辨析
  - 第二个为visitElementX方法的多态辨析
- Visitor模式的最大缺点在于扩展类层次结构（增添新的Element子类），会导致Visitor类的改变
- 因此Visitor适用于“Element类层次结构稳定，而其中的操作却经常面临频繁改动”
