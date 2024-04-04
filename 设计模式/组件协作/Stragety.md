## Stragety

### 一、模式定义

- 定义一系列算法，把它们一个个封装起来，并且使它们可以互相替换（变化）。
- 该模式使得算法可独立于使用它的客户程序（稳定）而变化（扩展，子类化）



### 二、示例背景

- 在电子商务系统，计算订单的税率，假设该系统支持不同国家的税种，需要根据不同国家的订单来确定不同的税法计算规则（算法），从而计算税额

#### 1、结构化设计

```c++
enum TaxBase {
	CN_Tax,
	US_Tax,
	DE_Tax,
	FR_Tax       //新增的变化
};

class SalesOrder{
    TaxBase tax;
public:
    double CalculateTax(){
        //...
        
        if (tax == CN_Tax){
            //CN***********
        }
        else if (tax == US_Tax){
            //US***********
        }
        else if (tax == DE_Tax){
            //DE***********
        }
		else if (tax == FR_Tax){  //变化后额外增加的代码
			//...
		}

        //....
     }
    
};
```

- 一开始定义了三种税种的枚举，分别为中国、美国、德国的税种CN_Tax，US_Tax，DE_Tax
- CalculateTax为计算税额的方法，根据税种的不同进入不同的分支采用不同的计算方法计算税额
- 若产生了新的需求，即系统更新后支持了法国订单。
  - 则需要修改枚举类型，新增FR_Tax税种
  - 并在实现SalesOrder的类中的计算税额的方法中新增新的流程控制分支。

#### 2、Strategy

```c++
class TaxStrategy{
public:
    virtual double Calculate(const Context& context) = 0;
    virtual ~TaxStrategy(){}		//基类写成虚析构函数
};

//不同的子类分别重写计算税额的虚函数Calculate()
class CNTax : public TaxStrategy{
public:
    virtual double Calculate(const Context& context){
        //***********
    }
};

class USTax : public TaxStrategy{
public:
    virtual double Calculate(const Context& context){
        //***********
    }
};

class DETax : public TaxStrategy{
public:
    virtual double Calculate(const Context& context){
        //***********
    }
};



//扩展子类，而非修改源代码
//*********************************
class FRTax : public TaxStrategy{
public:
	virtual double Calculate(const Context& context){
		//.........
	}
};


class SalesOrder{
private:
    TaxStrategy* strategy;								//父类指针指向子类对象产生多态

public:
    SalesOrder(StrategyFactory* strategyFactory){		//通过工厂模式，根据需要生成特定的子类对象
        this->strategy = strategyFactory->NewStrategy();
    }
    ~SalesOrder(){
        delete this->strategy;							//生成的堆对象，需要在析构中释放
    }

    public double CalculateTax(){
        //...
        Context context();								//生成计算税额所需要的上下文参数
        
        double val = 
            strategy->Calculate(context); 				//多态调用，依赖strategyFactory返回的类型找到对应方法，计算税额
        //...
    }
    
};

```

- SalesOrder是稳定的，TaxStrategy也是稳定的，不同税种子类的算法是变化的
- 通过扩展的方式，增加子类，实现了稳定和变化的分离



### 三、模式理解

- Strategy及其子类为组件提供了一系列可重用的算法，使类型可以在运行时根据需要在各个算法之间进行切换
- Strategy提供了用条件判断语句以外的另一种选择，消除条件判断语句就是在解耦合。
  - if-else出现时，且可能会产生新的分支和变化，考虑Strategy模式，以扩展的方式支持未来变化
  - 除非分支是决定稳定不变的，绝大多数情况下需要考虑Strategy模式
  - if-else很多时，某种条件下大部分的代码一直不会被使用，比如该系统仅在中国使用，其他分支就没有意义
    - 但是代码都被装载到代码段中了，代码段在运行时会加载到内存里，其中
      - 最好的是加载到CPU的高级缓存里，执行最快
      - 若代码段过长，会放到主存里，甚至极端情况下，内存过下时会放到虚拟内存里（硬盘）
      - 很多if-else代码没有真正使用，但被迫加载到CPU的高级缓存里，其他代码就不能很好在缓存中执行
- 如果Strategy没有实例变量，那么各个上下文可以共享一个Strategy对象，从而节省开销（通过单例模式）