@[toc]
## 回声客户端的完美实现

### 回声客户端出现的问题

在上一节基于TCP的服务器端、回声客户端中，存在问题：

如果数据太大，操作系统就有可能把数据分成多个数据包发送到客户端，客户端有可能在尚未收到全部数据包时就调用read函数

问题出在客户端，而不是服务器端，先来对比一下客户端与服务器端I/O相关代码

**服务器端**

```c++
while((str_len = read(cknt_sock, message, BUF_SIZE)) != 0)
    write(clnt_sock, message, str_len);
```

**客户端**

```c+
write(sock, message, strlen(message))
str_len = read(sock, message, BUF_SIZE - 1);
```

客户端与服务器端都调用read和write函数，实际上回声客户端将100%接受自己传输的数据，只不过接收数据时单位出现问题

回声客户端传输的是字符串，而且是通过调用write函数一次性发送的，之后调用read函数，等待接收自己传输的字符串

这里的问题就出现在客户端收到所有字符串数据，需要等待多长时间？等待之后调用read函数是否可以一次性读取完毕？

### 回声客户端问题解决方法

因为可以提前确定接收数据的大小，所以这个问题不难解决，如果之前传输了100字节长的字符串，则在接收时循环调用read函数读取100字节结课

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
	int str_len, recv_len, recv_cnt;
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
		fputs("Input message(Q to quit): ", stdout);
		fgets(message, BUF_SIZE, stdin);
		
		if(!strcmp(message,"q\n") || !strcmp(message,"Q\n"))
			break;

		str_len=write(sock, message, strlen(message));
		
		recv_len=0;
		while(recv_len<str_len)
		{
			recv_cnt=read(sock, &message[recv_len], BUF_SIZE-1);
			if(recv_cnt==-1)
				error_handling("read() error!");
			recv_len+=recv_cnt;
		}
		
		message[recv_len]=0;
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

在之前的示例中只调用了一次read函数，上述示例为了接收所有传输的数据而循环调用read函数

```c++
while(recv_len<str_len)
{
	recv_cnt=read(sock, &message[recv_len], BUF_SIZE-1);
	if(recv_cnt==-1)
		error_handling("read() error!");
	recv_len+=recv_cnt;
}
```

如果将上述代码改成：

```c++
while(recv_len != str_len)
{
	recv_cnt=read(sock, &message[recv_len], BUF_SIZE-1);
	if(recv_cnt==-1)
		error_handling("read() error!");
	recv_len+=recv_cnt;
}
```

接收的数据大小应该和传输的相同，所以recv_len中保存的数据等于str_len中保存的数据，即可结束while循环，读者们肯定认为这种循环写法更符合逻辑，但是可能因为某种异常情况导致死循环，所以尽可能的使用 while(recv_len<str_len) ，即使发生异常情况也不会导致无限循环

## TCP原理

### TCP套接字中的I/O缓冲

TCP套接字的数据收发无边界，服务器端即使调用一次write函数传输40字节的数据，客户端也有可能通过4次read函数调用每次读取10字节，但是问题又出现了，服务器端一次性传输了40字节，而客户端居然可以缓慢的分批接收，，客户端接收10字节后，剩下的30字节在什么地方等着呢？

事实上，write函数调用后并非立即传输数据，read函数调用后也并非马上接收数据。更准确的来说，write函数调用瞬间，数据将移到输出缓冲；read函数调用瞬间，从输入缓冲读取数据。

![在这里插入图片描述](https://img-blog.csdnimg.cn/c5c2b3ec4a514da2b118cadd118cf037.png)



调用write函数时，数据将移到输出缓冲，在适当的时候传向对方的输入缓冲，这时对方调用read函数从输入缓冲读取数据

**I/O缓冲特性如下**

- I/O缓冲在每个TCP套接字中单独存在
- I/O缓冲在创建套接字时自动生成
- 即使关闭套接字也会继续传递输出缓冲中遗留的数据
- 关闭套接字将丢失输入缓冲中的数据

### TCP内部工作原理1：与对方套接字的连接

TCP套接字从创建到消失所经过程分为三步：

1.与对方套接字建立连接

2.与对方套接字交换数据

3.断开与对方套接字的连接

TCP在实际通信过程中也会经过三次对话过程，因此该过程又称为三次握手

![在这里插入图片描述](https://img-blog.csdnimg.cn/23f054de01504621900045717de013a1.png)






套接字是以全双工方式工作的，也就是说明套接字可以双向传递数据

**1、** 请求连接的主机A向主机B传递如下信息：

[SYN] SEQ: 1000, ACK:  - 

该消息中SEQ为1000，ACK为空，SEQ为1000的含义如下：

*现在传递的数据包序号为100，如果接收无误，请通知我向你传递1001号数据包*

**2、** 这是首次请求连接时使用的消息，又称SYN，表示收发数据前传输的同步消息，接下来主机B向A传递如下消息：

[SYN+ACK] SEQ: 2000, ACK: 1001

此时SEQ为2000，ACK为1001，而SEQ为2000的含义如下：

*现在传递的数据包序号为2000，如果接收无误，请通知我向你传递2001号数据包*

而ACK 1001的含义如下：

刚才传输的SEQ为1000的数据包接收无误，现在请传递SEQ为1001的数据包

对主机A首次传输的的数据包确认消息（ACK 1001）和为主机B传输数据做准备的同步消息（SEQ 200）捆绑发送，因此此种类型的消息又称为 SYN + ACK

**3、** 收发数据前向数据包分配序号，并向对方通报此序号，这都是为了防止数据丢失所做的准备。通过向数据包分配序号并确认，可以在数据丢失时马上查看并重传丢失的数据包，所以TCP可以保证可靠的数据传输。最后观察主机A向B传输的消息：

[ACK] SEQ : 1001, ACK: 2001

TCP连接过程中发送数据包时需要分配序号，在此之前的序号100的基础上+1，此时该数据包传递如下消息：

*已正确收到传输的SEQ为2000的数据包，现在可以传输SEQ为2001的数据包*

### TCP内部工作原理2：与对方主机的数据交换

通过第一步三次握手过程完成了数据交换准备，下面就开始正式收发数据

![在这里插入图片描述](https://img-blog.csdnimg.cn/015a2357b3de47438dbf04382cce8e89.png)

首先主机A通过1个数据包发送100个字节的数据，数据包的SEQ为1200，主机B为了确认这点，向主机A发送了ACK1301消息

此时的ACK号为1301而非1201，原因在于ACK号的增量为传输的数据字节数，假设每次ACK号不加传输的字节数，这样虽然可以确认数据包的传输，但是不能明确100字节却不正确传递还是丢失，因此按照如下公式传递ACK消息：

**ACK 号  --- SEQ号 + 传递的字节数 + 1**

与三次握手协议相同，最后+1是为了告知对方下次要传递的SEQ号

下面分析传输过程中数据包消失的情况

![在这里插入图片描述](https://img-blog.csdnimg.cn/62145f37fa97469580685013c1977841.png)


SEQ1301数据包向主机B传递100字节数据，但中间发生错误，主机B未收到，一段时间后，主机A仍未收到对于SEQ 1301的ACK确认，因此重传改数据包，为了完成数据包重传，TCP套接字启动计时器以等待ACK回答，相应计时器发生超时则重传

### TCP内部工作原理3：断开与套接字的连接

先由套接字A向套接字B传递断开连接的消息，套接字B发出确认收到的消息，然后向套接字A传递可以断开连接的消息，套接字A同样发出确认消息

![在这里插入图片描述](https://img-blog.csdnimg.cn/dee36f1100cc4bb6a9ebe8b16813879d.png)


数据包内的FIN表示断开连接，双方各发送一次FIN消息后断开连接，此过程经历四个阶段，因此又称为四次握手

---
这是《TCP/IP网络编程》专栏的第五篇文章，欢迎各位读者订阅！

更多资料点击 [GitHub](https://github.com/Shy2593666979) 欢迎各位读者去Star

⭐学术交流群Q 754410389 持续更新中~~~
