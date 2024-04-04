## Memento备忘录模式

## 一、简介

#### 1、动机

- 在软件构建过程中，某些对象的状态在转换过程中，可能由于某种需要，要求程序能够回溯到对象之前处于某个点时的状态
  - 对象级别的REDO和UNDO
- 如果使用一些公有接口让其他对象得到对象的状态，便会暴露对象的细节实现
- 如何实现对象状态的良好保存与恢复，但同时不会因此破坏对象本身的封装性

#### 2、定义

- 在不破坏封装性的前提下，捕获一个对象的内部状态，并在该对象之外保存这个状态
- 这样以后就可以将该对象恢复到原先保存的状态
  - 类似快照的思想

![image](https://user-images.githubusercontent.com/106053649/176901071-677dff6a-b8fb-4c9f-afd1-a3e4aa21a478.png)

- CareTaker即使用备忘录的人，在下方示例代码中即为main函数调用



### 二、示例

```c++
class Memento
{
    string state;
    //..
public:
    Memento(const string & s) : state(s) {}
    string getState() const { return state; }
    void setState(const string & s) { state = s; }
};



class Originator
{
    string state;
    //....
public:
    Originator() {}
    Memento createMomento() {
        Memento m(state);
        return m;
    }
    void setMomento(const Memento & m) {
        state = m.getState();
    }
};



int main()
{
    Originator orginator;
    
    //捕获对象状态，存储到备忘录
    Memento mem = orginator.createMomento();
    
    //... 改变orginator状态
    
    //从备忘录中恢复
    orginator.setMomento(memento);
    
}
```

- 其中，Originator为需要保存状态的对象
- 从main函数具体来看
  - 首先通过调用createMomento()将现有状态塞入一个Memento对象，即将当前对象内存状态进行快照
  - orginator状态随着程序的运行，状态不断地改变
  - 若在某一时刻需要状态回滚，orginator对象调用setMomento(memento)从备忘录中恢复状态

- 这种方式没有破坏orginator类的封装性，仍旧保持了信息隐藏
- 在该对象之外保存了这个状态



### 三、总结

- 备忘录（Memento）存储原发器（Originator）对象的内部状态，在需要时恢复原发器状态
- Memento模式的核心是信息隐藏，即Originator需要向外界隐藏细节，保持其封装性，但同时又需要将状态保持到外界（Memento）

- 现代语言运行时都具有相当的对象序列化支持，因此往往采取序列化方案实现Memento模式
