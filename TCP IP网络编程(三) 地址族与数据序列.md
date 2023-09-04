## 分配给套接字的IP地址与端口号

### 网络地址

IP地址分为两类：

- IPv4		4字节地址族
- IPv6        16字节地址族	

IPv4和IPv6的差别主要是表示在IP地址所用的字节数，目前通用的地址族为IPv4，而IPv6是为了应对IP地址耗尽的问题而提出的标准，目前主要还是使用IPv4

IPv4标准的4字节IP地址分为网络地址和主机地址，且分为A、B、C、D等类型。

```c++
A类		网络ID 主机ID 主机ID 主机ID
B类		网络ID 网络ID 主机ID 主机ID
C类		网络ID 网络ID 网络ID 主机ID
D类		网络ID 网络ID 网络ID 网络ID （多播IP地址）
```

### 网络地址分类与主机地址边界

通过IP地址的第一个字节即可判断网络地址占用的字节数

- A类地址的首字节范围：0-127
- B类地址的首字节范围：128-191
- C类地址的首字节范围：191-223

还有另一种表述方式

- A类地址的首位以0开始
- B类地址的首位以10开始
- C类地址的首位以110开始

## 地址信息的表示

### 表示 IPv4地址的结构体

```c++
struct sockaddr_in
{
    sa_family_t     sin_family;  //地址族
    uint16_t        sin_port;    //16位TCP/UDP端口号
    struct in_addr  sin_addr;    //32位IP地址
    char            sin_zero[8]; //不使用
};
```

该结构体中提到了另外一个结构体 in_addr 定义为：

```c++
struct in_addr
{
    in_addr_t        s_addr;     //32位IPv4地址
}
```

上述两个结构体包含一些数据类型，uint16_t、in_addr_t等类型可以参考POSIX。POSIX是为UNIX系统操作设立的标准

​																					**POSIX中定义的数据类型**

| 数据类型名称 |      数据类型说明      |  声明头文件  |
| :----------: | :--------------------: | :----------: |
|    int8_t    |    signed 8-bit int    | sys/types.h  |
|   uint8_t    |   unsigned 8-bit int   | sys/types.h  |
|   int16_t    |   signed 16-bit int    | sys/types.h  |
|   uint16_t   |  unsigned 16-bit int   | sys/types.h  |
|   int32_t    |   signed 32-bit int    | sys/types.h  |
|   uint32_t   |  unsigned 32-bit int   | sys/types.h  |
| sa_family_t  | 地址族(address family) | sys/socket.h |
|  socklen_t   | 长度(length of struct) | sys/socket.h |
|  in_addr_t   |   IP地址,，uint32_t    | netinet/in.h |
|  in_port_t   |    端口号，uint16_t    | netinet/in.h |

### 结构体sockaddr_in 的成员分析

**成员 sin_family**

​	每种协议族适用的协议族均不同，比如IPv4使用4字节地址族，IPv6使用16字节地址族

​																				**地址族**

|  地址族  |               含义               |
| :------: | :------------------------------: |
| AF_INET  |    IPv4网络协议中使用的地址族    |
| AF_INET6 |    IPv6网络协议中使用的地址族    |
| AF_LOCAL | 本地通信中采用的UNIX协议的地址族 |

AF_LOCAL是为了说明具有多种地址族而添加的

**成员sin_port**

该成员保存16位端口号，重点在于，它以网络字节序保存

**成员sin_addr**

该成员保存32位IP地址信息，也以网络字节序保存，同时管擦结构体in_addr

**成员sin_zero**

无特殊含义，只是为使结构体sockaddr_in的大小与sockaddr结构体保存一致而插入的成员，必须填充为0

## 网络字节序与地址变换

### 字节序与网络字节序

CPU保存数据的方式有两种，意味着CPU解析数据的方式也分为两种

- 大端序：高位字节存放到低位地址
- 小端序：高位字节存放到高位地址

在通过网络传输数据时约定统一方式，这种约定称为网络字节序，统一为大端序

总结来说，将数据数组转发成大端序格式再进行网络传输，所有计算机接收数据时应该识别该数据是网络字节序格式，小端序系统传输时应该转化成大端序排列方式

### 字节序转换

填充结构体 sockadr_in 前将数据转换成网络字节序，介绍帮助转换字节序的函数

```c
unsigned short htons(unsigned short)
unsigned short ntohs(unsigned short)
unsigned long htonl(unsigned short)
unsined long ntohl(unsigned long)
```

**函数名的含义**

- htons中的 h 代表主机(host)字节序
- htons中的 n 代表网络(network)字节序
- htons中的 s 指的是short
- htons中的 l 指的是long(Linux中long类型占用4个字节)

```c++
#include <stdio.h>
#include <arpa/inet.h>

int main(int argc, char *argv[])
{
	unsigned short host_port=0x1234;
	unsigned short net_port;
	unsigned long host_addr=0x12345678;
	unsigned long net_addr;
	
	net_port=htons(host_port);
	net_addr=htonl(host_addr);
	
	printf("Host ordered port: %#x \n", host_port);
	printf("Network ordered port: %#x \n", net_port);
	printf("Host ordered address: %#lx \n", host_addr);
	printf("Network ordered address: %#lx \n", net_addr);
	return 0;
}
```

下面这就是在小端序CPU中运行的结果。如果在大端序CPU中运行，则变量值不会改变。大部分朋友都会得到类似的运行结果，因为Intel和AMD系列的CPU都采用小端序标准。

```shell
gcc endian_conv.c -o conv
./conv
输出：
Host ordered port : 0x1234
Network ordered port : 0x3412
Hostordered address : 0x12345678
Network ordered address : 0x78563412
```



## 网络地址的初始化与分配

### 将字符串信息转换为网络字节序的整数型

​	对于IP地址的表示，我们熟悉的是点分十进制表示法而非整数型数据表示法。幸运的是，有函数会帮我们将字符串形式的IP地址转换成32位整数型数据，在转换类型的同时进行网络字节序转换。

```c++
#include<arpa/inet.h>
in_addr_t inet_addr(const char * string);
		//成功时返回32位大端序整数型值，失败时返回INADDR_NONE。
```

该函数的调用过程

```c++
#include <stdio.h>
#include <arpa/inet.h>

int main(int argc, char *argv[])
{
	char *addr1="127.212.124.78";
	char *addr2="127.212.124.256";

	unsigned long conv_addr=inet_addr(addr1);
	if(conv_addr==INADDR_NONE)
		printf("Error occured! \n");
	else
		printf("Network ordered integer addr: %#lx \n", conv_addr);
	
	conv_addr=inet_addr(addr2);
	if(conv_addr==INADDR_NONE)
		printf("Error occureded \n");
	else
		printf("Network ordered integer addr: %#lx \n\n", conv_addr);
	return 0;
}
```

从结果可以看出，inet_addr函数不仅可以把IP地址转成32位整数型，而且可以检测无效的IP地址

inet_aton函数与inet_addr函数在功能上完全相同，也将字符串形式IP地址转换为32位网络字节序整数并返回。不同的是该函数利用了in_addr结构体，且使用频率更高。

```c++
#include<arpa/inet.h>
int inet_aton(const char * string, struct in_addr * addr);
		成功时返回1，失败时返回0
		参数1：string，含有需转换的IP地址信息的字符串地址值。
		参数2：addr，将保存转换结果的in_addr结构体变量的地址值。
```

该函数的调用过程

```c++
#include <stdio.h>
#include <stdlib.h>
#include <arpa/inet.h>
void error_handling(char *message);

int main(int argc, char *argv[])
{
	char *addr="127.232.124.79";
	struct sockaddr_in addr_inet;
	
	if(!inet_aton(addr, &addr_inet.sin_addr))
		error_handling("Conversion error");
	else
		printf("Network ordered integer addr: %#x \n", addr_inet.sin_addr.s_addr);
	return 0;
}

void error_handling(char *message)
{
	fputs(message, stderr);
	fputc('\n', stderr);
	exit(1);
}
```

调用inet_addr函数，返回转换后的IP地址信息还需保存到sockaddr_in结构体中声明的in_addr结构体变量。而inet_aton函数则不需此过程，因为该函数会自动把结果存入该结构体变量。



还有一个函数，与 inet_aton() 正好相反，它可以把网络字节序整数型IP地址转换成我们熟悉的字符串形式

```c++
#include <arpa/inet.h>
char *inet_ntoa(struct in_addr adr);
	成功返回转换的字符串地址值，失败返回-1
```

- 该函数将通过参数传入的整数型IP地址转换为字符串格式并返回

- 但要小心，返回值为 char 指针，返回字符串地址意味着字符串已经保存在内存空间，但是该函数未向程序员要求分配内存，而是在内部申请了内存保存了字符串。也就是说调用了该函数候要立即把信息复制到其他内存空间。因为，若再次调用inet_ntoa函数，则有可能覆盖之前保存的字符串信息
- 总之，再次调用 inet_ntoa 函数前返回的字符串地址是有效的。若需要长期保存，则应该将字符串复制到其他内存空间

给出该函数的调用示例

```c++
#include <stdio.h>
#include <string.h>
#include <arpa/inet.h>

int main(int argc, char *argv[])
{
	struct sockaddr_in addr1, addr2;
	char *str_ptr;
	char str_arr[20];
   
	addr1.sin_addr.s_addr=htonl(0x1020304);
	addr2.sin_addr.s_addr=htonl(0x1010101);
	
	str_ptr=inet_ntoa(addr1.sin_addr);
	strcpy(str_arr, str_ptr);
	printf("Dotted-Decimal notation1: %s \n", str_ptr);
	
	inet_ntoa(addr2.sin_addr);
	printf("Dotted-Decimal notation2: %s \n", str_ptr);
	printf("Dotted-Decimal notation3: %s \n", str_arr);
	return 0;
}

```

### 网络地址初始化

服务器端套接字创建过程中常见的网络地址信息初始化方法：

```c++
struct sockaddr_in addr;
char *serv_ip = "211.217.168.13";          // 声明 IP 地址字符串
char *serv_port = "9190";                  // 声明端口号字符串
memset(&addr, 0, sizeof(addr));            // 结构体变量 addr 的所有成员初始化为 0，主要是为了将 sockaddr_in 的成员 sin_zero 初始化为 0。
addr.sin_family = AF_INET;                 // 指定地址族
addr.sin_addr.s_addr = inet_addr(serv_ip); // 基于字符串的 IP 地址初始化
addr.sin_port = htons(atoi(serv_port));    // 基于字符串的端口号初始化
```

### INADDR_ANY

初始化地址信息

```c++
struct sockaddr_in addr;
char * serv_port = "9190";
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
add.sin_addr.s_addr = htonl(INADDR_ANY);
addr.sin_port = htons(atoi(serv_port));
```

与之前方式最大的区别在于，利用INADDR_ANY分配服务器端的IP地址若采用这种方式，则可自动获取运行服务器端的计算机IP地址，不必亲自输入。而且，若同一计算机中已分配多个IP地址，则只要端口号一致，就可以从不同IP地址接收数据。

### 向套接字分配网络地址

前面了解了sockaddr_in结构体的初始化方法，接下来就把初始化的地址信息分配给套接字。bind函数负责这项操作

```c++
#include<sys/socket.h>
int bind(int sockfd, struct sockaddr * myaddr, socklen_t addrlen);
	成功时返回0，失败时返回-1。
	参数1：sockfd，要分配地址信息（IP地址和端口号）的套接字文件描述符
	参数2：myaddr，存有地址信息的结构体变量地址值。
	参数3：addrlen，第二个结构体变量的长度。
```

如果此函数调用成功，则将第二个参数指定的地址信息分配给第一个参数中的相应套接字。

```c++
int serv_sock;
struct sockaddr_in serv_addr;
char * srev_port = "9190";/* 创建服务器端套接字（监听套接字）*/
serv_sock = socket(PF_INET, SOCK_STREAM, 0);/* 地址信息初始化 */
memset(&serv_addr, 0, sizeof(serv_addr));
serv_addr.sin_family = AF_INET;
serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);
serv_addr.sin_port = htons(atoi(serv_port));/* 分配地址信息 */
bind(serv_sock, (struct sockaddr * )&serv_addr, sizeof(serv_addr));
```

服务器端代码结构如上

