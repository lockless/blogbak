---
layout: post
title: "DPDK编程指南（五）"
date: 2017-09-06 12:00
comments: true
tags: 
	- 数据平面
	- DPDK
key: "5"
---
# 5.内存池库
内存池是固定大小的对象分配器。在DPDK中，它由名称唯一标识，并且使用mempool操作来存储空闲对象。默认的mempool操作是基于ring的。它提供了一些可选的服务，如per-core缓存和对齐帮助，以确保对象被填充，方便将他们均匀扩展到DRAM或DDR3通道上。

这个库由报文缓冲区库使用。

<!-- more -->

## 5.1.Cookies
在调试模式（CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG使能）中，将在内存块的开头和结尾处添加cookies。 分配的对象包含保护字段，以帮助调试缓冲区溢出。

## 5.2.Stats
在调试模式（CONFIG_RTE_LIBRTE_MEMPOOL_DEBUG使能）中，从池中获取/释放的统计信息存放在mempool结构体中。统计信息是per-lcore的，避免并发访问统计计数器。

## 5.3.内存对齐限制
根据硬件内存配置，可以通过在对象之间添加特定的填充来大大提高性能。其目的是确保每个对象开始于不同的内存通道上，并在内存中排列，以便实现所有通道负载均衡。

特别是当进行L3转发或流分类时，报文缓冲对齐尤为重要。此时访问报文的前64B，因此可以通过在将对象的起始地址分布到不同的信道上来提升性能。

DIMM上的rank数目是可访问DIMM完整数据位宽的独立DIMM集合的数量。 由于他们共享相同的路径，因此rank不能被同时访问。 DIMM上的DRAM芯片的物理布局无需与rank数目相关。

当运行应用程序时，EAL命令行选项提供了添加内存通道和rank数目的能力。
命令行必须始终指定处理器的内存通道数目。

不同DIMM架构的对齐实例如下图所示。

![Figure 5 1 Two Channels and Quad-ranked DIMM Example](http://upload-images.jianshu.io/upload_images/7246758-825cbe4098e7807f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在这个例子中，假设报文时16块64B存储就不成立了。
Intel® 5520芯片组有三个通道，因此，在大多数情况下，对象之间不需要填充。（除了大小为n x 3 x 64B的块）。

![Figure 5 2 hree Channels and Two Dual-ranked DIMM Example](http://upload-images.jianshu.io/upload_images/7246758-c4fe48e6110a07ad.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

当创建一个新内存池时，用户可以指定使用此功能。

## 5.4.本地缓存
考虑CPU的使用率，由于每个访问需要compare-and-set (CAS)操作，所以多核访问内存池环形缓冲器中的空闲节点成本比较高。为了避免对内存池ring的访问请求太多，内存池分配器可以维护per-core cache，并通过实际内存池中具有较少锁定的缓存对内存池ring执行批量请求。通过这种方式，每个core都可以访问自己空闲对象的缓存（带锁），只有当空闲缓存被填充时，内核才需要将这些对象重新放回到缓冲池ring，或者当本地缓存空时，从缓冲池中获取更多对象。

虽然这意味着一些buffer可能在某些core的本地缓存上处于空闲状态，但是core可以无锁访问其自己的缓存提供了性能上的提升。

缓存由一个小型的per-core表及其长度组成。可以在创建池时启用/禁用此缓存。

缓存大小的最大值是静态配置，并在编译时定义的(CONFIG_RTE_MEMPOOL_CACHE_MAX_SIZE)。

![Figure 5 3 A mempool in Memory with its Associated Ring](http://upload-images.jianshu.io/upload_images/7246758-ce4669e3c0f81539.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

不同于per-lcore缓存，应用程序可以通过接口 rte_mempool_cache_create()，rte_mempool_cache_free()和rte_mempool_cache_flush() 创建和管理外部缓存。这些用户拥有的缓存可以被显式传递给 rte_mempool_generic_put() 和 rte_mempool_generic_get() 。接口 rte_mempool_default_cache() 返回默认内部缓存。与默认缓存相反，用户拥有的高速缓存可以由非EAL线程使用。

## 5.5.内存池操作
这允许外部存储子系统，如外部硬件存储管理系统和基于软件的存储管理与DPDK一起使用。

mempool 处理包括两方面：
* 添加你自己新的mempool操作代码。这是通过添加mempool ops代码，并使用 MEMPOOL_REGISTER_OPS 宏来实现的。
* 使用新的API调用 rte_mempool_create_empty() 及 rte_mempool_set_ops_byname() 用于创建新的mempool，并制定用户要使用的操作。

在同一个应用程序中可能会使用几个不同的mempool处理。可以使用 rte_mempool_create_empty() 创建一个新的mempool，然后用 rte_mempool_set_ops_byname() 将mempool指向相关的 mempool处理回调（ops）结构体。

传统的应用程序可能会继续使用旧的 rte_mempool_create() API调用，它默认使用基于ring的mempool处理。这些应用程序需要修改为新的mempool处理。

对于使用 rte_pktmbuf_create()的应用程序，有一个配置设置（RTE_MBUF_DEFAULT_MEMPOOL_OPS），允许应用程序使用另一个mempool处理。

## 5.6.用例
需要高性能的所有分配器应该使用内存池实现。 以下是一些使用实例：
* [报文缓冲区库]()  
* [环境抽象层]()
* 任何需要在程序中分配固定大小对象，并将被系统持续使用的应用程序

同时发布于简书：[http://www.jianshu.com/p/d69d8511cf63](http://www.jianshu.com/p/d69d8511cf63) 。