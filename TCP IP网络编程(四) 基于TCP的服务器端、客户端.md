
## 理解TCP、UDP

### TCP/IP协议栈

​																**TCP/IP协议栈**

![请添加图片描述](https://img-blog.csdnimg.cn/27175fda09a64d71b6eb120801ae45e5.png)


TCP/IP协议栈共分为4层，可以理解为数据收发分成了4个层次化过程。

​											**TCP协议栈**

![在这里插入图片描述](https://img-blog.csdnimg.cn/22aa0b33105c4be7b65fb227198090aa.png)


​										**UDP协议栈**

![在这里插入图片描述](https://img-blog.csdnimg.cn/5e40bea55ea548208e74877cef2d8231.png)


### 链路层

链路层是物理连接领域标准化的结果，也是最基本的领域，专门定义LAN、WAN、MAN等网络标准。两台主机通过网络进行数据交换，这需要像下图所示的物理连接，链路层就负责这些标准。

![在这里插入图片描述](https://img-blog.csdnimg.cn/67779bb2e56a46c8b158302b0b145e1f.png)


### IP层

IP协议是面向消息的，不可靠的协议。每次传输数据时会帮我们选择路径，但并不一致。如果传输中发生路径错误，则选择其他路径。但如果发生数据丢失或错误，则无法解决。IP协议无法应对数据错误。

### TCP/UDP层

IP层解决数据传输中路径选择问，只需要按照此路径传输数据即可。TCP和UDP层以IP层提供的路径信息为基础完成实际的数据传输，故该层又称为传输层。TCP可以保证可靠的数据传输，但它发送数据时以IP层为基础。

![在这里插入图片描述](https://img-blog.csdnimg.cn/69b5943e169b47c0a25141f22e2e468c.png)


TCP和UDP存在于IP层之上，决定主机之间的数据传输方式，TCP协议确认后向不可靠的IP协议赋予可靠性。

### 应用层

以上类容是套接字通信过程中自动处理的。选择数据传输路径、数据确认过程都被隐藏到套接字内部。但掌握了这些理论，才能编写出符合需求的网络程序。

向大家提供的工具就是套接字，大家只需要利用套接字编写出程序即可。编写软件的过程中，需要根据程序特点决定服务器端和客户端之间的数据传输规则，这便是应用层协议。网络编程的大部分内容就是设计并实现应用层协议。

## 实现基于TCP的服务器端、客户端

### TCP服务器端的默认函数调用顺序

```c++
1、socket()    		创建套接字
2、bind()			分配套接字地址
3、listen()			等待连接请求状态
4、accept()			允许连接
5、read()/write()	数据交换
6、close()			断开连接
```

### 进入等待连接请求状态

我们已经调用bind函数给套接字分配了地址，接下来通过调用listen函数进入等待连接请求状态，只有调用了listen函数，客户端才能进入可发出连接请求的状态，这时客户端才可以调用connect函数。

```c++
#include<sys/socket.h>

int listen(int sock, int backlog);
	成功返回0，失败返回-1
    sock	为希望进入等待连接请求状态的文件描述符，传递的描述符套接字参数为服务器端套接字
    backlog 连接请求等待队列的长度，若为5，则队列长度为5，表示最多使5个连接请求进入队列
```

等待连接请求状态是指客户端请求连接时，受理连接前一直使连接处于等待状态，客户端连接请求本身也是网络中收到的一种数据，而想要接受就需要套接字。

### 受理客户端连接请求

调用listen函数后，有新的连接请求，则应按序受理。受理请求意味着进入可接受数据的状态，此时就需要套接字来接受数据，但服务器端的套接字在做门卫，不能再充当接受数据的角色。因此需要另外一个套接字，该套接字不需要亲自创建，accept函数将会创建套接字并连接到发起请求的客户端。

```c++
#include <sys/socket.h>

int accept(int sock, struct sockaddr* addr, socklen_t* addrlen);
	成功返回创建的套接字文件描述符，失败返回-1
    sock  	服务器套接字的文件描述符
	addr  	保存发起连接请求的客户端地址信息的变量的地址，调用函数后会向该变量填充客户端地址信息
	addrlen	第二个参数addr结构体的长度，调用函数后会向该变量填充客户端地址长度
```

accept函数受理连接请求等待队列中待处理中的客户端连接请求。函数调用成功时，accept函数内部将产生用于数据I/O的套接字，并返回文件描述符，套接字使自动创建的，并且自动与发起连接请求的客户端建立连接。

### TCP客户端的默认函数调用顺序

TCP客户端函数的调用顺序

```c++
1、socket()			创建套接字
2、connect()			请求连接
3、read()/write()	交换数据
4、closr()			断开连接
```

与服务器端相比，区别就在于请求连接，它使创建客户端套接字后向服务器端发起的连接请求，服务器端调用listen函数后创建请求等待队列，之后客户端即可请求连接。

```c++
#include<sys/socket.h>

int connect(int sock, struct sockaddr* servaddr, socklen_t addrlen);
	成功返回0，失败返回-1
     sock  		客户端套接字文件描述符
	 servaddr	保存目标服务器地址信息的变量地址值
	 addrlen	以字节为单位，传递第二个参数的地址变量的长度
```

客户端调用connect函数后，发生以下情况才会返回

- 服务器端接受连接请求
- 发生断网等异常情况而中断连接请求

接受连接请求并不是服务器端调用accept函数，其实是服务器端把连接请求信息记录到等待队列中，因此connect函数返回后并不立即进行数据交换

### 基于TCP的服务器端、客户端函数调用关系

![在这里插入图片描述](https://img-blog.csdnimg.cn/263fcf400ef8419bbcdb049dc04b0761.png)


总体流程如下：

​	服务器端创建套接字后连续调用bind、listen函数进入等待状态，客户端通过调用connect函数发起连接请求，客户端侄女等到服务器端调用listen函数之后才能调用connect发起连接请求，也要主义客户端调用connect函数前，服务器端可能率先调用accept函数，此时服务器端调用accept函数进入阻塞状态，知道客户端调用connect函数为止。

## 实现迭代服务器端、客户端

### 实现迭代服务器端

![在这里插入图片描述](https://img-blog.csdnimg.cn/5fb4b20367e243eb9e97e1976f06cefd.png)


实现迭代服务器端最简单的办法就是插入循环语句反复调用accept函数。循环最后的close(client)关闭的调用accept函数创建的套接字，意味着结束了针对某一客户端的服务，此时如果还想服务于其他客户端，就要重新调用accept函数。目前同一时刻只能服务于一个客户端，学完进程和线程后，就可以编写同时服务于多个客户端的服务器端。

### 迭代回声服务器端、客户端

回声服务器端以及配套的回声客户端的程度的基本运行方式：

- 服务器端在同一时刻只与一个客户端相连，并提供回声服务。
- 服务器端依次向5个客户端提供服务并退出。
- 客户端接受用户的输入字符串并发送到服务器端。
- 服务器端将收到的字符串数据传回客户端，即“回声”。
- 两端之间的字符串回声一直执行到客户端输入Q为止。

**首先介绍满足以上要求的回声服务器端**

*echo_server.c*

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 1024
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int serv_sock, clnt_sock;
	char message[BUF_SIZE];
	int str_len, i;
	
	struct sockaddr_in serv_adr;
	struct sockaddr_in clnt_adr;
	socklen_t clnt_adr_sz;
	
	if(argc!=2) {
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	
	serv_sock=socket(PF_INET, SOCK_STREAM, 0);   
	if(serv_sock==-1)
		error_handling("socket() error");
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_adr.sin_port=htons(atoi(argv[1]));

	if(bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr))==-1)
		error_handling("bind() error");
	
	if(listen(serv_sock, 5)==-1)
		error_handling("listen() error");
	
	clnt_adr_sz=sizeof(clnt_adr);

	for(i=0; i<5; i++)
	{
		clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_adr, &clnt_adr_sz);
		if(clnt_sock==-1)
			error_handling("accept() error");
		else
			printf("Connected client %d \n", i+1);
	
		while((str_len=read(clnt_sock, message, BUF_SIZE))!=0)
			write(clnt_sock, message, str_len);

		close(clnt_sock);
	}

	close(serv_sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

**运行结果**

```shell
gcc echo_server.c -o eserver
./eserver 9190
输出：
Connecten client 1
Connecten client 2
Connecten client 3
```

**回声客户端代码**

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 1024
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sock;
	char message[BUF_SIZE];
	int str_len;
	struct sockaddr_in serv_adr;

	if(argc!=3) {
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_STREAM, 0);   
	if(sock==-1)
		error_handling("socket() error");
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=inet_addr(argv[1]);
	serv_adr.sin_port=htons(atoi(argv[2]));
	
	if(connect(sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr))==-1)
		error_handling("connect() error!");
	else
		puts("Connected...........");
	
	while(1) 
	{
		fputs("Input message(Q to quit): ", stdout); 	//如果输入Q说明结束while循环
		fgets(message, BUF_SIZE, stdin);
		
		if(!strcmp(message,"q\n") || !strcmp(message,"Q\n"))  //检验message是否为Q/q
			break;

		write(sock, message, strlen(message));
		str_len=read(sock, message, BUF_SIZE-1);
		message[str_len]=0;
		printf("Message from server: %s", message);
	}
	
	close(sock);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

**运行结果**

```shell
gcc echo_client.c -o eclient
./eclient 192.168.233.20 9190
输出：
Connected ....
Input message: hello
Message from server: hello
Input message : Q
```

