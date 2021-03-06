---
layout: post
title: "TIPC协议"
date: 2017-09-11 18:00
comments: true
tags: 
	- Linux
	- 网络
key: "1"
---
# 1.TIPC协议概述
TIPC是爱立信开源的透明进程通信协议，一般用于集群系统中。
虽然tipc是基于socket实现的，但是与一般的socket还有所区别。平时我们使用socket，无论是TCP也好，UDP也好，用来标识一对socket的通信，无非是用两个socket的IP地址和端口号。比如使用UDP的socket，要发送一个datagram到另一个socket，需要指定对端的地址，这个地址是由对端设备的IP和端口号组成的。Socket是在内核中进行管理，当内核检测到socket有数据可读时，就会通知拥有这个socket的进程去读取数据。
这种实现由一种不方便，就是需要指定对端的地址，我们必须知道这个socket在哪台设备上，设备IP是多少，使用的端口号是什么，才能发送数据。
TIPC解决了这个问题。使用TIPC，我们在创建socket的时候，在内核中注册自己的服务类型，那么在发送端，只需要指定服务类型就可以由内核路由到相应的socket。这个时候，对应用层来讲，对端地址仅仅是一个服务类型。显然，内核维护着这样一张路由表，可以根据服务类型去找到对应的socket。每台设备都有这样的路由表，他们的信息就能够像普通路由表一样共享到整个集群网络中去，所有设备都可以进行socket查找。因此，有了TIPC，我们无需关心socket使用了哪个IP，哪个端口。

<!-- more -->

Tipc还具有如下特性：
*  有些时候，多个进程提供相同的服务，仅仅是为了负载均衡或冗余备份等原因，这种情况下可以用一个整数变量instance来标识不同的socket，但是指定同样的服务类型。此时，socket是由service type和instance共同指定的。发送数据的时候只需要指定service type和一个instance值即可。也可以指定service type和instance的一个区间，这种情况就是broadcast你的datagram
* 管理tipc路由表的是内核中的name server进程。他维护着集群中所有的tipc socket。在发送datagram给某个socket之前，可以向他请求


# 2.实验结果
本实验在Ubuntu 14.04环境中测试，两台虚拟机搭建集群环境
虚拟机操作系统需要先加载tipc模块，使用命令
```
modprobe tipc
```
编译并运行tipcutils，方便我们对tipc进行配置
tipcutils代码可以在github上获取https://github.com/parbhu/tipcutils/tree/96413b283861d271bff23f3098565da58536c628
或者
https://github.com/TIPC/tipcutils

## 2.1.集群配置
配置tipc的网卡和地址，配置成功之后，两台设备上都可以发现tipc邻居，具体结果如下图所示

![neighbor-1.png](http://upload-images.jianshu.io/upload_images/7246758-e71e9d143e49d8a2.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
 
![neighbor-2.png](http://upload-images.jianshu.io/upload_images/7246758-6655e339c18ce00d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

此时就可以通过tipc进行通信了。

## 2.2.通信测试
以hello_world模块进行测试
* 1.1.2设备当成集群的服务端，运行server_tipc程序
* 1.1.3设备当成集群的客户端，运行client_tipc程序

```
//服务端可客户端定义应该一样
#define SERVER_TYPE  18888
#define SERVER_INST  17
```

#### 服务端
根据参数创建socket之后，就一直处于监听socket，等待数据到来，并对数据进行处理
```
	server_addr.family = AF_TIPC;
	server_addr.addrtype = TIPC_ADDR_NAMESEQ;
	server_addr.addr.nameseq.type = SERVER_TYPE;
	server_addr.addr.nameseq.lower = SERVER_INST;
	server_addr.addr.nameseq.upper = SERVER_INST;
	server_addr.scope = TIPC_ZONE_SCOPE;

	sd = socket(AF_TIPC, SOCK_RDM, 0);

	if (0 != bind(sd, (struct sockaddr *)&server_addr, sizeof(server_addr))) {
		printf("Server: failed to bind port name\n");
		exit(1);
	}

	if (0 >= recvfrom(sd, inbuf, sizeof(inbuf), 0,
	                  (struct sockaddr *)&client_addr, &alen)) {
		perror("Server: unexpected message");
	}
	printf("Server: Message received: %s !\n", inbuf);

	if (0 > sendto(sd, outbuf, strlen(outbuf)+1, 0,
	                (struct sockaddr *)&client_addr, sizeof(client_addr))) {
		perror("Server: failed to send");
	}
```

#### 客户端

创建socket之后主动发送数据，监听socket
服务端调用wait_for_server来等待server连上
```
	wait_for_server(SERVER_TYPE, SERVER_INST, 10000);

	sd = socket(AF_TIPC, SOCK_RDM, 0);

	server_addr.family = AF_TIPC;
	server_addr.addrtype = TIPC_ADDR_NAME;
	server_addr.addr.name.name.type = SERVER_TYPE;
	server_addr.addr.name.name.instance = SERVER_INST;
	server_addr.addr.name.domain = 0;

	if (0 > sendto(sd, buf, strlen(buf)+1, 0,
	                (struct sockaddr*)&server_addr, sizeof(server_addr))) {
		perror("Client: failed to send");
		exit(1);
	}

	if (0 >= recv(sd, buf, sizeof(buf), 0)) {
		perror("Client: unexpected response");
		exit(1);
	}
```
#### 运行结果

![server.png](http://upload-images.jianshu.io/upload_images/7246758-e2271529e6446cc9.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![client.png](http://upload-images.jianshu.io/upload_images/7246758-8579c92ba7506525.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

同时发布于简书：[TIPC协议](http://www.jianshu.com/p/7fb89e3d19af) 。