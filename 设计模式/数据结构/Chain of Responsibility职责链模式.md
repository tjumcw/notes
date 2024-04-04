## Chain of Responsibility职责链模式

### 一、简介

#### 1、动机

- 在软件构建过程中，一个请求可能被多个对象处理，但是每个请求在运行时只能有一个接受者
- 如果显式指定，将必不可少地带来请求发送者与接收者的紧耦合
- 如何使请求的发送者不需要指定具体的接受者，让请求的接受者自己在运行时决定来处理请求，从而使两者解耦

#### 2、定义

- 使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系
- 将这些对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止



### 二、示例

#### 1、背景

- 有三种请求类别，不同的处理者只能受理特定的请求类别

#### 2、示例代码

- 首先给出有关请求的代码，为三种枚举类型以及请求类的设计

```c++
#include <iostream>
#include <string>

using namespace std;

enum class RequestType
{
    REQ_HANDLER1,
    REQ_HANDLER2,
    REQ_HANDLER3
};

class Reqest
{
    string description;
    RequestType reqType;
public:
    Reqest(const string & desc, RequestType type) : description(desc), reqType(type) {}
    RequestType getReqType() const { return reqType; }
    const string& getDescription() const { return description; }
};
```

- Reqest对象包含了有关请求的一些信息，通过get()方法能在外界拿到相关的成员字段信息



- 需要为不同的请求类型设计不同的处理函数Handler，根据面向接口编程，设计其抽象基类ChainHandler：

```c++
class ChainHandler{
    
    ChainHandler *nextChain;		//指向自身的指针，由于其是接口抽象类，形成了多态的链表节点指针
    void sendReqestToNextHandler(const Reqest & req)	//若下一个节点不空，将请求分派给下一个节点处理
    {
        if (nextChain != nullptr)
            nextChain->handle(req);
    }
protected:
    virtual bool canHandleRequest(const Reqest & req) = 0;		//定义的虚函数，运行时判断请求能否处理
    virtual void processRequest(const Reqest & req) = 0;		//定义的虚函数，具体的请求处理业务逻辑
public:
    ChainHandler() { nextChain = nullptr; }
    void setNextChain(ChainHandler *next) { nextChain = next; }	//设定下一个节点
    
   
    void handle(const Reqest & req)			//整体处理逻辑，如果当前对象能处理，就处理；不能处理，将请求分给下个节点
    {
        if (canHandleRequest(req))
            processRequest(req);
        else
            sendReqestToNextHandler(req);
    }
};
```

- 基类基本定义了主要逻辑，构造函数先将同类型的多态指针置为nullptr
- 在handle()中，得到一个请求，若能处理能自己处理，不行就转发给下一个节点处理
  - 其中，通过setNextChain(ChainHandler *next)设置下一个节点



- 分别设计具体的请求处理类，其都继承了抽象基类并实现了接口

```c++
class Handler1 : public ChainHandler{
protected:
    bool canHandleRequest(const Reqest & req) override
    {
        return req.getReqType() == RequestType::REQ_HANDLER1;
    }
    void processRequest(const Reqest & req) override
    {
        cout << "Handler1 is handle reqest: " << req.getDescription() << endl;
    }
};
        
class Handler2 : public ChainHandler{
protected:
    bool canHandleRequest(const Reqest & req) override
    {
        return req.getReqType() == RequestType::REQ_HANDLER2;
    }
    void processRequest(const Reqest & req) override
    {
        cout << "Handler2 is handle reqest: " << req.getDescription() << endl;
    }
};

class Handler3 : public ChainHandler{
protected:
    bool canHandleRequest(const Reqest & req) override
    {
        return req.getReqType() == RequestType::REQ_HANDLER3;
    }
    void processRequest(const Reqest & req) override
    {
        cout << "Handler3 is handle reqest: " << req.getDescription() << endl;
    }
};

```



- 具体调用示例如下：

```c++
int main(){
    Handler1 h1;
    Handler2 h2;
    Handler3 h3;
    h1.setNextChain(&h2);
    h2.setNextChain(&h3);
    
    Reqest req("process task ... ", RequestType::REQ_HANDLER3);
    h1.handle(req);
    return 0;
}
```

- 创建了三个具体处理类，其实也可把setNextChain的逻辑放在构造函数中实现
- 因为nextChain是虚基类类型的指针，所以相当于父类指针指向具体的子类对象，因为子类is-a父类
- 形成了h1->h2->h3的链式结构
- 创建了一个Request对象，并给了REQ_HANDLER3类型的RequestType
- 通过Handler1对象h1调用handle(req)，从链表最开始的节点开始一直传递直到能处理为止



### 3、总结

#### 1、结构

![image](https://user-images.githubusercontent.com/106053649/176989976-e4f98644-243e-4990-b1a3-7069660d534c.png)

- 红色稳定，蓝色变化，可通过扩展子类来增加新的处理方式

#### 2、总结

- Chain of Responsibility模式的应用场合在于“一个请求可能有多个接受者，但是最后真正的接受者只有一个”
- 这时候请求发送者与接受者的耦合有可能出现“变化脆弱”的症状，职责链的目的就是将二者解耦，从而更好地应对变化
- 应用了Chain of Responsibility模式后，对象的指责分派将更具灵活性。可以在运行时多态添加/修改请求的处理职责
- 若请求传递到职责链的默契为仍得不到处理，应该有一个合理的缺省机制
