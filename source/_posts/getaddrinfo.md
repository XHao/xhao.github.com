layout: post
title: getaddrinfo探秘
tags: linux
date: 2018-09-27

---
在现在流行的服务化调用场景中，为了避免服务的单点问题，会将服务部署成一个集群，然后利用服务注册中心来解决服务发现问题。

为了解决服务注册中心的单点问题，有时候会选择古老的dns轮询。

通过将一个域名关联多条A记录，利用域名服务器每次的不同返回，达到软负载的效果。

关于jdk的dns解析，网上有很多文章介绍，有对的地方也有错的。
<!--more-->
首先看看常用的jdk方法

```java
// netty Bootstrap Line 97
public ChannelFuture connect(String inetHost, int inetPort) {
    return connect(new InetSocketAddress(inetHost, inetPort));
}
```

这段代码入参是hostname，获得socket连接，在`InetSocketAddress`中自动完成了dns的工作——`InetAddress.getByName(hostname)`

jdk中dns查询分2步完成，首先是检查addressCache；如果没有发现，则触发一次网络dns查询。

`InetAddressCachePolicy`是address的缓存，默认是30s的缓存时间，可以通过`networkaddress.cache.ttl`来修改。这个cache有这样几个特点：

1. 如果你通过security property的方式设了值，则在运行期不可以修改
2. 如果运行期修改ttl，不能比以前小

从代码上猜测，主要是因为dns查询本身损耗性能，加入这种安全限制，能避免三方库随意设置，导致应用出现瓶颈。

如果想在运行期自由控制dns缓存时间，就只能通过反射这种方式了

```java
public class Test{

    public static void main(String[] args) throws Exception{
        try {
            Field f = sun.net.InetAddressCachePolicy.class.getDeclaredField("cachePolicy");
            f.setAccessible(true);
            int cache = (int) f.get(sun.net.InetAddressCachePolicy.class);
            System.err.println("before ttl is "+cache);
            f.set(sun.net.InetAddressCachePolicy.class, 0);
            cache = (int) f.get(sun.net.InetAddressCachePolicy.class);
            System.err.println("after ttl is "+cache);
        } catch (NoSuchFieldException | SecurityException | IllegalArgumentException | IllegalAccessException e) {
        }
        while(true){
            Arrays.asList(InetAddress.getAllByName("xhao.io")).forEach(System.out::println);
            System.out.println("-------------");
            Thread.sleep(5000L);
        }
    }
}
```

ok，当我们将ttl换成0之后，跑跑看，会发现在不同的机器上结果不一样。

macos high sierra

![mac dns 查询](/img/dns-mac.jpg)

CentOS Linux release 7.5.1804

![centos dns 查询](/img/dns-centos.jpg)

因为我们已经调整了dns缓存的时间，所以每次打印的结果都是通过dns查询得来的。

但是在mac上感觉ip顺序是固定的，而centos上却是无序的；换句话说，在mac上无法实现dns轮询，而centos上还可以。具体原因是什么呢？关于这个，查阅了一些资料，但都说的不清楚，有一个可能性是关于RFC 3484

> [getaddrinfo](https://www.systutorials.com/docs/linux/man/3-getaddrinfo/): The sorting function used within getaddrinfo() is defined in RFC 3484; the order can be tweaked for a particular system by editing /etc/gai.conf (available since glibc 2.5).

网上一些关于RFC 3484的说明是：

1. 给ip排序的目的是想尽量使用ipv6，方便ipv6设备的推广
2. 应该少依赖dns轮询，这不可靠
3. ip排序有一定的规则，会实现与否要看本地库（这只是建议，不强制）

在mac上并不打算继续追查这个问题了，偷懒认为它是按照标准实现的。但是linux上，可以用trace工具看看`getaddrinfo`的逻辑。所以找了台机器跑一跑下面的程序：`strace ./a.out`

```c
#include <netdb.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
/*
 * ===  FUNCTION
 * ====================================================================== Name:
 * main Description:
 * =====================================================================================
 */
int main(int argc, char *argv[])
{

  struct addrinfo hints, *res = NULL;
  char *host = "xhao.io";
  memset(&hints, 0, sizeof(hints));
  hints.ai_flags = AI_CANONNAME;
  hints.ai_family = AF_UNSPEC;

  int getaddrinfo_error = getaddrinfo(host, NULL, &hints, &res);
  if (getaddrinfo_error)
  {
    printf("fail %s", gai_strerror(getaddrinfo_error));
  }
  else
  {
    struct addrinfo *iterator = res;
    char prev[100], addrstr[100];
    void *ptr;
    while (iterator != NULL)
    {
      if (iterator->ai_family == AF_INET)
      { /* AF_INET */
        ptr = &((struct sockaddr_in *)iterator->ai_addr)->sin_addr;
      }
      else
      {
        ptr = &((struct sockaddr_in6 *)iterator->ai_addr)->sin6_addr;
      }
      inet_ntop(iterator->ai_family, ptr, addrstr, 100);
      if (strcmp(prev, addrstr) != 0)
      {
        printf("Ipv%d is %s\n", iterator->ai_family == AF_INET ? 4 : 6,
               addrstr);
        strncpy(prev, addrstr, 100);
      }
      iterator = iterator->ai_next;
    }
  }

  return EXIT_SUCCESS;
} /* ----------  end of function main  ---------- */
```

我们会发现结果和jdk中的结果相同，基本可以判断并没有之前所说的排序过程。

同时我们还发现，jdk是调用了系统命令来查询dns，这和我们在jdk源码中看到的差不多(jdk这部分代码有一大坨，看起来只是在整理和排序)

我们还发现getaddrinfo是一个很重的命令，因为它每次都会去发请求到dns服务器上，并没有任何缓存。这应该就是jdk自己做了缓存的原因，毕竟每次都查询dns服务器，对性能是有影响的。

关于java dns相关的，大概就是这些内容了。总之dns轮询不一定可靠，需要慎重对待。
