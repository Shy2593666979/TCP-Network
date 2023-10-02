
## 一、理解UDP

### 1.UDP套接字的特点

如果只考虑可靠性，TCP确实比UDP好，但是UDP在结构上比TCP更简洁，UDP不会发送类似ACK的应答消息，也不会像SEQ那样给数据包分配序号，因此ＵＤＰ的性能有时比TCP高出许多，在编程中实现UDP也比TCP更加简单。

虽然UDP的可靠性比不上TCP，但是不会频繁的发生数据损毁，在更重视性能而非可靠性的情况下，UDP是一种不错的选择。

### 2.UDP内部工作原理

与TCP不同，UDP不会进行流量控制

![在这里插入图片描述](https://img-blog.csdnimg.cn/6a9d410c960145c5be9f4e05e5f10151.png)


IP的作用就是让离开主机B的数据包准确传递到主机A，但是把UDP包最终交给主机A的某一UDP套接字的过程就是由UDP完成的，UDP最重要的作用就是根据端口号传到主机的数据包交付给最终的UDP套接字。

### 3.UDP的高效使用

虽然大部分网络编程都基于TCP实现，但也有一些是基于UDP实现的，接下来考虑何时使用UDP更高效，网络传输特性导致信息丢失频发，可若要传递压缩文件，则必须使用TCP，因为压缩文件只要丢失一点就无法打开。但是通过网络实时传输视频的情况有所不同，对于媒体数据而言，丢失一部分不会产生太大问题，最多会出现短暂的视频模糊，但是需要提供实施服务，速度成为非常重要的因素。

TCP比UDP慢的原因通常有下面两点：

- 收发数据前后进行的连接设置及清除过程
- 收发数据过程中为保证可靠性而添加的流控制

如果收发的数据量小但需要频繁连接时，UDP比TCP更高效

## 二、实现基于UDP的服务器端、客户端

### 1.UDP中的服务端和客户端没有连接

UDP的服务器端、客户端不像TCP那样在连接状态下交换数据，因此与TCP不同，无需经过连接过程，不必调用TCP连接过程中调用的listen函数和accept函数，UDP中只有创建套接字的过程和数据交换过程。

### 2.UDP服务器端和客户端均只需要一个套接字

TCP中套接字之间应该是一对一的关系，如果向10个客户端提供服务，则除了守门的服务器套接字外，还需要10个套接字，但在UDP中，不管是服务器端还是客户端都只需要1个套接字。

![在这里插入图片描述](https://img-blog.csdnimg.cn/7cfc1df17d2941628ded114198693eaf.png)


图中显示一个UDP套接字能和多台主机通信。

### 3.基于UDP的数据I/O函数

创建好TCP套接字后，传输数据无需再添加地址信息，因为TCP套接字将保持与对方套接字的连接。TCP套接字知道目标地址信息，但UDP套接字不会保持连接状态，因此每次传输数据都要添加目标地址信息，这相当于寄信件时填写收件地址。

**填写地址并传输数据时调用的UDP相关函数**

```c++
#include<sys/socket.h>

ssize_t sendto(int sock, void *buff, size_t nbytes, int flags,struct sockaddr *to,socklen_t addrlen);
	成功返回传输的字节数，失败时返回-1
    sock		用于传输数据的UDP套接字文件描述符
    buff		保存待传输数据的缓冲地址值
    nbytes		待传输的数据长度，以字节为单位
    flags		可选项参数，若没有则传递0
    to			存有目标地址信息的sockaddr结构体变量的地址值
    addrlen		传递给参数to的地址值结构体变量长度
```

**接收UDP数据的函数**

```c++
#include<sys/socket.h>

ssize_t recvfrom(int sock, void *buff, size_t nbytes, int flags, struct sockaddr *from, socklen_t *addrlen);
	成功返回接收的字节数，失败返回-1
    sock		用于传输数据的UDP套接字文件描述符
    buff		保存待传输数据的缓冲地址值
    nbytes		待传输的数据长度，以字节为单位
    flags		可选项参数，若没有则传递0
    to			存有发送端地址信息的sockaddr结构体变量的地址值
    addrlen		传递给参数from的地址值结构体变量长度
```

### 4.基于UDP的回声服务器端、客户端

UDP不同于TCP，不存在连接请求和受理过程，因此在某种意义上无法明确区分客户端和服务器端，只是因为提供服务所以叫服务器端

***uecho_server.c***

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int serv_sock;
	char message[BUF_SIZE];
	int str_len;
	socklen_t clnt_adr_sz;
	
	struct sockaddr_in serv_adr, clnt_adr;
	if(argc!=2){
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	
	serv_sock=socket(PF_INET, SOCK_DGRAM, 0);
	if(serv_sock==-1)
		error_handling("UDP socket creation error");
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(serv_sock, (struct sockaddr*)&serv_adr, sizeof(serv_adr))==-1)
		error_handling("bind() error");

	while(1) 
	{
		clnt_adr_sz=sizeof(clnt_adr);
		str_len=recvfrom(serv_sock, message, BUF_SIZE, 0, 
								(struct sockaddr*)&clnt_adr, &clnt_adr_sz);
		sendto(serv_sock, message, str_len, 0, 
								(struct sockaddr*)&clnt_adr, clnt_adr_sz);
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

接下来介绍与上述服务器端协同工作的客户端，与TCP客户端不同，不存在connect函数调用

***uecho_client.c***

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sock;
	char message[BUF_SIZE];
	int str_len;
	socklen_t adr_sz;
	
	struct sockaddr_in serv_adr, from_adr;
	if(argc!=3){
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_DGRAM, 0);   
	if(sock==-1)
		error_handling("socket() error");
	
	memset(&serv_adr, 0, sizeof(serv_adr));
	serv_adr.sin_family=AF_INET;
	serv_adr.sin_addr.s_addr=inet_addr(argv[1]);
	serv_adr.sin_port=htons(atoi(argv[2]));
	
	while(1)
	{
		fputs("Insert message(q to quit): ", stdout);
		fgets(message, sizeof(message), stdin);     
		if(!strcmp(message,"q\n") || !strcmp(message,"Q\n"))	
			break;
		
		sendto(sock, message, strlen(message), 0, 
					(struct sockaddr*)&serv_adr, sizeof(serv_adr));
		adr_sz=sizeof(from_adr);
		str_len=recvfrom(sock, message, BUF_SIZE, 0, 
					(struct sockaddr*)&from_adr, &adr_sz);

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

### 5.UDP客户端套接字的地址分配

UDP客户端缺少把IP和端口分配给套接字的过程。TCP客户端调用connect函数自动完成此过程，而UDP中连能承担相同功能的函数调用语句都没有。

UDP程序中，调用sento函数传输数据前应完成对套接字的地址分配工作，因此调用bind函数。bind函数不区分TCP或者UDP，也就是说，在UDP程序中同样可以调用。另外如果调用sendto函数时发现尚未分配地址信息，则在首次调用sendto函数时给相应套接字自动分配IP和端口。而且此时分配的地址一直保留到程序结束为止，因此也可用来与其他UDP套接字进行数据交换。IP用主机IP，端口号选尚未使用的任意端口号。

调用sendto函数时自动分配IP和端口号，UDP客户端中无需额外的地址分配过程。

## 三、UDP的数据传输特性和调用connect函数

TCP传输的数据不存在数据边界，表示“数据传输过程中调用I/O函数的次数不具有任何意义”。

UDP数据传输中存在数据边界，传输中调用I/O函数的次数非常重要。因此，输入函数的调用次数应和输出函数的调用次数完全一致，这样才能保证接收全部已发送数据。

***bound_host1.c***

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sock;
	char message[BUF_SIZE];
	struct sockaddr_in my_adr, your_adr;
	socklen_t adr_sz;
	int str_len, i;

	if(argc!=2){
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_DGRAM, 0);
	if(sock==-1)
		error_handling("socket() error");
	
	memset(&my_adr, 0, sizeof(my_adr));
	my_adr.sin_family=AF_INET;
	my_adr.sin_addr.s_addr=htonl(INADDR_ANY);
	my_adr.sin_port=htons(atoi(argv[1]));
	
	if(bind(sock, (struct sockaddr*)&my_adr, sizeof(my_adr))==-1)
		error_handling("bind() error");
	
	for(i=0; i<3; i++)
	{
		sleep(5);	// delay 5 sec.
		adr_sz=sizeof(your_adr);
		str_len=recvfrom(sock, message, BUF_SIZE, 0, 
								(struct sockaddr*)&your_adr, &adr_sz);     
	
		printf("Message %d: %s \n", i+1, message);
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

***bound_host2.c***

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

#define BUF_SIZE 30
void error_handling(char *message);

int main(int argc, char *argv[])
{
	int sock;
	char msg1[]="Hi!";
	char msg2[]="I'm another UDP host!";
	char msg3[]="Nice to meet you";

	struct sockaddr_in your_adr;
	socklen_t your_adr_sz;
	if(argc!=3){
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_DGRAM, 0);   
	if(sock==-1)
		error_handling("socket() error");
	
	memset(&your_adr, 0, sizeof(your_adr));
	your_adr.sin_family=AF_INET;
	your_adr.sin_addr.s_addr=inet_addr(argv[1]);
	your_adr.sin_port=htons(atoi(argv[2]));
	
	sendto(sock, msg1, sizeof(msg1), 0, 
					(struct sockaddr*)&your_adr, sizeof(your_adr));
	sendto(sock, msg2, sizeof(msg2), 0, 
					(struct sockaddr*)&your_adr, sizeof(your_adr));
	sendto(sock, msg3, sizeof(msg3), 0, 
					(struct sockaddr*)&your_adr, sizeof(your_adr));
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

**UDP数据报：** UDP套接字传输的数据包又称为数据报，实际上数据报也属于数据包的一种。只是与TCP包不同，其本身可以成为1个完整数据。这与UDP的数据传输特性有关，UDP中存在数据边界，1个数据包即可成为有关完整数据，因此称为数据报。

### 1.已连接UDP套接字与未连接UDP套接字

TCP套接字中需要注册待传输数据的目标IP和端口号，而UDP无需注册，因此通过sendto函数传输数据的过程分为三个阶段

- 第一阶段：向UDP套接字注册目标IP和端口号
- 第二阶段：传输数据
- 第三阶段：删除UDP套接字中注册的目标地址信息

每次调用sendto函数时重复上述过程，每次都变更地址，因此可以重复利用同一UDP套接字向不同的目标传输，这种未注册目标地址信息的套接字成为未连接套接字，注册了目标地址信息地址的套接字成为连接connected套接字，UDP默认为未连接套接字

如果IP为192.168.233.20向端口100准备了三个数据，调用了3次sendto函数进行传输

此时需要重复3次上述阶段，因此需要与同一主机进行长时间通信，将UDP套接字变为已连接会提高效率

### 2.创建已连接的UDP套接字

针对UDP套接字调用connect函数并不意味着要与对方UDP套接字连接，这只是向UDP套接字注册目标IP和端口信息。

之后就可以跟TCP一样，每次调用sendto函数只需传递数据，因为已经指定了收发对象，所以不仅可以使用sendto函数、recvfrom函数，还可以使用write函数、read函数进行通信

----
这是《TCP/IP网络编程》专栏的第六篇文章，欢迎各位读者订阅！

更多资料点击 [GitHub](https://github.com/Shy2593666979/TCP-Network/tree/main) 欢迎各位读者去Star

⭐学术交流群Q 754410389 持续更新中~~~
