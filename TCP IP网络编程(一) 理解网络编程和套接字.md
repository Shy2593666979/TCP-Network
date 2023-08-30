## 理解网络编程和套接字

### 网络编程和套接字概要

网络编程就是编写程序使两台联网的计算机相互交换数据

为了与远程计算机进行数据传输，需要连接因特网，而编程种的套接字就是用来连接该网络的工具。

### 构建套接字

1.调用soecket函数创建套接字

```c++
#include<sys/socket.h>
int socket(int domain, int type, int protocol);
	
	——>成功返回文件描述符，失败返回-1
```

2.调用bind函数给套接字分配地址

```c++
#include<sys/socket.h>
int bind(int sockfd, struct sockaddr *myaddr, socklen_t addrlen);

	——>成功返回0，失败返回-1
```

3.调用listen函数将套接字转化为可接受连接的状态

```c++
#include<sys/socket.h>
int listen(int sockfd, int backlog);

	——>成功时返回0，失败返回-1
```

4.调用accept函数接受连接请求

```c++
#include<sys/socket.h>
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);

	——>成功时返回文件描述符，失败返回-1
```

**网络编程中接受连接请求的套接字创建过程总结如下：**

1.调用soecket函数创建套接字

2.调用bind函数给套接字分配地址

3.调用listen函数将套接字转化为可接受连接的状态

4.调用accept函数接受连接请求

### 编写 Hello World 服务器端

服务器端（server）是能受理连接请求的程序

*hello_server.c*

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char *argv[])
{
	int serv_sock;
	int clnt_sock;

	struct sockaddr_in serv_addr;
	struct sockaddr_in clnt_addr;
	socklen_t clnt_addr_size;

	char message[]="Hello World!";
	
	if(argc!=2){
		printf("Usage : %s <port>\n", argv[0]);
		exit(1);
	}
	
	serv_sock=socket(PF_INET, SOCK_STREAM, 0);
	if(serv_sock == -1)
		error_handling("socket() error");
	
	memset(&serv_addr, 0, sizeof(serv_addr));
	serv_addr.sin_family=AF_INET;
	serv_addr.sin_addr.s_addr=htonl(INADDR_ANY);
	serv_addr.sin_port=htons(atoi(argv[1]));
	
	if(bind(serv_sock, (struct sockaddr*) &serv_addr, sizeof(serv_addr))==-1 )
		error_handling("bind() error"); 
	
	if(listen(serv_sock, 5)==-1)
		error_handling("listen() error");
	
	clnt_addr_size=sizeof(clnt_addr);  
	clnt_sock=accept(serv_sock, (struct sockaddr*)&clnt_addr,&clnt_addr_size);
	if(clnt_sock==-1)
		error_handling("accept() error");  
	
	write(clnt_sock, message, sizeof(message));
	close(clnt_sock);	
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

### 构建请求连接套接字

客户端程序只有调用socket函数创建套接字和调用connect函数向服务器发出连接请求两个步骤

*hello_client.c*

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <sys/socket.h>

void error_handling(char *message);

int main(int argc, char* argv[])
{
	int sock;
	struct sockaddr_in serv_addr;
	char message[30];
	int str_len;
	
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
	
	str_len=read(sock, message, sizeof(message)-1);
	if(str_len==-1)
		error_handling("read() error!");
	
	printf("Message from server: %s \n", message);  
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

**请求连接函数**

```c++
#include<sys/socket.h>
int connect(int sockfd,struct sockaddr *serv_addr, socklen_t addrlen);

	——>成功返回0，失败返回-1
```

### 在Linux平台下运行

```shell
gcc hello_server.c -o hserver
./hserver 9190
```

现在已经打开了服务端，但是调用的accept函数还没有返回，接下来运行客服端

```shell
gcc hello_client.c -o hclient
./hclient 192.168.233.20 9190
输出: Hello World!
```

## 基于Linux的文件操作

### 打开文件

```c++
#include <sys/type.h>
#include <sys/stat.h>
#include <fcnt.h>

int open(const char *path, int flag);

	——>成功返回文件描述符，失败返回-1
    path   文件名的字符串地址
    flag   文件打开模式信息
```

**文件打开模式**

```
打开模式				含义
O_CREAT			   必要时创建文件
O_TRUNC			   删除全部现有数据
O_APPEND		   维持现有数据，保存末尾
O_RDONLY		   只读打开
O_WRONLY	   	   只写打开
O_RDWR			   读写打开
```

### 关闭文件

```c++
#include<unistd.h>

int close(int fd);

	——>成功返回0，失败返回-1
```

### 将数据写入文件

```c++
#include<unistd.h>
ssize_t write(int fd, const void *buf, size_t nbytes);

	——>成功时返回写入的字节数，失败返回-1
        buf保存要传输数据的缓冲地址
        nbytes要传输的字节数
```

函数定义中，size_t是通过typedef声明的unsigned int类型，对于ssize_t来说s代表signed，ssize_t为signed int类型

*lower_open.c*

```c++
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

void error_handling(char* message);

int main(void)
{
	int fd;
	char buf[]="Let's go!\n";
	
	fd=open("data.txt", O_CREAT|O_WRONLY|O_TRUNC);
	if(fd==-1)
		error_handling("open() error!");
	printf("file descriptor: %d \n", fd);

	if(write(fd, buf, sizeof(buf))==-1)
		error_handling("write() error!");

	close(fd);
	return 0;
}

void error_handling(char* message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}

```

### 读取文件中的数据

与write函数对应，read函数是用来接受数据的

```c++
#include<unistd.h>

ssize_t read(int fd, void *buf, size_t nbytes)
    
    ——>成功返回接受的字节数，失败返回-1
```

*lower_read.c*

```c++
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

#define BUF_SIZE 100

void error_handling(char* message);

int main(void)
{
	int fd;
	char buf[BUF_SIZE];
	
	fd=open("data.txt", O_RDONLY);
	if( fd==-1)
		error_handling("open() error!");
	
	printf("file descriptor: %d \n" , fd);
	
	if(read(fd, buf, sizeof(buf))==-1)
		error_handling("read() error!");

	printf("file data: %s", buf);
	
	close(fd);
	return 0;
}

void error_handling(char* message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

**ps:**

 	在文中出现好多次void error_handling(char* message)函数，这个是打印错误信息的，也可以用perror函数，例如

```c++
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>

int main()
{
    int fd = open(data.txt, O_RDONLY);
    if(fd == -1)
    {
        perror("open:");
    }
}

```

