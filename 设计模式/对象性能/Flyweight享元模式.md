## Flyweight享元模式

### 一、简介

#### 1、动机

- 在软件系统中采用纯粹对象方案的问题在于大量细粒度的对象会很快充斥在系统中，从而带来很高的运行时代价
  - 主要指内存需求方面的代价
- 避免大量细粒度对象问题的同时，让外部客户程序仍然能够透明地使用面向对象的方式进行操作

#### 2、定义

- 运用共享技术有效地支持大量细粒度的对象
  - 利用容器实现对象池pool
  - 若有该对象直接拿来用
  - 没有再创建，加入对象池



### 二、实例

#### 1、背景

- 实现一个字体类，通过一个string的key（如字体名字）对应到具体的字体
- 绝大多数字符使用的字体都是一样的，如一篇几万字的论文也就用了那么几种字体



#### 2、享元模式实现

```c++
class Font {
private:

    //unique object key
    string key;
    
    //object state
    //....
    
public:
    Font(const string& key){
        //...
    }
};

class FontFactory{
private:
    map<string,Font* > fontPool;	//维护一个字体对象池
    
public:
    Font* GetFont(const string& key){

        map<string,Font*>::iterator item=fontPool.find(key);
        
        if(item!=footPool.end()){
            return fontPool[key];
        }
        else{
            Font* font = new Font(key);
            fontPool[key]= font;
            return font;
        }

    }
    
    void clear(){
        //...
    }
};
```

- GetFont(const string& key)通过传入key获取字体对象
  - 若有，直接返回字体供使用
  - 若无，创建新对象并加入对象池



### 三、总结

- Flyweight主要解决面向对象的代价问题，一般不触及面向对象的抽象性问题（大部分设计模式）
- Flyweight通过采用对象共享的做法来降低系统中对象的个数，从而降低细粒度对象给系统带来的内存压力，在具体实现方面，要注意对象状态的处理