## Template Method

### 一、概述

- 算法的骨架是稳定的，而将一些步骤延迟（变化）到子类中。
  - 延迟的手法就是通过虚函数实现晚绑定
- 子类不改变（复用）算法结构，重定义（override重写）某些特定步骤
  - 支持子类变化，更灵活，易扩展



### 二、示例

- 背景：考虑在一个已有框架Framework上开发一个应用程序application

#### 1、结构化设计

```c++
class Framework{
public:
    void step1(){
        //...
    }  
    void step3(){
        //...
    }
    void step5(){
        //...
    }
};

class Application{
public:
    bool step2(){
        //...
    }
    void step4(){
        //...
    }
};

int main(){
    Framework work;
    Application app;
    
    work.step1();
    
    if(app.step2()){
        work.step3();
    }
    
    for(int i = 0; i < 4; i++){
        app.step4();
    }
    
    work.step5();
}
```

- 代码很简单，就是应用程序开发的人员在开发过程中需要调用框架开发人员设计的方法，并且自己实现整个应用程序的流程
- 框架在前，程序开发在后，由程序开发人员调用框架方法，早绑定



#### 2、Template method实现

```c++
class Framework{
public:
    //稳定 template method
    void run(){
        
        step1();
        
        if(step2()){	//可变 ==> 支持虚函数的多态调用
            step3();
        }
        
        for(int i = 0; i < 4; i++){
            step4();	//可变 ==> 支持虚函数的多态调用
        }
        
        step5();
    }
    virtual ~Framework(){}

protected:
    void step1(){
        //...
    }
    void step3(){
        //...
    }
    void step5(){
        //...
    }
    
    virtual bool step2() = 0;
    virtual void step5() = 0;
};

class Application : public Framework{
protected:
    virtual bool step2(){
        //...子类重写父类虚函数
    }
    virtual void step4(){
        //...子类重写父类虚函数
    }
};

int main(){
    Framework* work = new Application();  //父类指针指向子类对象，多态
    work->run();
    
    delete work;
}
```

- 在修改后的实现中，应用程序执行流程由框架开发人员决定，应用程序开发人员只需要重写某几个特定步骤的虚函数实现即可
- 实现了稳定和变化的分离
  - 首先开发程序的流程即run()方法流程是稳定的，其中step1，3，5也是稳定的
  - 将不稳定的可能根据需求发生变化的step2，4写成虚函数，交给子类去重写
  - 虽然框架在前，程序开发在后，但多态实现的反向控制机制实现了框架调用应用的晚绑定

