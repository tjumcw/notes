## Factory Method

### 一、简介

#### 1、类属

- 属于对象创建模式，绕开new，避免对象创建（new）过程中所导致的紧耦合（依赖具体类）。

#### 2、动机

- 软件系统中，经常面临创建对象的工作；由于需求的变化，需要创建的对象的具体类型经常变化。

#### 3、定义

- 定义一个用于创建对象的接口，让子类决定实例化哪一个类。Factory Method使得一个类的实例化延迟到子类
  - 目的：解耦
  - 手段：虚函数



### 二、示例

#### 1、背景

- 以观察者模式的文件分割器作为例子，需要实现对不同文件的分割，如二进制文件、文本文件等等
- 主要矛盾是MainForm中创建对象，针对不同的需求，创建对象的类型会发生变化



#### 2、基础代码

- 首先考虑需要实现多种文件类型的分割器，很容易想到定义一个接口虚基类，实现如下（仅展示骨架代码，没考虑内存管理）：
  - 定义多个继承虚基类的不同分割器子类
  - 其中每个不同子类需要重写split()的虚函数

```c++
class ISplitter{
public:
    virtual void split()=0;
    virtual ~ISplitter(){}
};

class BinarySplitter : public ISplitter{
    
};

class TxtSplitter: public ISplitter{
    
};

class PictureSplitter: public ISplitter{
    
};

class VideoSplitter: public ISplitter{
    
};
```



- 针对上述的分割器定义，可以修改MainForm类创建对象的代码如下

```c++
class MainForm : public Form
{
public:
	void Button1_Click(){
        
		ISplitter * splitter=
            new BinarySplitter();//依赖具体类
        
        splitter->split();

	}
};
```

- 首先将Button1_Click()中创建对象的代码修改，根据依赖倒置原则，发现
  - 等号左边可以修改为依赖抽象基类ISplitter
  - 但等号右边依旧依赖具体类，仍旧违背依赖于抽象不依赖于细节的原则



#### 3、优化和重构

- 思考C++创建对象的几种方法，并根据第三种方法进行尝试
  - BinarySplitter bs();
  - BinarySplitter* bs = new BinarySplitter();
  - 通过函数返回值得到对象

```c++
class SplitterFactory(){
public:
    ISplitter* createSplitter(){
        return new BinarySplitter();
    }
};

class MainForm : public Form
{
public:
	void Button1_Click(){
        
        SplitterFactory factory;
		ISplitter * splitter=
            factory.createSplitter();//依赖具体类
        
        splitter->split();

	}
};
```

- 看起来MainForm类挺合理，但实际上它依赖factory，但factory的创建依赖BinarySplitter对象

- 这种间接的依赖关系，本质上创建对象还是编译时依赖



- 思考如何将编译时依赖变为运行时依赖  -> 虚函数
  - 通过虚函数可以延迟createSplitter()的具体实现，将其延迟到运行时
  - 需要对SplitterFactory创建具体子类

```c++
class SplitterFactory(){
public:
    virtual ISplitter* createSplitter() = 0;
    virtual ~SplitterFactory()
};

class MainForm : public Form
{
public:
	void Button1_Click(){
        
        SplitterFactory* factory;
		ISplitter * splitter=
            factory->createSplitter();//依赖具体类
        
        splitter->split();

	}
};

//具体工厂
class BinarySplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new BinarySplitter();
    }
};

class TxtSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new TxtSplitter();
    }
};

class PictureSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new PictureSplitter();
    }
};

class VideoSplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new VideoSplitter();
    }
};
```

- 修改为上述代码后，依赖关系理顺了，仔细看Button1_Click()方法，调用时
  - 创建一个工厂虚基类SplitterFactory的指针factory
  - ISplitter的父类指针指向由factory调用createSplitter()创建的某个具体对象
  - 现在factory还是一个没有指向具体类型的父类指针，所以需要小做修改



```c++
class MainForm : public Form
{
    SplitterFactory* factory;
public:
    MainForm(SplitterFactory* factory){
        this->factory = factory;
    }
	void Button1_Click(){
        
		ISplitter * splitter=
            factory->createSplitter();//依赖具体类
        
        splitter->split();

	}
};
```

- 这样，未来外界传入一个具体的分割器对象工厂，如BinarySplitterFactory，通过Button1_Click()可以反复创建BinarySplitter对象
- 按上述代码修改后，MainForm不依赖于具体类，而把所有变化放到其他局部的地方



### 三、总结

- Factory Method模式用于隔离类对象的使用者和具体类型之间的耦合关系。
  - 面对一个经常变化的具体类型，紧耦合关系（new）会导致软件的脆弱
- Factory Method模式通过面向对象的手法，将所要创建的具体对象工作延迟到子类。
  - 实现一种扩展（而非更改）的策略，较好地解决了这种紧耦合关系
    - 通过增加其他具体的实现类（具体Splitter）及其对应工厂即可
    - 需要什么工厂将其作为参数传参给MainForm构造函数
- Factory Method模式解决“单个对象”的需求变化。缺点在于要求创建方法/参数相同。