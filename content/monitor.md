# 第十章：监控CentOS的系统状态

## 如何查看系统变量
- 执行`env`可以查看系统的环境变量，如主机的名称、当前用户的SHELL类型、当前用户的家目录、当前系统所使用的语言等。
- 执行`set`可以看到系统当前所有的变量，其中包括了：
    - 系统的所有预设变量，这其中既包括了`env`所显示的环境变量，也包含了其它许多预设变量。
    - 用户自定义的变量。

## 监控系统的状态
### 使用w命令查看当前系统整体上的负载
使用`w`命令可以查看当前系统整体上的负载：
```
# w
 20:33:11 up 309 days, 10:03,  1 user,  load average: 0.00, 0.01, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
root     pts/0    113.102.224.86   20:33    3.00s  0.00s  0.00s w
```
需要关注的是第一行的最后一个部分——**load average**，这里的3个数字分别表示了系统在1分钟/5分钟/15分钟内的平均负载值，值越大说明服务器压力就越大。

那么，如何看负载是不是太满了呢？其实这个值是与服务器的物理CPU做对比的，那么只要负载值不超过物理CPU数量即可；如当前服务器有两个CPU，那么就尽量不要让负载值超过2。

### 用vmstat命令查看系统具体负载
`vmstat`命令打印的结果共分为6部分：procs、memory、swap、io、system和cpu，其中又有许多的细分字段，这里我们重点关注r、b、si、so、bi、bo、wa字段。

```
# vmstat 1 5
procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
 r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
 0  0      0 707116  15392 177284    0    0    81     5  110  267  0  0 99  1  0
 0  0      0 707100  15392 177284    0    0     0     0  121  274  1  0 99  0  0
 0  0      0 706712  15392 177284    0    0     0     0  107  254  0  1 99  0  0
 0  0      0 706696  15392 177284    0    0     0    40   94  235  0  0 100  0  0
 0  0      0 706712  15392 177284    0    0     0     0   93  231  0  0 100  0  0
```

- r(run)：表示正在运行或等待CPU时间片的进程数，**该数值如果长期大于服务器CPU的个数，则说明CPU资源不够用了**。
- b(block)：表示等待资源(I/O、内存等)的进程数。举个例子，当磁盘读写非常频繁时，写数据就会非常慢，此时CPU运算很快就结束了，但进程需要把计算的结果写入磁盘，这样进程的任务才算完成，因此这个任务只能慢慢等待磁盘了。**该数值如果长时间大于1，则需要查一下具体是缺的哪项资源**。
- si和so：分别表示由交换区写入内存的数据量以及由内存写入交换区的数据量；**一般情况下，si、so的值都为0，如果si、so的值长期不为0，则表示系统内存不足**，需要借用磁盘上的交换区，由于这往往对系统性能影响极大，因此需要考虑是否增加系统内存。
- bi和bo：分别表示从块设备读取数据的量和往块设备写入数据的量；**如果这两个值很高，那么表示磁盘I/O压力很大**。
- wa：表示I/O等待所占用CPU的时间百分比。**wa值越高，说明I/O等待越严重。如果wa值超过20%，说明I/O等待严重。**
 
另外，`vmstat`命令后可带两个数字，第一个数字表示每多少秒打印一次结果，第二个数字表示总共打印多少次结果；如果只有第一个数字，则会不停地打印结果，直到你终止该命令。

### 用top命令显示进程所占的系统资源
`top`命令的结果有很多信息，但我们主要用它来监控进程所占的系统资源。top命令的结果每隔3秒变1次，它的特点是把占用系统资源（CPU、内存、磁盘I/O等）最高的进程放到最前面。
```
  PID USER      PR  NI    VIRT    RES    SHR S %CPU %MEM     TIME+ COMMAND
    1 root      20   0   41060   3576   2396 S  0.0  0.4   0:00.89 systemd
    2 root      20   0       0      0      0 S  0.0  0.0   0:00.00 kthreadd
    3 root      20   0       0      0      0 S  0.0  0.0   0:00.00 ksoftirqd/0
```
这里面我们主要关注RES（所占内存大小）、%CPU、%MEM（占用内存的百分比）、COMMAND这4个字段。

另外，如果需要一次性打印系统资源的使用情况，可以使用`top - bn1`。

### 监控网卡流量

#### 使用sar命令查看网卡流量历史记录
使用`sar`命令前可能需要先进行安装：`yum install -y sysstat`。

使用方法是：`sar -n DEV`，第一次使用时会报错，因为还没有生成相应的数据记录。打印出来的结果里有很多字段，我们关注`rxpck/s`和`rxkB/s`。
- `rxpck/s`表示网卡每秒收取的包的数量，如果数值大于4000则考虑是被攻击了。
- `rxkB/s`表示网卡每秒收取的数据量（单位为KB）。

#### 使用nload命令监控网卡实时流量
使用`nload`前需进行安装：`yum install -y epel-release;yum install -y nload`。

使用起来也很简单，直接使用`nload`命令则可动态显示当前的网卡流入/流出的流量。

### 使用free命令查看内存使用状况
为了检查内存是否够用，除了`vmstat`外，我们还可以使用更直接有效的`free`命令：`free -h`。

```
# free -h
              total        used        free      shared  buff/cache   available
Mem:           992M        141M        462M        516K        388M        714M
Swap:          1.0G          0B        1.0G
```
- total：内存总量，相当于used+free+buff/cache=used+available。
- used：已真正使用的内存量。
- free：剩余（未被分配）的内存量。
- shared：不关注。
- buff/cache：缓解CPU和I/O速度差距所用的内存缓存区，由系统预留出来备用，但如果剩余内存都不够用了，那么这部分也是可以挪用出来供服务来使用的。
- available：可用内存，相当于free+buff/cache。

### 使用ps命令查看系统进程
与`top`命令类似，`ps`命令也是用来查看系统具体进程占用资源的情况；由于`top`命令本身是动态的，而`ps`命令是非动态的（相当于执行命令时的一个快照），因此`ps`命令的功能实际上更接近于`top -bn1`。

### 使用netstat命令查看网络情况
`netstat`的功能非常强大，这里举两个实际使用场景：`netstat -lnp`（打印当前系统启动哪些端口）和`netstat -an`（打印网络连接状况）。

```
# netstat -an
Proto Recv-Q Send-Q Local Address           Foreign Address         State  
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN     
tcp        0      0 172.18.63.215:36492     140.205.140.205:80      ESTABLISHED
```

```
netstat -lnp
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 0.0.0.0:3306            0.0.0.0:*               LISTEN      1604/mysqld         
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      1636/httpd          
tcp        0      0 0.0.0.0:21              0.0.0.0:*               LISTEN      1639/vsftpd         
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      674/sshd            
tcp        0      0 0.0.0.0:443             0.0.0.0:*               LISTEN      1636/httpd  
```
