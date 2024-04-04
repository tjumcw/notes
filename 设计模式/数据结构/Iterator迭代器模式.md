## Iterator迭代器模式

### 一、简介

#### 1、动机

- 在软件构建过程中，集合对象内部结构常常变化各异（vector，list，map等等）
- 但对于这些集合对象，我们希望在不暴露其内部结构的同时，可以让外部客户代码透明地访问其中包含的元素
- 同时这种“透明遍历”也为“同一种算法在多种集合对象上进行操作”提供了可能
  - 迭代器隔离容器与算法，同时所有容器通过迭代器取元素，封装了其内部结构

- 使用面向对象技术将这种遍历机制抽象为“迭代器对象”为“应对变化中的集合对象”提供了一种方式
  - 现在C++的迭代器是利用模板和泛型编程的思想实现的

#### 2、定义

- 提供一种方法顺序访问一个聚合对象（集合）中的各个元素，而又不暴露（稳定）该对象的内部表示



### 二、示例

```c++
//迭代器抽象类
template<typename T>
class Iterator
{
public:
    virtual void first() = 0;
    virtual void next() = 0;
    virtual bool isDone() const = 0;
    virtual T& current() = 0;
};


//定义的元素集合，即容器
template<typename T>
class MyCollection{
    
public:
    
    Iterator<T> GetIterator(){
        //...
    }
    
};

//特定的集合迭代器类，实现了迭代器抽象类的接口
template<typename T>
class CollectionIterator : public Iterator<T>{
    MyCollection<T> mc;
public:
    
    CollectionIterator(const MyCollection<T> & c): mc(c){ }
    
    void first() override {
        
    }
    void next() override {
        
    }
    bool isDone() const override{
        
    }
    T& current() override{
        
    }
};

void MyAlgorithm()
{
    MyCollection<int> mc;
    
    Iterator<int> iter= mc.GetIterator();	//返回的是一个Iterator<T>类型
    
    for (iter.first(); !iter.isDone(); iter.next()){
        cout << iter.current() << endl;
    }
    
}

```

- 感觉上面GetIterator()应该返回Iterator*<T>类型，不然没法多态



- 按我理解修改为

```c++
template<typename T>
class MyCollection{
    
public:
    
    Iterator*<T> GetIterator(){
        //...
    }
    
};

void MyAlgorithm()
{
    MyCollection<int> mc;
    
    Iterator*<int> iter= mc.GetIterator();	//返回的是一个Iterator*<T>类型
    
    for (iter->first(); !iter->isDone(); iter->next()){
        cout << iter->current() << endl;
    }
    
}
```

- 这样才能让抽象类的父类指针iter指向具体子类的对象，实现多态
  - 以支持各种不同容器的迭代器，示例代码只给了一种
- 实际上模板是一种编译时多态，效率高于运行时多态



### 三、总结

- 迭代抽象：访问一个聚合对象的内容而不暴露它的内部表示

- 迭代多态：为遍历不同的几何结构提供一个同一的接口，从而支持同样的算法在不同的集合结构上进行操作

- 迭代器的健壮性考虑：遍历的同时更改迭代器所在的集合结构，会导致问题

  - ```c++
    map<int, int> m;
    //...对m进行插值
    for(auto a : m){
        if(a.first == 1){
            m.erase(a.first);
        }
    }
    ```

  - 这样若在遍历过程中发现有key为1的元素，调用erase后map结构发生变化，会出错（自己试的会直接退出Loop）