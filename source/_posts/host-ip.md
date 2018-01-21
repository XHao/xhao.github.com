layout: post
title: java中的getHostname 
tags: [java, linux]
date: 2016-04-16

---
有时候在linux服务器上，java程序启动时会出现`unknown host name`的问题。解决这类问题的关键是，搞清楚jdk获取ip和hostname的方式以及linux机器的ip、hostname（这里经常会涉及到一些命令，诸如：hostnamectl、nslookup、dig、ifconfig等）。
<!--more-->
下面这个是大家最常用的：

### 通过`InetAddress.getLocalHost`来获得Address

```
Returns the address of the local host. This is achieved by retrieving the name of the host from the system, then resolving that name into an InetAddress.

Note: The resolved address may be cached for a short period of time.
```

它的实现分3个部分：

1.  先是`impl.getLocalHostName()`取得hostname。这是一个native方法，在linux内核上是通过调用内核函数`gethostname`完成。而hostname是和下面有关的

```bash
[hao.xie@hao-xie-vm ~]$ hostname #hostname返回当前值
hao-xie-vm
[hao.xie@vhao-xie-vm ~]$ more /proc/sys/kernel/hostname #当前值，或者说修改后生效
hao-xie-vm
[hao.xie@hao-xie-vm ~]$ more /etc/sysconfig/network #这个文件需要重点说下，reboot的时候，系统会读取它，并给hostname赋值，但是在运行过程中修改它不会立即生效，需要配合hostname命令生效
# Created by anaconda
```

2. 接着，jdk又去查了缓存的localhost信息（5s有效的cache），如果没有缓存才开始取address
3. 取Address的方式是`InetAddress.getAddressesFromNameService`，按字面意思就是向dns来反向查询自己

### `InetAddress.getLocalHost.getHostName`获取hostname

```
Gets the host name for this IP address.

If this InetAddress was created with a host name, this host name will be remembered and returned; otherwise, a reverse name lookup will be performed and the result will be returned based on the system configured name lookup service. If a lookup of the name service is required, call getCanonicalHostName.
```

描述很清楚，就是再通过ip反向来查hostname，原因和上面一样。

### 我遇到过的问题

我之前在服务器上发现hostname返回的是`A`，但是`InetAddress.getLocalHost.getHostName`返回异常`unknown host name : host name B `。原因就是2次dns的结果差异造成`hostname A -> ip1；ip1 -> B`这种尴尬的事情。这个时候就需要nslookup工具来检查，最后在dns服务器上进行修改了。

### 如何在java代码中获取本机ip（非127.0.0.1）

有一些推荐的方式，比如通过遍历网卡来选择

```java
public static InetAddress getLocalHostLANAddress() throws UnknownHostException, SocketException {
		try {
			InetAddress candidateAddress = null;
			// Iterate all NICs (network interface cards)...
			Enumeration<NetworkInterface> ifaces = NetworkInterface.getNetworkInterfaces();
			while (ifaces.hasMoreElements()) {
				NetworkInterface iface = (NetworkInterface) ifaces.nextElement();
				for (@SuppressWarnings("rawtypes")
				Enumeration inetAddrs = iface.getInetAddresses(); inetAddrs.hasMoreElements();) {
					InetAddress inetAddr = (InetAddress) inetAddrs.nextElement();
					if (!inetAddr.isLoopbackAddress()) {
						if (inetAddr.isSiteLocalAddress()) {
							return inetAddr;
						} else if (candidateAddress == null) {
							candidateAddress = inetAddr;
						}
					}
				}
			}
			if (candidateAddress != null) {
				return candidateAddress;
			}
			// At this point, we did not find a non-loopback address.
			// Fall back to returning whatever InetAddress.getLocalHost()
			// returns...
			InetAddress jdkSuppliedAddress = InetAddress.getLocalHost();
			if (jdkSuppliedAddress == null) {
				throw new UnknownHostException("The JDK InetAddress.getLocalHost() method unexpectedly returned null.");
			}
			return jdkSuppliedAddress;
		} catch (Exception e) {
			UnknownHostException unknownHostException = new UnknownHostException(
					"Failed to determine LAN address: " + e);
			unknownHostException.initCause(e);
			throw unknownHostException;
		}
	}
```

或者通过建立tcp socket来反向获取本机ip

```java
public static String trickyIp() throws IOException {
    try (final Socket socket = new Socket()) {
        socket.connect(new InetSocketAddress("www.google.com", 80));
        return socket.getLocalAddress().getHostAddress();
    }
}
```

### 在java代码中获得[hostname](/2016/09/hostname)

`hostname(1)`应该是唯一的选择（错误的网络环境配置导致上面的例子），所以在java代码中就变成了如何调用系统调用了，通过jni或者jna都是可以的，比如

```java
// 通过jna
interface CLibrary extends Library {
    CLibrary INSTANCE = (CLibrary) Native.loadLibrary("c", CLibrary.class);

    int gethostname(byte[] hostname, int bufferSize);
}

public static String hostname() throws UnknownHostException {
        if (Platform.isWindows()) {
            return Kernel32Util.getComputerName();
        } else {
            byte[] hostnameBuffer = new byte[255];
            int result = CLibrary.INSTANCE.gethostname(hostnameBuffer, hostnameBuffer.length);
            if (result != 0) {
                return InetAddress.getLocalHost().getHostName();
            }
            return Native.toString(hostnameBuffer);
        }
    }
```
