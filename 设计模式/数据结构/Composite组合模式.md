## Composite组合模式

### 一、简介

#### 1、动机

- 软件在某些情况下，客户代码过多地依赖于对象容器的内部实现结构
- 对象容器内部实现结构（而非抽象接口）的变化将引起客户代码的频繁变化
- 带来了代码的维护性、扩展性等弊端

- 如何将”客户代码与复杂的对象容器结构“解耦，让对象容器自己来实现自身的复杂结构
  - 使得客户代码像处理简单对象一样来处理复杂的对象容器

#### 2、定义

- 将对象组合成树形结构以表示”部分-整体“的层次结构
- Composite使得用户对单个对象和组合对象的使用具有一致性（稳定）



### 二、示例

#### 1、示例代码

```c++
#include <iostream>
#include <list>
#include <string>
#include <algorithm>

using namespace std;

class Component
{
public:
    virtual void process() = 0;
    virtual ~Component(){}
};

//树节点
class Composite : public Component{
    
    string name;
    list<Component*> elements;		//多态对象容器
public:
    Composite(const string & s) : name(s) {}
    
    void add(Component* element) {
        elements.push_back(element);
    }
    void remove(Component* element){
        elements.remove(element);
    }
    
    void process(){
        
        //1. process current node
          
        //2. process leaf nodes
        for (auto &e : elements)
            e->process(); //多态递归调用，如果e是Composite，会继续调process()遍历自己的容器，如果是Leaf则直接调Leaf的process()         
    }
};

//叶子节点
class Leaf : public Component{
    string name;
public:
    Leaf(string s) : name(s) {}
            
    void process(){
        //process current node
    }
};

//客户程序，接受Component作为参数，抽象类的父类引用可接受两种具体子类
void Invoke(Component & c){
    //...
    c.process();
    //...
}


int main()
{

    Composite root("root");
    Composite treeNode1("treeNode1");
    Composite treeNode2("treeNode2");
    Composite treeNode3("treeNode3");
    Composite treeNode4("treeNode4");
    Leaf leaf1("leaf1");
    Leaf leaf2("leaf2");
    
    root.add(&treeNode1);
    treeNode1.add(&treeNode2);
    treeNode2.add(&leaf1);
    
    root.add(&treeNode3);
    treeNode3.add(&treeNode4);
    treeNode4.add(&leaf2);
    
    Invoke(root);
    Invoke(leaf2);
    Invoke(treeNode3);
  
}
```

- 其中，Composite和Leaf都继承自Component，表示is-a关系，即都可以加入容器list<Component*> elements;

- 根据main函数，构建了一棵树
  - root下有treeNode1
    - treeNode1下有treeNode2
      - treeNode2下有leaf1
  - root下还有treeNode3
    - treeNode3下有treeNode4
      - treeNode4下有leaf2

- 根据上面的设计，发现调用Invoke()的处理具有一致性，无论其是什么节点
- 若注释了

```c++
for (auto &e : elements)
            e->process(); //多态递归调用，如果e是Composite，会继续调process()遍历自己的容器，如果是Leaf则直接调Leaf的process()   
```

- 则需要在Invoke中分别处理
  - Composite树节点有自己的处理方式，处理完还要处理子节点，会把数据结构暴露给客户程序
  - Leaf叶子节点也有自己的处理方式

- 故这段代码将内部的数据结构封装进去了，且对外部程序提供了一致的访问方法（不论是单个对象还是组合对象）



### 三、总结

#### 1、结构

![image](https://user-images.githubusercontent.com/106053649/176916421-296eba28-d8e7-4150-b8fa-cc961bbaf619.png)

- 其中Add，Remove不论放在父类、子类都很尴尬
  - 如果放在子类，必须实现，不然也是抽象类。但Leaf节点实现上述两个方法没有意义
    - 实现成空，客户可以调用，但啥也没干相当于欺骗客户
    - 抛出异常也不合理，父类提供了接口，子类抛出异常，违背is-a关系
  - 如果按照示例代码，将其仅放在Composite中，也有点问题
    - 在调用Add方法时需要判断类型，因为只有Composite具有Add方法

- 但这都不是大问题，重点在于对于单个对象和组合对象，为客户提供了一致的访问方法

#### 2、总结

- Composite模式采用树形结构来实现普遍存在的对象容器，从而将”一对多“的关系转化为”一对一“的关系
  - Invoke函数内只需传一个抽象基类的指针（引用）即可，无需考虑到底是树节点还是叶子节点
- 使得客户代码可以一致地（复用）处理对象和对象容器，无需关心处理的是单个对象，还是组合的对象容器
- 将”客户代码与复杂的对象容器结构“解耦是Composite的核心思想
- 解耦之后，客户代码将与纯粹的抽象接口——而非对象容器的内部实现结构——发生依赖，从而更能”应对变化“
