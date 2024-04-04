# Unix网络编程——I/O多路复用

- I/O多路复用使得程序能同时监听多个文件描述符，提高程序的性能
- Linux下实现I/O多路复用的系统调用主要有select、poll、epoll



### 常见I/O模型

- #### 阻塞等待（BIO，blocking）

- accept、read等函数（没数据的时候一直等待）

- 好处：不占用CPU时间片

- 缺点：同一时刻只能处理一个操作，效率低

- 采用多线程或者多进程可以解决上述缺点，能支持并发

- 但是线程或进程会消耗资源，且调度时消耗CPU资源

- #### 非阻塞，忙轮询（NIO，non-blocking）

- 比如设置read函数不阻塞，但需要不断的循环检测read是否有数据到达

- 优点：提高了程序的执行效率

- 缺点：需要占用更多的CPU和系统资源，每次循环O(n)系统调用

  - 有n个客户端连接进来了，每次循环都需要对n个客户端调用read判断是否有数据
  - select虽然也需要遍历一遍，但直接遍历数组标志位为1即可判断，无需通过系统调用read

- ### I/O多路复用

- 委托内核检测哪些文件描述符数据到达了（读写缓冲区）

- select/poll知道有几个fd的事件，epoll能知道是哪几个fd发生事件

- 不需要多线程或多进程能处理多客户端连接



### select

- #### 主旨思想

  - 首先构造一个关于fd的列表，将需要监听的fd添加到该列表中
  - 调用系统函数，监听列表中的fd，直到这些fd的一个或多个进行了I/O该函数返回
    - 该函数是阻塞的
    - 该函数对fd的检测是由内核完成的
  - 返回时，遍历列表可知有哪些fd需要进行I/O操作

- #### select函数

- ```c++
  int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds,
             struct timeval *timeout)
  ```

- nfds表示委托内核检测的最大文件描述符加1（若最大的fd为101，for到102即可，后面不检测）

- readfds表示要检测读的fd的集合，传入传出参数

  - 一般都是检测读操作，即表示接收到数据即对方发过来数据

- writefds表示要检测写的fd的集合，传入传出参数

  - 一般不检测，委托内核检测写缓冲区是不是还可以往里写（不满就可以写）

- ```c++
  sizeof(fd_set) = 128	//128字节即1024位，1024个0/1最多检测1024个fd
  ```

- exceptfds表示检测发生异常的文件描述符的集合，一般也不用

- timeout表示设置的超时时间，有long的tv_sec和tv_usec

  - NULL表示永久阻塞，直到检测到fd有变化
  - 都为0表示不阻塞
  - 其他表示为阻塞对应的时间

- 返回值为-1表示出错，返回0表示未检测到（若永久阻塞不可能返回0），返回n表示检测到n个fd变化

- #### 设置fd_set值的函数

- ```c++
  void FD_CLR(int fd, fd_set* set);	//将fd对应标志位设为0
  int FD_ISSET(int fd, fd_set* set);	//判断fd对应标志位是否为1
  void FD_SET(int fd, fd_set* set);	//设置fd对应标志位为1
  void FD_ZERO(int fd, fd_set* set);	//fd_set一共有1024位，全部初始化为0
  ```

- #### 示例程序（服务端）

- ```c++
  #include <stdio.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <string.h>
  #include <sys/select.h>
  using namespace std;
  
  int main(){
      int lfd = socket(AF_INET, SOCK_STREAM, 0);
      sockaddr_in saddr;
      saddr.sin_family = AF_INET;
      saddr.sin_port = htons(9999);
      saddr.sin_addr.s_addr = INADDR_ANY;
      
      bind(lfd, (sockaddr*)&saddr, sizeof(saddr));
      listen(lfd, 8);
      
      fd_set rdset, tmp;
      FD_ZERO(&rd_set);
      FD_SET(lfd, &rd_set);	//若监听的fd有数据，即表示有客户端要连接，所以要加入集合委托内核检测
      int maxfd = lfd;
      
      while(1){
          tmp = rd_set;
          int ret = select(maxfd + 1, &tmp, NULL, NULL, NULL);
          if(ret == -1){
              perror("select");
              exit(-1);
          }else if(ret == 0){
              continue;
          }else if(ret > 0){
              if(FD_ISSET(lfd, &tmp)){
                  //表示有新的客户端连接进来了
                  sockaddr_in caddr;
                  int len = sizeof(caddr);
                  int cfd = accept(lfd, (sockaddr*)&caddr, &len);
                  FD_SET(cfd, &rd_set);
                  maxfd = maxfd > cfd ? maxfd : cfd;
              }
              for(int i = lfd + 1; i <= maxfd; i++){	//lfd肯定是最前面的，因为内核每次会分配当前最小的fd
                  if(FD_ISSET(i, &tmp)){
                      //表示fd对应的客户端发来了数据
                      char buf[1024] = {0};
                      int len = read(i, buf, sizeof(buf));
                      if(len == -1){
                          perror("read");
                          exit(-1);
                      }else if(len == 0){
                          printf("client closed\n");
                          close(i);
                          FD_CLR(i, &rd_set);
                      }else if(len > 0){
                          printf("read buf = %s\n", buf);
                          write(i, buf, strlen(buf) + 1);		//回写，无伤大雅的操作
                      }
                  }
              }	
          }
      }
      close(lfd);
      return 0;
  }
  ```

- 需要定义两个fd_set，一个表示需要监听的fd集合rd_set（不通过select交给内核操作），另一个表示传入select后内核修改后的集合tmp

- rd_set可以通过FD_SET和FD_CLR函数操作，根据传入select的tmp的遍历结果，改rd_set（增加或减少需要监听的fd）

- #### select缺点

- 每次调用select都需要把fd集合从用户态拷贝到内核态，若fd很多时开销会很大

- 内核需要遍历fd，fd很多时开销也很大

- select支持的文件描述符数量太少了，默认是1024，unsigned long 的128大小的数组，一共128 * 8 = 1024个字节

- fds集合不能重用，每次都要重置



### poll（对select的改进）

- #### poll函数

- ```c++
  int poll (struct pollfd *__fds, nfds_t __nfds, int __timeout);
          (1) fds : 结构体数组,需要检测的文件描述符集合,传入传出参数
          (2) nfds : 第一个参数数组中最后一个有效元素的下标
          (3)timeout : 0表示不阻塞,-1表示永久阻塞，>0表示阻塞时长
  ```

- 返回值同select()，其他与select类似，主要区别是第一个参数为pollfd类型的结构体数组

- ```c++
  struct pollfd
      {
          int fd;					/* 委托内核检测的fd  */
          short int events;		/* 委托内核检测什么事件  */
          short int revents;		/* 检测到的fd实际发生事件 */
      };
  
  //若需同时检测读、写事件
  struct pollfd myfd;
  myfd.fd = 5;
  myfd.events = POLLIN | POLLOUT;
  ```

- #### 与select的对比

- 克服了select的1024限制，传入的pollfd*指针对应的数组可以任意大

- 克服了select对于检测集合不能重用的缺点，每次修改revents即可，遍历也只需判断revents

- 每次调用仍需拷贝到内核，内核仍需遍历fd（均在fd很多时开销大）

- #### 示例程序（服务端）

- ```c++
  #include <stdio.h>
  #include <arpa/inet.h>
  #include <unistd.h>
  #include <stdlib.h>
  #include <string.h>
  #include <poll.h>
  
  int main() {
      // 创建socket
      int lfd = socket(PF_INET, SOCK_STREAM, 0);
      struct sockaddr_in saddr;
      saddr.sin_port = htons(9999);
      saddr.sin_family = AF_INET;
      saddr.sin_addr.s_addr = INADDR_ANY;
  
      // 绑定
      bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));
  
      // 监听
      listen(lfd, 8);
  
      // 初始化检测的文件描述符数组
      struct pollfd fds[1024];
      for(int i = 0; i < 1024; i++) {
          fds[i].fd = -1;
          fds[i].events = POLLIN;
      }
      fds[0].fd = lfd;
      int nfds = 0;		
  
      while(1) {
  
          // 调用poll系统函数，让内核帮检测哪些文件描述符有数据
          int ret = poll(fds, nfds + 1, -1);
          if(ret == -1) {
              perror("poll");
              exit(-1);
          } else if(ret == 0) {
              continue;
          } else if(ret > 0) {
              // 说明检测到了有文件描述符的对应的缓冲区的数据发生了改变
              if(fds[0].revents & POLLIN) {
                  // 表示有新的客户端连接进来了
                  struct sockaddr_in cliaddr;
                  int len = sizeof(cliaddr);
                  int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
  
                  // 将新的文件描述符加入到集合中
                  for(int i = 1; i < 1024; i++) {		//需要遍历数组，有空的最小的fd赋给客户端
                      if(fds[i].fd == -1) {
                          fds[i].fd = cfd;
                          fds[i].events = POLLIN;
                          break;
                      }
                  }
  
                  // 更新最大的文件描述符的索引
                  nfds = nfds > cfd ? nfds : cfd;
              }
  
              for(int i = 1; i <= nfds; i++) {	//0索引表示监听的文件描述符，只需检测到当前最大索引
                  if(fds[i].revents & POLLIN) {
                      // 说明这个文件描述符对应的客户端发来了数据
                      char buf[1024] = {0};
                      int len = read(fds[i].fd, buf, sizeof(buf));
                      if(len == -1) {
                          perror("read");
                          exit(-1);
                      } else if(len == 0) {
                          printf("client closed...\n");
                          close(fds[i].fd);
                          fds[i].fd = -1;
                      } else if(len > 0) {
                          printf("read buf = %s\n", buf);
                          write(fds[i].fd, buf, strlen(buf) + 1);
                      }
                  }
              }
  
          }
  
      }
      close(lfd);
      return 0;
  }
  ```



### epoll

- #### epoll_create

- 通过epoll_create在内核中创建了一个类型为eventpoll（即称为epoll）的实例

- 返回指向该内核区中epoll实例的fd，不能直接操作（内核区），通过系统调用API

- ```c++
  int epfd = epoll_create(int num);	//num可以任意，大于0即可
  ```

- #### eventpoll结构体

- ```c++
  struct eventpoll{
     	//...
     	struct rb_root rbr;         //记录需要内核检测的文件描述符，红黑树结构
     	struct list_head rdlist;    //双向链表，rdlist表示ready_list就绪列表，存检测到改变的fd
     	//...
  };
  ```

- 首先，红黑树结构遍历效率比线性数组高

- 其次，select/poll需要把要检测的fd传入某个容器，并拷贝到内核中处理

- 现在是直接把需要检测的fd传入内核中的epoll实例中的红黑树（通过epoll_ctl），节省开销

  - 但检测到的rd_list中的fd需要从内核拷贝到用户（开销比前者小很多）

- ```c++
  int epoll_ctl(epfd, op, fd, *epev); 
     	(1)epfd : eoll实例的文件描述符
      (2)op   : 操作类型，如EPOLL_CTL_ADD,即将fd放到epfd对应的epoll实例的rbr中，用于监听
      (3)fd   : 需要加入或删除到epoll实例rbr中的文件描述符
      (4)epev : 描述监听fd哪种事件，若op为删除则直接传NULL即可，为epoll_event类型的结构体
      (5)return : = 0 成功, -1表示失败
  ```

- #### epoll_event结构体

- ```c++
  struct epoll_event{
   	uint_32 events;         //需要监听的事件，常用EPOLLIN
     	epoll_data_t data;      //联合体，一般只用到data.fd
  };
  ```

- #### epoll_wait（检测函数）

- ```c++
  int epoll_wait(epfd, *events, maxevents, timeout);
    	(1)epfd      : epoll实例的文件描述符,内核中的epoll实例的红黑树保存了所有需要检测的fd
      (2)events    : 传出参数,保存了发生了变化的文件描述符的信息
      (3)maxevents : 第二个参数结构体数组的大小
      (4)timeout   : 阻塞时间,0表示不阻塞,-1表示阻塞, > 0表示阻塞时长
      (5)return    : > 0 返回发生变化的文件描述符个数, -1表示失败
  ```

- #### 整体流程

- 通过epoll_create()创建一个epoll实例（即eventpoll的结构体对象，其在内核中）得到epfd

- 将需要监听的fd通过epoll_ctl()加入epoll实例的红黑树中，并通过epoll_event对象设置监听事件

  - 一开始传入epoll的就是单个服务端绑定的监听lfd

- 调用epoll_wait()监听内核区中epoll实例中红黑树所存的fd其对应的事件，返回检测到的fd数量

  - 需要创建一个epoll_event数组epevs传入epoll_wait()函数，其为传入传出参数
  - rdlist中的fd会拷贝到该数组中返回用户区，遍历就可以操作所有发生事件的fd

- 通过epoll_wait()返回的len去遍历epevs，通过epevs[i].data.fd取得每个检测到事件的fd

  - 若为lfd，则调用accept()得到客户端fd，并通过epoll_ctl加入epoll实例中
  - 若不为lfd，则对发生的事件进行处理（一般都是监听EPOLLIN，即收到数据）

- #### 示例代码（服务端）

- ```c++
  #include <stdio.h>
  #include <stdlib.h>
  #include <unistd.h>
  #include <arpa/inet.h>
  #include <sys/epoll.h>
  #include <string.h>
  using namespace std;
  
  int main(){
      int lfd = socket(AF_INET, SOCK_STREAM, 0);
      sockaddr_in saddr;
      saddr.sin_family = AF_INET;
      saddr.sin_port = htons(9999);
      saddr.sin_addr.s_addr = INADDR_ANY;
      bind(lfd, (sockaddr*)&saddr, sizeof(addr));
      
      int epfd = epoll_create(1234);
      listen(lfd, 8);
      
      epoll_event epev;
      epev.data.fd = lfd;
      epev.events = EPOLLIN;  
      epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &epev);
      
      epoll_events epevs[1024];
      
      while(1){
          int num = epoll_wait(epfd, &epevs, 1024, -1);	//-1表示永久阻塞，直到检测到至少一个fd事件发生
          if(ret == -1){
              perror("epoll");
              exit(-1);
          }
          for(int i = 0; i < num; i++){
              int curfd = epevs[i].data.fd;
              if(curfd == lfd){
                  sockaddr_in caddr;
                  int len = sizeof(caddr);
                  int cfd = accept(lfd, (sockaddr*)&caddr, &len);
                  
                  epev.data.fd = cfd;
                  epev.events = EPOLLIN;
                  epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &epev);
              }else{
                  char buf[1024] = {0};
                  int len = read(curfd, buf, sizeof(buf));
                  if(len == -1){
                      perror("read");
                      exit(-1);
                  }else if(len == 0){
                      printf("client closed\n");
                      epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                      close(curfd);
                  }else if(len > 0){
                      printf("read buf = %s\n", buf);
                      write(curfd, buf, strlen(buf) + 1);
                  }
              }
          }
      }
      close(lfd);
      close(epfd);
      return 0;
  }
  ```

- #### epoll的两种工作模式