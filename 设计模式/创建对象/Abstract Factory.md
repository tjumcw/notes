## Abstract Factory

### 一、简介

#### 1、动机

- 在软件系统中，经常面临着“一系列相互依赖的对象”的创建工作（Connect、Command、DataReader）
- 同时，由于需求的变化，往往存在更多系列对象的创建工作（SQL Server、MySQL、Orcale）
- 需要提供一种“封装机制”避免客户程序和这种“多系列具体 对象创建工作”的紧耦合

#### 2、定义

- 提供一个接口，让该接口负责创建一系列“相关或者相互依赖的对象”，无需指定它们具体的类。



### 二、示例

#### 1、背景

- 实现一个数据访问层，在数据访问层需要创建一系列的对象，比如
  - SQLserver的数据库，需要针对其创建：
    - 数据库连接的对象SqlConnection
    - 数据库的命令对象SqlCommand
    - 数据库的数据读取对象SqlDataReader
  - 还有其他数据库，如MySQL，Orcale等等，每个都需要创建其相应的对象
  - 其中各个对象的创建存在依赖关系，如SqlDataReader的创建依赖SqlCommand对象的函数调用
    - 即对象创建有要求，所有的对象在同一个类型中必须是针对同一种数据库的对象，比如
      - 连接的对象是SqlServer，命令对象是MySQL，数据读取对象是Orcale，那类的功能就没法正确实现了
      - 要保证，在一个具体类中，传入的对象必须是具有依赖关系的针对同一种数据库的对象



#### 2、基础代码

```c++
class EmployeeDAO{ 
public:
    vector<EmployeeDO> GetEmployees(){
        SqlConnection* connection =
            new SqlConnection();
        connection->ConnectionString = "...";

        SqlCommand* command =
            new SqlCommand();
        command->CommandText="...";
        command->SetConnection(connection);

        SqlDataReader* reader = command->ExecuteReader();
        while (reader->Read()){

        }

    }
};
```

- 首先审视上方代码，在GetEmployees()中创建了一系列跟SqlServer相关的类，有
  - 连接类SqlConnection
  - 命令类SqlCommand
  - 数据读取类SqlDataReader
- 其中SqlDataReader的创建依赖于SqlCommand对象的ExecuteReader()函数调用
- 而且由实际业务需求也可知，GetEmployees()函数内的三种对象必须是针对同一数据库的，不然没有意义
  - 拿着SqlServer的连接对象去获得MySQL的命令对象，在调用函数产生Orcale的数据读取对象，没有任何实际意义



- 根据面向接口编程的原则，首先对可能用到的各个类设计抽象层
  - 考虑有3钟数据库，分别为SqlServer，MySQL，Orcale
  - 每种数据库都需要有对应的三种类：连接类，命令类及数据读取类
  - 设计虚基类接口

```c++
//数据库访问有关的接口基类
class IDBConnection{
    //一些virtual方法，不写出了
};

class IDBCommand{
    //一些virtual方法，不写出了
};

class IDBDataReader{
    //一些virtual方法，不写出了
};


//支持SQL Server的具体类型
class SqlConnection : public IDBConnection{
  	//重写虚函数  
};

class SqlCommand : public IDBCommand{
  	//重写虚函数  
};

class SqlDataReader : public IDBDataReader{
    //一些virtual方法，不写出了
};

//支持MySQL的具体类型
```



- 可以在上述接口类及具体类实现的基础上，根据面向接口编程的原则，将具体类的声明类型改为接口类

```c++
class EmployeeDAO{ 
public:
    vector<EmployeeDO> GetEmployees(){
        IDBConnection* connection =
            new SqlConnection();
        connection->ConnectionString = "...";

        IDBCommand* command =
            new SqlCommand();
        command->CommandText="...";
        command->SetConnection(connection);

        IDBDataReader* reader = command->ExecuteReader();
        while (reader->Read()){

        }

    }
};
```

- 继续审视上方代码，发现new带来的编译时依赖没有消除，等号左边依赖抽象，但等号右边依赖具体实现



- 根据上节课的工厂方法，创建一系列生产对象的抽象工厂

```c++
class IDBConnectionFactory{
    virtual IDBConnection* CreateDBConnection() = 0;
};

class IDBCommandFactory{
    virtual IDBCommand* CreateDBCommand() = 0;
};

class IDBDataReaderFactory{
    virtual IDBDataReader* CreateDBDataReader() = 0;
};
```



- 每个抽象工厂都要有其对应的具体工厂，以下仅展示SqlServer相关类设计，其余数据库完全类似

```c++
class SqlConnectionFactory : public IDBConnectionFactory{
  	  virtual IDBConnection* CreateDBConnection(){
          return new SqlConnection();
      }
};

class SqlCommandFactory : public IDBCommandFactory{
  	  virtual IDBCommand* CreateDBCommand(){
          return new SqlCommand();
      }
};

class SqlDataReaderFactory : public IDBDataReaderFactory{
  	  virtual IDBDataReader* CreateDBDataReader(){
          return new SqlDataReader();
      }
};

//...省略其他数据库类型，如Orcale，MySQL，均类似
```



- 根据上述工厂虚基类以及具体工厂的设计，对应修改EmployeeDAO类的代码，需要新增工厂成员

```c++
class EmployeeDAO{ 
    	IDBConnectionFactory* dbConnectionFactory;
        IDBCommandFactory* dbCommandFactory;
        IDBDataReaderFactory* dbDataReaderFactory;
public:
    vector<EmployeeDO> GetEmployees(){
        IDBConnection* connection =
            dbConnectionFactory->CreateDBConnection();
        connection->ConnectionString = "...";

        IDBCommand* command =
            dbCommandFactory->CreateDBCommand();
        command->CommandText="...";
        command->SetConnection(connection);

        IDBDataReader* reader = command->ExecuteReader();
        while (reader->Read()){

        }

    }
};
```

- 看起来依赖关系都理清楚了，但要注意，GetEmployees()各个类对象的相关性

  - ```c++
    command->SetConnection(connection);
    ```

  - ```c++
    IDBDataReader* reader = command->ExecuteReader();
    ```

  - 所以传进来的三个工厂对象所创建的对象必须是同一系列，即必须是同一数据库的三种对象，三个工厂生产的对象必须是同一系列



- 继续考虑上述代码，发现EmployeeDAO类中

- ```c++
  class EmployeeDAO{ 
      	IDBConnectionFactory* dbConnectionFactory;
          IDBCommandFactory* dbCommandFactory;
          IDBDataReaderFactory* dbDataReaderFactory;
  		//......
  };
  ```

- 这三个工厂特别有相关性，生产的对象互相有依赖，若将三个工厂放到一个工厂类中

```c++
//合并之前的三个工厂
class IDBConnectionFactory{
    virtual IDBConnection* CreateDBConnection() = 0;
};

class IDBCommandFactory{
    virtual IDBCommand* CreateDBCommand() = 0;
};

class IDBDataReaderFactory{
    virtual IDBDataReader* CreateDBDataReader() = 0;
};

//对其进行合并得到
class IDBFactory{
    virtual IDBConnection* CreateDBConnection() = 0;
    virtual IDBCommand* CreateDBCommand() = 0;
    virtual IDBDataReader* CreateDBDataReader() = 0;
}
```



- 按照上述对抽象基类的处理，将所有的具体工厂类也进行合并

```c++
//合并前
class SqlConnectionFactory : public IDBConnectionFactory{
  	  virtual IDBConnection* CreateDBConnection(){
          return new SqlConnection();
      }
};

class SqlCommandFactory : public IDBCommandFactory{
  	  virtual IDBCommand* CreateDBCommand(){
          return new SqlCommand();
      }
};

class SqlDataReaderFactory : public IDBDataReaderFactory{
  	  virtual IDBDataReader* CreateDBDataReader(){
          return new SqlDataReader();
      }
};

//合并后
class SqlDBFactory : public IDBFactory{
    
  	  virtual IDBConnection* CreateDBConnection(){
          return new SqlConnection();
      }
    
      virtual IDBCommand* CreateDBCommand(){
          return new SqlCommand();
      }
    
      virtual IDBDataReader* CreateDBDataReader(){
          return new SqlDataReader();
      }
    
};

//其他Orcale及MySQL同理
```



- 现在工厂合并完之后，对EmployeeDAO类的三个工厂字段也可以修改为一个工厂

```c++
class EmployeeDAO{ 
    	IDBFactory* dbFactory;
public:
    vector<EmployeeDO> GetEmployees(){
        IDBConnection* connection =
            dbFactory->CreateDBConnection();
        connection->ConnectionString = "...";

        IDBCommand* command =
            dbFactory->CreateDBCommand();
        command->CommandText="...";
        command->SetConnection(connection);					//类的关联性得到了保证

        IDBDataReader* reader = command->ExecuteReader();	//类的关联性得到了保证
        while (reader->Read()){

        }

    }
};
```

- 实际使用类似上一节工厂模式，也是在EmployeeDAO的构造函数中传入对应的具体工厂如SqlDBFactory，这样
  - IDBFactory的父类指针dbFactory指向了子类的SqlDBFactory对象，多态
  - IDBConnection类型的父类指针指向由dbFactory（实际是多态创建）生产的SqlConnection对象



### 三、总结

- 如果没有应对“多系列对象创建”的需求变化，则可直接用简单的工厂模式，无需用抽象工厂
- “系列对象”指的是在某一特定系列下的对象之间有相互依赖、或作用的关系。不同系列的对象之间不能有相互依赖
  - SqlServer之间的三个Connection、Command、DataReader对象相互依赖
  - 不同数据库之间的对象无依赖，如SqlServer和Connection和MySQL的Command之间不存在依赖，也无法产生对应关系

- Abstract Factory模式主要在于应对“新系列”的需求变动。其缺点在于难以应对“新对象”的需求变动。
  - 即可以增加新系列，如新增对Redis的支持
  - 但若在IDBFactory类中增加新对象，那就不太适合了，因为抽象基类要求是稳定的，不然代码维护成本高