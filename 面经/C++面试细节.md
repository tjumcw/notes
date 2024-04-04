# C++面试细节

### 1、C语言模拟封装类

首先理解类的封装

- 其实是将数据和行为整合到一起，形成高内聚（数据的处理依靠自身的函数，不依赖全局函数）
- 这样一个类对象既有自己的数据，也有处理数据的方法，是一个完整的模块
- C语言中的struct只能封装数据，不能封装函数

>思考如何把函数封装进结构体：
>
>- struct内只能存放数据类型，函数其实可以通过函数指针来调用，函数指针是一种类型

```c++
// 设计一个简单类
class Base{
	int val;
    void setVal(int x){
        this->val = x;
    }
};

// 其对应的struct版本如下：
struct Base;
typedef void (*fp_setVal)(Base* b, int x);
struct Base{
  	int val;
    fp_setVal fp;
};

void setVal(Base* b, int x){	//必须传指针，不然值传递内部不会修改
    b->val = x;
}

int main(){
    Base b;
    b.fp = setVal;
    b.fp(&b, 3);
    cout << b.val << endl;
}
```



### 2、何时需要考虑重写拷贝构造函数

- 存在需要分配的资源，可能是堆内存，或是文件的句柄，为了避免重复释放，需要重写拷贝构造函数



### 3、delete[]和delete

#### 1、malloc和free

- malloc只负责一件事，就是分配内存，传进去多少字节就分配多少字节，如

```c++
char* buf = (char*)malloc(10);
free(buf)
```

>注意事项：
>
>- 像是malloc和new（本质还是调的malloc），其动态申请的内存在堆上，但函数返回的地址即取得的指针仍在栈上，不过指针指向的空间是动态分配出来的堆内存
>- 所以char*类型的buf是在栈上的，但其指向的内存在堆中
>- buf的值是堆内存的首地址，本身在栈上（指针的值存的是地址）
>- 同理delete ptr或是free(ptr)也是去释放ptr所指向的内存



#### 2、new和delete

- new是运算符，会调用operator new，而operator new内部会调用malloc分配内存，然后调用构造函数
- delete是运算符，会先调用析构函数，再调用operator delete，operator delete内部调用free释放内存



#### 3、区别

- new/delete是运算符，由编译器支持，malloc和free是函数，需要包括malloc.h头文件
- new和delete会调用构造函数和析构函数，malloc和free只是简单的申请和释放内存
  - new先operator new分配完再构造，delete先析构再operator delete释放内存
  - 在运算符重载中可以自己定义内存分配策略，甚至不做内存分配，但一般就是调用malloc和free

- 因为malloc和free是库函数，无法承担自定义类型的对象构造和析构
- malloc需要做类型转换，其返回的是void*类型的指针，还需要指定大小



#### 4、delete和delete[]

##### 1）析构函数角度（侯捷分析）

- 首先delete和delete[]都是先调用析构函数再释放内存
- delete只会调用一次析构函数，delete[]会调用多次析构函数
- 对于对象内部带指针，且本身通过new获取空间的，是在对象内部析构函数进行内存释放的
- 若是new了这样的对象数组，如：

```c++
string* str = new string[3];
```

- delete和delete[]都能释放上句话申请的空间
- 但delete只能释放掉第一个对象内部new出来的内存，后两个对象的析构函数没有被调用



##### 2）编译器实现角度

- new[]在分配内存时会在最开始额外申请4个字节写入当前数组的长度，用于delete[]时多次调用析构函数
- delete甚至连这4个字节都不删，会直接从内存申请的数组的首地址开始释放（不包括额外的4字节）
  - 虽然padding在最上面，中间隔了一个额外的4字节，但是从首地址向下层padding找，也能释放该段内存

- delete[]编译器就知道上面可能有4个字节的偏移量，就去获取数组长度信息，并循环调用析构函数后在释放内存



### 4、异常处理机制

- C++通过try—catch机制实现异常处理
- try中的代码若出现异常，catch就能捕获到异常并处理异常
- 也可以通过throw主动抛出异常，会被catch捕获并处理
- 可以有多个catch，类似switch case结构，针对不同的异常类型处理，cathc(...)表示接收所有类型的异常

```c++
int main(){
    try{
        throw 1;
        throw 1.0f;
        throw "Hello";
    }
    catch(int n){
        cout << n << endl;
    }
    catch(float n){
        cout << n << endl;
    }
    catch(...){
        cout << "error" << endl;
    }
}
```

- 上述代码，注释掉各个throw单独运行，可以发现对于int和float类型的异常有对应的catch，对于string类型的异常会被catch(...)捕捉并处理



- 异常是一层一层往外抛的：
  - 当抛出一个异常后，程序暂停当前函数的执行过程并立即开始寻找与异常匹配的catch子句。
  - 当throw出现在一个try语句块内时，检查与该try块关联的catch子句。
  - 如果找到了匹配的catch，就使用该catch处理异常。
  - 如果没找到匹配的catch且该try语句嵌套在其它try块中，则继续检查与外层try匹配的catch子句。
  - 如果还是找不到匹配的catch，则退出当前的函数，在调用当前函数的外层函数中继续寻找
  - 栈展开过程沿着嵌套函数的调用链不断查找，直到找到了与异常匹配的catch子句为止：或者也可能一直没找到匹配的catch，则退出主函数后查找过程终止。
  - 当找不到匹配的catch时，程序将调用标准库函数terminate，terminate负责终止程序的执行过程。



- C++语言本身或者标准库抛出的异常都是 exception 的子类，称为标准异常（Standard Exception）
- 可以通过下面的语句来捕获所有的标准异常：

```c++
try{
    //可能抛出异常的语句
}catch(exception &e){
    //处理异常的语句
}
```

>通过exception类：
>
>- 抛出的异常均是其子类，子类is-a父类，符合语法，通过其封装的函数可以获取一些异常的信息
>- 比如 e.waht()，可以通过具体类重写父类what的虚函数，对不同的情况取得不同的异常信息



### 5、大量TIME_WAIT

- TIMEWAIT状态本身和应用层的客户端或者服务器是没有关系的，主动关闭连接一方会进入该状态。
- 对于服务端的每个端口，都能接受大量客户端的连接
- 若是服务器只==被动关闭==连接，不用考虑该问题

> 关键点：
>
> - 在==高并发短连接==的TCP服务器上，当服务器处理完请求后立刻按照主动正常关闭连接。这个场景下，会出现大量socket处于TIMEWAIT状态。如果客户端的并发量持续很高，此时部分客户端就会显示连接不上。
> - 很重要的一点是的TIME_WAIT指的是断开连接一方的那个==端口==，不是指机器
>   - 服务器若主动断开连接，只是该连接之前占用的端口会进入TIME_WAIT，其他端口正常

- 高并发会让服务器在短时间范围内同时被占用大量端口
- 短连接表示业务处理+数据传输的时间远小于TIMEWAIT超时的时间
- 由于服务器端主动关闭连接，其对应的端口会进入TIMEWAIT状态

- 综合高并发及短连接会使服务器因端口资源不足而拒绝为一部分客户服务
  - 同时，这些端口都是服务器临时分配，无法用SO_REUSEADDR选项解决这个问题



解决方法：

- 排查程序本身bug
- 尝试通过setsockopt()设置SO_REUSEADDR

- 调整内核参数，让服务器能够快速回收和重用那些TIME_WAIT的资源
- 客户端将短连接改为长连接
  - HTTP 请求的头部， connection 设置为 keep-alive， 保持存活一段时间



### 6、自动类型转换

**1、转换按数据长度增加的方向进行，以保证精度不降低。**

如int型和long型运算时，先把int量转成long型后再进行运算

- 若两种类型的字节数不同，转换成字节数高的类型
- 若两种类型的字节数相同，且一种有符号，一种无符号，则转换成无符号类型==（无符号精度高）==

**2、所有的浮点运算都是以双精度进行的，即使是两个float单精度量运算的表达式，也要先转换成double型**

**3、在赋值运算中，赋值号两边量的数据类型不同时，赋值号右边量的类型将转换为左边量的类型。**

**4、如果遇到无符号数与有符号数之间的操作，编译器会自动转化为无符号数来进行处理**

```c
unsigned int a = 20;
signed int b = -130;
// 实验证明b>a，也就是说－130>20
```



### 7、嵌入式为什么用C不用C++

- C++的类虽然利用泛型能适配各种场景，但更占用内存，加载更慢，嵌入式要求快
- 单片机的资源有限，占用更大的内存不利于开发
- 嵌入式开发大都是Linux系统，用C++很多地方调用的也是C相关的系统调用去操作操作系统



### 8、一个.h定义的变量能被多个.c文件用吗

- 首先，在.h中通过extern关键字声明变量
- 然后在其中一个包含了该.h文件的.c文件中定义该变量，即赋值
- 所有引用该.h的.c文件都可以使用该变量
- extern对应的关键字是 static，被它修饰的全局变量和函数只能在本模块中使用
- 不仅是变量，通过extern声明的函数也可以在本模块或其它模块中使用



### 9、进程和线程

#### 1、简单区分

- **线程**：进程内部的一条执行路径或序列，一个进程可以包含多条线程。（CPU调度和执行的基本单位，针对task_struct）

- **进程**：一个正在进行的程序 （资源分配的基本单位）

#### 2、深入理解

- 从Linux内核的角度，它并没有线程这个概念，Linux把所有的线程都当作进程来实现。

- 线程仅仅被视为一个与其他进程共享某些资源的进程。每个线程都拥有唯一隶属于自己的task_struct。
- 所以在内核中，它看起来就就像是一个普通的进程（只是该进程和其他一些进程共享某些资源，如地址空间）。

![image-20220815232716444](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220815232716444.png)

>关键数据结构：
>
>- 创建一个进程需要一堆数据结构, 进程控制块（task_struct）、进程地址空间（mm_struct）以及页表的创建，虚拟地址和物理地址就是通过页表建立映射的。
>- 对于线程，其也是用task_struct来表征的，将其视为与父task_struct共享进程地址空间和页表的进程即可
>  - 这里的父task_struct即可视为用来申请系统资源的进程
>  - 本质上Linux不区分进程、线程，均视为tack_struct，一个进程至少有一个task_struct（即主线程）
>- 因此，所谓的进程并不是通过task_struct来衡量的，除了task_struct之外，一个进程还要有进程地址空间、文件、信号等等，合称为一个进程。
>- **因此，站在内核角度来理解进程：承担分配系统资源的基本实体，叫做进程。**



#### 3、进程和线程对比

##### 1）线程优点

- 创建一个新线程的代价要比创建一个新进程小得多。
- 切换代价小（不需要切换地址空间和页表）。
- 线程占用的资源要比进程少很多。
- 能充分利用多处理器的可并行数量。
- CPU密集型任务，将计算任务分解到多个线程分别执行
- IO密集型，创建多个线程IO，合理利用CPU资源让其他线程处理逻辑

##### 2）进程优点

- 进程有独立的地址空间，一个进程崩溃后，在保护模式下不会对其它进程产生影响
- 线程没有单独的地址空间，一个线程死掉就等于整个进程死掉，所以多进程的程序更健壮
- 多进程的性能上限更高，因为进程占用的资源更多，成本高则上限高
- UNIX环境下，其进程调度效率是很高的，多进程调度开销和多线程调度开销没有显著区别

>注意点：
>
>- 进程栈在进程的栈区，即主线程采用的栈是进程地址空间中原生的栈（向下生长的）
>- 其余线程采用的栈就是在共享区中开辟的，不是向下生长的，只是先进后出



#### 4、最终总结

- 虽然老生常谈，但是总结来说：**进程是资源分配的基本单位，线程是CPU调度和执行的基本单位**

>面试回答：
>
>- 进程是资源分配的基本单位，因为进程在创建时，操作系统会为每个进程分配虚拟地址空间，需要创建进程控制块，创建页表（为虚拟地址和物理地址建立映射），每个进程的地址空间都是独立的。
>- 线程是CPU调度和执行的基本单位，相对来说进程并不纯粹是程序执行的逻辑，或者说进程不单单是只有程序逻辑（主线程）就够了，很重要的一点是需要有虚拟地址空间等系统资源才能称为一个进程，而线程更像是一段代码逻辑或是一段CPU上执行的命令序列，是程序执行的基本单位。
>- 因为对于内核来说，是不区分进程和线程的，其在内核中都是用task_struct结构体来表征的，无非就是进程创建时必然有一个主线程对应的task_struct，每个其他线程创建也会对应一个task_struct，可以将其理解为：
>  - 进程创建时，有一个父task_struct，可视为用来申请系统资源的进程
>  - 之后创建的所有线程对应到子task_struct，也可将其视为与父task_struct共享进程地址空间的进程（进程线程无所谓）
>  - 所以对于内核来说其不做任何区分，对CPU来说也无所谓区不区分，就是待执行的命令序列。只不过原生主线程占用地址空间的栈，之后创建的线程栈来自共享区
>
>进程和线程调度、通信开销：
>
>- 线程间调度开销小，其只需要保存简单的上下文信息和寄存器信息即可，进程需要切换虚拟地址空间，开销大。
>- 进程由于其独立性（虚拟地址空间都是独立的），进程间通信开销也比较大，一般都涉及到文件fd
>  - 如匿名管道，虽无文件实体，但也有对应的pipe[2]数组，对应到读端和写端的文件描述符
>  - 有名管道（mkfifo）和socket（sockaddr_un）类似，均对应到文件实体
>  - mmap的内存映射一块磁盘区域，返回fd，不同进程通过打开同一个fd通信
>  - 共享内存也是将进程中的内存共享，消息队列也需要额外的数据结构，信号涉及内核态切换，开销也大
>- 线程通信很简单，线程间共享全局变量，通过线程同步可以安全地读取数据进行通信，且本身线程同步也可视为通信，开销小
>
>多进程和多线程：
>
>- 多进程资源消耗大，成本高，但性能上限更高
>- 进程有独立的地址空间，一个进程崩溃后，不会对其他进程产生影响，更健壮
>- 线程没有单独的地址空间，一个线程死掉就等于整个进程死掉
>- 线程创建开销小、调度、通信开销小，能充分利用并发的优势



### 10、一致性哈希算法

#### 1、背景介绍

- 首先在分布式缓存的背景下，假设有3台服务器分别为S0,S1,S2，同时有三万张图片需要缓存
- 为了让图片均匀分布，以图片key利用hash函数求hashcode，再对服务器数量取模决定数据缓存在哪
- 下次在用同样的方法就能得到图片所在服务器并查看图片

>上述算法即为哈希算法

- 但若是增加新的服务器，总数到了4台，对同一张图片取模求得的服务器编号就变了
  - 如原先6 % 3 = 0，存在S0上。现在6 % 4 = 2，访问S2查看图片却查不到了
- 服务器数量变化导致不能正常从缓存中获取数据，此时程序就向后端服务器请求数据。由于这种情况下大量缓存同时失效，导致缓存雪崩，后端服务器很有可能因为压力过大而被压垮。



#### 2、一致性哈希算法

- 首先考虑一个圆环，假设这个圆环上有2^32个节点，假设依旧是3台服务器A,B,C
- 利用hash函数求每个服务器所在位置，如Hash(A) % 2 ^ 32求得一个环上的整数代表服务器A，其他同理映射
- 对于需要缓存的记录，依旧利用该方法映射到哈希环上，并从该位置沿顺时针查找遇到的第一个服务器作为缓存服务器
- 由于hash结果不变，在服务器数量不变的情况下，依旧可以用相同方法找到对应服务器查看图片

>此时考虑增加服务器，如原先A,B,C的情况下插入D，如下图

![image-20220816205715378](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220816205715378.png)

- 可以看到，原先映射到C到A之间的图片会缓存到A上，现在插入D后，原先C顺时针到D之间的部分即属于服务器A缓存的图片会失效，但其他从D顺时针到C的图片依旧能对应到正确的服务器上

- 增加服务器时，并不是所有缓存失效，只是部分缓存失效，减少了后端服务器的压力



#### 3、一致性哈希算法的问题和改进

由于哈希函数映射的不确定性，可能发生哈希倾斜，即大部分缓存会集中到一台服务器上，如下图：

![image-20220816210133112](C:\Users\mcw\AppData\Roaming\Typora\typora-user-images\image-20220816210133112.png)

>解决方法：
>
>- 尽可能增加服务器的数量，让缓存尽可能均匀的分布到服务器上
>- 虽然真实服务器只有3台，但可以增加物理节点对应的虚拟节点，如A1,A2,A3,落到该部分的图片会缓存到真实服务器A上
>- 进行缓存读写时，先找到虚拟节点，从虚拟节点对应到真实节点后再进行缓存数据的读取



### 11、函数指针

- 一般通过typedef来定义
- 有两种不同的调用方式，p()或(*p)()，建议用第一种

```c++
#include <bits/stdc++.h>
using namespace std;

class Person;

//定义一个函数指针类型,即pfn表示一个返回值为void，函数参数为空的函数指针类型
//pfn是一个重新定义的类型，其存的是地址，声明该类型时加了*表示其本身是指针类型，存的值为地址
typedef int (*pfn)();  

int helper(){
    cout << "is called" << endl;
    return 0;
}

int main(){
    pfn p = helper;     //函数名表示地址，即对指针类型的变量初始化
    pf = helper;
    int c = p();
    int d = (*p)();		//两种调用方式都可以
}
```



- 也可以不通过typedef来声明类型，每次用到直接定义一个，调用方式相同

```c++
int main(){
    int (*pf)();
    pf = helper;
    int c = pf();
    int d = (*pf)();
}
```



### ==12、类的指针理解==

- 类指针声明以及赋值都不会调用类的构造函数
  - 声明仅是表明这是一个该类的指针，占几个字节罢了（可以随意类型转换），不初始化则为野指针
  - 若用一个对象的地址给该对象指针赋值，也不会调用构造，声明类对象时调过一次，赋值只是把指给指针罢了
- 类指针只有在new关键字中才会调用构造函数（new内部实现机制就是先分配内存，再调用构造函数）

>指针理解：
>
>- 指针只是表示一种存地址的变量，其存的值是其他变量的地址，其也有自己的地址
>- 把指针看成指向一块内存区域首地址的变量，可以通过类型转换验证一些想法

```c++
class Base{
public:
    Base(){
        a = 1, b = 2;
    }
    int a;
    int b;
};

int main(){
    int a = 3;
    Base* b;
    b = (Base*)&a;
    cout << b->a << endl;
    cout << b->b << endl;
}

// 打印结果如下：
zsf@zsf:~/Linux$ ./a
3
-2007512692
zsf@zsf:~/Linux$ ./a
3
-1933190596
zsf@zsf:~/Linux$ ./a
3
-785997268
zsf@zsf:~/Linux$ ./a
```

- 可见，对于原先Base*类型的指针，其指向的对象应该是8个字节
- 通过将int的地址强转为Base*类型赋值给b，b的首地址就指向了变量a内存首地址
- 通过Base*指针取得内部字段的值，也是通过从指针指向的地址对用的内存中取出对应的字节数
- 对于成员变量a，取出4个字节，强转后的地址对应内存的前四个字节，正好存的是int的值，为a
- 对于该int变量后续的4个字节，是未知的，存的可能是任意值，也可见每次结果都不同



### ==13、虚函数==

> 理解：
>
> - 每个声明了虚函数的类都有一个虚表，该类的所有对象共有一个，是static类型的。
> - 虚表是一个存函数指针的数组，每个元素都是一个虚函数对应的函数指针。
> - 虚表中会按照父类虚函数声明的顺序，把成员函数放到虚表中。
> - 子类会继承这个虚表，即复制一个一样的虚表，为该子类所独有且被所有子类对象共用的
>   - 复制表示虚表的内存地址和父类的不一样，不是copy-on-write，是复制了一份一样的
> - 若子类重写了某个虚函数，会发生函数覆盖：
>   - 即在父类对应虚函数的位置上的函数指针被替换成子类重写的函数指针。
> - 每个有虚函数的类对象，其在对象内存分布中，首4个字节会是虚函数指针，指向该类的虚表
> - 同一类的不同对象的虚函数指针的值，即指针变量存的地址相同（都是该类虚函数表的首地址）
> - 虚表是只读的，不存在线程安全问题，多父类继承时会有多个虚表0-3字节，4-7字节依次往后



#### 1、手动寻址调用虚函数（64位）

- 首先理解强制类型转换，如下：

```c++
long a = 3;
long* q = (long*)a;		//表示声明一个指针变量，且其值被初始化为=右边的值，等号右边把3强转为指针的值
cout << q << endl;		//q是一个存long类型数据地址的变量，上一步把3强转为long*类型的地址赋值给q了

//结果为0x3，即把3作为对应地址赋值给q	cout << *q;会段错误，因为其没有指向的内存空间
```



- 进行手动寻址

```c++
#include <bits/stdc++.h>
using namespace std;

class Person;	//前向声明
typedef void (Person::*pfn)();

class Person{
public:
    virtual void speak(){
        cout << "Person is speak" << endl;
    }
    void eat();
    int type;       //内存对齐后Person类对象大小为16，即8 + 4对齐到16
};

class Chinese : public Person{
public:
    void speak(){
        cout << "Chinese is speak" << endl;
    }
};

class English : public Person{
public:
    void speak(){
        cout << "English is Speak" << endl;
    }
};

union funcPtr
{
    unsigned long num;
    pfn funPtr;
};

int main(){
    Person *p[3];
    Person p1;
    p[0] = &p1;
    Chinese c;
    p[1] = &c;
    English e;
    p[2] = &e;
    
    for(int i = 0; i < 3; i++){
        unsighed long num = *(long*)p[i];
        funcPtr ptr;
        ptr.num = *(unsighed long*)num;		//用到了上述说的强制类型转换
        (p[i]->*(ptr.funPtr))()
    }
}

/*
zsf@zsf:~/Linux$ ./test 
Person is speak
Chinese is speak
English is Speak
*/
```



==首先第一步==，将p[i]的值，即对象地址或是说this指针指向的地址的前8个字节取出来

```c++
unsigned long num = *(unsigned long*)p[i];	//64位系统下指针为8个字节
```

- 之所以转为long就是因为其大小8个字节，通过*解引用拿到的是前八个字节的值，即虚函数指针的值
- 虚函数指针的值即虚函数表的地址，但要注意该值是通过解引用变成long类型的变量，现在是数值不表示地址



==其次第二步==，将该数值转为地址并同样通过解引用取前8个字节，得到虚函数表前8个字节的值

```c++
 ptr.num = *(unsigned long*)num;		//因为不能直接转为函数指针类型，通过共用体转化
 //pfn pfunc = (pfn)*(unsighed long*)num; 取出前8个字节转为pfn类型，会报错
```

- 虚函数表前8个字节存的是虚函数指针的地址，此时利用共用体不需要把该8个字节再做类型转换

- 因为8个字节的值是虚函数指针的地址，虽然赋给num，但共用体所有字段共用内

- 所以该8个字节也对应到函数指针的类型，函数指针的值=地址没问题，不需要值到指针再解引用了



==第三步==，调用对象的成员函数（通过该虚函数指针）

```c++
// 通过函数指针的另一种调用方式即 (*p)()的方式而不是pfn()的方式，具体如下
(p[i]->*(ptr.funPtr))();
```

- 首先对ptr.funPtr加括号表示其为一个整体，可视作上述中的p，表示函数指针

- 根据上述调用方式，加一个\*再加括号，所以先外加一个*形成\*p的结构
- 由于是对象的成员函数指针，不指定对象没法调用，故得通过对象指针调用表示其作用域
- 即形成了p[i]->*(ptr.funPtr)的结构，其整体表示为上述形式的\*p
- 最后对其加()后，再在其后加一个括号表示调用，就形成了上述调用形式



#### 2、虚函数的直接调用

通过父类指针直接加特定类的作用域限定符直接调用对应虚函数，无需动态绑定。



### 14、类和struct的tips

- C的struct不允许有函数，C++的struct表现和class几乎等同，唯一就是默认属性为public（class为private）
- C++的struct可以继承，可以定义虚函数，可以用public等关键字



- 常量一旦定义就不能修改，所以声明时需要直接初始化
- 静态成员函数没有this指针，this指针本身是一个指针常量，指针指向不能改但是指向的内存可以改（为T * const类型）
- const成员函数限定其对应的this指针为const T * const类型，故指针的值和指针指向的值均不能修改
- 可以通过修改this间接更改const成员函数内成员变量的值，代码如下所示：

```c++
class Base{
public:
    int num;
};

void Base::getVal() const{
    Base * const p = (Base * const)this;
    p-> num = 10;
    return this->num;
}
```



### 15、Malloc的线程安全和可重入

- 线程安全就是多个线程并发执行相同代码，程序的结果依然是正且一致确的
- 可重入指函数简单来说就是可以被中断的函数
  - 即在这个函数执行的任何时刻中断它，转入 OS 调度下去执行另外一段代码，执行完返回原函数时不会有错误。
- 可重入的要求比线程安全高，**可重入必定是线程安全的，但线程安全不一定可重入**。

>==结论：malloc线程安全，但不可重入==
>
>- malloc针对的是共享的堆空间内存（临界资源）进行操作，肯定需要加锁，不然也不会用，必定线程安全
>  - 这个锁必须是递归锁
>- 因为如果当程序调用malloc函数时收到信号，在信号处理函数里再调用malloc函数，如果使用一般的锁就会造成死锁
>- 虽然递归锁能保证线程安全，但是不能保证它的可重入性。
>- 程序调用malloc函数时收到信号，在信号处理函数里再调用malloc函数就可能破坏共享的内存链表等资源，因而是不可重入的。



### 16、继承时析构为什么虚函数

- ==哪怕没有多态==，只有要继承关系，基类的析构函数一定要声明为虚析构函数

- 考虑普通继承，声明一个父类指针指向一个子类对象

```c++
class Base{
public:
    Base(){
        cout << "Base ctor " << endl;
    }
    ~Base(){
        cout << "Base dector" << endl;
    }
};

class Son : public Base{
public:
    Son(){
        cout << "Son ctor" << endl;
    }
    ~Son(){
        cout << "Son dector" << endl;
    }
};

int main(){
    Base* son = new Son;
    delete son;
}

/*结果为
Base ctor 
Son ctor
Base dector
*/
```

>分析：
>
>- 派生类对象构造时先构造父类，再构造子类，析构时相反，先调用子类析构再调用父类析构
>- 若基类不声明为虚析构，则该父类指针或引用指向子类对象（普通继承也一样），在delete时先调用析构函数在释放内存
>- 第一步调用析构函数不会发生动态绑定，传入delete的指针是基类类型，只会调基类的析构函数就认为结束了（因为基类一定是最后析构的），派生类存在内存泄漏可能（若析构做了delete操作）
>- 调完析构



思考为什么构造函数不能声明为虚函数：

- 声明的虚函数会存在该类的虚表中，通过对象内存地址首四字节的虚函数指针寻址找到
- 若是构造函数声明为虚函数，也需要通过对象内存地址进行寻址调用构造函数
- 但构造函数都还没完成，内存中也就不存在这一个对象，即无法通过对象地址寻找虚表中构造函数并调用
- 典型的一个先有鸡还是先有蛋的问题



### 17、友元

- 友元的声明可以在类内任意地方（public、private、protected），被声明的函数或类可以访问本类的私有成员

#### 1、友元普通函数

```c++
class A{
public:
    friend void foo();
private:
    int num;
};

void foo(){
    A a;
    a.num = 1;
}
```



#### 2、友元成员函数

- 友元成员函数有2个要求：
  - 被声明为友元成员函数对应的类B要在类A前面声明
  - 友元成员函数的实现要写在类外

```c++
class B{
public:
    void foo();
};

class A{
public:
    friend void B::foo();
private:
    int num;
};

void B::foo(){
    A a;
    a.num = 3;
}
```



#### 3、友元类

- 被声明成友元类的类B可以访问朋友类A的所有私有数据

```c++
class B{
public:
    void foo();
    void foo2();
};

class A{
public:
    friend B;
private:
    int num;
};

void B::foo(){
    A a;
    a.num = 3;
}

void B::foo2(){
    A a;
    a.num = 3;
}
```



#### 4、分析

- 友元破坏了类的封装性和数据的隐藏性
- 不推荐使用，在某些方面如运算符重载时可酌情使用



#### 18、转换函数

- 类内声明的转换函数可以将本来转换为另一种类型
- 不能对转换函数指定返回值类型，但内部可以return

```c++
class Test{
public:
    operator bool(){
        if(num == 1){
            return true;
        }
        return false;
    }
    operator int(){
        return num;
    }
private:
    int num;
};

int main(){
    Test a;
    if(a){
        cout << 1 << endl;
    }else{
        cout << 0 << endl;
    }
}
```

>注意：
>
>- 若不定义转换函数则if语句会报错



### 19、函数重载、重写、重定义

#### 1、函数重载

- 函数名相同、参数必须不同（可以是个数、类型）
- virtual关键字可有可无（类内的话）
- ==必须在同一个作用域里，如都是全局函数或都是成员函数==

#### 2、函数重写（函数覆盖）

- 其实就是派生类重写基类的虚函数，必须与基类的函数名、参数等完全一致
- ==本质是发生了虚函数表中的函数覆盖，即派生类的函数指针覆盖了复制来的虚表中的基类虚函数指针==

#### 3、函数重定义（函数隐藏）

- ==不同作用域定义的同名函数之间才能发生函数隐藏==
  - 派生类的函数屏蔽了与其同名的基类函数
  - 类成员函数屏蔽了与其同名的全局外部函数
- 派生类的函数与基类的函数完全相同，但基类没有virtual关键字，此时基类被隐藏（有的话是覆盖）
- 派生类的函数与基类的函数同名但参数列表不同，不管基类有没有virtual，都被隐藏

>关键理解：
>
>- 派生类的函数与基类的函数同名，但是参数不同。此时，若基类无virtual关键字，基类的函数将被隐藏。
>- 虽然函数名相同参数不同应称之为重载，但这里不能理解为重载（==作用域不同==）
>- 通过作用域的显式调用可以访问被隐藏的函数



### 20、强制类型转换

强制类型转换是有一定风险的，但有的转换不一定安全，比如把**整形**转换为**指针**，把**基类指针**转换成**派生类指针**，把一种**函数指针**转换成另一种**函数指针**，把**常量指针**转换成**非常量指针**。

#### 强制转换运算符

C++引入了四种功能不同的强制类型转换运算符以进行强制类型转换：

- const_cast
- static_cast
- reinterpret_cast
- dynamic_cast

#### 1、const_cast

- 可用于const以及volatile属性的去除，四种转换中唯一去除const属性的运算符
- 只能进行指针类型、引用类型、this指针的转换

```c++
class test{
public:
    void foo(int n) const{
        //3、this指针是T* const类型，const函数中this指针是const T* const类型，转换this指针
        const_cast<test* const>(this)->num = n;
    }
private:
    int num;
};

int main(){

    //1、指针转换
    const int n = 5;
    int* p = const_cast<int*>(&n);
    *p = 123;

    //2、引用转换
    const string str = "hello world";
    string s = const_cast<string&>(str);
    s = "sas";

    test t;
    t.foo(1);
}
```



#### 2、static_cast

- 本质上就是隐式转换，给它显式的表现出来（更明确）
- 可以直接对类型操作而不是const_cast的指针、引用、this指针
- 可以进行低风险的转换，高风险的不行（与直接隐式转换几乎一致）

```c++
class CInt{
public:
    operator int(){
        return num;
    }
    int num;
};

int main(){
    int n;
    double f = 10.0f;

    //低风险的转换，就是精度的变化罢了，等价于 f = n;
    f = static_cast<float>(n);

    //低风险的转换，void*指针的转换
    void* p = nullptr;
    int* pN = static_cast<int*>(p);

    //转换运算符的方式,类内要定义转换函数
    CInt c;
    int num = static_cast<int>(c);

    //高风险转换,相当于直接 k = kk;也不会允许，这种int转int*的方式必须强转
    int kk;
    int* k = static_cast<int*>(kk);     //会报错
}
```



#### 3、dynamic_cast

用于具有**虚函数的基类**与**派生类**之间的**指针或引用**的转换

- **基类必须具备虚函数**
  - dynamic_cast是运行时类型检查（==RTTI==），而这个信息与类的**虚函数表**关系紧密。

- **运行时检查，转型不成功则返回一个空指针**

- **非必要不要使用dynamic_cast（确定类型安全就不用），有额外的函数开销**

>常见的转换方式：
>
>- 基类指针转派生类指针（必须用dynamic_cast）
>  - 其他转换编译也不报错，运行时直接操作指针访问了超过基类大小的其他未知内存，危险
>  - 使用dynamic_cast在运行时会报错，虽然报错但防止了危险行为
>- 派生类指针转基类指针（可以使用dynamic_cast，但是**更推荐使用static_cast**）

```c++
class Father{
public:
    virtual void foo();
    int father;
};

class Son : public Father{
public:
    void foo();
    int son;
};

int main(){
    Father f;
    Son s;

    Father* pf = &f;
    Son* ps = &s;

    //向下转换，父类转子类，不安全,运行时会报错
    ps = dynamic_cast<Son*>(pf);

    //向上转换，子类转父类，安全
    pf = static_cast<Father*>(ps);
}
```



#### 4、reinterpret_cast

- 其实和强制转换没有任何区别，就是规范化了强转的格式，后果自负

```c++
int main(){
    int p =1;
    int* pf = (int*)p;
    //和上述的强转效果一致
    int* kf = reinterpret_cast<int*>(p);
}
```



### 21、function、bind

**function是函数对象包装器，通过封装函数，使得函数也表现为一个对象**

>支持四种函数的封装：
>
>- 普通函数
>- 匿名函数
>- 成员函数，需要有一个额外的this指针作为参数
>- 仿函数，也需要有对应类的this指针作为参数



#### 1、普通函数

```c++
void foo1(int n){
    cout << n << endl;
}

int main(){
    function<void(int)> f = foo1;
    f(3);
}
```



#### 2、匿名函数

```c++
int main(){
    function<void(int)> f = [](int n) -> void{
        cout << n << endl;
    };
    f(3);
}
```



#### 3、成员函数

注意点：

- 需要传入this指针，调用时也需要传入特定对象的地址
- 需要加作用于限定符以及取地址符

```c++
class Test{
public:
    void foo2(int n){
        cout << n << endl;
    }
};

int main(){
    function<void(Test*, int)> f = &Test::foo2;
    Test t;
    f(&t, 3);
}
```



#### 4、仿函数

- 仿函数是一种表现像函数的类，重载了()运算符，通过对象加（参数）达到类似函数调用的效果

```c++
class Test{
public:
    void operator()(int n){
        cout << n << endl;
    }
};

int main(){
    function<void(Test*, int)> f = &Test::operator();
    Test t;
    f(&t, 3);
}
```



#### 5、bind去看侯捷老师C++11笔记



### 22、引用计数

#### 1、深拷贝和浅拷贝

- 深拷贝安全，但内存消耗比较大
- 浅拷贝面临重复释放内存的危险，但是节约内存（不修改时）

- 若实现如下，则t3在初始化时调用了拷贝构造函数，发生浅拷贝，程序结束时重复释放报错

```c++
class Test{
public:
    Test(int n){
        this->num = new int(n);
    }

    Test(Test& test){
        this->num = test.num;
    }

    Test& operator=(Test& test){
        this->num = test.num;
        return *this;
    }
    ~Test(){
        if(num != nullptr){
            delete num;
            num = nullptr;
        }
    }
private:
    int* num;

};

int main(){
    Test t1(3);
    Test t2(3);
    Test t3(t2);
}
```



#### 2、引用计数

- 对上述代码进行改造，类内增加一个引用计数，每当发生拷贝构造或拷贝赋值时增加
- 析构时判断引用计数是否为0，为0时才真正去释放内存

```c++
class Test{
public:
    Test(int n){
        this->num = new int(n);
        this->ref = new int(1);
    }

    Test(Test& test){
        this->num = test.num;
        this->ref = test.ref;
        (*ref)++;
    }

    Test& operator=(Test& test){
        this->num = test.num;
        this->ref = test.ref;
        (*ref)++;
        return *this;
    }

    ~Test(){
        if(num != nullptr && --(*ref) == 0){
            delete num;
            num = nullptr;
            delete ref;
        }
    }
private:
    int* num;
    int* ref;    //用static的话，t1会被t2影响，肯定不对。直接int每个引用计数无关，也不对
};

int main(){
    Test t1(3);
    Test t2(3);
    Test t3(t2);
}
```

>几个关键点：
>
>- ref必须是int*类型
>  - 若是int类型，则每个对象的引用计数无关，无法解决浅拷贝
>  - 若是static int类型也不行，所有对象都共用一个引用计数，就上述例子的t1就被t2和t3影响了
>- 拷贝赋值函数需要特别处理，因为拷贝赋值指的是对一个已经初始化的对象用另一个对象赋值（侯捷面向对象(上)笔记）
>  - 需要先判断是否指向同一块内存，是的话无需处理
>  - 不是的话先要对原来的引用计数减一，赋值后再对共同的引用计数加一



#### 3、引用计数类

```c++
class Ref{
public:
    Ref(int n){
        this->num = new int(n);
        ref = 1;
    }

    ~Ref(){
        if(num != nullptr){
            delete num;
            num = nullptr;
        }
    }

    void addRef(){
        this->ref++;
    }

    void delRef(){
        if(--ref == 0){
            delete this;    //此时会调用析构
        }
    }

private:
    int* num;
    int ref;    //此处可以用ref，因为封装用类的指针表示即可
};

class Test{
public:
    Test(int n){
        this->ref = new Ref(n);
    }

    Test(Test& test){
        this->ref = test.ref;
        this->ref->addRef();
    }

    Test& operator=(Test& test){
        if(test.ref == this->ref){
            return *this;
        }
        this->ref->delRef();
        this->ref = test.ref;
        this->ref->addRef();
        return *this;
    }

    ~Test(){
        this->ref->delRef();
    }
private:
    Ref* ref;
};

int main(){
    Test t1(3);
    Test t2(3);
    Test t3(t2);
}
```

>关键点：
>
>- Ref类内直接用int型的计数值即可，因为具体类中会用引用计数类的指针
>- ref为0时直接delete this，会自动去调析构函数
>- 需要把具体的数据即int* num放入了引用计数类中，这样才能对同一资源分配同一ref进行管理



采用引用计数始终存在的问题：

- 共享的资源一旦被修改了，所有引用该资源的对象都会受影响
- 具体类的数据需要放在引用计数类中，类之间的耦合度太高，复用性太差

==针对资源修改问题采用写时拷贝来解决==：

```c++
void modfiy(int number){
    this->ref->delRef();			//先让原先的引用计数减一
    this->ref = new Ref(number);	//将该类的引用计数重新初始化为新的资源
}
```



### 23、简易智能指针

基本思想：

- 表现得像一种指针（重载指针的一些运算符）
- 能完成资源的自动释放（析构函数）



#### 1、智能指针雏形

```c++
class Student{
public:
    void show(){
        cout << num << endl;
    }
private:
    int num;
};


class SmartPtr{
public:
    SmartPtr(Student* stu){
        this->m_stu = stu;
    }
    ~SmartPtr(){
        if(this->m_stu != nullptr){
            delete m_stu;
            m_stu = nullptr;
        }
    }
    Student* operator->(){
        return this->m_stu;
    }
    Student& operator*(){
        return *m_stu;
    }
private:
    Student* m_stu;
};


int main(){
    Student* stu = new Student();
    SmartPtr sp(stu);
    //stu->show();
    //本来应该是sp->stu->show()，重载运算符后sp->返回stu，所以正确语法为sp->->show()
    sp->show();     //两个连续的->编译器会合并为一个
    (*sp).show();
}
```

>初步总结：
>
>- 实现了简单的资源自动释放以及指针运算符重载（还有很多没重载比如operator bool）
>- 不能对于拷贝构造产生的浅拷贝处理，也就是会重复释放，有以下几种方式处理：
>  - 删除拷贝构造和拷贝赋值实现（auto_ptr及unique_ptr的实现方式）
>  - 使用移动构造函数（auto_ptr及unique_ptr的实现方式）
>  - ==引用计数+写时拷贝（shared_ptr的实现方式）==



#### 2、简易版智能指针

```c++
class SmartPtr;

class Student{
public:
    void show(){
        cout << num << endl;
    }
private:
    int num;
};

class Ref{
public:
    friend class SmartPtr;
    Ref(Student* stu){
        this->m_stu = stu;
        ref = 1;
    }

    ~Ref(){
        cout << "ref de ctor" << endl;
        if(m_stu != nullptr){
            delete m_stu;
            m_stu = nullptr;
        }
    }

    void addRef(){
        this->ref++;
    }

    void delRef(){
        if(--ref == 0){
            delete this;    //此时会调用析构
        }
    }

private:
    Student* m_stu;
    int ref;    //此处可以用ref，因为封装用类的指针表示即可
};

class SmartPtr{
public:
    SmartPtr(){
        ref = nullptr;
    }
    SmartPtr(Student* stu){
        this->ref = new Ref(stu);
    }
    SmartPtr(SmartPtr& sp){
        this->ref = sp.ref;
        this->ref->addRef();
    }
    SmartPtr& operator=(SmartPtr& sp){
        if(this->ref == sp.ref){
            return *this;
        }
        if(this->ref != nullptr){
            this->ref->delRef();
        }
        this->ref = sp.ref;
        this->ref->addRef();
        return *this;
    }
    ~SmartPtr(){
        cout << "de ctor" << endl;
        ref->delRef();
    }
    Student* operator->(){
        return this->ref->m_stu;
    }
    Student& operator*(){
        return *(ref->m_stu);
    }
private:
    Ref* ref;
};


int main(){
    SmartPtr sp(new Student());
    SmartPtr sp2(sp);
    SmartPtr sp3;
    sp3 = sp2;
}
```

>注意事项：
>
>- 会打印三次=="de ctor"==后再打印=="ref de ctor"==
>- 引用计数类管理实际的资源，然后智能指针类采用委托的方式组合一个引用计数类的指针管理引用计数类
>- 上述代码还没有实现写时拷贝的功能，可以再添加，同时智能指针重载运算符也变为多级了



#### 3、模板类实现

```c++
template<class T>
class SmartPtr;

class Student{
public:
    void show(){
        cout << num << endl;
    }
private:
    int num;
};

template<class T>
class Ref{
public:
    friend class SmartPtr<T>;
    Ref(T* stu){
        this->m_stu = stu;
        ref = 1;
    }

    ~Ref(){
        cout << "ref de ctor" << endl;
        if(m_stu != nullptr){
            delete m_stu;
            m_stu = nullptr;
        }
    }

    void addRef(){
        this->ref++;
    }

    void delRef(){
        if(--ref == 0){
            delete this;    //此时会调用析构
        }
    }

private:
    T* m_stu;
    int ref;    //此处可以用ref，因为封装用类的指针表示即可
};

template<class T>
class SmartPtr{
public:
    SmartPtr(){
        ref = nullptr;
    }
    SmartPtr(T* stu){
        this->ref = new Ref<T>(stu);
    }
    SmartPtr(SmartPtr& sp){
        this->ref = sp.ref;
        this->ref->addRef();
    }
    SmartPtr& operator=(SmartPtr& sp){
        if(this->ref == sp.ref){
            return *this;
        }
        if(this->ref != nullptr){
            this->ref->delRef();
        }
        this->ref = sp.ref;
        this->ref->addRef();
        return *this;
    }
    ~SmartPtr(){
        cout << "de ctor" << endl;
        ref->delRef();
    }
    T* operator->(){
        return this->ref->m_stu;
    }
    T& operator*(){
        return *(ref->m_stu);
    }
private:
    Ref<T>* ref;
};


int main(){
    SmartPtr<Student> sp(new Student());
    SmartPtr<Student> sp2(sp);
    SmartPtr<Student> sp3;
    sp3 = sp2;
}
```



### 24、C++的智能指针

#### 1、auto_ptr

- 只实现了基本的析构时释放功能和指针运算符的重载，不能有2个auto_ptr指向同一个对象（重复释放）
- 所有权唯一（unique_ptr也是），当对一个auto_ptr对象拷贝或赋值时，左边对象获取所有权，右边对象失去所有权
- 不能用于数组，即不能创建一个存放auto_ptr的vector容器（关键是没有拷贝和赋值操作）

##### 1）不报错，指向的是不同的地址（虽然内存的值相同）

```c++
int main(){
    auto_ptr<int> ap(new int(3));
    auto_ptr<int> ap2(new int(3));
}
```

##### 2）报错，指向同一个对象

```c++
int main(){
    int* p = new int(3);
    auto_ptr<int> ap(p);
    auto_ptr<int> ap2(p);	//拷贝构造，两个auto_ptr指向同一个对象，重复释放，报错
}
```

##### 3）不报错，转移所有权（拷贝构造也转移）

```c++
int main(){
    int* p = new int(3);
    auto_ptr<int> ap(p);
    auto_ptr<int> ap2;
    ap2 = ap;				//发生赋值时，所有权转移到左边，右边对象失去所有权
}
```



#### 2、unique_ptr

- 禁用了拷贝构造函数以及拷贝赋值函数，仅支持有参构造，移动构造和移动赋值
- 支持STL容器操作，但调用时如push_back需要用move语义
- 其他的实现方式与auto_ptr类似，也是转移所有权（所有权唯一）

##### 1、显式地指出了move的语义

```c++
int main(){
    unique_ptr<int> uptr(new int(3));
    //unique_ptr<int> uptr2 = uptr;		报错
    unique_ptr<int> uptr2 = move(uptr);
}
```



#### ==3、shared_ptr与weak_ptr==

- shared_ptr是一种强指针，weak_ptr是一种弱指针
- shared_ptr实现方式和之前自己实现的智能指针类似，带引用计数以及写时拷贝功能

注意如下会报错：

```c++
int main(){
    int* p = new int(3);
    shared_ptr<int> sp(p);
    shared_ptr<int> sp2(p);		//会报错，应该改为shared_ptr<int> sp2(sp);
}
```

>原因：
>
>- 都用同一个指针进行有参构造初始化，两个shared_ptr对象的内部引用计数都为1，且互不相关
>- 析构时每个shared_ptr都会对内部管理的资源进行释放，发生重复释放
>- 关键在于资源的申请用一个shared_ptr的有参构造，其他shared_ptr用已存在的shared_ptr初始化



#### 1）循环引用

```c++
class Person;
class Son;

class Person{
public:
    Person(){}

    void set(shared_ptr<Son> pSon){
        m_son = pSon;
    }
    ~Person(){
        cout << "person de ctor" << endl;
    }
private:
    shared_ptr<Son> m_son;
};

class Son{
public:
    Son(){}

    void set(shared_ptr<Person> pPerson){
        m_person = pPerson;
    }
    ~Son(){
        cout << "son de ctor" << endl;
    }
private:
    shared_ptr<Person> m_person;
};

int main(){
    Person* p = new Person();
    Son* s = new Son();
    shared_ptr<Person> person(p);
    shared_ptr<Son> son(s);
    person->set(son);
    son->set(person);
    cout << "son.ref : " << son.use_count() << endl;		//输出2
    cout << "person.ref : " << person.use_count() << endl; 	//输出2
}
```

>分析：
>
>- 由于循环引用，释放资源时都依赖互相导致都不能释放，定义的两个析构函数也都没有被调用



#### 2）weak_ptr解决循环引用

- 随意将Person类或者Son类中的一个shared_ptr成员变量改为weak_ptr类型即可
- 比如将Son中的shared_ptr改为weak_ptr，输出如下：

```bash
son.ref : 1
person.ref : 2
son de ctor
person de ctor
```



### 25、空指针调用成员函数

```c++
class Test{
public:
    Test() = default;
    ~Test(){
        cout << "!" << endl;
    }
    void p1(){
        cout << "p1" << endl;
    }
    virtual void p2(){
        cout << "p2" << endl;
    }
};

int main(){
    {
        Test* t = nullptr;
        t->p1();
        t->p2();
    }
    cout << "i am ok" << endl;
    return 0;
}
```

>结论：调p1能打印，调p2报错，输出i am ok但不输出!
>
>- 成员函数地址在编译器确定了，t->p1()等价于A::p1(this),this即t
>- 调用时没有任何操作涉及this指针，p1执行必然没问题
>- 虚函数的调用是间接寻址，需要通过this->vptr才行，this是空指针不指向具体对象没法找到vptr，报错，去掉虚函数就可以调用p2了
>- 同理若哪怕没有virtual关键字，但p1内部对读取或修改了成员变量，也会报错
>- 不会析构，指针只是申明表示其类型（告诉编译器），实际上就是个存地址的变量，不涉及类对象初始化，也不存在析构