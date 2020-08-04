# Socket网络编程

## TCP客户/服务器模型

### 回射客户/服务器模型

**echosrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h> 

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void do_service(int conn) {
	//接收
	char recvbuf[1024];
	while(1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int ret = read(conn, recvbuf, sizeof(recvbuf));
		if(ret == 0) {
			printf("client close\n");
			break;
		} 
		else if(ret == -1)
			ERR_EXIT("read");
		fputs(recvbuf, stdout);
		write(conn, recvbuf, ret);
	} 	
}

int main() {
	//创建套接字 
	int listenfd;
	if((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//绑定本机的任意地址 
    
    int on = 1;
    if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
	
	//绑定
	if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	//监听
	if(listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	//连接	
	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	
	pid_t pid; 
	while(1) {
		if((conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen)) < 0)
			ERR_EXIT("accept"); 
		printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
		
		pid = fork();
		if(pid == -1)
			ERR_EXIT("fork");
		if(pid == 0) {
			//子进程不需要处理监听 
			close(listenfd);
			do_service(conn);
			exit(EXIT_SUCCESS);
		}
		else
			//父进程不需要处理连接 
			close(conn);
	} 
	
	return 0;
} 
```

**echocli.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	if(connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
		
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	while(fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		write(sock, sendbuf, strlen(sendbuf));
		read(sock, recvbuf, sizeof(recvbuf));
		fputs(recvbuf, stdout);
		memset(sendbuf, 0, sizeof(sendbuf)); 
		memset(recvbuf, 0, sizeof(recvbuf));
	} 
	
	close(sock);
	
	return 0;
} 
```

### 点对点聊天程序实现

**p2pcrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h> 

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void handler(int sig) {
	printf("recv a sig = %d\n", sig);
	exit(EXIT_SUCCESS);
}

int main() {
	//创建套接字 
	int listenfd;
	if((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//绑定本机的任意地址 
    
    int on = 1;
    if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
	
	//绑定
	if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	//监听
	if(listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	//连接	
	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	
	if((conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen)) < 0)
		ERR_EXIT("accept"); 
	printf("ip = %s  port = %d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
	
	pid_t pid;
	pid = fork();
	if(pid == -1)
		ERR_EXIT("fork");
	if(pid == 0) {
		//子进程用于发送数据
		signal(SIGUSR1, handler); 
		char sendbuf[1024] = {0};
		while(fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
			write(conn, sendbuf, strlen(sendbuf));
			memset(sendbuf, 0, sizeof(sendbuf));
		} 
		printf("child close\n");
		exit(EXIT_SUCCESS);
	}	
	else {
		//父进程用于接收数据 
		char recvbuf[1024];
		while(1) {
			memset(recvbuf, 0, sizeof(recvbuf));
			int ret = read(conn, recvbuf, sizeof(recvbuf));
			if(ret == -1)
				ERR_EXIT("read");
			else if(ret == 0) {
				printf("peer close\n");
				break;
			} 
			fputs(recvbuf, stdout);
		} 	
		printf("parent close\n");
		kill(pid, SIGUSR1);
		exit(EXIT_SUCCESS);		
	}

	close(conn);
	close(listenfd);

	return 0;
} 
```

**p2pcli.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void handler(int sig) {
	printf("recv a sig = %d\n", sig);
	exit(EXIT_SUCCESS);
}

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	if(connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
		
	pid_t pid;
	pid = fork();
	if(pid == -1)
		ERR_EXIT("fork");
	if(pid == 0) {
		//子进程用于接收数据 
		char recvbuf[1024] = {0};
		while(1) {
			memset(recvbuf, 0, sizeof(recvbuf));
			int ret = read(sock, recvbuf, sizeof(recvbuf));
			if(ret == -1)
				ERR_EXIT("read");
			else if(ret == 0) {
				printf("peer close\n");
				break;
			}	
			fputs(recvbuf, stdout);
		} 
		printf("child close\n");
		close(sock);	
		kill(getppid(), SIGUSR1);	
	}
	else {
		//父进程用于发送数据 
		signal(SIGUSR1, handler);
		char sendbuf[1024] = {0}; 
		while(fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
			write(sock, sendbuf, strlen(sendbuf));
			memset(sendbuf, 0, sizeof(sendbuf)); 
		} 	
		printf("parent close\n");
		close(sock);
	}
		
	close(sock);
	
	return 0;
} 
```

### 改进回射客户/服务器模型

**封装定长接收readn函数、定长发送writen函数、自定义协议解决TCP的粘包问题**

**echosrv1.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h> 

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

struct packet {
	int len;
	char buf[1024];
};

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

void do_service(int conn) {
	//接收
	struct packet recvbuf;
	int n;
	while(1) {
		memset(&recvbuf, 0, sizeof(recvbuf));
		int ret = readn(conn, &recvbuf.len, 4);
		if(ret == -1)
			ERR_EXIT("read");
		else if(ret < 4) {
			printf("client close\n");
			break;
		} 
		
		n = ntohl(recvbuf.len);
		ret = readn(conn, recvbuf.buf, n);
		if(ret == -1)
			ERR_EXIT("read");
		else if(ret < n) {
			printf("client close\n");
			break;
		} 
		fputs(recvbuf.buf, stdout);
		writen(conn, &recvbuf, 4 + n);
	} 	
}
int main() {
	//创建套接字 
	int listenfd;
	if((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//绑定本机的任意地址 
    
    int on = 1;
    if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
	
	//绑定
	if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	//监听
	if(listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	//连接	
	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	
	pid_t pid; 
	while(1) {
		if((conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen)) < 0)
			ERR_EXIT("accept"); 
		printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
		
		pid = fork();
		if(pid == -1)
			ERR_EXIT("fork");
		if(pid == 0) {
			//子进程不需要处理监听 
			close(listenfd);
			do_service(conn);
			exit(EXIT_SUCCESS);
		}
		else
			//父进程不需要处理连接 
			close(conn);
	} 
	
	return 0;
} 
```

**echocli1.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

struct packet {
	int len;
	char buf[1024];
};

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	if(connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
	
	struct packet sendbuf;
	struct packet recvbuf;
	memset(&sendbuf, 0, sizeof(sendbuf));
	memset(&recvbuf, 0, sizeof(recvbuf));
	int n;
	while(fgets(sendbuf.buf, sizeof(sendbuf.buf), stdin) != NULL) {
		n = strlen(sendbuf.buf);
		sendbuf.len = htonl(n);
		writen(sock, &sendbuf, 4 + n);
		
		int ret = readn(sock, &recvbuf.len, 4);
		if(ret == -1)
			ERR_EXIT("read");
		else if(ret < 4) {
			printf("client close\n");
			break;
		} 
		
		n = ntohl(recvbuf.len);
		ret = readn(sock, recvbuf.buf, n);
		if(ret == -1)
			ERR_EXIT("read");
		else if(ret < n) {
			printf("client close\n");
			break;
		} 
		fputs(recvbuf.buf, stdout);
		memset(&sendbuf, 0, sizeof(sendbuf));
		memset(&recvbuf, 0, sizeof(recvbuf));
	} 
	
	close(sock);
	
	return 0;
} 
```

**利用readline函数解决TCP粘包问题**

**echosrv2.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h> 

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

ssize_t recv_peek(int sockfd, void *buf, size_t len) {//接收数据后不将数据从缓冲区移除 
	while(1) {
		int ret = recv(sockfd, buf, len, MSG_PEEK);
		if(ret == -1 && errno == EINTR)//被信号中断
			continue;
		return ret;
	} 
}

ssize_t readline(int sockfd, void *buf, size_t maxline) {//只能用于套接口 
	int ret;
	int nread;
	char *bufp = (char *)buf;
	int nleft = maxline;
	while(1) {
		ret = recv_peek(sockfd, bufp, nleft);
		if(ret < 0)//不用再进行中断判断，因为recv_peek函数内部已经进行了 
			return ret;
		else if(ret == 0)//对方关闭
			return ret; 
		nread = ret;
		int i;//判断已接收到的缓冲区中是否有\n 
		for(i = 0; i < nread; ++i) {
			if(bufp[i] == '\n') {
				ret = readn(sockfd, bufp, i + 1);//将数据从缓冲区移除 
				if(ret != i + 1) 
					exit(EXIT_FAILURE);
				return ret;
			}
		}
		
		if(nread > nleft)
			exit(EXIT_FAILURE);
		
		nleft -= nread;
		ret = readn(sockfd, bufp, nread);//还没遇到\n的数据也从缓冲区移除
		if(ret != nread) 
			exit(EXIT_FAILURE);
			
		bufp += nread;
	}
	return -1;
}

void echo_srv(int conn) {
	//接收
	char recvbuf[1024];
	while(1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int ret = readline(conn, recvbuf, 1024);
		if(ret == -1)
			ERR_EXIT("readline");
		if(ret == 0) {
			printf("client close\n");
			break;
		} 
		
		fputs(recvbuf, stdout);
		writen(conn, recvbuf, strlen(recvbuf));
	} 	
}

int main() {
	//创建套接字 
	int listenfd;
	if((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//绑定本机的任意地址 
    
    int on = 1;
    if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
	
	//绑定
	if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	//监听
	if(listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	//连接	
	struct sockaddr_in peeraddr;
	socklen_t peerlen = sizeof(peeraddr);
	int conn;
	
	pid_t pid; 
	while(1) {
		if((conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen)) < 0)
			ERR_EXIT("accept"); 
		printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
		
		pid = fork();
		if(pid == -1)
			ERR_EXIT("fork");
		if(pid == 0) {
			//子进程不需要处理监听 
			close(listenfd);
			echo_srv(conn);
			exit(EXIT_SUCCESS);
		}
		else
			//父进程不需要处理连接 
			close(conn);
	} 
	
	return 0;
} 
```

**echocli2.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

ssize_t recv_peek(int sockfd, void *buf, size_t len) {//接收数据后不将数据从缓冲区移除 
	while(1) {
		int ret = recv(sockfd, buf, len, MSG_PEEK);
		if(ret == -1 && errno == EINTR)//被信号中断
			continue;
		return ret;
	} 
}

ssize_t readline(int sockfd, void *buf, size_t maxline) {//只能用于套接口 
	int ret;
	int nread;
	char *bufp = (char *)buf;
	int nleft = maxline;
	while(1) {
		ret = recv_peek(sockfd, bufp, nleft);
		if(ret < 0)//不用再进行中断判断，因为recv_peek函数内部已经进行了 
			return ret;
		else if(ret == 0)//对方关闭
			return ret; 
		nread = ret;
		int i;//判断已接收到的缓冲区中是否有\n 
		for(i = 0; i < nread; ++i) {
			if(bufp[i] == '\n') {
				ret = readn(sockfd, bufp, i + 1);//将数据从缓冲区移除 
				if(ret != i + 1) 
					exit(EXIT_FAILURE);
				return ret;
			}
		}
		
		if(nread > nleft)
			exit(EXIT_FAILURE);
		
		nleft -= nread;
		ret = readn(sockfd, bufp, nread);//还没遇到\n的数据也从缓冲区移除
		if(ret != nread) 
			exit(EXIT_FAILURE);
			
		bufp += nread;
	}
	return -1;
}

void echo_cli(int sock) {
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	while(fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		writen(sock, sendbuf, strlen(sendbuf));
		
		int ret = readline(sock, recvbuf, sizeof(recvbuf));
		if(ret == -1)
			ERR_EXIT("readline");
		else if(ret == 0) {
			printf("client close\n");
			break;
		} 
		
		fputs(recvbuf, stdout);
		memset(sendbuf, 0, sizeof(sendbuf));
		memset(recvbuf, 0, sizeof(recvbuf));
	} 
	
	close(sock);
}

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1"); 
	
	if(connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
	
	struct sockaddr_in localaddr;
	socklen_t addrlen = sizeof(localaddr);
	if((getsockname(sock, (struct sockaddr*)&localaddr, &addrlen)) < 0)
		ERR_EXIT("getsockname");
	printf("ip = %s  port = %d\n", inet_ntoa(localaddr.sin_addr), ntohs(localaddr.sin_port));
	
	echo_cli(sock);
	
	return 0;
} 
```

## 获取本机ip地址

**gethost.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

/*
struct hostent {
	char  *h_name;
	char **h_aliases;
	int    h_addrtype;
	int    h_length;
	char **h_addr_list;
*/

int getlocalip(char *ip) {
	char host[100] = {0};
	//获取本机主机名 
	if(gethostname(host, sizeof(host)) < 0) 
		return -1;
		
	struct hostent *hp;
	if((hp = gethostbyname(host)) == NULL) 
		return -1;
	strcpy(ip, inet_ntoa(*(struct in_addr*)hp->h_addr));
	return 0;
}	

int main() {
	char host[100] = {0};
	//获取本机主机名 
	if(gethostname(host, sizeof(host)) < 0) 
		ERR_EXIT("gethostname");
		
	//获取主机中所有ip
	struct hostent *hp;
	if((hp = gethostbyname(host)) == NULL) 
		ERR_EXIT("gethostbyname");
	
	int i = 0;
	while(hp->h_addr_list[i] != NULL) {
		printf("%s\n", inet_ntoa(*(struct in_addr*)hp->h_addr_list[i]));
		++i;
	}
	
	char ip[100];
	if(getlocalip(ip) < 0) 
		ERR_EXIT("getlocalip");
	printf("localip = %s\n", ip);
	return 0;
}
```

## select函数管理多个文件描述符(I/O)

### select函数实现单进程下的多个客户端并发

**echosrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <sys/wait.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <netdb.h> 

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

ssize_t recv_peek(int sockfd, void *buf, size_t len) {//接收数据后不将数据从缓冲区移除 
	while(1) {
		int ret = recv(sockfd, buf, len, MSG_PEEK);
		if(ret == -1 && errno == EINTR)//被信号中断
			continue;
		return ret;
	} 
}

ssize_t readline(int sockfd, void *buf, size_t maxline) {//只能用于套接口 
	int ret;
	int nread;
	char *bufp = (char *)buf;
	int nleft = maxline;
	while(1) {
		ret = recv_peek(sockfd, bufp, nleft);
		if(ret < 0)//不用再进行中断判断，因为recv_peek函数内部已经进行了 
			return ret;
		else if(ret == 0)//对方关闭
			return ret; 
		nread = ret;
		int i;//判断已接收到的缓冲区中是否有\n 
		for(i = 0; i < nread; ++i) {
			if(bufp[i] == '\n') {
				ret = readn(sockfd, bufp, i + 1);//将数据从缓冲区移除 
				if(ret != i + 1) 
					exit(EXIT_FAILURE);
				return ret;
			}
		}
		
		if(nread > nleft)
			exit(EXIT_FAILURE);
		
		nleft -= nread;
		ret = readn(sockfd, bufp, nread);//还没遇到\n的数据也从缓冲区移除
		if(ret != nread) 
			exit(EXIT_FAILURE);
			
		bufp += nread;
	}
	return -1;
}

void echo_srv(int conn) {
	//接收
	char recvbuf[1024];
	while(1) {
		memset(recvbuf, 0, sizeof(recvbuf));
		int ret = readline(conn, recvbuf, 1024);
		if(ret == -1)
			ERR_EXIT("readline");
		if(ret == 0) {
			printf("client close\n");
			break;
		} 
		
		fputs(recvbuf, stdout);
		writen(conn, recvbuf, strlen(recvbuf));
	} 	
}

void handle_sigchld(int sig) {
	//捕获子进程的初始状态
	//wait(NULL); 
	while(waitpid(-1, NULL, WNOHANG) > 0);//可以等待所有子进程，大于0表示等待到了一个子进程 
}

int main() {
	//signal(SIGCHLD, SIG_IGN);
	signal(SIGCHLD, handle_sigchld);
	//创建套接字 
	int listenfd;
	if((listenfd = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);//绑定本机的任意地址 
    
    int on = 1;
    if(setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on)) < 0)
        ERR_EXIT("setsockopt");
	
	//绑定
	if(bind(listenfd, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	//监听
	if(listen(listenfd, SOMAXCONN) < 0)
		ERR_EXIT("listen");
	
	//连接	
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int conn;

	int i; 
	int client[FD_SETSIZE];//保存多个客户端连接信息
	int maxi = 0;//最大的不空闲的位置 
	
	for(i = 0; i < FD_SETSIZE; ++i)
		client[i] = -1;//-1表示空闲的 
	 
	int nready;//检测到的事件个数 
	int maxfd = listenfd; 
	fd_set rset;
	fd_set allset;
	FD_ZERO(&rset);
	FD_ZERO(&allset);
	FD_SET(listenfd, &allset);
	
	while(1) {
		rset = allset;
		nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
		if(nready == -1) {
			if(errno == EINTR)//被信号中断
				continue;
			ERR_EXIT("select");
		}
		if(nready == 0)//超时，现在timeout设置为NULL，故不可能发生
			continue; 
		if(FD_ISSET(listenfd, &rset)) {
			peerlen = sizeof(peeraddr);//必须设置一个初始值 
			conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen);
			if(conn == -1)
				ERR_EXIT("accept"); 
			//保存到某个空闲位置 
			for(i = 0; i < FD_SETSIZE; ++i) {
				if(client[i] < 0) {
					client[i] = conn;
					if(i > maxi)
						maxi = i;
					break;
				}		
			} 
			if(i == FD_SETSIZE) {
				fprintf(stderr, "too mang clients\n");
				exit(EXIT_FAILURE);
			}
			printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
			
			FD_SET(conn, &allset);
			if(conn > maxfd)
				maxfd = conn;
			
			if(--nready <= 0)//检测到的事件已经处理完了，应继续监听，没必要处理下面的代码 
				continue;	
		}
		
		//已连接套接口产生了数据
		for(i = 0; i <= maxi; ++i) {
			conn = client[i];
			if(conn == -1)
				continue;
			if(FD_ISSET(conn, &rset)) {
				char recvbuf[1024] = {0};
				int ret = readline(conn, recvbuf, 1024);
				if(ret == -1)
					ERR_EXIT("readline");
				if(ret == 0) {
					printf("client close\n");
					FD_CLR(conn, &allset);
					client[i] = -1;
				} 
				
				fputs(recvbuf, stdout);
				writen(conn, recvbuf, strlen(recvbuf));
				
				if(--nready <= 0)
					break;
			}	
		}
	}
	return 0;
} 
```

**echocli.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

ssize_t readn(int fd, void *buf, size_t count) {//ssize_t 有符号整数  size_t 无符号整数 
	size_t nleft = count;//剩余的字节数 
	ssize_t nread;//已接收的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nread = read(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nread == 0)//对等方关闭了 
			return count - nleft;
		bufp += nread; 
		nleft -= nread;
	}
	return count;
}

ssize_t writen(int fd, const void *buf, size_t count) {
	size_t nleft = count;//剩余要发送的字节数 
	ssize_t nwritten;//已发送的字节数 
	char *bufp = (char*)buf;
	
	while(nleft > 0) {
		if((nwritten = write(fd, bufp, nleft)) < 0) {
			if(errno == EINTR)//被信号中断 
				continue;
			return -1;
		}
		else if(nwritten == 0)
			continue;
		bufp += nwritten; 
		nleft -= nwritten;
	}
	return count;	
}

ssize_t recv_peek(int sockfd, void *buf, size_t len) {//接收数据后不将数据从缓冲区移除 
	while(1) {
		int ret = recv(sockfd, buf, len, MSG_PEEK);
		if(ret == -1 && errno == EINTR)//被信号中断
			continue;
		return ret;
	} 
}

ssize_t readline(int sockfd, void *buf, size_t maxline) {//只能用于套接口 
	int ret;
	int nread;
	char *bufp = (char *)buf;
	int nleft = maxline;
	while(1) {
		ret = recv_peek(sockfd, bufp, nleft);
		if(ret < 0)//不用再进行中断判断，因为recv_peek函数内部已经进行了 
			return ret;
		else if(ret == 0)//对方关闭
			return ret; 
		nread = ret;
		int i;//判断已接收到的缓冲区中是否有\n 
		for(i = 0; i < nread; ++i) {
			if(bufp[i] == '\n') {
				ret = readn(sockfd, bufp, i + 1);//将数据从缓冲区移除 
				if(ret != i + 1) 
					exit(EXIT_FAILURE);
				return ret;
			}
		}
		
		if(nread > nleft)
			exit(EXIT_FAILURE);
		
		nleft -= nread;
		ret = readn(sockfd, bufp, nread);//还没遇到\n的数据也从缓冲区移除
		if(ret != nread) 
			exit(EXIT_FAILURE);
			
		bufp += nread;
	}
	return -1;
}

void echo_cli(int sock) {
	fd_set rset;
	FD_ZERO(&rset);
	
	//循环检测是否产生了可读事件
	int nready;
	int maxfd;
	int fd_stdin = fileno(stdin);//标准输入的文件描述符 
	if(fd_stdin > sock)
		maxfd = fd_stdin;
	else
		maxfd = sock;
		
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	
	while(1) {
		FD_SET(fd_stdin, &rset);
		FD_SET(sock, &rset);
		nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
		if(nready == -1)
			ERR_EXIT("select");
		if(nready == 0)
			continue;
		if(FD_ISSET(sock, &rset)) {
			int ret = readline(sock, recvbuf, sizeof(recvbuf));
			if(ret == -1)
				ERR_EXIT("readline");
			else if(ret == 0) {
				printf("server close\n");
				break;
			} 
			
			fputs(recvbuf, stdout);
			memset(recvbuf, 0, sizeof(recvbuf));	
		}
		if(FD_ISSET(fd_stdin, &rset)) {
			if(fgets(sendbuf, sizeof(sendbuf), stdin) == NULL)
				break;
			writen(sock, sendbuf, strlen(sendbuf));	
			memset(sendbuf, 0, sizeof(sendbuf));
		}
	} 
	close(sock);
}

int main() {
	//创建套接字 
	int sock;
	int i;
	if((sock = socket(PF_INET, SOCK_STREAM, IPPROTO_TCP)) < 0)
		ERR_EXIT("socket");
	
	//地址初始化	
	struct sockaddr_in servaddr; //IPv4地址结构
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188); 
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	if(connect(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("connect");
	
	struct sockaddr_in localaddr;
	socklen_t addrlen = sizeof(localaddr);
	if((getsockname(sock, (struct sockaddr*)&localaddr, &addrlen)) < 0)
		ERR_EXIT("getsockname");
	printf("ip = %s  port = %d\n", inet_ntoa(localaddr.sin_addr), ntohs(localaddr.sin_port));

	echo_cli(sock);
	
	return 0;
} 
```



