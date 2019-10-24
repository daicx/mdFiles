---
title: IO多路复用
date: 2019-05-15 00:19:42
tags:
- io多路复用
categories:
- 策略
- 通信策略
---

# I/O多路复用技术原理

## 1.简介

I/O操作指的是应用层去操作内核层.例如:我们写的程序去执行read()操作,读取服务器里面的数据.基于此操作衍生出几种操作模型.

<iframe frameborder="0" style="width:100%;height:463px;" src="https://www.draw.io/?lightbox=1&highlight=0000ff&layers=1&nav=1&page-id=08KH_gHkmrdjplk5lw3A&title=IO%E6%A8%A1%E5%9E%8B.drawio#R%3Cmxfile%20modified%3D%222019-05-10T03%3A38%3A19.157Z%22%20host%3D%22www.draw.io%22%20agent%3D%22Mozilla%2F5.0%20(Windows%20NT%2010.0%3B%20Win64%3B%20x64)%20AppleWebKit%2F537.36%20(KHTML%2C%20like%20Gecko)%20Chrome%2F74.0.3729.131%20Safari%2F537.36%22%20etag%3D%22kWXTK2WFBNGjQhbpSYhw%22%3E%3Cdiagram%20id%3D%22KZTyC1CdyC2-qKRh1v-L%22%20name%3D%22%E7%AC%AC%201%20%E9%A1%B5%22%3E3Vldc5s4FP01ekzGIITh0di03d3uNjPZaZu%2BySADGxm5WMR2f%2F1eCfFl48RJE6epJ5NIV1dC3HPu0bWC8HS5fV%2FQVfq3iBlH9ijeIjxDtj0mGH4rw64ykNG4MiRFFlcmqzVcZz%2BYMY6Mtcxitu45SiG4zFZ9YyTynEWyZ6NFITZ9t4Xg%2FaeuaMIODNcR5YfWL1ks08rq2ePW%2FoFlSVo%2F2XL9amRJa2fzJuuUxmLTMeEQ4WkhhKxay%2B2UcRW7Oi7VvHdHRpuNFSyXp0y4uaX%2F%2FBn%2Blbqfv1ExdkvyvZxfmFXuKC%2FNCyPb5bBesBCwLOxa7kwo3O%2BlqAcu1hqoCThgZ7VtB6GVqL9%2FwD4%2BoZCgyQgFRDcs5If14jBarV95mwA1j7Il2yp7KpccDBY0CwaPpHPtMIL%2BSmS51NiSAJEZWGgpRbUtPYHyLMmhzdlCLXXHCpkBrBNjlmIF1vWKRlme%2FKs6swun2YryZtujgbYa%2BID2TCyZLHbgYibYnuF8TXnT3bT8sbAhRdrhTk0UaiibNCu3qELDAPsIkMlBfAtR5jGLTSxFIVORiJzyj0JHQsXvPyblzuSjCm0fDrbN5NdO%2B0YtdYn3PmZwtjUP0p1dp3PFigzekRW1LYf3%2FdrtVAuTutsupXv1WtULsvggl%2FcwgyCIsojYPcGyjcjQImHyHj88zIGCcSqzu%2F4%2Bnh1Ra%2FwCkDaxvySkH37%2Fofjr3j6YMV2nekNdwowubdLjjPVokrwC2Jb7k2ibqVdKtVqlaGRgVyvAngRUL2Bm7XGm2cbTaWQfqn%2FooGCifvb5BcfXSjXLJZ9EUhQdTf1I54xfgfjKTChtnQspxRIcuBoIaHSbaHZOBVfzYDW80J%2BjutyhpSglz3I2bc73lgI%2FpdJHY9%2BR6SGVdl9KpeuKpxNzxfFr020zOmytwQsJObk%2FKV9fpfGJieu%2FpkrjgfQiCMTUc1UDksy%2FT8cVJJs0k%2BwaqhQ1uoEKu4%2FeIuO8k1UxZd4iUnWNLMQt64y4kcfmi%2BfJHGztZY59mDmWfc7UcYd1zEfBCIUe8gLVDl1VgPrT36zYPAEMcs5i07IOAnxGGWul66arXA%2FI2GW%2F3nlKsfOM2uafqG32q1ag%2BNeB%2BYESsobZ6oJsPQXkXkX7BhEfrkKx25cQPN7ThmpfL1aF%2Bqcck6aB9VXCDPneGzw4He9XOzitoW8ArjokPX%2Fo5JyicIx8MDrK6IVoou92vLECR4HmoEn4ex2w2OmDZhHnELTRWU9YMghaYKs7tmOgQc4oSCGLQuSRPpIAoKdLpePIK%2BcJ8jTskI2Bzkygs%2B%2FrpYjymXgDXICJQaDcjn6xfNPssPdSuqmNO%2BywnbOyY6gYBtjfIQ%2FrpMUowOrqtqVMg3%2FTGCSRq0Z9SyMJDjPlFhDlf1AJcJ6t1urEr28NIi7K%2BNGyvFhEEVBnQJaxi30cPxOGfQjdgevaIVH2Ho8gdNv7%2Fur4bP9pgsP%2FAQ%3D%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%22idACs95ClMfavqlKaeNe%22%20name%3D%22%E7%AC%AC%202%20%E9%A1%B5%22%3E7Vxdc6M2FP01ekxGH4iPR2M73c5sp9lJZ3b7SIxs08XIxXLi7K%2BvBJJtQLGdxECTmM0k6AoE6Nx7dHVAC8hwsfktj5bzP3jMUoBhvAFkBDD2KJG%2FleGpNFDolYZZnsSlCe0Md8kvpo1QW9dJzFaVAwXnqUiWVeOEZxmbiIotynP%2BWD1sytPqVZfRjDUMd5MobVq%2FJ7GYl1Yfezv7F5bM5ubKyA3KmkVkDtZPsppHMX%2FcM5ExIMOcc1HuLTZDlqq%2BM%2F3y7SFJp9Hjd%2Fjl5htf32YPv35Mr8rGbl5yyvYRcpaJ8zZNdDc8ROladxgYe2AwVD9jB4Q%2B8AMwpmDggIHumJV4Mp2b83UWM3UBBEj4OE8Eu1tGE1X7KL1J2uZikerqaZKmQ57yvDiXTKn6J%2B0rkfOfbK%2FGLTZ1Bs%2FEnr3cpP2B5SKREA%2FSZJbJOsHVpfSTyDq2qWF%2FpOPQFk0ZBYwvmMif5Hm6FV%2F3kQ4Ax3HL8uPOnVyso2S%2B70qOo91Yu%2FBs2%2FQOJrmjkXoJau4R1AbAH35y1AJYQU1GvAU1aEPNbws1ejzW4AW1fdQQORU10hZquAkadlOhe6wCkfvvmpuKq1UxEg7kAThYbnaVcm%2Bm%2Fv4u7%2BZPMA5AMALBuIAdgpAWO6iwuGDgq31pkUU%2FNNeVJ5aXLhtqOIqERFS9IWfybqL74gAoy0ueZKLoKBoCOpKWaC14ecfFCZEGO2VT8awPrKT3JdnsL1UYXTktekXdLVzS8AoMLV5hPOXsTtGMThbLrEMXM54x1evbgFV9znMx5zOeRelXXvSY6ud%2FmBBPOmdSEFRhY5tE%2FNjb%2F1s1dU1qm64cbfSFisLTXuGW5Yl8bJYbWya74Md%2BoWyYmuKuqaJk2iqfWT3oK7CVncXX%2BYQd4kedHUb5jB1qz7f7Ss7SSCQP1Zs7O%2FIOOUDMb8RZQ2CQfgOw14FbxdY7gm1RqrcXR6v5dsD5v6BPgj7hJ5YhvMi35E%2FdL%2BR8Yal214t0MBFqYN0S6dfonqW3knFFwhWh3nMh%2BEIekKqKMJr8nBVetT%2BEF9uzZLznTnwt0iSTg7yZUMGzJgRtkTyitbHfhQ2WJxaSd9sieeQeZPldoI931vORfpUM0GEyMIGP9qMevSbm2w5z%2F9Qw7zPKfUuUyySMAt9VOzLWA%2B%2Bs%2BXkcMX86sYbjxGf30xbDjuDjYYeoZXrbXtxBO8kGIJTTI19lwnJfpscyKw6GnyH9bUDUnBXRLtNfRDvNfyHYz3%2FpK6iw%2F%2BQWGe2z7exWn3qrfHznQo5fc6Gg5hvlfemzau6xvY03eAyyUqpiUlJMeeUk2FcxHeJiyuuqyFbK49Fwf6e0S90TaNemdLRHuzYpuAEIHhb7EpbxTqpSlhEYNFnhA3IxpRXcsEMbuPmdcrHfKRd%2FEi0CBSfSNX7GXbpJVM2o39VAfD6BwvfoyxDvX6E42SfK9z59%2BYS5zXevUcQ%2BhB6xjc4DCqEDmxrF9vptsf%2F25ZBhfzMP7EujwIcJ4KJRvCbQ8am5OupVi8T2nPqjyhTkupp5EUN0vQkV2PJu8HMLFahOj02IOhUqcPNtzUWoOEJ%2BTleZ7zNKhVf1IWK%2B1OpIqTDPf1EqjFJBj%2Bc83SoV2PYhzUWpaCgVpBpIli%2BgOlUq8OH3aRel4nV87Z3K170mqxR1Cv6n%2FpTiZJcotcve5i82wfk9ChWMIgdC2%2BAcYghhL0IFqZL%2F9v1Pb0LFYZm6daHCQbgPqcLGLa2H%2F8natddOBg%2Brr7aoeUVimiifoJHBNxpCTnC4obanAjYl9cMKLKS2zMHCGN3KK%2BTyHchhecVxm%2B8eO5VXSMdJ3QeQV8yChvbT9dPkFYd2%2ByGIef6LvLKVV6qDnC2ou5VXiO0j54u8UidjUstyLB%2FwdCqvEHu2Et4Anyhg5O%2BQqBVJCi1pHzW5O02T5UpRtpltTVK%2Bjl8cTdPpRG62aCIuCUjcJiq1r6pgc6UQsq0UMot%2Bz79e5Bk5uYlKsdbLb%2FLaB0AFVVHBXnMFLUIWVLy2FtA6NgXZioqjAmUA1UI9JUyEasWmTPpDywqwD4eTQ%2FC1ZTCyxg99MVKyuFs7X6YXu%2F%2BAgIz%2FAw%3D%3D%3C%2Fdiagram%3E%3Cdiagram%20id%3D%2208KH_gHkmrdjplk5lw3A%22%20name%3D%22%E7%AC%AC%203%20%E9%A1%B5%22%3E7Vpbc6M2FP41ekxGIK6PYJNtO%2Bl003R226cdBWSbroxckBM7v74SCMxFcexNbHY38WQc6Ug%2BgL7vXHQEQJPl5kOOV4vfWUIoMGGyAWgKTNO1kfiWgm0lsKFbCeZ5mlQiYye4TR%2BJEkIlXacJKToTOWOUp6uuMGZZRmLekeE8Zw%2FdaTNGu1dd4TkZCG5jTIfSz2nCF5XUM92d%2FBeSzhf1lQ3Hr0aWuJ6snqRY4IQ9tEQoAmiSM8ar1nIzIVSuXb0uRTx7XFzc%2FBlx9%2Bb6%2FuY3%2F9OX64tK2dUxP2keIScZf13VSOm%2Bx3StFkw9LN%2FWK5izdZYQqcUAKHxYpJzcrnAsRx8EZYRswZdUDc9Yxq%2FwMqWSLp9InuAMK7HihmHKfkrphFGWl5dAs9nMjGMhL3jOvpLWSOLcObYjRtR9kpyTTQ%2FZZ5bFaLASHCdsSXi%2BFb9TWkxfEVzx21doP%2BzIYiFFlkWLKMhVE7Ei6LxRvQNBNBQOR2BiDiD5Vej5A0Q2CCzgByDyQOiC4EpJvCmIXOBbIPAG2InF4l2AclKkj%2FiunABFf8XSjJePYIfAngoJXnNWKKxkl6bzTLQpmUlVEoFUmFegxJxJBhSCEGk2%2F0t2phdWF3HTOyF%2BqDZPhR%2ByB%2FgZDhziZ50KPgMNUCCJcEmqm7GMSBgao5IgsJwv2JxlmF6zcgnlwv9LON%2BqJZSYdHEkm5T%2F3Wr%2FI1Vdot5HDU436kJlZ9vqfCR5Kp6b5LUsE2sg9V7AS2jataTSbtXdnb6yt233%2BhoTXCwa76FnhVyfb%2BCEWGO2zmOyz7%2BpiIPzOdmnz9FzLCcU8%2FS%2Be3Ovzhh0XsbAS7vNmW%2BniFDUI4hvv5ghJEsCGfZFN6a4KNK4El6llA5JVAaT74VEyByVRZpI7lCu1qtDL%2Be%2FNasHLipXH4gJhrfa7AZFa17%2BjywQBvJPqRN3V2lU433uijxpJZvrJQ1iLgN5EzSu8R2hH0V04SmTweOOcc6WYgKVAyGOv85LmndTA%2FF5MvC0%2BM3WnKaZSB7qRBJqHE4v8YDQhmSmSzwgNKbhRGnQ3c%2BpAloTrVRAM%2BEwI0GagOacLKD5Iwa02lU94Z4aV9T1Q8%2B5oadi0lnciXOgOzGe4Ml53Im5PyjtAI520lPFKOMNk2DUmFLvkkciQccTuAe6AqPNAuMZDhyVsX5XxECj5hpwwIsT%2BX%2Fz0ABwCaHXtX%2Bn2v78bNnowZ4DjskQ5%2BXZqKvPRm0gdhmeUxY%2FIhA6ZX4aggCVVRAohw5MVI8ubO1JJBObeImlSyQ98w45p6xg9SsgTem1XQIxz5oxwiH6bdzEXsJ3g5fjcXyhkRgCKFcHk%2B%2B4CJ8SJruX1xuavP7MMGmKvwOYXsFsfmiYmr3WeDANC8JDmCZvHKamHD8eTNbLQ97TBRgfhFAW%2Fr1QtiMH%2BBHwJ4dGup%2F%2FGMDsBUFDc47jnvUY4BUyIFPQoeSBjhFTEMA695mUZ0JXwK%2BoMQGeryGLdBpQ%2BIofijWtnPr1WWOhHmtsjRexzkobT%2BPsHRCaIDD2Yfv2sHNgDztriJ3u4M88FXSmLut9oaG%2BOVStQ9IveFZYdenXu0UeYJGm441skZqU7N0ijy0veL1M20JjW6QmtXo3xD2GaGkqQvZZEdNlNUMjEw1hmrY6cfacNwhdU7yrfag3sg9F%2BuqD8JgekoiJ7xDJ%2FUrbjQqvKoNjs2915WtrnlXOn0hsKw1BuBvqQy1uLV0V8jypfp8gpmydHFWo1dUsIPQwhLqaBYS2C%2Bt3Bs554m%2F1ElldYVC3d%2FWOx1x0dy%2BslmOtt35R9D8%3D%3C%2Fdiagram%3E%3C%2Fmxfile%3E"></iframe>



## 2.堵塞I/O

通常IO操作都是阻塞I/O的，也就是说当你调用read时，如果没有数据收到，那么线程或者进程就会被挂起，直到收到数据。阻塞的意思，就是一直等着。阻塞I/O就是等着数据过来，进行读写操作。应用的函数进行调用，但是内核一直没有返回，就一直等着。应用的函数长时间处于等待结果的状态，我们就称为阻塞I/O。每个应用都得等着，每个应用都在等着，浪费啊！很像现实中的情况。大家都不干活，等着数据过来，过来工作一下，没有的话继续等着。

## 3.非堵塞I/O

非阻塞IO很简单，通过fcntl（POSIX）或ioctl（Unix）设为非阻塞模式，这时，当你调用read时，如果有数据收到，就返回数据，如果没有数据收到，就立刻返回一个错误，如EWOULDBLOCK。这样是不会阻塞线程了，但是你还是要不断的轮询来读取或写入。相当于你去查看有没有数据，告诉你没有，过一会再来吧！应用过一会再来问，有没有数据？没有数据，会有一个返回。但是依旧很不好。应用必须得过一会来一下，问问内核有木有数据啊。这和现实很像啊！好多情况都得去某些地方问问好了没有？木有，明天再过来。明天，好了木有？木有，后天再过来。。。。。忙碌的应用。。。。

## 4.I/O多路复用

多路复用是指使用一个线程来检查多个文件描述符（Socket）的就绪状态，比如调用select和poll函数，传入多个文件描述符（FileDescription，简称FD），如果有一个文件描述符（FileDescription）就绪，则返回，否则阻塞直到超时。得到就绪状态后进行真正的操作可以在同一个线程里执行，也可以启动线程执行（比如使用线程池）。虾米意思？就是派一个代表，同时监听多个文件描述符是否有数据到来。等着等着，如有有数据，就告诉某某你的数据来啦！赶紧来处理吧。有没有很感动，一个人待着，帮了很多人。医院的黄牛，一个人排队，大家只要把钱给它，它就会把号给需要的人.

### 5.I/O多路复用实现方式

#### 5.1 select 函数

```
int select (int n, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select 函数监视的文件描述符分3类，分别是writefds、readfds、和exceptfds。调用后select函数会阻塞，直到有描述副就绪（有数据 可读、可写、或者有except），或者超时（timeout指定等待时间，如果立即返回设为null即可），函数返回。当select函数返回后，可以 通过遍历fdset，来找到就绪的描述符。

select目前几乎在所有的平台上支持，其良好跨平台支持也是它的一个优点。select的一 个缺点在于单个进程能够监视的文件描述符的数量存在最大限制，在Linux上一般为1024，可以通过修改宏定义甚至重新编译内核的方式提升这一限制，但 是这样也会造成效率的降低。

#### 5.2 poll函数

```
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

不同与select使用三个位图来表示三个fdset的方式，poll使用一个 pollfd的指针实现。

```
struct pollfd {
    int fd; /* file descriptor */
    short events; /* requested events to watch */
    short revents; /* returned events witnessed */
};
```

​	pollfd结构包含了要监视的event和发生的event，不再使用select“参数-值”传递的方式。同时，pollfd并没有最大数量限制（但是数量过大后性能也是会下降）。 和select函数一样，poll返回后，需要轮询pollfd来获取就绪的描述符。

注:

**从上面看，select和poll都需要在返回后，`通过遍历文件描述符来获取已经就绪的socket`。事实上，同时连接的大量客户端在一时刻可能只有很少的处于就绪状态，因此随着监视的描述符数量的增长，其效率也会线性下降。**

#### 5.3 epoll函数

epoll是在2.6内核中提出的，是之前的select和poll的增强版本。相对于select和poll来说，epoll更加灵活，没有描述符限制。epoll使用一个文件描述符管理多个描述符，将用户关系的文件描述符的事件存放到内核的一个事件表中，这样在用户空间和内核空间的copy只需一次。

epoll操作过程需要三个接口，分别如下：

```
int epoll_create(int size)；//创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
int epoll_wait(int epfd, struct epoll_event * events, int maxevents, int timeout);
```

**1. int epoll_create(int size);**
创建一个epoll的句柄，size用来告诉内核这个监听的数目一共有多大，这个参数不同于select()中的第一个参数，给出最大监听的fd+1的值，`参数size并不是限制了epoll所能监听的描述符最大个数，只是对内核初始分配内部数据结构的一个建议`。
当创建好epoll句柄后，它就会占用一个fd值，在linux下如果查看/proc/进程id/fd/，是能够看到这个fd的，所以在使用完epoll后，必须调用close()关闭，否则可能导致fd被耗尽。

2.int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)；
函数是对指定描述符fd执行op操作。

- epfd：是epoll_create()的返回值。

- op：表示op操作，用三个宏来表示：添加EPOLL_CTL_ADD，删除EPOLL_CTL_DEL，修改EPOLL_CTL_MOD。分别添加、删除和修改对fd的监听事件。

- fd：是需要监听的fd（文件描述符）

- epoll_event：是告诉内核需要监听什么事，struct epoll_event结构如下：

  ```
  struct epoll_event {
    __uint32_t events;  /* Epoll events */
    epoll_data_t data;  /* User data variable */
  };
  
  //events可以是以下几个宏的集合：
  EPOLLIN ：表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；
  EPOLLOUT：表示对应的文件描述符可以写；
  EPOLLPRI：表示对应的文件描述符有紧急的数据可读（这里应该表示有带外数据到来）；
  EPOLLERR：表示对应的文件描述符发生错误；
  EPOLLHUP：表示对应的文件描述符被挂断；
  EPOLLET： 将EPOLL设为边缘触发(Edge Triggered)模式，这是相对于水平触发(Level Triggered)来说的。
  EPOLLONESHOT：只监听一次事件，当监听完这次事件之后，如果还需要继续监听这个socket的话，需要再次把这个socket加入到EPOLL队列里
  
  ```

  **3. int epoll_wait(int epfd, struct epoll_event \* events, int maxevents, int timeout);**
  等待epfd上的io事件，最多返回maxevents个事件。
  参数events用来从内核得到事件的集合，maxevents告之内核这个events有多大，这个maxevents的值不能大于创建epoll_create()时的size，参数timeout是超时时间（毫秒，0会立即返回，-1将不确定，也有说法说是永久阻塞）。该函数返回需要处理的事件数目，如返回0表示已超时。

epool 工作模式:

epoll对文件描述符的操作有两种模式：**LT（level trigger）**和**ET（edge trigger）**。LT模式是默认模式，LT模式与ET模式的区别如下：
　　**LT模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序可以不立即处理该事件`。下次调用epoll_wait时，会再次响应应用程序并通知此事件。
　　**ET模式**：当epoll_wait检测到描述符事件发生并将此事件通知应用程序，`应用程序必须立即处理该事件`。如果不处理，下次调用epoll_wait时，不会再次响应应用程序并通知此事件。

##### LT模式

LT(level triggered)是缺省的工作方式，并且同时支持block和no-block socket.在这种做法中，内核告诉你一个文件描述符是否就绪了，然后你可以对这个就绪的fd进行IO操作。如果你不作任何操作，内核还是会继续通知你的。

##### ET模式

ET(edge-triggered)是高速工作方式，只支持no-block socket。在这种模式下，当描述符从未就绪变为就绪时，内核通过epoll告诉你。然后它会假设你知道文件描述符已经就绪，并且不会再为那个文件描述符发送更多的就绪通知，直到你做了某些操作导致那个文件描述符不再为就绪状态了(比如，你在发送，接收或者接收请求，或者发送接收的数据少于一定量时导致了一个EWOULDBLOCK 错误）。但是请注意，如果一直不对这个fd作IO操作(从而导致它再次变成未就绪)，内核不会发送更多的通知(only once)

ET模式在很大程度上减少了epoll事件被重复触发的次数，因此效率要比LT模式高。epoll工作在ET模式的时候，必须使用非阻塞套接口，以避免由于一个文件句柄的阻塞读/阻塞写操作把处理多个文件描述符的任务饿死。

##### epoll总结

在 select/poll中，进程只有在调用一定的方法后，内核才对所有监视的文件描述符进行扫描，而**epoll事先通过epoll_ctl()来注册一 个文件描述符，一旦基于某个文件描述符就绪时，内核会采用类似callback的回调机制，迅速激活这个文件描述符，当进程调用epoll_wait() 时便得到通知**。(`此处去掉了遍历文件描述符，而是通过监听回调的的机制`。这正是epoll的魅力所在。)

**epoll的优点主要是一下几个方面：**
\1. 监视的描述符数量不受限制，它所支持的FD上限是最大可以打开文件的数目，这个数字一般远大于2048,举个例子,在1GB内存的机器上大约是10万左 右，具体数目可以cat /proc/sys/fs/file-max察看,一般来说这个数目和系统内存关系很大。select的最大缺点就是进程打开的fd是有数量限制的。这对 于连接数量比较大的服务器来说根本不能满足。虽然也可以选择多进程的解决方案( Apache就是这样实现的)，不过虽然linux上面创建进程的代价比较小，但仍旧是不可忽视的，加上进程间数据同步远比不上线程间同步的高效，所以也不是一种完美的方案。

1. IO的效率不会随着监视fd的数量的增长而下降。epoll不同于select和poll轮询的方式，而是通过每个fd定义的回调函数来实现的。只有就绪的fd才会执行回调函数。

> 如果没有大量的idle -connection或者dead-connection，epoll的效率并不会比select/poll高很多，但是当遇到大量的idle- connection，就会发现epoll的效率大大高于select/poll。

参考:

<https://zhuanlan.zhihu.com/p/65013389>

<https://segmentfault.com/a/1190000003063859>

