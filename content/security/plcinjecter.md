Title: 利用Modbus PLC作为攻击载荷分发系统
Date: 2017-07-18 14:36
Tags: 工控技术,PLC注入
Authors: chansim


##一、	介绍

随着互联网+、工业4.0的推进，以前的网络边界被逐渐打破，工业控制系统的安全问题越来越突出，近期针对控制器的攻击层出不穷，如 [PLC逻辑炸弹](http://icsmaster.com/news/logicblob.html)、[PLC勒索软件](http://icsmaster.com/security/ClearEnergy.html)，[PLC蠕虫](http://icsmaster.com/security/plcblaster.html)，相关的技术我们一直在跟踪分析。

本文主要介绍怎么利用Modbus PLC作为攻击载荷，将恶意代码隐藏在控制器(programmable logic controllers)中，从而躲避当前的主流的网络攻击检测，提高攻击的隐蔽性，使得网络攻击难发现以及难溯源。下文中主要使用是施耐德电气的控制器TM221，该设备使用的主要的通信协议是Modbus Tcp，被中小型制造工厂广泛使用，以实现制造工艺的自动化。由于Modbus Tcp协议存在多个安全问题，利用Modbus Tcp协议可以实现任意上传和下载内存数据，因此，可以将攻击载荷上传至PLC内存中，从而躲避常规的安全检测。


##二、	利用modbus-cli操作plc寄存器

本文中我们使用的控制器，如下图所示：

![Alt text](/static/images/plcinjector/2.png)

使用关键字“TM221ME16R”，在Shodan可以查询到当前暴露在互联网上的施耐德电气，如下图所示。**注意：本文仅做技术研究，相关工具请务在真实工业环境中使用，否则后果自负。** 

![Alt text](/static/images/plcinjector/1.png)

Modbus-cli是一个命令行（cli）工具，使用此工具可以对PLC 寄存器进行读写操作，使用方法如下图所示：

运行指令参数：modbus [OPTIONS] SUBCOMMAND [ARG]

![Alt text](/static/images/plcinjector/3.png)

我们至少有两种方法获取寄存器地址里面的值，分别为：Schneider address和Modicon addess。 如下表所示，Modicon addess从地址前的％M开始。

| Data type | Data size | Schneider address | Modicon addess | Parameter |
| --------- | --------- | ----------------- | -------------- | --------- |
| Word      | 16 bits    | %MW100           | 400101         | --word    |
| Interger  | 16 bits    | %MW100           | 400101         | --int     |
| Floating point  | 32 bits    | %MF100     | 400101         | --float   |
| Double word   | 32 bits    | %MD100       | 400101         | --word    |
| Boolean(coils)      | 1 bits    | %M100   | 101            | N/A       |

modbus read <IP> %MW100 10，意思是读取以%MW100开头的十个值；我们可以通过modbus-cli从plc中读取指定的寄存器中的值，执行如下图所示：

![Alt text](/static/images/plcinjector/4.png)

为了达到相同的效果，也可以换种形式来获取，如下图所示：

![Alt text](/static/images/plcinjector/5.png)

读取线圈（coils）的值，命令如下

    modbus read 192.168.1.31 %M100 10 

可以看出线圈全都处于关闭状态，如下图所示：

![Alt text](/static/images/plcinjector/6.png)

modbus-cli不仅可以读取plc寄存器数据还可以更改，如若被不法份子利用直接可以更改里面的状态值，命令如下：

    modbus write 192.168.1.31 %M100 1 1 1 1 1 1 1 1 1 1

执行结果如下图所示

![Alt text](/static/images/plcinjector/7.png)

修改PLC寄存器的值时，可能会导致上位机和终端设备的异常运行，下图所示是执行前后

正常运行状态如下图所示：

![Alt text](/static/images/plcinjector/8.png)

修改寄存器后组态界面出现异常

![Alt text](/static/images/plcinjector/9.gif)


##三、	利用ISF攻击框架进行PLC注入

利用plc存储payload/shellcode优势：
由于payload存放在PLC的内存中，所以加大了取证分析的难度。此外，一旦payload被取出，其内容直接被写入内存，这样也加大了取证难度。
此外，我认为Modbus Stager在某些ICS环境中也是非常有用的，因为这些环境下Modbus之外的协议会引起人们的警觉，并且WinHTTP / WinInet stager也不是最适用的。所以，在这种情况下，只需要一个和modbus处理器（stager），读取plc里面的数据并执行。
攻击流程如下：

![Alt text](/static/images/plcinjector/10.png)

1.	attacker将payload/shellcode上传至modbus plc，如果没设定上传得初始地址，那么就从x00开始，在这里其实也可以使用其他工具或者交互脚本，如实验一里面的modbus-cli。攻击者一般会选择第三方的plc，因为第三方的plc具有良好的匿名性质，跟踪难度较大，无须将payload上传至服务器。
2.	attacker 利用其他方式漏洞或钓鱼等将stager传入工控环境的windows机器并触发运行。
3.	stager运行之后会与modbus plc数据交互，读取plc保持寄存器里面的payload/shellcode并执行。stager是由汇编编译主要功能是和modbus plc建立socket通信接收payload并在内存执行，总共大小只有1KB，难以被沙软检测，更何况是安全相对薄弱的工控环境。


Payload 生成：
Payload是“Functional Keylogger to File Null-Free Shellcode” 长度为0x0259，在生成的时候前四个字节代表shellcode的长度，如图2-2-2所示：

![Alt text](/static/images/plcinjector/11.png)

为了将payload上传至plc中，plcInjectPayload会根据加载的控制策略不同，对plc的可用内存大小的要求也有所变化，因此该脚本首先检查它们是否有足够的内存空间来存放相应的payload。为了检测内存的大小，可以发送操作功能代码为03（读取保持寄存器）的Modbus请求，尝试从某个地址读取特定记录（每个记录长度为16比特）。如果收到一个0x83异常，那么说明这个PLC对于我们来说是无法使用的。

Modbus 协议的功能代码如下图所示

![Alt text](/static/images/plcinjector/13.png)

使用ISF下载控制器寄存器，执行操作如下

![Alt text](/static/images/plcinjector/12.gif)

读取到的内容如下图所示

![Alt text](/static/images/plcinjector/14.png)

上传payload至plc，在本次实验没有指定地址，就从0x00开始上传。上传之后的结果如图所示：

![Alt text](/static/images/plcinjector/15.png)

执行stager 获取plc保持寄存器里面payload加载至内存，并执行：

Stager入口点：

![Alt text](/static/images/plcinjector/16.png)


Socket连接，对应的连接数据如下
  0x1F01A8C0 -> 192.168.1.31 
  0xF6010002 -> port :502

![Alt text](/static/images/plcinjector/17.png)

接收0x259个字节：

![Alt text](/static/images/plcinjector/18.png)


该stager分三次接收数据：

![Alt text](/static/images/plcinjector/19.png)

接收的payload写入内存并执行：

![Alt text](/static/images/plcinjector/20.png)

##四、参考

[1] Modbus Stager: Using PLCs as a payload/shellcode distribution system  [链接](http://www.shelliscoming.com/2016/12/modbus-stager-using-plcs-as.html)
[2] Functional Keylogger [链接](https://www.exploit-db.com/exploits/39794/)
[4] ISF攻击框架 [链接](https://github.com/w3h/isf)


