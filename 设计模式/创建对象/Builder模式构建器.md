## Builder模式构建器

## 一、简介

#### 1、动机

- 在软件系统中，有时候面临着“一个复杂对象”的创建工作，其通常由各个部分的子对象用一定的算法构成
- 由于需求的变化，这个复杂对象的各个部分经常面临着剧烈的变化，但是将它们组合在一起的算法却相对稳定

#### 2、定义

- 将一个复杂对象的构建与其表示相分离，使得同样的构建过程（稳定）可以创建不同的表示（变化）。



### 二、示例

#### 1、背景

- 游戏里建房子，包括茅草屋，砖瓦房、豪华房子等
- 建房子有几个固定流程，地基、窗户、天花板、门等等
- 但不同房子相同流程的每个部分实现细节不一样



#### 2、基础代码

```c++
class House{
public:
    void Init(){
        
        this->BuildPart1();
        
        for(int i = 0; i < 4; i++){
            this->BuildPart2();
        }
        
        bool flag = this->BuildPart3();
        
        if(flag){
            this->BuildPart4();
        }
        
        this->BuildPart5();
    }
    
    virtual ~House(){}
protected:
    virtual void BuildPart1() = 0;
    virtual void BuildPart2() = 0;
    virtual void BuildPart3() = 0;
    virtual void BuildPart4() = 0;
    virtual void BuildPart5() = 0;
};
```

- 注意，不能将流程放在构造函数中，即

```c++
	House(){
        
        this->BuildPart1();
        
        for(int i = 0; i < 4; i++){
            this->BuildPart2();
        }
        
        bool flag = this->BuildPart3();
        
        if(flag){
            this->BuildPart4();
        }
        
        this->BuildPart5();
    }
```

- 因为在C++的构造函数中，若调用虚函数则为静态绑定，不会去调用子类
  - 因为子类构造函数会调用父类的构造函数
  - 若允许实现动态绑定，则子类调用House构造函数，在构造函数中调用子类虚函数的override版本时
  - 导致子类都还没完成构造函数，就要调用子类的虚函数，不合理



- 在上述基类的基础上，实现一些具体类并重写虚函数，如下：

```c++
class StoneHouse : public House{
protected:
    virtual void BuildPart1(){
        
    }
    virtual void BuildPart2(){
        
    }
    virtual void BuildPart3(){
        
    }
    virtual void BuildPart4(){
        
    }
    virtual void BuildPart5(){
        
    }
}

int main(){
    House* pHouse = new StoneHouse();
    pHouse->Init();
}
```



#### 3、重构和优化

- 实际上已经完成了Builder模式整个流程，但若对象过于复杂，有很多与构建无关的其他方法与实现
- 将与构建流程相关的字段拆分出去，分成两个抽象基类，

```c++
class House{
  	//...还有很多其他与构造无关的其他功能及实现  
};

class HouseBuilder {
public:
    void Init(){

        this->BuildPart1();

        for (int i = 0; i < 4; i++){
            this->BuildPart2();
        }

        bool flag=this->BuildPart3();

        if(flag){
            this->BuildPart4();
        }

        this->BuildPart5();
    }    
    House* GetResult(){
        return pHouse;
    }
    virtual ~HouseBuilder(){}
protected:
    
    House* pHouse;
	virtual void BuildPart1()=0;
    virtual void BuildPart2()=0;
    virtual void BuildPart3()=0;
    virtual void BuildPart4()=0;
    virtual void BuildPart5()=0;
	
};

```

- 其中，HouseBuilder专门管理构建，其他无关的将其放到House中并分离



- 同时，不同的具体House类型继承抽象基类House，重写虚函数
- 不同的具体HouseBuilder继承抽象基类HouseBuilder，重写虚函数

```c++
class StoneHouse: public House{
    
};

class StoneHouseBuilder: public HouseBuilder{
protected:
    
    virtual void BuildPart1(){
        //pHouse->Part1 = ...;
    }
    virtual void BuildPart2(){
        
    }
    virtual void BuildPart3(){
        
    }
    virtual void BuildPart4(){
        
    }
    virtual void BuildPart5(){
        
    }
    
};
```





- 审视代码，继续考虑将HouseBuilder中稳定的部分拆分出去，形成一个稳定的构建流程类

```c++
class HouseDirector{
    
public:
    HouseBuilder* pHouseBuilder;
    
    HouseDirector(HouseBuilder* pHouseBuilder){
        this->pHouseBuilder=pHouseBuilder;
    }
    
    House* Construct(){
        
        pHouseBuilder->BuildPart1();
        
        for (int i = 0; i < 4; i++){
            pHouseBuilder->BuildPart2();
        }
        
        bool flag=pHouseBuilder->BuildPart3();
        
        if(flag){
            pHouseBuilder->BuildPart4();
        }
        
        pHouseBuilder->BuildPart5();
        
        return pHouseBuilder->GetResult();
    }
};

class StoneHouseBuilder: public HouseBuilder{
protected:
    
    virtual void BuildPart1(){
        //pHouse->Part1 = ...;
    }
    virtual void BuildPart2(){
        
    }
    virtual void BuildPart3(){
        
    }
    virtual void BuildPart4(){
        
    }
    virtual void BuildPart5(){
        
    }
    
};
```

- 最终流程如下：
  - 通过HouseDirector的构造函数，传某种具体的HouseBuilder
  - HouseBuilder类型的父类指针指向该具体的Builder，在Construct()函数中调用该具体Builder重写的虚函数
  - 执行流程，流程中会对HouseBuilder类中的House类型指针进行操作并更改状态
  - 流程结束后，调HouseBuilder的GetResult()返回构建完的具体House





### 三、总结

- Builder模式主要用于“分步骤构建一个复杂的对象”。
  - 在这其中“分步骤”是一个稳定的算法
  - 复杂对象的各个部分则经常变化
- 变化点在哪里，封装哪里。Builder模式主要在于应对“复杂对象各个部分”的频繁需求变动
  - 缺点在于难以应对“分步骤构建算法”的需求变动（即示例的构建流程）