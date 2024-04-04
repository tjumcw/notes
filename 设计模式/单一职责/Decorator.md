## Decorator

### 一、简介

#### 1、动机

- 在某些情况下我们可能会“过度地使用继承来扩展对象的功能”，由于继承为类型引入的静态特质，使得这种扩展方式缺乏灵活性

  - 静态特质：结构化设计中由继承实现

  - ```c++
    FileStream::Read(number);//读文件流
    ```

  - 动态特质：装饰模式修改后由组合实现

  - ```c++
    stream->Read(number);//读文件流
    ```

- 随着子类的增多（扩展功能的增多），各种子类的组合（扩展功能的组合）会导致更多子类的膨胀

#### 2、需求

- 如何使“对象功能的扩展”能够根据需求动态实现
- 如何避免“扩展功能的增多”带来的子类膨胀问题
- 如何使任何“功能扩展带来的变化”的影响降到最低

#### 3、定义

- 动态（组合）地给一个对象增加一些额外的职责
- 就增加功能而言，Decorator模式比生成子类（继承）更为灵活
  - 消除重复代码
  - 减少子类个数



### 二、示例

#### 1、设计场景

- 设计一个IO的库，针对流操作，包括文件流，网络流，内存流
- 每个流都需要扩展一些功能，如对流加密，对流缓存等
- 思考流的设计，需要有一个基类（接口类），并提供read、seek、write等公共操作作为纯虚函数
- 每个单独的流类，如文件流需要重写3个虚函数，实现自己的功能
- 考虑扩展操作（加密、缓存），如加密，需要对流有一个主体的操作，才能加密
- 针对不同的流，其加密操作都是一样的，但是不同流需要加密的主体操作是不一样的
  - 文件流和网络流各自重写了基类三个公共操作虚函数，
  - 假设对文件流和网络流的read过程加密（其他过程加密同理）
    - 具体的read方法是不相同的，但加密的过程是相同的（在该背景下）
    - 可能对于同一文件流的不同操作主体，如read和write的加密方式不同，比方说read是在read文件流前加密，write是在write前后都加密
    - 但不同流的同一操作主体，如read对read，write对write，其加密方式都是一致的
- 具体看以下结构化设计的实现方式



#### 2、结构化设计

```c++
//业务操作
class Stream{
public:
    virtual char Read(int number)=0;
    virtual void Seek(int position)=0;
    virtual void Write(char data)=0;
    
    virtual ~Stream(){}
};

//主体类
class FileStream: public Stream{
public:
    virtual char Read(int number){
        //读文件流
    }
    virtual void Seek(int position){
        //定位文件流
    }
    virtual void Write(char data){
        //写文件流
    }

};

class NetworkStream :public Stream{
public:
    virtual char Read(int number){
        //读网络流
    }
    virtual void Seek(int position){
        //定位网络流
    }
    virtual void Write(char data){
        //写网络流
    }
    
};

class MemoryStream :public Stream{
public:
    virtual char Read(int number){
        //读内存流
    }
    virtual void Seek(int position){
        //定位内存流
    }
    virtual void Write(char data){
        //写内存流
    }
    
};

//扩展操作
class CryptoFileStream :public FileStream{
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        FileStream::Read(number);//读文件流
        
    }
    virtual void Seek(int position){
        //额外的加密操作...
        FileStream::Seek(position);//定位文件流
        //额外的加密操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        FileStream::Write(data);//写文件流
        //额外的加密操作...
    }
};

class CryptoNetworkStream : :public NetworkStream{
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        NetworkStream::Read(number);//读网络流
    }
    virtual void Seek(int position){
        //额外的加密操作...
        NetworkStream::Seek(position);//定位网络流
        //额外的加密操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        NetworkStream::Write(data);//写网络流
        //额外的加密操作...
    }
};

class CryptoMemoryStream : public MemoryStream{
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        MemoryStream::Read(number);//读内存流
    }
    virtual void Seek(int position){
        //额外的加密操作...
        MemoryStream::Seek(position);//定位内存流
        //额外的加密操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        MemoryStream::Write(data);//写内存流
        //额外的加密操作...
    }
};

class BufferedFileStream : public FileStream{
    //...
};

class BufferedNetworkStream : public NetworkStream{
    //...
};

class BufferedMemoryStream : public MemoryStream{
    //...
}


//既加密又缓冲
class CryptoBufferedFileStream :public FileStream{
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        //额外的缓冲操作...
        FileStream::Read(number);//读文件流
    }
    virtual void Seek(int position){
        //额外的加密操作...
        //额外的缓冲操作...
        FileStream::Seek(position);//定位文件流
        //额外的加密操作...
        //额外的缓冲操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        //额外的缓冲操作...
        FileStream::Write(data);//写文件流
        //额外的加密操作...
        //额外的缓冲操作...
    }
};


void Process(){

        //编译时装配
    CryptoFileStream *fs1 = new CryptoFileStream();

    BufferedFileStream *fs2 = new BufferedFileStream();

    CryptoBufferedFileStream *fs3 =new CryptoBufferedFileStream();

}
```

类的关系如下图

![image](https://user-images.githubusercontent.com/106053649/176860121-adce712c-1b4a-4897-a8d5-fb5fcf87386c.png)

其中，类的规模巨大（m种操作，n种类型的流），如红字所示。



#### 3、优化和重构

- 重新审视代码，发现所有涉及加密的操作都是一样的，只是各个类的主体操作不一样（自己各自重写的虚函数），代码如下(只展示read加密，其他类似)：

```c++
class CryptoFileStream :public FileStream{
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        FileStream::Read(number);//读文件流
        
    }
};

class CryptoNetworkStream : :public NetworkStream{
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        NetworkStream::Read(number);//读网络流
    }
};

class CryptoMemoryStream : public MemoryStream{
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        MemoryStream::Read(number);//读内存流
    }
};
```

- 若将继承关系转化为组合关系，则代码修改如下：

```c++
class CryptoFileStream {
    FileStream* stream;
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
        
    }
};

class CryptoNetworkStream {
    NetworkStream* stream;
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        stream->Read(number);//读网络流
    }
};

class CryptoMemoryStream {
    MemoryStream* stream;
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        stream->Read(number);//读内存流
    }
};
```

- 由继承改为组合后，发现代码主体格外的类似，结合面向对象的多态特性，可继续修改代码为：

```c++
class CryptoFileStream {
    Stream* stream;//= new FileStrean();
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
        
    }
};

class CryptoNetworkStream {
    Stream* stream;//= new NetworkStream();
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        stream->Read(number);//读网络流
    }
};

class CryptoMemoryStream {
    Stream* stream;//= new MemoryStream();
public:
    virtual char Read(int number){
        
        //额外的加密操作...
        stream->Read(number);//读内存流
    }
};
```

- 上述代码实现了编译时复用，变化延迟到运行时用多态的方式支持变化
- 再仔细思考，现在三个类除了名字一样，其他完全一样，可以合并，如下：

```c++
class CryptoStream {
    Stream* stream;//= new FileStream();
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
        
    }
};
```

- 仔细思考，考虑到接口规范，因为CryptoStream中Read()为虚函数，还需要继承Stream
- 通过Stream基类，为CryptoStream类定义了接口规范

```c++
class CryptoStream : public Stream{
    Stream* stream;//= new FileStream(); 
public:
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
        
    }
};
```

- 最终整合为：

```c++
//业务操作
class Stream{

public:
    virtual char Read(int number)=0;
    virtual void Seek(int position)=0;
    virtual void Write(char data)=0;
    
    virtual ~Stream(){}
};

//主体类
class FileStream: public Stream{
public:
    virtual char Read(int number){
        //读文件流
    }
    virtual void Seek(int position){
        //定位文件流
    }
    virtual void Write(char data){
        //写文件流
    }

};

class NetworkStream :public Stream{
public:
    virtual char Read(int number){
        //读网络流
    }
    virtual void Seek(int position){
        //定位网络流
    }
    virtual void Write(char data){
        //写网络流
    }
    
};

class MemoryStream :public Stream{
public:
    virtual char Read(int number){
        //读内存流
    }
    virtual void Seek(int position){
        //定位内存流
    }
    virtual void Write(char data){
        //写内存流
    }
    
};

//扩展操作


class CryptoStream: public Stream {
    
    Stream* stream;//...

public:
    CryptoStream(Stream* stm):stream(stm){
    
    }
    
    
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
    }
    virtual void Seek(int position){
        //额外的加密操作...
        stream::Seek(position);//定位文件流
        //额外的加密操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        stream::Write(data);//写文件流
        //额外的加密操作...
    }
};



class BufferedStream : public Stream{
    
    Stream* stream;//...
    
public:
    BufferedStream(Stream* stm):stream(stm){
        
    }
    //...
};


void Process(){

    //运行时装配
    FileStream* s1=new FileStream();
    CryptoStream* s2=new CryptoStream(s1);
    
    BufferedStream* s3=new BufferedStream(s1);
    
    BufferedStream* s4=new BufferedStream(s2);

}
```

- 最终根据“多个子类有相同字段，需要往上提”，即每个子类中的Stream*
- 创建一个中间类DecoratorStream，将Stream*放入其中，并稍微修改下继承关系，最终如下：

#### 4、Decorator模式

```c++
//业务操作
class Stream{

public:
    virtual char Read(int number)=0;
    virtual void Seek(int position)=0;
    virtual void Write(char data)=0;
    
    virtual ~Stream(){}
};

//主体类
class FileStream: public Stream{
public:
    virtual char Read(int number){
        //读文件流
    }
    virtual void Seek(int position){
        //定位文件流
    }
    virtual void Write(char data){
        //写文件流
    }

};

class NetworkStream :public Stream{
public:
    virtual char Read(int number){
        //读网络流
    }
    virtual void Seek(int position){
        //定位网络流
    }
    virtual void Write(char data){
        //写网络流
    }
    
};

class MemoryStream :public Stream{
public:
    virtual char Read(int number){
        //读内存流
    }
    virtual void Seek(int position){
        //定位内存流
    }
    virtual void Write(char data){
        //写内存流
    }
    
};

//扩展操作

DecoratorStream: public Stream{
protected:
    Stream* stream;//...
    
    DecoratorStream(Stream * stm):stream(stm){
    
    }
    
};

class CryptoStream: public DecoratorStream {
 

public:
    CryptoStream(Stream* stm):DecoratorStream(stm){
    
    }
    
    
    virtual char Read(int number){
       
        //额外的加密操作...
        stream->Read(number);//读文件流
    }
    virtual void Seek(int position){
        //额外的加密操作...
        stream::Seek(position);//定位文件流
        //额外的加密操作...
    }
    virtual void Write(byte data){
        //额外的加密操作...
        stream::Write(data);//写文件流
        //额外的加密操作...
    }
};



class BufferedStream : public DecoratorStream{
    
    Stream* stream;//...
    
public:
    BufferedStream(Stream* stm):DecoratorStream(stm){
        
    }
    //...
};


void Process(){

    //运行时装配
    FileStream* s1=new FileStream();
    
    CryptoStream* s2=new CryptoStream(s1);
    
    BufferedStream* s3=new BufferedStream(s1);
    
    BufferedStream* s4=new BufferedStream(s2);

}
```

- 梳理完后，有一些特性：
  - 文件流、网络流、内存类的类定义没有发生任何变化，它们可以单独行使行为
  - 扩展操作如加密流必须传流对象（如文件流），这些扩展操作是在谁的基础上再去做（装饰模式的含义）
- 见下图类关系

![image](https://user-images.githubusercontent.com/106053649/176860344-ddd6064d-4746-42f8-8722-d394c690f793.png)

- 注：DecoratorStream中有一个Stream指针，指向左边三个流对象中的某一个



### 三、总结

- 通过采用组合而非继承的手法， Decorator模式实现了运行时动态扩展对象的能力，且可以根据需要多占多个功能
  - 避免了使用继承带来的“灵活性差”
  - 避免了使用继承带来的“多子类衍生问题”
- Decorator类在接口上表现为is-a Component关系，即继承了Component类所具有的接口
  - 继承是为了完善接口（Component接口）的规范
- 但在实现上又表现为has-a Component关系，即类内有一个Component类对象
  - 组合是为了支持将来具体的实现类
- Decorator模式应用的要点是在于解决“主体类在多个方向上的扩展功能”——即装饰的含义
