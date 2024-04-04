# Unix网络编程——网络基础

### 字节序

- 大端字节序：内存低地址存高位字节

- 小端字节序：内存低地址存低位字节

- 注解：

  - 内存地址从左往右递增（从低位到高位）
  - 字节流如0x01020304（最左边是高位字节，最右边是低位字节，按数字理解即可越前面位数越高）
  - 0x表示16进制，16机制即4位，所以2个16进制数表示1个字节（1个字节8位）
  - 0x01020304表示4个字节

- 检测大小端（Union联合体，共用内存）

- ```c++
  #include <iostream>
  #include <bits/stdc++.h>
  using namespace std;
  
  union test{
      int value;	//4个字节
      char ch;	//1个字节
  };
  
  int main(){
      test t;
      t.value = 0x01020304;
      if(t.ch == 0x01) cout<<"大端存储"<<endl;
      else if(t.ch == 0x04) cout<<"小端存储"<<endl;
      return 0;
  }
  ```

- 网络字节序是大端字节序，不同主机的字节序不同，网络通信前统一为大端

- ```c++
  sockaddr_in addr;
  addr.sin_family = AF_INET;
  
  //点分十进制转化为网络字节序pton
  inet_pton(AF_INET, "172.18.85.192", &addr.sin_addr.s_addr);
  
  //主机字节序转化为网络字节序htons,s表示short，该函数一般用来转端口
  addr.sin_port = htons(20202);
  //htonl对long处理，转IP地址(IP4位，点分十进制每个点都是8个字节),一般直接用inet_pton
  ```



### socket地址

- 表示为结构体sockaddr，是对IP以及端口号的一个封装（想通信必须知道端口和IP的信息，一次性发过去）

- #### 通用socket地址

- ```c++
  #include <bits/socket.h>
  struct sockaddr{
    	sa_family_t sa_family;		//协议族，基本是用PF_INET/AF_INET
      char sa_data[14];			//用于存放socket地址值
  };
  typedef unsigned short int sa_family_t;
  ```

- sa_data若存IPV4地址，只需6个字节（2个字节的端口号和4个字节的IP地址）

- sa_data存不下IPV6地址（26个字节），故增加了新的通用socket地址sockaddr_storage

- #### 专用socket地址（实际使用的）

- 现在使用其他好用的socket地址，sockaddr退化为（void*）的作用，用来做类型转化

- ```c++
  struct sockaddr_in{
    	sa_family_t sa_family;
      in_port_t sin_port;			//ushort,端口号
      struct in_addr sin_addr;	//int,IP地址
      //...剩余填充部分
  };
  
  struct in_addr{
    	in_addr_t s_addr;	//int,IP地址  
  };
  ```

- 所有专用socket地址需要转化为通用地址类型sockaddr，因为socket编程接口全都是sockaddr



### socket相关函数

```c++
#include <arpa/inet.h>

int socket(int domain, int type, int protocol);
//int fd = socket(AF_INET, SOCK_STREAM, 0);

int bind(int sockfd, const struct sockaddr* addr, socklen_t addr_len);
//int ret = bind(lfd, (sockaddr *)&addr, sizeof(addr));

int listen(int sockfd, int backLog);
//ret = listen(lfd, 8);

int accept(int sockfd, const struct sockaddr* addr, socklen_t& addr_len);
//int clientfd = accept(lfd, (sockaddr *)&client, &len);

int connect(int sockfd, const struct sockaddr* addr, socklen_t addr_len);
//int ret = connect(fd, (sockaddr *)&addr, sizeof(addr));

ssize_t write(int fd, const void* buf, size_t count);
ssize_t read(int fd, void* buf, size_t count);
```



### 实现简单的TCP通信

- #### server端（不做error判断）

- ```c++
  #include <bits/stdc++.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <stdio.h>
  #include <stdlib.h>
  using namespace std;
  
  int main(){
      int fd = socket(AF_INET, SOCK_STREAM, 0); 	//创建用于监听的fd
      sockaddr_in addr;
      addr.sin_family = AF_INET;
      //inet_pton(AF_INET, "172.18,85,192", &addr.sin_addr.s_addr);
      addr.sin_addr.s_addr = INADDR_ANY;	//0.0.0.0,服务端才能这么写
      addr.sin_port = htons(1234);
      int ret = bind(fd, (sockaddr*)&addr, sizeof(addr));
      
      sockaddr_in caddr;		//传入传出参数，accept后更改了
      socklen_t len = sizeof(caddr);
      int cfd = accept(fd, (sockaddr*)&caddr, &len);
      
      char IP[16];	//点分十进制，最多15，加一个结束符
      inet_ntop(AF_INET, &caddr.sin_addr.s_addr, sizeof(IP));
      unsigned short port = ntohs(caddr.sin_port);
      printf("IP is %s, port is %d\n", IP, port);
      
      char recvBuf[1024];
      while(1){
          int num = read(cfd, recvBuf, sizeof(recvBuf));
          if(num == -1){
              perror("read");
              exit(-1);
          }else if(num > 0){
              printf("recv is %s\n", recvBuf);
          }else if(num == 0){
              printf("client is closed\n");
              break;
          }
          const char* data = "I am server";
          write(cfd, data, strlen(data));
      }
      close(cfd);
      close(fd);
  }
  ```

- #### client端（不做error判断）

- ```c++
  #include <bits/sdtc++.h>
  #include <unistd.h>
  #include <stdio.h>
  #include <srdlib.h>
  #include <arpa/inet.h>
  using namespace std;
  
  int main(){
      int fd = socket(AF_INET, SOCK_STREAM, 0);
      sockaddr_in addr;
      addr.sin_family = AF_INET;
      inet_pton(AF_INET, "172.18.85.192", &addr.sin_addr.s_addr);
      addr.sin_port = htons(1234);
      int ret = connect(fd, (sockaddr*)&addr, sizeof(addr));
      
      char buf[1024];
      while(1){
          const char* data = "I am client";
          write(fd, data, strlen(data));
          int num = read(fd, buf, sizeof(buf));
          if(num == -1){
              perror("read");
              exit(-1);
          }else if(num > 0){
              printf("client recv data is %s\n", buf);
          }else if(num == 0){
              printf("server is closed\n");
              break;
          }
      }
      close(fd);
  }
  ```

- 这段代码只能接受一个客户端的连接和通信
- 服务端若没有客户端连接就会一直accept阻塞，若有则accept后进入while循环



### 服务器并发——多进程

- 一个父进程，多个子进程

- 父进程负责等待并接受客户端连接（while循环内accept）

- 子进程：完成通信（一旦接收一个客户端连接，fork一个子进程处理）

- 需要考虑父进程回收子进程（SIGCHLD信号）

- ```c++
  #include <stdio.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <string.h>
  #include <signal.h>
  #include <wait.h>
  #include <errno.h>
  
  //信号处理函数，收到SIGCHLD就调用waitpid()回收子进程(具体看进程相关的md文件)
  void recyleChild(int arg) {
      while(1) {
          int ret = waitpid(-1, NULL, WNOHANG);
          if(ret == -1) {
              // 所有的子进程都回收了
              break;
          }else if(ret == 0) {
              // 还有子进程活着
              break;
          } else if(ret > 0){
              // 被回收了
              printf("子进程 %d 被回收了\n", ret);
          }
      }
  }
  
  int main() {
  
      struct sigaction act;
      act.sa_flags = 0;				//表示用第一个回调
      sigemptyset(&act.sa_mask);		//清空临时阻塞信号集
      act.sa_handler = recyleChild;
      // 注册信号捕捉
      sigaction(SIGCHLD, &act, NULL);
  
      // 创建socket
      int lfd = socket(PF_INET, SOCK_STREAM, 0);
      if(lfd == -1){
          perror("socket");
          exit(-1);
      }
      struct sockaddr_in saddr;
      saddr.sin_family = AF_INET;
      saddr.sin_port = htons(9999);
      saddr.sin_addr.s_addr = INADDR_ANY;
  
      // 绑定
      int ret = bind(lfd,(struct sockaddr *)&saddr, sizeof(saddr));
      if(ret == -1) {
          perror("bind");
          exit(-1);
      }
      // 监听
      ret = listen(lfd, 128);
      if(ret == -1) {
          perror("listen");
          exit(-1);
      }
  
      // 不断循环等待客户端连接
      while(1) {
  
          struct sockaddr_in cliaddr;
          int len = sizeof(cliaddr);
          // 接受连接
          int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &len);
          if(cfd == -1) {
              if(errno == EINTR) {  //accept阻塞等待连接，此时若有子进程被回收去内核执行信号处理的回调函数，会出现读/写中断错误即EINTR
                  continue;
              }
              perror("accept");
              exit(-1);
          }
  
          // 每一个连接进来，创建一个子进程跟客户端通信
          pid_t pid = fork();
          if(pid == 0) {
              // 子进程
              // 获取客户端的信息
              char cliIp[16];
              inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, cliIp, sizeof(cliIp));
              unsigned short cliPort = ntohs(cliaddr.sin_port);
              printf("client ip is : %s, prot is %d\n", cliIp, cliPort);
  
              // 接收客户端发来的数据
              char recvBuf[1024];
              while(1) {
                  int len = read(cfd, &recvBuf, sizeof(recvBuf));
  
                  if(len == -1) {
                      perror("read");
                      exit(-1);
                  }else if(len > 0) {
                      printf("recv client : %s\n", recvBuf);
                  } else if(len == 0) {
                      printf("client closed....\n");
                      break;
                  }
                  write(cfd, recvBuf, strlen(recvBuf) + 1);
              }
              close(cfd);
              exit(0);    // 退出当前子进程
          }
  
      }
      close(lfd);
      return 0;
  }
  ```

- 子进程回收通过信号来实现，通过软中断机制切换内核态回收在回到中断处执行

  - 如果在父进程的while中回收，wait肯定不行，阻塞，waitpid感觉也不好
  - 通过信号，自己定义处理函数交给内核负责回收，上下文切换也交给内核

- 因为发生了软中断（SIGCHLD信号产生切换内核态），会出现读/写中断错误即EINTR，需判断



### 服务器并发——多线程

```c++
#include <stdio.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <pthread.h>
#include <iostream>
using namespace std;

class sockInfo{
public:
    int cfd;
    pthread_t tid;
    sockaddr_in addr;
    sockInfo(){
        this->cfd = -1;
        this->tid = -1;
    }
    void sock(int cfd, pthread_t tid, sockaddr_in addr){
        this->cfd = cfd;
        this->tid = tid;
        this->addr.sin_addr.s_addr = addr.sin_addr.s_addr;
        this->addr.sin_port = addr.sin_port;
    }
};

sockInfo sockInfos[128];

void * work(void * arg){
    //需要cfd, client, tid
    sockInfo * sInfo = (sockInfo *)arg;
    char clientIP[16]; 
    inet_ntop(AF_INET, &(sInfo->addr.sin_addr.s_addr), clientIP, sizeof(clientIP));
    unsigned short port = ntohs(sInfo->addr.sin_port);
    printf("client IP : %s Port is : %d\n", clientIP, port);
    char buf[1024];
    while(1){
        int len = read(sInfo->cfd, buf, sizeof(buf));
        if(len == -1){
            perror("read");
            exit(-1);
        }else if(len > 0){
            printf("server receive data : %s\n", buf);
        }else if(len == 0){
            printf("client is closed\n");
            break;
        }
        write(sInfo->cfd, buf, sizeof(buf));
    }
    close(sInfo->cfd);
    return NULL;
}

int main(){
    //cout<<sInfo[33].cfd<<" "<<(int)sInfo[33].tid<<endl;
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    if(lfd == -1){
        perror("socket");
        exit(-1);
    }
    
    sockaddr_in addr;
    addr.sin_family = AF_INET;
    //inet_pton(AF_INET, "172.18.85.192", &addr.sin_addr.s_addr);
    addr.sin_addr.s_addr = INADDR_ANY;
    addr.sin_port = htons(9999);
    int ret = bind(lfd, (sockaddr *)&addr, sizeof(addr));
    if(ret == -1){
        perror("bind");
        exit(-1);
    }

    ret = listen(lfd, 128);
    if(ret == -1){
        perror("listen");
        exit(-1);
    }

    while(1){
        sockaddr_in client;
        int len = sizeof(client);
        int cfd = accept(lfd, (sockaddr *)&client, (socklen_t *)&len);
        sockInfo * sInfo;
        for(int i = 0; i < 128; i++){
            if(sockInfos[i].cfd == -1){
                sInfo = &sockInfos[i];
                break;
            }
            if(i == 127){
                sleep(1);
                i--;
            }
        }
        (*sInfo).sock(cfd, -1, client);  
        cout<<"cfd: "<<sInfo->cfd<<endl;
        pthread_create(&sInfo->tid, NULL, work, sInfo);
        pthread_detach(sInfo->tid);
    }
    close(lfd);
    return 0;
}
```



### 服务器并发——IO多路复用（很重要）

- select、poll、epoll（后续每个单独讲）



### 半关闭和端口复用

- 半关闭状态能接收数据，但不能发数据（通过shutdown()可实现半关闭连接）
  - 实现单向通信
  - shutdown(int sockfd, int how)
    - how : 关读、关写、读写都关闭三种选项
- close()中止一个连接，是减少文件描述符的引用计数，并不直接关闭连接（引用计数为0才关闭）
- 如果一个socket被多个进程共享，close每关闭一次，计数减一，直到计数为0时，才关闭

- 考虑以下场景

  - 服务端占据端口号9999并运行，接受一个客户端连接
  - 关闭服务端，查看状态（netstat -anp | grep 9999）其位于FIN_WAIT2
  - 再关闭客户端，服务端收到客户端的断开请求，进入TIME_WAIT
    - 不清楚的话回忆四次挥手
    - 想要断开连接的一方发送断开请求，此时处于FIN_WAIT1
    - 收到另一方的确认后进入CLOSE_WAIT，还能继续发数据
    - 此时发起断开方进入FIN_WAIT2，不能关（因为另一方可能要发）
    - 直到收到另一方的断开连接请求（该方变成LAST_ACK），收到ACK就关闭连接
    - 最初的发起方进入TIME_WAIT状态，2倍报文时间后关闭连接
  - 此时在2倍报文时间内，该端口9999被占用，若想再开服务器并绑定9999会失败
    - 可能系统崩了导致该情况出现，再开服务器的请求很合理
    - 所以需要避免这种绑定失败

- 通过端口复用技术来实现（setsockopt()可以设置socket属性）

  - 防止服务器重启时之前绑定的端口还未释放
  - 程序突然退出而系统没有释放端口

- ```c++
  int optval = 1;
  setsockopt(lfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));
  ```


- 设置完后，在TIME_WAIT期间qitaserver也可以绑定9999端口