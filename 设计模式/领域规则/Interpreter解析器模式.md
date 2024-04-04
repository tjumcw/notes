## Interpreter解析器模式

### 一、简介

#### 1、动机

- 在软件构建过程中，如果某一特定领域的问题比较复杂，类似的结构不断重复出现
  - 如果使用普通的编程方式来实现将面临非常频繁的变化
- 可将特定领域的问题表达为某种语法规则下的句子，然后构建一个解释器来解释这样的句子

#### 2、定义

- 给定一个语言，定义它的文法的一种表示，并定义一种解释器，这个解释其使用该表示来解释语言中的句子



### 二、示例

#### 1、背景

- 实现对字符串型的加减运算表达式的计算

#### 2、代码

```c++
#include <iostream>
#include <map>
#include <stack>

using namespace std;

class Expression {
public:
    virtual int interpreter(map<char, int> var)=0;
    virtual ~Expression(){}
};

//变量表达式
class VarExpression: public Expression {
    
    char key;
    
public:
    VarExpression(const char& key)
    {
        this->key = key;
    }
    
    int interpreter(map<char, int> var) override {
        return var[key];
    }
    
};

//符号表达式
class SymbolExpression : public Expression {
    
    // 运算符左右两个参数
protected:
    Expression* left;
    Expression* right;
    
public:
    SymbolExpression( Expression* left,  Expression* right):
        left(left),right(right){
        
    }
    
};

//加法运算
class AddExpression : public SymbolExpression {
    
public:
    AddExpression(Expression* left, Expression* right):
        SymbolExpression(left,right){
        
    }
    int interpreter(map<char, int> var) override {
        return left->interpreter(var) + right->interpreter(var);
    }
    
};

//减法运算
class SubExpression : public SymbolExpression {
    
public:
    SubExpression(Expression* left, Expression* right):
        SymbolExpression(left,right){
        
    }
    int interpreter(map<char, int> var) override {
        return left->interpreter(var) - right->interpreter(var);
    }
    
};



Expression*  analyse(string expStr) {
    
    stack<Expression*> expStack;
    Expression* left = nullptr;
    Expression* right = nullptr;
    for(int i=0; i<expStr.size(); i++)
    {
        switch(expStr[i])
        {
            case '+':
                // 加法运算
                left = expStack.top();
                right = new VarExpression(expStr[++i]);
                expStack.push(new AddExpression(left, right));
                break;
            case '-':
                // 减法运算
                left = expStack.top();
                right = new VarExpression(expStr[++i]);
                expStack.push(new SubExpression(left, right));
                break;
            default:
                // 变量表达式
                expStack.push(new VarExpression(expStr[i]));
        }
    }
   
    Expression* expression = expStack.top();

    return expression;
}

void release(Expression* expression){
    
    //释放表达式树的节点内存...
}

int main(int argc, const char * argv[]) {
    
    
    string expStr = "a+b-c+d-e";
    map<char, int> var;
    var.insert(make_pair('a',5));
    var.insert(make_pair('b',2));
    var.insert(make_pair('c',1));
    var.insert(make_pair('d',6));
    var.insert(make_pair('e',10));

    
    Expression* expression= analyse(expStr);
    
    int result=expression->interpreter(var);
    
    cout<<result<<endl;
    
    release(expression);
    
    return 0;
}
```

- 首先实现了接口基类Expression，里面定义了一个纯虚函数

- ```c++
  virtual int interpreter(map<char, int> var) = 0;
  ```

- 其次分别实现了变量表达式类VarExpression和符号表达式SymbolExpression，其中

  - 变量表达式即表示数值型，即叶子节点

  - 符号表达式有左右两个节点，均为表达式（可以是变量表达式也可以是符号表达式）

    - ```c++
      Expression* left;
      Expression* right;
      ```

    - 基类指针实现上述的多态

- 符号表达式按其符号分类又分为加法和减法，分别继承符号表达式基类SymbolExpression实现了AddExpression和SubExpression

  - 各自重写了最根的基类的虚函数interpreter(map<char, int> var)

- 具体实现流程结合类的实现以及analyse()函数即可理解，主要利用多态递归调用
- 具体怎么实现与解析器模式无关，主要在于体现模式的思想



### 三、总结

- Interpreter模式的应用场合是Interpreter模式应用中的难点
- 只有满足“业务规则频繁变化，且类似的结构不断重复出现，并且容易抽象为语法规则的问题”才适合Interpreter模式

- 使用Interpreter模式来表示文法规则，从而可以使用面向对象技巧来方便地“扩展”文法
- Interpreter模式比较适合简单的文法表示，对于复杂的文法表示，Interpreter模式会产生比较大的类层次结构（如SQL语句）