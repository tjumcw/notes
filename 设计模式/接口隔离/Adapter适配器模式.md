## Adapter适配器模式

### 一、简介

#### 1、动机

- 在软件系统中，由于应用环境的变化，常常需要将“一些现存的对象“放在新的环境中应用
  - 但新环境要求的接口是这些现存对象所不满足的
- 如何应对这种”迁移的变化“
  - 既能利用现有对象的良好实现
  - 又能满足新的应用环境所要求的接口

#### 2、定义

- 将一个类的接口转换成客户希望的另一个接口
- 使得原本由于接口不兼容而不能一起工作的那些类可以一起工作



### 二、示例

#### 1、类图

![image](https://user-images.githubusercontent.com/106053649/176858930-0d6733fc-5b8a-44e6-af38-c46a03a0ed45.png)

- 其中，红色稳定，蓝色变化

- Adapter继承Target接口，并组合老的类Adaptee



#### 2、基础代码

```c++
//目标接口（新街口）
class ITarget{
public:
    virtual void process() = 0;
};

//遗留接口（老接口）
class IAdaptee{
public:
    virtual void foo(int data) = 0;
    virtual int bar() = 0;
};

class OldClass : public IAdaptee{
    virtual void foo(int data){
        //...
    }
    virtual int bar(){
        //...
    }
}
```



#### 3、以Adapter模式演化

- 由于某种内在实现层面的关联性，可以把Adaptee转成Target

```c++
class Apapter : public ITarget{		//继承接口表示遵循其定义的规范
protected:
    IAdaptee* pAdaptee;
public: 
    Apapter(IAdaptee* pAdaptee){
        this->pAdaptee = pAdaptee;
    }
    
    virtual void process(){
        int data = pAdaptee->bar();
        pAdaptee->foo(data);
    }
};
```

- 在process过程中展示了其示例转换过程，实际实现可能非常复杂



- 具体调用如下：

```c++
int main(){
    IAdaptee* pAdaptee = new OldClass();
    ITarget* pTarget = new Apapter(pAdaptee);
    pTarget->process()
}
```

- 实现了拿着一个旧的类，塞到Adapter里面，当作一个面向新接口的实现去使用



### 三、总结

- Adapter模式主要应用于“希望复用一些现存的类，但是接口又与服用环境要求不一致的情况”
- 在遗留代码复用、类库迁移等方面非常有用
