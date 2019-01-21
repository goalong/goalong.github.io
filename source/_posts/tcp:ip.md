---
title: 计算机网络笔记
date: 2018-05-24
tags: 
	- 网络 

---


### udp
无连接的面向数据报的运输层协议，不提供可靠性，只是发送出数据报，不保证能到达目的地，不用在客户端和服务器之间建立连接，也没有超时重发等机制，故而传输速度很快。

每个数据报是一个独立的信息，包括完整的原地址和目的地址，能否到达目的地及到达的时间，内容的正确性等都不能被保证。类似于写信。

一般用于多点通信和实时的数据业务

适用场景：
语音广播
视频
qq
dns
股票市场 航空信息等

### socket
socket.socket(AddressFamily, Type)
AddressFamily: AF_INET(internet间进程间通信）， AF_UNIX(同一台机器的进程间通信）
Type: 套接字类型，SOCK_STREAM（流式套接字，tcp), SOCK_DGRAM(数据报套接字，udp)

ip地址 端口号 协议可以标识网络中的进程

udp服务端：

	import socket 
	server_sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
	server_addr = ('', 8000)
	server_sock.bind(server_addr)
	while True:
		receive_data, client_addr = server_sock.recvfrom(1024):
		msg = receive_data.decode("utf-8")
		if msg == "quit":
			server_sock.close()
			break
			
udp客户端：

	import socket 
	client_sock = socket.socket(AF_INET, SOCK_DGRAM)
	server_addr = ("127.0.0.1", 8000)
	data = input("")
	while data:
		client_sock.sendto(data.encode("utf-8"), server_addr)
		data = input("")
	client_sock.close()
	
没有为客户端绑定端口，操作系统会随机分配


### TCP

特点：
1. 面向连接，先建立连接才能进行传输，双方为连接分配必要的系统内核资源
2. 基于数据流， 发送端进行多次写操作时，tcp模块先把数据放到tcp发送缓冲区，
3. 可靠传输，发送应答机制，每个报文需要接收方的应答才认为传输成功；超时重传， 发送端发出一个报文段之后就启动定时器，如果在定时时间内没有收到应答就重新发送这个报文段； 错误校验， tcp用一个校验和函数来检验数据是否有错误，发送和接收都要计算校验和；流量控制和阻塞管理， 流量控制用来避免主机发送的过快而使接收方来不及完全收下

tcp与udp的不同点

面向连接
有序数据传输
重发丢失的数据包
舍弃重复的数据包
无差错的数据传输
阻塞／流量控制

### 三次握手

![tcp图示](https://aceld.gitbooks.io/python/content/assets/tcp07.png)

tcp用三次握手创建一个连接， tcp通信的报文称为段

1. 客户端发送一个带syn的tcp报文到服务器，syn位表示连接请求，序号是一个随机数，每发一个数据字节，序号要加1，这样接收端可以根据序号排列数据包的正确顺序，也可以发现丢包，mss表示最大段尺寸，客户端声明最大段尺寸，建议服务端发来的段不要超过这个长度
2. 服务器端响应客户端，是三次握手的第二个报文段，同时带ack标识和syn标示，表示是对刚才syn的回应，同时发送的syn询问客户端是否准备好进行数据通信。
服务器发出段2，也带有SYN位，同时置ACK位表示确认，确认序号是1001，表示“我接收到序号1000及其以前所有的段，请你下次发送序号为1001的段”，也就是应答了客户端的连接请求，同时也给客户端发出一个连接请求SYN，序号是8000（实际也是一个随机数，此处以8000为例），同时声明最大尺寸为1024。

3. 客户必须再次回应服务器端一个ACK报文，这是报文段3，客户端发出段3，对服务器的连接请求进行应答，确认序号是8001。

这个过程中，客户端和服务端分别给对方发送了连接请求，也应答了对方的连接请求，其中服务端的请求和应答在一个段中发出，因此共有三个段用于建立连接，成为三次握手，在建立连接的同时，双方协商了一些信息，如发送序号的初始值，最大段尺寸等

### 数据传输
1. 客户端发出段4，包含从序号1001开始的20个字节数据。
2. 服务器发出段5，确认序号为1021，对序号为1001-1020的数据表示确认收到，同时请求发送序号1021开始的数据，服务器在应答的同时也向客户端发送从序号8001开始的10个字节数据。
3. 客户端发出段6，对服务器发来的序号为8001-8010的数据表示确认收到，请求发送序号8011开始的数据。

在数据传输过程中，ACK和确认序号是非常重要的，应用程序交给TCP协议发送的数据会暂存在TCP层的发送缓冲区中，发出数据包给对方之后，只有收到对方应答的ACK段才知道该数据包确实发到了对方，可以从发送缓冲区中释放掉了，如果因为网络故障丢失了数据包或者丢失了对方发回的ACK段，经过等待超时后TCP协议自动将发送缓冲区中的数据包重发。

### 四次挥手

由于TCP连接是可以双向通信的（全双工），因此每个方向都必须单独进行关闭。

这原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个 FIN只意味着这一方向上没有数据流动，一个TCP连接在收到一个FIN后仍能发送数据。首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。

1. 客户端发出段7，FIN位表示关闭连接的请求。
2. 服务器发出段8，应答客户端的关闭连接请求。
3. 服务器发出段9，其中也包含FIN位，向客户端发送关闭连接请求。
4. 客户端发出段10，应答服务器的关闭连接请求。

建立连接的过程是三次握手，而关闭连接通常需要4个段，服务器的应答和关闭连接请求通常不合并在一个段中，因为有连接半关闭的情况，这种情况下客户端关闭连接之后就不能再发送数据给服务器了，但是服务器还可以发送数据给客户端，直到服务器也关闭连接为止。


连接释放的时候，实际上是其中一方，如A提出了释放请求，B收到后进行了报文确认，那么A到B的连接就释放了，这个时候，B还是可以向A发送数据，而且A会直接接收，到了B数据发送完了，B再向A发送请求释放的请求，B进入超时等待阶段，如果A及时发送连接中断确认，那么B就释放了B到A的连接，如果没有收到，超过一段时间，自动释放。


### tcp服务端客户端实现

服务端

	import socket
	server_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	server_sock.bind(("", 8000))
	
	server_sock.listen(120)
	client_sock, client_addr = server_sock.accept()
	recv_data = client_sock.recv(1024)
	client_sock.send("got it".encode())
	client_sock.close()
	server_sock.close()

客户端

	import socket
	# 创建客户端socket用以跟服务器连接通信
	# tcp协议对应为SOCK_STREAM
	client_sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
	
	# connect方法用来连接服务器
	server_addr = ("127.0.0.1", 8000)
	client_sock.connect(server_addr)
	
	# 提示用户输入要发送的数据
	msg = input("请输入要发送的内容：")
	# send()方法想服务器发送数据
	client_sock.send(msg.encode())
	
	# recv()接收对方发送过来的数据，最大接收1024个字节
	recv_data = client_sock.recv(1024)
	print("收到了服务器的回应信息：%s" % recv_data.decode())
	
	# 关闭客户端套接字
	client_sock.close()
	
	
### TCP十种状态

### TCP长连接短连接
长连接可以省去较多的连接建立和关闭的操作
短连接对于服务器的管理较为简单，所有的连接都是有效的连接

应用场景：
长连接多用于操作频繁，点对点的通讯，而且连接数不能太多情况， 比如数据库连接
web的http服务一般用短连接

### 服务器模型



	





