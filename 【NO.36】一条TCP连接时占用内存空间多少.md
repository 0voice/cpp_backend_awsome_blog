# 【NO.36】一条TCP连接时占用内存空间多少?

计算网络连接所占用的[内存](https://link.zhihu.com/?target=https%3A//so.csdn.net/so/search%3Fq%3D%E5%86%85%E5%AD%98%26spm%3D1001.2101.3001.7020)，需要知道申请哪些内存，以及一条TCP连接需要占用多少内存空间。sock_inode_cache、tcp_sock、dentry、file 内存对象占用多少空间？

## 1.内存分配关系

### 1.1 物理内存分配关系

![img](https://pic3.zhimg.com/80/v2-ac9916dd2db25b1b6c623d9c4f11b87a_720w.webp)

### 1.2 TCP申请对象之间关系

![img](https://pic2.zhimg.com/80/v2-057e68e58cddac8511de9385b74e6361_720w.webp)

## 2.实测内存开销

### 2.1 客户端内核参数调整

```text
# vi /etc/sysctl.conf
#端口号
net.ipv4.ip_local_port_range = 5000     65000
net.ipv4.tcp_tw_reuse = 0
#net.ipv4.tcp_tw_recycle        = 0
#time_wait状态
net.ipv4.tcp_max_tw_buckets = 60000
fs.file-max=210000
fs.nr_open=210000
 
 
# sysctl -p
```



```text
#vim /etc/security/limits.conf
 
*       soft    nofile  1010485
*       hard    nofile  1011000
```

### 2.2 服务端内核参数调整

```text
# vi /etc/sysctl.conf
net.core.rmem_max = 8388608
net.ipv4.tcp_rmem = 4096        131072  8388608
 
net.core.wmem_max = 8388608
net.ipv4.tcp_wmem = 4096        16384   8388608
 
 
fs.file-max=2100000
fs.nr_open= 2100000
net.core.somaxconn = 1024
 
# sysctl -p
```



```text
#vim /etc/security/limits.conf
 
*       soft    nofile  1010485
*       hard    nofile  1011000
```

服务端与客户端的缓存清理

```text
echo "3" > /proc/sys/vm/drop_caches
```



## 3.服务端开销计算

### 3.1 slab内存在连接前后输出以及计算

```text
root@wy-virtual-machine:~# cat /proc/meminfo | grep Slab
Slab:             147724 kB
root@wy-virtual-machine:~# 
root@wy-virtual-machine:~# cat /proc/meminfo | grep Slab
Slab:             180136 kB
root@wy-virtual-machine:~# 
root@wy-virtual-machine:~# cat /proc/meminfo | grep Slab
Slab:             195816 kB
```

180136 - 147724 = 32412 / 10000 = 3.2412K ESTABLISH

195816 - 180136 = 15680 / 10000 = 1.568K CLOSE_WAIT

### 3.2 TCP对象占用内存

slabtop

```text
 Active / Total Objects (% used)    : 877387 / 896605 (97.9%)
 Active / Total Slabs (% used)      : 31483 / 31483 (100.0%)
 Active / Total Caches (% used)     : 119 / 167 (71.3%)
 Active / Total Size (% used)       : 298213.95K / 308366.00K (96.7%)
 Minimum / Average / Maximum Object : 0.01K / 0.34K / 8.00K
 
  OBJS ACTIVE  USE OBJ SIZE  SLABS OBJ/SLAB CACHE SIZE NAME                   
155210 155210 100%    0.02K    913      170      3652K lsm_file_cache
 99712  99712 100%    0.03K    779      128      3116K kmalloc-32
 90048  90048 100%    0.19K   4288       21     17152K dentry
 64480  64480 100%    0.12K   2015       32      8060K kernfs_node_cache
 60352  60352 100%    0.25K   3772       16     15088K filp
 56069  56069 100%    0.81K   2951       19     47216K sock_inode_cache
 50050  50050 100%    2.19K   3575       14    114400K TCP
 46949  45281  96%    0.20K   2471       19      9884K vm_area_struct
 34764  33034  95%    0.62K   2897       12     23176K inode_cache
```

2.19K（TCP）+ 0.25K（flip）+0.19K（dentry）+0.81K（sock_inode_cache） = 3.44K



TCP对象与slab内存比较 稍微差了一点点

CLOSE_WAIT状态开销也不小有1.56 k

### 3.3 TCP对象内存开销总结

| 缓存             | 大小  | 对象                |
| ---------------- | ----- | ------------------- |
| sock_inode_cache | 0.81K | struct socket_alloc |
| TCP              | 2.19K | struct tcp_sock     |
| dentry           | 0.19K | struct dentry       |
| file             | 0.25K | struct file         |

## 4.参数设置

### 4.1 linux打开文件限制配置

1）进程级别，单个进程打开个数 soft nofile 与 fs.nr_open

2）系统级别，fs.file-max，这个参数不限制root用户

- 如果设置soft nofile 那么 hard nofile也要同步设置
- 如果设置hard nofile ,fs.nr_open也要一起调整
- 如果hard nofile设置比fs.nr_open大，会导致用户无法登录

设置参考

```text
# vi /etc/sysctl.conf
fs.file-max=1100000
fs.nr_open= 1100000  #设置比hard大
 
#sysctl -p
 
# vi /etc/security/limits.conf
*       soft    nofile  1000000
*       hard    nofile  1000000
```

### 4.2 TCP内存设置

读写设置，每个TCP 建立连接大概3.44K，在设置总内存时乘上数量

```text
#sysctl -a 
 
net.core.rmem_max = 8388608
net.ipv4.tcp_rmem = 4096        131072  8388608
 
net.core.wmem_max = 8388608
net.ipv4.tcp_wmem = 4096        16384   8388608
```

### 4.3 查看连接状态

```text
# netstat -antp
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN      689/systemd-resolve 
tcp        0      0 127.0.0.1:631           0.0.0.0:*               LISTEN      837/cupsd           
tcp6       0      0 ::1:631                 :::*                    LISTEN      837/cupsd
```

### 4.4 其他

- netstat -antp | grep EST |wc -l 查看单个状态并统计数量
- slabtop 查看slab对象内存的申请情况，以及使用量

原文地址：https://zhuanlan.zhihu.com/p/579154045   

作者：linux