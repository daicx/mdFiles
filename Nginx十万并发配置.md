---
title: Nginx十万并发配置
date: 2019-04-19 23:56:04
tags:
- Nginx并发配置
categories:
- Nginx
- Nginx十万并发
---

# Nginx 数十万并发设置

[![img](1.png)

<!-- more -->](1.png)

## 1.系统参数

```
[root@node101 ~]# ulimit -a  
core file size          (blocks, -c) 0                        　　　　　　　　 #core文件的最大值为100 blocks。
data seg size           (kbytes, -d) unlimited               　　　　       #进程的数据段可以任意大。
scheduling priority             (-e) 0                        　　　　　　      #指定调度优先级
file size               (blocks, -f) unlimited               　　　　　　       #文件可以任意大。
pending signals                 (-i) 31152                   　　　　　　 　 #最多有31152个待处理的信号。
max locked memory       (kbytes, -l) 64                        　　　　   #一个任务锁住的物理内存的最大值为64KB。
max memory size         (kbytes, -m) unlimited               　　 　　#一个任务的常驻物理内存的最大值。
open files                      (-n) 1024                    　　　　　　　　   #一个任务最多可以同时打开1024的文件。
pipe size            (512 bytes, -p) 8                        　　　　　　     #管道的最大空间为4096字节。
POSIX message queues     (bytes, -q) 819200                    　　   #POSIX的消息队列的最大值为819200字节。
real-time priority              (-r) 0                        　　　　　　　　  #指定实时优先级
stack size              (kbytes, -s) 8192                    　　　　　　　　#进程的栈的最大值为8192字节。
cpu time               (seconds, -t) unlimited                　　　　　　#进程使用的CPU时间。
max user processes              (-u) 31152                    　　　　　　#当前用户同时打开的进程（包括线程）的最大个数为31152。
virtual memory          (kbytes, -v) unlimited                　　　　　#没有限制进程的最大地址空间。
file locks                      (-x) unlimited                　　　　　　　　#所能锁住的文件的最大个数没有限制。
```

## 2.nginx参数调优

### worker进程数是否合理

```
worker_processes auto;
```

nginx worker进程数量，建议和cpu核数相当.

分析: worker_processes是worker进程的数量，默认值为auto，这个优化值受很多因素的影响，如果不确定的话，将其设置为CPU内核数是一个不错的选择

### worker连接数是否合理

```
worker_connections 1024;
```

单个worker进程可服务的客户端数量（根据并发要求进行更改）

分析:worker_connections 设置了一个worker进程可以同时打开的链接数，有高并发需求时，按照需求进行设定。链接**最大数目= worker_processes \* worker_connections**

### worker可打开最大文件数是否合理

```
worker_rlimit_nofileulimit -n
```

Nginx 最大可用文件描述符数量linux可同时打开最大文件数

分析:worker_rlimit_nofile为Nginx单个worker进程最大可用文件描述符数量，和链接数相同；**最大数目= worker_processes  \* worker_rlimit_nofile**。
 同时需要配置操作系统的 "ulimit -n XXXX"，或者在 /etc/security/limits.conf 中配置。 来达到对应配置。
 配置数量按照需求情况设定，不建议配置较高的值。

### multi_accept

```
multi_accept on;
```

是否尽可能的接受请求（建议打开）

分析: 

multi_accept 的作用是告诉 nginx 在收到新链接的请求通知时，尽可能接受链接。当然，得让他开着。

### 读写方式是否合理

```
sendfile on;
```

直接从磁盘上读取数据到操作系统缓冲（建议打开）

分析:

​	在 sendfile 出现之前，为了传输这样的数据，需要在用户空间上分配一块数据缓存，使用 read() 从源文件读取数据到缓存，然后使用 write() 将缓存写入到网络。
 sendfile() 直接从磁盘上读取数据到操作系统缓冲。由于这个操作是在内核中完成的，sendfile() 比 read() 和 write() 联合使用要更加有效率。

### tcp_nopush

```
tcp_nopush on;
```

在一个包中发送全部头文件（建议打开,默认为打开状态）

分析:

配置 nginx 在一个包中发送全部的头文件，而不是一个一个发送。
这个选项使服务器在 sendfile 时可以提前准备 HTTP 首部，能够达到优化吞吐的效果。

### tcp_nodelay

```
tcp_nodelay on;
```

配置 nginx 不缓存数据，快速发送小数据

分析:

不要缓存 data-sends （关闭 [Nagle 算法](http://blog.csdn.net/yuan1125/article/details/51536490)），这个能够提高高频发送小数据报文的实时性。**系统存在高频发送小数据报文的时候，打开它。**

### 客户端在keep-alive的请求数量是否合理

```
keepalive_requests 100000;

```

建议配置较大的值

分析:

设置通过"一个存活长连接"送达的最大请求数（默认是100，建议根据客户端在"keepalive"存活时间内的总请求数来设置）
当送达的请求数超过该值后，该连接就会被关闭。（通过设置为5，验证确实是这样）
**建议设置为一个较大的值**

### 客户端通信超时时间是否合理

```
keepalive_timeout   65;

```

超时时间.

分析:

​	配置连接 keep-alive 超时时间，服务器将在超时之后关闭相应的连接。
 指定了与客户端的 keep-alive 链接的超时时间。服务器会在这个时间后关闭链接。
 keep-alive设置过小客户端和服务器会频繁建立连接；设置过大由于连接需要等待keep-alive才会关闭，所以会造成不必要的资源浪费。

### 后端服务器超时时间是否合理

```
proxy_connect_timeout //默认60s

proxy_read_timeout //默认60s

proxy_send_timeout //默认60s

```

默认60s

分析: 

proxy_connect_timeout :后端服务器连接的超时时间_发起握手等候响应超时时间

proxy_read_timeout:连接成功后*等候后端服务器响应时间*其实已经进入后端的排队之中等候处理（也可以说是后端服务器处理请求的时间）

proxy_send_timeout :后端服务器数据回传时间_就是在规定时间之内后端服务器必须传完所有的数据

### reset_timedout_connection配置是否合理

```
reset_timedout_connection on;

```

服务器在客户端停止发送应答之后关闭连接.

分析:

​	允许server在client停止响应以后关闭连接,释放分配给该连接的内存。当有大并发需求时，建议打开。

### request body读超时时间是否合理

```
client_body_timeout 10;

```

默认60s.

分析;

该指令设置请求体（request body）的读超时时间。仅当在一次readstep中，没有得到请求体，就会设为超时。超时后，nginx返回HTTP状态码408(“Request timed out”)

### types_hash_max_size

```
types_hash_max_size 2048;

```

types_hash_max_size越小，消耗的内存就越小，但散列key的冲突率可能上升。

分析;

​	ypes_hash_max_size影响散列表的冲突率。types_hash_max_size越大，就会消耗更多的内存，但散列key的冲突率会降低，检索速度就更快。types_hash_max_size越小，消耗的内存就越小，但散列key的冲突率可能上升。

### 日志记录是否合理

```
access_log  /var/log/nginx/access.log  main;      #默认打开

```

access_log和error_log
access_log建议关闭，降低磁盘IO提高速度error_log

### 压缩选用是否合理

```
# nginx默认不进行压缩处理
    gzip on;
    gzip_disable "msie6";
    gzip_proxied any;
    gzip_comp_level 9;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;
设置数据压缩设置禁止压缩是否压缩基于请求、响应的压缩压缩等级压缩类型

```

gzip  ：设置nginx gzip压缩发送的数据，**建议打开** 

gzip_disable：为指定的客户端禁用gzip功能，(IE5.5和IE6 SP1使用msie6参数来禁止gzip压缩 )指定哪些不需要gzip压缩的浏览器(将和User-Agents进行匹配),依赖于PCRE库

gzip_proxied：Nginx作为反向代理的时候启用，根据某些请求和应答来决定是否在对代理请求的应答启用gzip压缩，是否压缩取决于请求头中的“Via”字段，指令中可以同时指定多个不同的参数，意义如下（

无特殊需求建议设置为any）： 

- off(关闭所有代理结果的数据的压缩)
- expired - 启用压缩，如果header头中包含 "Expires" 头信息
- no-cache - 启用压缩，如果header头中包含 "Cache-Control:no-cache" 头信息
- no-store - 启用压缩，如果header头中包含 "Cache-Control:no-store" 头信息
- private - 启用压缩，如果header头中包含 "Cache-Control:private" 头信息
- no_last_modified - 启用压缩,如果header头中不包含 "Last-Modified" 头信息
- no_etag - 启用压缩 ,如果header头中不包含 "ETag" 头信息
- auth - 启用压缩 , 如果header头中包含 "Authorization" 头信息
- any - 无条件启用压缩

gzip_comp_level：数据压缩的等级。等级可以是 1-9 的任意一个值，9 表示最慢但是最高比例的压缩

gzip_types：设置进行 gzip 的类型；对于多数以文本为主的站点来说，文本自身内容占流量的绝大部分。虽然单个文本体积并不算大，但是如果数量众多的话，流量还是相当可观。启用GZIP以后，可以大幅度减少所需的流量.

### linux系统是否启用epoll

```
use epoll;

```

Linux 关键配置，允许单个线程处理多个客户端请求。

分析:

​	Linux 关键配置，允许单个线程处理多个客户端请求。

### 缓存设置是否合理

```
open_file_cache max=200000 inactive=20s;//缓存最大数目及超时时间检测

open_file_cache_valid 30s; //缓存源文件是否超时的间隔时间

open_file_cache_min_uses 2;//缓存文件最小访问次数

open_file_cache_errors on;//缓存文件错误信息

```

## 3.服务器内核参数调优

**net.ipv4.tcp_max_tw_buckets = 6000**

timewait 的数量，默认是180000。

**net.ipv4.ip_local_port_range = 1024 65000**

允许系统打开的端口范围。

**net.ipv4.tcp_tw_recycle = 1**

启用timewait 快速回收。

**net.ipv4.tcp_tw_reuse = 1**

开启重用。允许将TIME-WAIT sockets 重新用于新的TCP 连接。

**net.ipv4.tcp_syncookies = 1**

开启SYN Cookies，当出现SYN 等待队列溢出时，启用cookies 来处理。

**net.core.somaxconn = 262144**

web 应用中listen 函数的backlog 默认会给我们内核参数的net.core.somaxconn 限制到128，而nginx 定义的NGX_LISTEN_BACKLOG 默认为511，所以有必要调整这个值。

**net.core.netdev_max_backlog = 262144**

每个网络接口接收数据包的速率比内核处理这些包的速率快时，允许送到队列的数据包的最大数目。

**net.ipv4.tcp_max_orphans = 262144**

系统中最多有多少个TCP 套接字不被关联到任何一个用户文件句柄上。如果超过这个数字，孤儿连接将即刻被复位并打印出警告信息。这个限制仅仅是为了防止简单的DoS 攻击，不能过分依靠它或者人为地减小这个值，更应该增加这个值(如果增加了内存之后)。

**net.ipv4.tcp_max_syn_backlog = 262144**

记录的那些尚未收到客户端确认信息的连接请求的最大值。对于有128M 内存的系统而言，缺省值是1024，小内存的系统则是128。

**net.ipv4.tcp_timestamps = 0**

时间戳可以避免序列号的卷绕。一个1Gbps 的链路肯定会遇到以前用过的序列号。时间戳能够让内核接受这种“异常”的数据包。这里需要将其关掉。

**net.ipv4.tcp_synack_retries = 1**

为了打开对端的连接，内核需要发送一个SYN 并附带一个回应前面一个SYN 的ACK。也就是所谓三次握手中的第二次握手。这个设置决定了内核放弃连接之前发送SYN+ACK 包的数量。

**net.ipv4.tcp_syn_retries = 1**

在内核放弃建立连接之前发送SYN 包的数量。

**net.ipv4.tcp_fin_timeout = 1**

如  果套接字由本端要求关闭，这个参数决定了它保持在FIN-WAIT-2 状态的时间。对端可以出错并永远不关闭连接，甚至意外当机。缺省值是60  秒。2.2 内核的通常值是180 秒，3你可以按这个设置，但要记住的是，即使你的机器是一个轻载的WEB  服务器，也有因为大量的死套接字而内存溢出的风险，FIN- WAIT-2 的危险性比FIN-WAIT-1 要小，因为它最多只能吃掉1.5K  内存，但是它们的生存期长些。

**net.ipv4.tcp_keepalive_time = 30**

当keepalive 起用的时候，TCP 发送keepalive 消息的频度。缺省是2 小时。

## 4.nginx.conf配置文件

```py
user root;

# cpu数量，建议使用默认
worker_processes auto;
pid /run/nginx.pid;

# 配置nginx worker进程最大打开文件数 
worker_rlimit_nofile 65535;

events {
        # 单个进程允许的客户端最大连接数
	worker_connections 20480;
	# multi_accept on;
}

http {

	##
	# Basic Settings
	##

	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;
	# server_tokens off;

	# server_names_hash_bucket_size 64;
	# server_name_in_redirect off;

	include /etc/nginx/mime.types;
	default_type application/octet-stream;

	##
	# SSL Settings
	##

	ssl_protocols TLSv1 TLSv1.1 TLSv1.2; # Dropping SSLv3, ref: POODLE
	ssl_prefer_server_ciphers on;

	##
	# Logging Settings
	##

	access_log /var/log/nginx/access.log;
	error_log /var/log/nginx/error.log;

	##
	# Gzip Settings
	##

	gzip on;
	gzip_disable "msie6";

	# gzip_vary on;
	# gzip_proxied any;
	# gzip_comp_level 6;
	# gzip_buffers 16 8k;
	# gzip_http_version 1.1;
	# gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;

	##
	# Virtual Host Configs
	##

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}

```

## 5.内核配置文件

```
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 4096 87380 4194304
net.ipv4.tcp_wmem = 4096 16384 4194304
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 262144
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024 65000

```

**vi /etc/sysctl.conf** CentOS5.5中可以将所有内容清空直接替换.

使配置立即生效可使用如下命令：
**/sbin/sysctl -p**

## 6.系统连接数

**linux 默认值 open files 和 max user processes 为 1024**

#ulimit -n

1024

\#ulimit Cu

1024

**问题描述：** 说明 server 只允许同时打开 1024 个文件，处理 1024 个用户进程

使用ulimit -a 可以查看当前系统的所有限制值，使用ulimit -n 可以查看当前的最大打开文件数。

新装的linux 默认只有1024 ，当作负载较大的服务器时，很容易遇到error: too many open files 。因此，需要将其改大。

**解决方法：**

使用 ulimit Cn 65535 可即时修改，但重启后就无效了。（注ulimit -SHn 65535 等效 ulimit -n 65535 ，-S 指soft ，-H 指hard)

有如下三种修改方式：

\1. 在/etc/rc.local 中增加一行 ulimit -SHn 65535 2. 在/etc/profile 中增加一行 ulimit -SHn 65535 3. 在**/etc/security/limits.conf** 最后增加：

```
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535

```

具体使用哪种，**在 CentOS 中使用第1 种方式无效果，使用第3 种方式有效果**，而在Debian 中使用第2 种有效果

\# ulimit -n

65535

\# ulimit -u

65535

备注：ulimit 命令本身就有分软硬设置，加-H 就是硬，加-S 就是软默认显示的是软限制

soft 限制指的是当前系统生效的设置值。 hard 限制值可以被普通用户降低。但是不能增加。 soft 限制不能设置的比 hard 限制更高。 只有 root 用户才能够增加 hard 限制值。