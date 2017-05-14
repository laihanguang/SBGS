# 目录
1——实验拓扑

2——代码运行

3——代码结构

----------


## 1 实验拓扑
![](http://i.imgur.com/sxMDbCO.png)

## 2 代码运行

### 2.0 启用ECN

打开/etc/sysctl.conf，在里面添加一行

	net.ipv4.tcp_ecn = 1
并且不能被“#”号注释

注：目前东主楼的四台物理机都已启用，无需再次设置。其他机器在修改此文件后**必须重启**以使命令生效！

### 2.1 设置限速命令

**接收端：**

在接收端宿主机，根据《Open VSwitch安装手册》启动ovs，并创建好网桥br0及设备tap1、tap2；

根据《ovs限速归纳》，新建qos，速率以bit为单位

	ovs-vsctl set port tap1 qos=@newqos -- --id=@newqos create qos type=linux-htb other-config:max-rate=100000000

接着使用

	ovs-vsctl list qos
获取相应qos的uuid，将uuid存储到接收端point目录下的cmd.py中作为全局变量

若要修改速率值则可使用如下命令，其中xxxxx为uuid

	ovs-vsctl set qos xxxxx other-config:max-rate=100000000


**发送端：**

在每台物理机启动三个虚拟ip

	ifconfig eth0:0 192.168.1.31/24 up
	ifconfig eth0:1 192.168.1.32/24 up
	ifconfig eth0:2 192.168.1.33/24 up

根据《TC限速归纳》，为每个虚拟ip地址设置tc限速命令，以下以eth0:0为例

	tc qdisc add dev eth0:0 root handle 1: htb r2q 1
	tc class add dev eth0:0 parent 1: classid 1:1 htb rate 30mbit
	tc filter add dev eth0:0 parent 1: protocol ip prio 16 u32 match ip src 192.168.1.31 flowid 1:1
其中tc qdisc命令只需设置一次，eth0:1及eth0:2时无需再次设置，每个虚拟ip的classid和flowid必须与其他ip不同

### 2.2 加载内核模块
在接收端每个VM中：
	
	sudo su
	cd rcv_hook/
	make clean
	make
	insmod s_hook4.ko

在每个发送端物理机：

	sudo su
	cd send_hook/
	make clean
	make
	insmod s_hook4.ko

### 2.3 运行用户态python获取内核数据

在接收端VM的rcv_hook目录下：

	python s_hook4_rcv.py

在发送端物理机的send_hook目录下：

	python s_hook4_send.py

### 2.4 运行SBGS核心算法

在接收端宿主机的point目录下（本次实验中为point2/）：

	python Run.py

在发送端宿主机的point目录下（本次实验中为point3/和point4/）：

	python Run.py

### 2.5 启动iperf发包

实验中iperf常用命令：

	-i  发送report到终端的时间间隔
	-t  发包总时间
	-B  指定虚拟ip
	-p  指定端口
	-n  指定发送的总数据量
	-u  发送UDP
	-b  指定UDP发送带宽
	-P  指定发送的流的数量

**注**：若要使用-P指定流的数量，则无法使用虚拟ip，必须使用真实ip（还须将代码预定义宏中的ip做相应修改），原因是虚拟ip实际上相当于真实ip的不同端口，而一个端口是无法发送多条流的。
### 2.6 重新编译

修改代码之后需要重新编译，在接收端VM（发送端物理机同理）的rcv_hook目录下：

	rmmod s_hook4
	make clean
	make
	insmod s_hook4.ko


## 3 代码结构

### 3.1 Makefile

编译文件

### 3.2 s_hook4.c

编写hook函数的内核代码，发送端与接收端大同小异。主要目的为统计包的数量、长度、种类，计算出流入、流出速率。

特别重要的是，若当前为TCP流，则会统计ECN包并计算出ECN比例。

将所有统计数据通过socket发送到用户态，由python程序负责接收。

### 3.3 s\_hook4\_rcv.py

运行在接收端VM。主要目的是使用socket接收从内核态发来的数据信息。

将TCP入、出速率，UDP入、出速率，ECN比例发送给接收端宿主机，由运行在宿主机的SBGS中的python负责接收。

将UDP入速率及ECN比例作为反馈信息发给发送端，由相应SBGS的python接收。

### 3.4 s\_hook4\_send.py

运行在发送端。主要目的是使用socket接收从内核态发来的数据信息。

将TCP入、出速率，UDP入、出速率，ECN比例发送给本机（127.0.0.1），由运行在本机的SBGS中的python负责接收。

