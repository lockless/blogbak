---
layout: post
title: "Interlaken协议（上）"
date: 2017-09-13 18:00
comments: true
tags: 
	- 网络
key: "1"
---
> 英文原文： Interlaken Protocol Definition - A Joint Specification of Cortina Systems and Cisco Systems

# 1.简介
网络应用中两种主流的芯片到芯片的高速网络传输协议是XAUI 和SPI4.2 。虽然SPI4.2在通道化、Burst大小可编程和每通道背压方面提供了重要的优势，但是接口的过大宽度限制了其可扩展性，并且协议的源同步性质降低了其有效覆盖范围。相反的，XAUI是一个狭窄的4通道接口，提供长距离，适合各种实现：PCB上的FR4、背板和电缆。但是作为一个基于数据包的接口，它缺乏通道化和流控制，限制了他的应用。而且这两种协议仅提供固定配置，限制了设计人员为应用定制接口容量的能力。

本文档定义了一种新的协议，Interlaken，他能够实现窄、高速、信道化分组接口的设计。

<!-- more -->

![Figure 1 1 XAUI versus SPI4.2 interfaces](http://upload-images.jianshu.io/upload_images/7246758-9063720af05d9262.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 2.应用
Interlaken有多种应用：
* 帧/MAC到NPU的接口或者L2/L3交换接口
* 线卡到交换网的接口

它也可以在多种介质上运行，包括PCB、背板或电缆。

![Figure 2 1 Framer/MAC to NPU/L2 or L3 switch](http://upload-images.jianshu.io/upload_images/7246758-d4c5a5cc9fca1180.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![Figure 2 2 Framer/MAC to NPU/L2 or L3 switch](http://upload-images.jianshu.io/upload_images/7246758-2751d7b6bca74e2e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

# 3.Interlaken协议
## 3.1.基本面
Interlaken是一种窄的，高速通道化的芯片到芯片接口。其特性可以描述为如下几点：
* 支持256个通信通道，或高达64K的通道扩展
* 用一个简单的控制字结构来描述数据包，功能类似于SPI4.2
* 频率可编程的连续的Meta Frame以保证通道对齐，同步扰码器，执行始终补偿，指示通道状态
* 协议与SerDes lanes数目和速率无关
* 包括带外和带内每通道流量控制选项，具有简单的ON/OFF语义
* 64B/67B数据编码和加扰
* 性能与通道数量成正比

## 3.2.基本概念
两种基本的数据结构定义了Interlaken协议：数据传输格式和Meta Frame。

数据传输格式显著依赖于SPI4.2协议概念。通过接口发送的数据被分割成Bursts，Burst是原始数据包的子集。每个Burst由两个控制字限定，前后各一个，并且控制字内的字段影响其后面或前面的用于诸如包起始，包结束，错误检测等功能的数据。每个Burst与逻辑channel相关，channel可以表示系统中的物理端口或一些其他逻辑上连接的数据流。报文通过一个或多个Burst顺序地发送，并且Burst的大小可以配置。通过将数据包分割成Burst，接口允许不同channel的数据交织传输以提升性能，降低延迟。

Meta Frame用于支持通过SerDes基础设施的数据传输。它包括一组四个唯一的控制字，用于提供通道对齐、扰频器初始化、始终补偿和诊断功能。Meta Frame利用数据传输的带内运行，使用控制字的特定格式来讲其与数据区分开。

下面详细描述数据传输格式和Meta Frame部分。
## 3.3.协议层
### 3.3.1.传输格式
数据通过可配置数量的SerDes通道在Interlaken接口传输。在本文档中，通道被定义为两个IC之间的单工串行链路。协议被设计为可以使用任何数量的lane，包括只有一个lane，但没有最大数目限制。实际使用中可以选择将他们的操作固定到特定数量的通道，没有要求支持可变通道数目。

通过接口发送的数据的基本单位是8字节字。选择该数字以符合为协议选择的64/67B编码，并且也是用于描绘Burst的控制字的大小。通过使基本传送单元等于控制字大小，容易调整接口的宽度。

数据和控制字被顺序地条带穿过lane，从lane 0开始，在lane M结束，并且下一个数据重复这样的操作。下图展示了这一过程：

![Figure 3 1 Lane Striping Example
](http://upload-images.jianshu.io/upload_images/7246758-3ff5292286693a2f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

64B/67B编码在每个通道上独立发生。传输通过两种基本数据类型实现：数据字和Burst/Idle控制字，他们通过64/67B成帧位进行区分。这两种数据字类型的格式如下图所示：

![Figure 3 2 Word Formats](http://upload-images.jianshu.io/upload_images/7246758-5a6d3b2ca6555854.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

数据和控制信息都是以位66～0的顺序传输的，框架层引入了4个附加控制字，详细信息后面将描述。

### 3.3.2.Burst结构
#### 3.3.2.1.数据传输过程
Interlaken的带宽在支持的channel上被划分为Bursts。数据包通过一个或多个Burst在接口上传输。Burst通过一个或多个控制字来描述。下一节将描述控制字的格式。

为了将任意大小的数据包分割成Burst，定义以下两个参数：
* BurstMax：Burst的最大大小（64B的倍数）
* BurstShort：Burst的最小大小（最小32B，增量为8B）

接口通常在控制字之后发送BurstMax长度的Burst。发送设备中的调度逻辑可以根据流控制状态信息来自有自由选择服务信道的顺序。Burst在每个信道上依次发送，直到报文被发送完，然后可以开始新的报文传输。

因为接口是信道化的，所以当每个信道上具有非常少量的数据时，分组结束可能在几个信道上背靠背发生。由于发送器和接受器存储器可以理想的设计，因此他们可能需要以非常高的速率来处理这中情况。为了减少接收器和发送器的负担，BurstShort参数保证了连续的Burst控制字之间的间隔。最小BurstShort间隔为32B，以8B为增量。

下图说明了BurstShort如何保证的最小间隔。通过在下一个Burst控制字之前添加额外的空闲控制字来强制执行BurstShort。在下面的例子中，空闲控制字1的EOP_Format指示EOP和最后数据字的大小，空闲控制字1的CRC24包括最后数据字和空闲控制字1。插入空闲控制字2和空闲控制字3以维持BurstShort，并且紧跟着的Burst控制字与其后发送的数据相关。

![Figure 3 3 BurstShort Guarantee Illustration](http://upload-images.jianshu.io/upload_images/7246758-3f3400f8583cd97b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

##### 3.3.2.1.1.Optional Scheduling Enhancement
上述简单的调度导致在报文长度和BurstMax的某些组合下，结束时会有带宽利用不到的情况。当报文长度 / BurstMax的值很小时，使得在最后一个BurstMax之后还剩余少量数据要传输，此时必须发送较多额外的空闲字以实施BurstShort保证。在最坏的情况下，浪费的带宽等于（BurstShort-1）字节。但是，通过在分组中向前查看，以识别EOP的位置，可以实现更为高效的调度。以下过程说明了一种这样的机制，并且作为用于优化接口性能的可选项提供。

本文档引入了一个新的参数：
* BurstMini：定义为32B的倍数，受限于BurstMini<=BurstMax/2；BurstMini>=BurstShort

为了说明此参数的用法目的，定义如下参数：
* Packet_length=数据包的总长度
* Packet_remainder=一旦数据传输开始，仍要发送的数据包中的数据量
* Data_transfer=当前Burst上传输的数据量
* i=传输数据包所需的Burst数

决定burst尺寸计算的决策算法如下：
```
packet_remainder = packet_length
for (x=1; x <= i; x++) {
    if (packet_remainder >= BurstMax + BurstMin) then
        data_transfer = BurstMax
    else
        if (packet_length MOD BurstMax < BurstMin) && (packet_remainder > BurstMax) then
            data_transfer = BurstMax - BurstMin
        else
            data_transfer = packet_remainder
    packet_remainder = packet_remainder - data_transfer
}
```
这个功能可以保证报文的最后一个Burst是BurstMini和BurstMax之间的尺寸，从而避免了多个短结束报文分段造成的背靠背的问题。然而，为了使算法正确的工作，BurstMini不能超过BurstMax的一半。

举个例子，长度为513B的报文将通过Interlaken进行传送，BurstMax=256B，BurstMin=64B，在这种情况下，发送3个Burst
* Burst1=BurstMax=256B
* Burst2=MurstMax-BurstMin=256-64=192B
* Burst3=packet_remainder=65B

如果报文长度只有511B，那么只要发送两个Burst
* Burst1=BurstMax=256B
* Burst2= packet_remainder =255B
具体实现可以根据需求调整BrustMax和BurstMini参数，服从上面的约束。

这个可选的算法旨在指导实现引导到传输Burst的有效机制。但是，只要发射机遵循用于报文分段的不同过程，只要遵循BurstShort和BurstMax参数，则对接收逻辑没有额外的负担。例如，可能存在从一种接口类型转换到另一种接口类型的情况，其中重新格式化Burst将施加不必要的负担。其他调度算法是可能的，并且设计者可以根据上面定义的约束自由创建它们。

##### 3.3.2.1.2.Control Word Format
通过8B控制字来描述Burst。通过使用bit[66-64]和bit[63]=’1’的控制码在数据流中识别控制字。Burst和Idle控制字格式如下图所示：

![Figure 3 4 Control Word Format](http://upload-images.jianshu.io/upload_images/7246758-0799e826040c0ebb.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Table 3.1 Idle/Burst Control Word Format

|Field	|Bit Position	|Function|
|:------|:--------------|:-------|
|Inversion	|66	        |用于指示bit[63-0]是否已经反转以限制运行差异；1=反转                                                                 |
|Farming	|65:64	    |64B/67B机制区分控制字和数据字；01=数据；10=控制                                                                     |
|Control	|63	        |1表示为idle/burst控制字，0表示Framing层控制字                                                                       |
|Type	    |62	        |设置为1时，通道号和SOP字段有效，并且在控制字后有一个数据Burst；如果设置为0，通道号和SOP字段无效，并且控制字后无数据 |
|SOP	    |61	        |数据包开始。如果设置为1，控制字之后的Burst表示数据包的开始，如果设置为0，表示Burst是数据包的中间或结尾              |
|EOP_Format	|60:57	    |该字段指示控制字之前的Burst，它编码如下: '1xxx'-包结束，其中位[59:57]定义Burst中最后8字节字中的有效字节数。位[59:57]被编码，使得'000'表示8字节有效，'001'表示1字节有效等，'111'表示7字节有效, 有效字节从位位置[63:56];'0000'-无包结束，无ERR;'0001'-错误和数据包结束;所有其他组合未定义。|
|Reset Calendar	        |56	    |如果设置为1，表示带内流控制状态代表了通道日历的开启|
|In-Band Flow Control	|55:40	|当前16个channel的1位流控制状态;如果设置为“1”，则channel是XON，如果设置为“0”，则channel是XOFF|
|Channel Number	        |39:32	|与控制字之后的Burst相关联的信道; 空闲控制字时全为0|
|Multiple-Use	        |31:24	|此字段可能有多种用途，具体取决于应用程序。如果需要超过256的附加信道，这8个比特可以用作信道号扩展，表示信道号的8个最低有效位。如果需要额外的带内流控制位，这些位可用于表示在位[55:40]中表示的16个日历条目之后的8个日历条目的流控制状态。这些位也可以保留用于超出本说明书范围的特定用途。|
|CRC24	                |23:0	|覆盖先前burst（如果有）和此控制字的CRC错误检查|

Burst控制字（Type ='1'）标识数据Burst的开始。每个突发数据传输必须以突发控制字开头，这表示SOP和通道号字段适用于紧随其后的数据。当突发控制字落在数据突发之间时，EOP_Format和CRC字段适用于紧接在前的数据，SOP和通道号字段适用于紧随其后的数据（意图是类似于SPI4.2突发控制语义的操作）。

当没有新的数据可用于发送时，空闲控制字（Type ='0'）始终被发送。由于流量控制信息必须始终发送到接收设备，流控制字段在Idle和Burst控制字中都是有效的，并且发送器总是在两种类型的控制字中发送有效的流量控制状态。

突发控制字的EOP_Format字段识别突发的最后一个数据字有多少字节有效。无效的字节由接收方丢弃。按照惯例，第一个有效字节出现在位字段[63:56]，位字段[55:48]等的第二个有效字节等。

通过24位CRC确保数据和控制完整性。根据脉冲串中的所有数据和控制字中的所有字段计算CRC24。
CRC24多项式：
```
x^24 + x^21 + x^20 + x^17 + x^15 + x^11 + x^9 + x^8 + x^6 + x5 + x + 1
```
CRC计算的细节在附录B中规定。

### 3.3.3.状态图
下面的状态图说明了主要的逻辑操作。

接口的每个接收SerDes通道按照下图进行操作。

![Figure 3 5 Receive Per-Lane State](http://upload-images.jianshu.io/upload_images/7246758-81ca28994a27c1ce.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

接口的接收端（所有通道捆绑在一起作为逻辑整体）根据以下操作：

![Figure 3 6 Receive Interface State](http://upload-images.jianshu.io/upload_images/7246758-44b906084f1fc66c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

一旦接口的接收端进入RX操作状态，接口就可以通过在所有通道上发信号通知ON状态来向发射机发布许可。

接口的发送侧（作为整个逻辑接口捆绑在一起的所有通道）根据以下操作：

![Figure 3 7 Transmit Interface State](http://upload-images.jianshu.io/upload_images/7246758-781cf091f5fb8c19.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 3.3.4.流控
Interlaken的一个重要特性是能够实现每通道背压。为了提供此功能，指定了两个选项：带外流控制接口和带内信道。语义上，流控制信息使用简单的开关机制来发信号通知在特定信道上的传输许可。
#### 3.3.4.1.协议
开/关流控制状态是由每个支持的通道的单个状态控制位决定的。按照惯例，使用1表示XON状态，指示传输器在该通道上可以允许发送数据。用0表示XOFF状态，指示传输器在该通道上停止传输数据。

一旦通道被指示为XON，则传输器可以发送数据，直到流控制状态变为XOFF为止。接收器选择在XON和XOFF之间切换的阈值是留给用户可编程的选项，并且取决于通道数目，接收缓冲器的深度和给定环境的流控延迟。

流控制信道可以选择被映射到日历上，使得流控制可以被映射到任何日历条目集合。作为示例，映射可以包括信道到日历条目的一对一映射，一对多映射以增加某些信道的频率，或者插入空字段以匹配具有不同频率的设备。

该日历结构还可以用于提供链路级的流控制，由此日历中的位表示在接口上作为整体发送的许可。链路状态的极性将与信道状态的极性相同，1指允许发送，0指停止发送。要启用此功能，可以为通道信息或链接信息配置每个日历条目。为了促进低延迟链路状态，接口需要提供足够的日历条目以在这些字的相同位位置中的每个Burst/Idle控制字中编程链路状态。

举个例子，使用超过16个通道，可以通过如下步骤来执行：
```
第一控制字：
日历条目0 =链接状态
日历条目1 =通道0状态
日历条目2 =通道1状态
...等等。
第二控制字：
日历条目15 =链接状态
日历条目16 =通道15状态
...等等。
```
使用此种方法，链路状态将始终出现在Burst/Idle控制字的bit[55]中。

#### 3.3.4.2.带外流控
为了支持需要单工操作的系统，定义了带外流量控制选项。它使用源同步接口来实现，具有如下信号：
* FC_CLK：流控数据同步时钟
* FC_DATA：流量控制状态信息（单个bit）
* FC_SYNC：用于识别流量控制日历开头的同步信号
这些信号的电平标准可以是LVDS或LVCMOS。其逻辑时序如下所示：

![Figure 3 8 Out-of-Band Logical Timing Diagram](http://upload-images.jianshu.io/upload_images/7246758-3cc95840b4e100b5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

带外流控通道通过4位CRC计算进行保护，涵盖多达64位的流量控制数据。CRC4多项式为：
```
x^4 + x + 1
```
CRC从MSB到LSB（CRC [3]到CRC [0]）。当通道数为64个或更少时，CRC4校验和紧跟在最后一个日历插槽之后，紧接着是日历插槽0的流程控制状态。当日历插槽的数量大于64时，每64位的流量控制状态CRC4校验和发生一次。对于最后一组日历插槽，CRC4校验和发生在最后一个支持的日历插槽之后，紧接着是日历插槽0状态。

如图所示，FC_CLK用于在上升沿和下降沿时处理FC_DATA。在最大速率为100 MHz的情况下，对于支持48个信道和24 Gbps的实现，最差情况：
```
FC_CLKperiod = 10 ns
Time in flight = (10 ns) / (2 bits/clk) * (48 channels + 4 CRC bits) = 260 ns
Data in flight = (260 ns) * (24 Gbps) = 780 bytes
```
对于支持256通道和24 Gbps的实现，最差情况：
```
Time in flight = (10 ns) / (2 bits/clk) * (256 channels + 16 CRC4 bits) = 1.36 µsec
Data in flight = (1.36 µsec) * (24 Gbps) = 4.08 KB
```

##### 3.3.4.2.1.带外流控接口时序
本节描述带外流控接口的时序参数。
为了提供最大采样窗口，FC_DATA和FC_SYNC信号以两倍于时钟频率的数据速率发送，并相对于FC_CLK信号以正交相位发送。注意：时钟的上升沿和下降沿的时序关系是相同的。

![Figure 3 9 Out-of-Band Flow Control Timing Diagram](http://upload-images.jianshu.io/upload_images/7246758-2bc67e9cd600359d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Table 3.2 Out-of-Band Flow Control Interface Timing

|Symbol	|Parameter	|Min	|Type	|Max	|Units|
|:------|:----------|:------|:------|:------|:----|
|F_clk  | FC_CLK时钟频率	                          |0   |	            |100  |MHz |
|	    | FC_CLK时钟占空比	                          |45  |	            |55	  |%   |
|T_isu	| 输入数据/同步建立时间w.r.t时钟沿。	      |0.75|		        |     | ns |
|T_ih	| 输入数据/同步保持时间w.r.t时钟沿。	      |0.75|		        |     | ns |
|T_on	| 下一个输出数据/同步无效时间w.r.t下一个时钟边沿的上一个正交点。|  |  |0.75 | ns |
|T_op	| 以前的输出数据/同步无效时间w.r.t. 上一个时钟沿的正交点。	    |  |  |0.75 | ns |

#### 3.3.4.3.带内流控
当使用此选项时，接收机利用在通过接口发送的控制字中发送的流量控制状态作为正常数据传输的一部分。该选项适用于需要最少数量的外部信号引脚的全双工实现。

如图所示，控制字的流控制字段为16位，位[55:40]。控制字的位[31:24]也可以用于流量控制的8位，总共为24位。这些状态位表示每个Interlaken日历通道的ON-OFF流量控制状态，当前日历项X位[55]，位[54]上的日历项X + 1，等等。为了同步日历的开始，复位日历位在空闲/突发控制字中提供;当该位为“1”时，通道0状态出现在位[55]中。当重置日历为“0”时，日历将继续从上一个控制字中停止的日期继续。一旦所有通道的状态都被通信，发送器将设置复位日历位并重复序列。在日历的最后一个控制字（即，当信道数量不是状态比特数的倍数）时不需要额外的比特，接收机将被忽略，并由发射机设置为0。

由于控制字CRC24覆盖流控制字段，因此不需要进行独立的错误检查，并且在此处不保留对带外选项执行的CRC4计算。

流控制信息始终以空闲和突发控制字的形式发送。

由于在每个突发数据传输之间发送控制字，流量控制信息的最差情况频率是每个最大突发长度一个消息。实现者要选择所需流量控制带宽所需的BurstMax。

作为性能计算的例子，对于具有256字节突发和48的接口通道，日历传输期间的飞行中的数据是：
Data in flight = (2 bursts) * (256 bytes/burst) + (2 control words) * (8 bytes/control word) = 528 Bytes

#### 3.3.4.4.全分组模式流控
虽然因Interlaken被优化以通过交织来自不同信道的传输进行操作，但它也适用于需要完整分组传输的应用。对于这些应用，发送设备简单地避免了从一个信道到另一个信道的切换，直到当前信道的分组完成传输。

在全分组模式下有两种流量控制的解释：在收到XOFF消息后立即停止传输，或在停止传输之前完成当前数据包。第一种解释减少了在响应流控制之前所需的接收机缓冲，以牺牲接口中的线路阻塞其他信道为代价; 第二个解释提供相反的权衡。 因为不同的应用程序需要不同的行为，所以这个规范在合规实现中留下了选择这两种解释的可能性。

#### 3.3.4.5.流控扩展
有些应用程序可能希望实现与Interlaken的XON / XOFF提供的不同的流控制方法; 例如，这可能涉及使用由发射机和接收机交换的显式信用。Interlaken不是试图直接指定这一点，而是将其作为可以使用附加信道来实施的更高层功能。通过将该扩展作为数据有效载荷的一部分，可以通过Interlaken接口设计和可靠地传输任何更高层的协议。