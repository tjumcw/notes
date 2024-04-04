# Linux指令

### 1、动态查看Linux日志

```bash
tail -f 文件名
```



### 2、查询端口

```bash
netstat -anp	#显示所有已开放端口
netstat -tunlp	#列出正在侦听的TCP或UDP端口，-tnlp表示TCP，-unlp表示UDP
netstat -anp | grep 8888 #一般配合grep抓取对应端口号
```



### 3、查看内存使用

#### 1、free

```bash
#free显示系统使用以及空闲的内存情况
```

#### 2、top

```bash
#top可以查看系统的负载，包括进程、CPU负载、内存使用等
```

#### 3、ps -aux

- 若按照内存消耗进行排序，top不好，是动态的阻塞的，用ps -aux

```bash
ps -aux | sort -k4nr #表示对第四列(k)即mem按数字(n)进行降序排序(r)
```

- 若再提取前5行数据

```bash
ps -aux | sort -k4nr | head -n 5	#head默认提取前10行，加-n参数可自定义行数
```

- 若再取出前5行的第四列数据单独查看

```bash
ps -aux | sort -k4nr | head -n 5 | awk -F " " '{print $4}' #默认是空格，不用-F也行
```

