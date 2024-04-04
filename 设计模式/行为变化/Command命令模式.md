## Command命令模式

### 一、简介

#### 1、动机

- 在软件构建过程中，“行为请求者”与“行为实现者”通常呈现一种“紧耦合”
- 但在某些场合，这种无法抵御变化的紧耦合是不合适的
  - 比如需要对行为进行“记录、撤销/重做（undo/redo）、事务”等处理

- 这种情况下，如何将“行为请求者”与“行为实现者”解耦
  - 将一组行为抽象为对象，可以实现二者之间的松耦合

#### 2、定义

- 将一个请求（行为）封装为一个对象，从而使可用不同的请求对客户进行参数化
- 对请求排队或记录请求日志，以及支持可撤销的操作



### 二、示例

#### 1、示例代码

```c++
#include <iostream>
#include <vector>
#include <string>
using namespace std;


class Command
{
public:
    virtual void execute() = 0;
};

class ConcreteCommand1 : public Command
{
    string arg;
public:
    ConcreteCommand1(const string & a) : arg(a) {}
    void execute() override
    {
        cout<< "#1 process..."<<arg<<endl;
    }
};

class ConcreteCommand2 : public Command
{
    string arg;
public:
    ConcreteCommand2(const string & a) : arg(a) {}
    void execute() override
    {
        cout<< "#2 process..."<<arg<<endl;
    }
};
           
class MacroCommand : public Command
{
    vector<Command*> commands;
public:
    void addCommand(Command *c) { commands.push_back(c); }
    void execute() override
    {
        for (auto &c : commands)
        {
            c->execute();
        }
    }
};
        
        
int main()
{

    ConcreteCommand1 command1(receiver, "Arg ###");
    ConcreteCommand2 command2(receiver, "Arg $$$");
    
    MacroCommand macro;
    macro.addCommand(&command1);
    macro.addCommand(&command2);
    
    macro.execute();

}
```

- 所有继承Command的具体类的对象，表达的就是一种行为对象
  - 其中string arg表示命令的一些参数，可以提供给实现的虚函数execute()
  - 虽然是对象，但只有一个execute()，表征的是行为
  - 对象较灵活
    - 对象可以做参数传递
    - 可以作为字段存储
    - 可以序列化
    - 可以保存在数据结构中

- MacroCommand类的设计类似Composite组合模式，将多个命令组合其中：
  - 组合命令本身又继承自命令对象
  - ConcreteCommand1和ConcreteCommand2相当于叶子节点类
  - 通过遍历集合内容器的方式多态递归调用execute()

- main函数中，有以下几个命令
  - ConcreteCommand1类型的command1
  - ConcreteCommand2类型的command2
  - MacroCommand的macro对象通过addCommand()方法构成了命令组合

- 通过macro去执行命令



### 三、总结

- Command模式的根本目的在于将“行为请求者”与“行为实现者”解耦
  - 将行为抽象为对象
- 实现Command接口的具体命令对象ConcreteCommand有时候根据需要可能会保存一些额外的状态信息
  - 示例中的string arg
- 通过使用Composite模式，可以将多个“命令”封装为一个“复合命令”MacroCommand
- Command模式类似C++中的函数对象（Functor仿函数），但两者定义行为接口的规范有所区别
  - Command以面向对象中的“接口-实现”来定义行为接口规范，更严格（函数名、参数、返回值与接口虚函数保持一致）
    - 虚函数实现的运行时绑定，有性能损失
  - C++函数对象以函数签名来定义行为接口规范，更灵活（只需要参数，返回值一致即可，名字无所谓）
    - 利用模板实现编译时绑定，性能更高