## Prototype原型模式

### 一、简介

#### 1、动机

- 在软件系统中，经常面临着“某些结构复杂的对象”的创建工作
- 由于需求的变化，这些对象经常面临着剧烈的变化，但是它们却拥有比较稳定一致的接口
- 向“客户程序（使用这些对象的程序）”隔离出“这些易变对象”
- 使得“依赖这些易变对象的客户程序”不随着需求改变而改变

#### 2、定义

- 使用原型实例指定创建对象的种类，然后通过拷贝（深拷贝）这些原型创建新的对象

#### 3、理解

- 与工厂方法最大的区别在于 -> 某些结构复杂的对象
  - 工厂方法通过多态new一个简单对象即可
  - 当对象比较复杂时，通过拷贝构造的clone()可以把对象的当前状态也取出来

- 何时原型何时工厂方法
  - 用工厂方法创建对象是不是简单的几步可以创建出来
  - 若是需要考虑对象很复杂的中间状态，并保留中间状态，用原型



### 二、示例

#### 1、背景

- 以之前的文件分割器为例子（依旧只给出骨架，没考虑规范和内存管理）

#### 

#### 2、基础代码

```c++
//抽象类
class ISplitter{
public:
    virtual void split()=0;
    virtual ~ISplitter(){}
};


//工厂基类
class SplitterFactory{
public:
    virtual ISplitter* CreateSplitter()=0;
    virtual ~SplitterFactory(){}
};



//具体类
class BinarySplitter : public ISplitter{
    
};

class TxtSplitter: public ISplitter{
    
};

class PictureSplitter: public ISplitter{
    
};

class VideoSplitter: public ISplitter{
    
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

class MainForm : public Form
{
    SplitterFactory*  factory;//工厂

public:
    
    MainForm(SplitterFactory*  factory){
        this->factory=factory;
    }
    
	void Button1_Click(){

        
		ISplitter * splitter=
            factory->CreateSplitter(); //多态new
        
        splitter->split();

	}
};

```



#### 3、原型模式的重构

- 将两个抽象类ISplitter及SplitterFactory合并，并修改CreateSplitter(),演化为：

```c++
class ISplitter{
public:
    virtual void split()=0;
    virtual ISplitter* clone()=0; //通过克隆自己创建对象
    virtual ~ISplitter(){}

};
```



- 在合并抽象类及抽象的工厂类的基础上，同理对具体类和具体工厂类也进行合并，如下
  - 实际上每个具体类需要重写split()，知道就行了省略了

```c++
//---------------------------合并前-----------------------
//具体类
class BinarySplitter : public ISplitter{

};

//具体工厂
class BinarySplitterFactory: public SplitterFactory{
public:
    virtual ISplitter* CreateSplitter(){
        return new BinarySplitter();
    }
};

//---------------------------合并后-----------------------
class BinarySplitter : public ISplitter{
    virtual ISplitter* clone(){
        return new BinarySplitter(*this);
    }
};
```

- 通过拷贝构造函数深拷贝自己(需要实现正确的拷贝构造函数)



- 将其他具体Splitter也如上进行合并，最后只留下如下类（不包括MainForm）
  - 注意，每个具体的类如BinarySplitter还要重写关键的业务方法split()，这里省略了

```c++
class ISplitter{
public:
    virtual void split() = 0;
    virtual ISplitter* clone() = 0; //通过克隆自己创建对象
    virtual ~ISplitter(){}

};

class BinarySplitter : public ISplitter{
    virtual ISplitter* clone(){
        return new BinarySplitter(*this);
    }
};

class TxtSplitter : public ISplitter{
    virtual ISplitter* clone(){
        return new TxtSplitter(*this);
    }
};

class PictureSplitter : public ISplitter{
    virtual ISplitter* clone(){
        return new PictureSplitter(*this);
    }
};

class VideoSplitterFactory : public ISplitter{
    virtual ISplitter* clone(){
        return new VideoSplitterFactory(*this);
    }
};
```



- 按照上述修改，需要对MainForm类进行修改

```c++
class MainForm : public Form
{
    ISplitter*  prototype;//原型对象，用于clone，不是拿来使用的

public:
    
    MainForm(ISplitter*  prototype){
        this->prototype=prototype;
    }
    
	void Button1_Click(){
    
		ISplitter * splitter=
            prototype->clone(); //通过克隆原型得到了一个新对象
        
        splitter->split();
	}
};

```

- 实质上就是合并了工厂基类和业务基类，同理合并了工厂具体类和业务具体类



### 三、总结

- Prototype模式同样用于隔离类对象的使用者和具体类型（易变类）之间的耦合关系，它同样要求这些“易变类”拥有“稳定的接口”。
- Prototype模式对于“如何创建易变类的实体对象”采用“原型克隆”的方法。
  - 可以非常灵活地动态创建“拥有某些稳定接口”的新对象
  - 所需工作仅仅是注册一个新类的对象（即原型），然后再任何需要的地方clone
- 有时会结合某些框架中的序列化来实现深拷贝（C++用拷贝构造就可以）