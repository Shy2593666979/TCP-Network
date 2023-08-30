## 套接字协议及其数据传输特性

### 关于协议

如果相隔比较远的两人进行通话，必须先决定通话方式，如果一方选择电话，另一方也必须选择电话，否则接受不到消息。

总之，协议就是为了完成数据交换而定好的约定。

### 创建套接字

```c++
#include <sys/socket.h>
int socket(int domain, int type, int protocol);

	- 成功返回文件描述符，失败返回-1
        domain		套接字中使用的协议族信息
        type		套接字数据传输类型信息
        protocol	计算机间通信中使用的协议信息
```

### 协议族

通过socket函数的第一个参数传递套接字中使用的协议分类信息，此协议分类信息成为协议族。

|   名称    |        协议族        |
| :-------: | :------------------: |
|  PF_INET  |   IPv4互联网协议族   |
| PF_INET6  |   IPv6互联网协议族   |
| PF_LOCAL  | 本地通信的UNIX协议族 |
| PF_PACKET |  底层套接字的协议族  |
|  PF_IPX   |   IPX Novell协议族   |

### 套接字类型1：面向连接的套接字（SOCK_STREAM）

传输方式特征如下：

- 传输过程中数据不会消失
- 按序传输数据
- 传输的数据不存在数据边界

收发数据的套接字内部有缓冲（buffer），就是字节数组。通过套接字传输的数据将保存到该数组，因此收到数据并不意味着马上调用read函数，只要不超过数组容量，有可能在数据填充缓冲后通过一次read函数调用读取全部，也有可能分成多次read函数调用进行读取。在面向连接的套接字中，read函数和write函数的调用次数并无太大意义，所以面向连接的套接字不存在数据边界。

**概括面向连接的套接字如下：** 可靠的、按序传递的、基于字节的面向连接的数据传输方式的套接字。



### 套接字类型2：面向消息的套接字（SOCK_DGRAM）

传输方式特征如下：

- 强调快速传输而非传输顺序
- 传输的数据可能丢失也夸你损毁
- 传输的数据有数据边界
- 限制每次传输的数据大小

**概括面向消息的套接字如下：** 不可靠、不按序传递的、以数据的高速传输为目的的套接字

### 协议的最终选择

前面已经通过socket函数的强两个参数传递了协议族信息和套接字数据传输方式，这些信息还不足以决定采用的协议，还需要传递第三个参数。

传递前两个参数即可创建所需套接字，所以大部分情况下可以向第三个参数传递0，除非遇到一下这种情况：

 “同一协议族中存在多个数据传输方式相同的协议”

- 如果是IPv4协议族面向连接的套接字第三个参数使用 IPPROTO_TCP

- 如果是IPv4协议族面向消息的套接字第三个参数使用 IPPROTO_UDP

### 面向连接的套接字：TCP套接字示例

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/types.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char* argv[])
{
	int sock;
	struct sockaddr_in serv_addr;
	char message[30];
	int str_len=0;
	int idx=0, read_len=0;
	
	if(argc!=3){
		printf("Usage : %s <IP> <port>\n", argv[0]);
		exit(1);
	}
	
	sock=socket(PF_INET, SOCK_STREAM, 0);
	if(sock == -1)
		error_handling("socket() error");
	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family=AF_INET;
	serv_addr.sin_addr.s_addr=inet_addr(argv[1]);
	serv_addr.sin_port=htons(atoi(argv[2]));
		
	if(connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr))==-1) 
		error_handling("connect() error!");

	while(read_len=read(sock, &message[idx++], 1))
	{
		if(read_len==-1)
			error_handling("read() error!");
		
		str_len+=read_len;
	}

	printf("Message from server: %s \n", message);
	printf("Function read call count: %d \n", str_len);
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

