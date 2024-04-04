## Singleton单例模式

### 一、简介

#### 1、动机

- 在软件系统中，经常有这样一些特殊的类，必须确保它们在系统中只存在一个实例，才能确保它们的逻辑正确性、以及良好的效率。
- 绕过常规的构造器，提供一种机制来保证一个类只有一个实例。

#### 2、定义

- 保证一个类仅有一个实例，并提供一个该实例的全局访问点。



### 二、实现

#### 1、基础代码

- 让构造函数和拷贝构造函数私有化，只有内部调用 
- 声明一个静态的类对象，全局初始化为nullptr
- 声明一个静态的成员函数用来生成这个单例

```c++
class Singleton{
private:
    Singleton();
    Singleton(const Singleton& other);
public:
    static Singleton* getInstance();
    static Singleton* m_instance;
};

Singleton* Singleton::m_instance=nullptr;

//线程非安全版本
Singleton* Singleton::getInstance() {
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}

```



#### 2、重构和优化

- 考虑到多线程背景，可能多个线程同时判断完进入if内部逻辑，这样不能保证单一
- 先考虑普通解决，加锁实现：

```c++
//线程安全版本，但锁的代价过高
Singleton* Singleton::getInstance() {
    Lock lock;
    if (m_instance == nullptr) {
        m_instance = new Singleton();
    }
    return m_instance;
}
```

- 当m_instance已经new完不是nullptr后，调用getInstance()只是读操作，后续也不存在写操作
  - 每次都需要持有锁再判断，其他线程需要等待，考虑到高并发环境下代价比较高



- 继续修改代码，实现双检查锁如下：

```c++
//双检查锁，但由于内存读写reorder不安全
Singleton* Singleton::getInstance() {
    
    if(m_instance==nullptr){
        Lock lock;
        if (m_instance == nullptr) {
            m_instance = new Singleton();
        }
    }
    return m_instance;
}
```

- 其中，第一个判断解决一旦有实例后，不再去持有锁
- 第二个判断用来保证单例
  - 可能在m_instance==nullptr的时候多个线程已经进入if内部代码
  - 一个线程持有锁，其他线程等待
  - 若没有第二个判断，这样依旧会创建多个实例



- 但由于内存读写reorder，仍旧不安全（即编译器对代码的优化，更改指令执行顺序）
  - new的过程可能分为三个阶段，如分配内存，再调构造器，将内存地址赋给m_instance
  - reorder后可能先分配内存，将内存地址赋给m_instance，最后再调构造器
  - 这样当一个线程拿到锁并用new创建对象时，
    - 如果先分配内存，将内存地址赋给m_instance，最后才调构造器
    - 其他线程在第一个判断时，若内存地址赋给m_instance后，还没调构造器
    - 此时m_instance不为nullptr，此时直接return，若该线程得到该对象，不对
    - 因为对象只分配了内存，没执行构造器，状态是不对的

- volatile关键字修饰变量，则不允许编译器对该变量的编译优化
- C++11之后的跨平台实现方式（有些平台没有volatile），双检查锁的改进

```c++
//C++ 11版本之后的跨平台实现 (volatile)
std::atomic<Singleton*> Singleton::m_instance;
std::mutex Singleton::m_mutex;

Singleton* Singleton::getInstance() {
    Singleton* tmp = m_instance.load(std::memory_order_relaxed);
    std::atomic_thread_fence(std::memory_order_acquire);//获取内存fence
    if (tmp == nullptr) {
        std::lock_guard<std::mutex> lock(m_mutex);
        tmp = m_instance.load(std::memory_order_relaxed);
        if (tmp == nullptr) {
            tmp = new Singleton;
            std::atomic_thread_fence(std::memory_order_release);//释放内存fence
            m_instance.store(tmp, std::memory_order_relaxed);
        }
    }
    return tmp;
}
```



### 三、总结

- Singleton模式中的实例构造器可以设置为protected以允许子类派生
- Singleton模式一般不要支持拷贝构造函数和clone接口，有可能导致多个对象实例
- 要考虑多线程环境下安全的Singleton