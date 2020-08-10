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

## 多个文件描述符(I/O)的管理

### select函数实现单进程下多个客户端并发

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
                    close(conn);
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
    
    int stdineof = 0;
	
	while(1) {
        if(stdineof == 0) 
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
			if(fgets(sendbuf, sizeof(sendbuf), stdin) == NULL) {
                stdineof = 1;
                break;
            }
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

### poll函数实现单进程下多个客户端并发

**pollsrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <poll.h>
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

void handle_sigpipe(int sig) {
	printf("recv a sig = %d\n", sig);
}

int main() {
	//signal(SIGPIPE, SIG_IGN);
	signal(SIGPIPE, handle_sigpipe);
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
	
	/*
	struct pollfd {
		int   fd;//文件描述符
		short events;//请求事件/感兴趣事件 
		short revents;//返回事件 
	};
	*/

	int i; 
	struct pollfd client[2048];//保存多个客户端连接信息
	int maxi = 0;//最大的不空闲的位置 
	
	for(i = 0; i < 2048; ++i)
		client[i].fd = -1;//-1表示空闲的 
	 
	int nready;//检测到的事件个数 
	client[0].fd = listenfd;
	client[0].events = POLLIN;//对可读事件感兴趣 
	
	while(1) {
		nready = poll(client, maxi + 1, -1); 
		if(nready == -1) {
			if(errno == EINTR)//被信号中断
				continue;
			ERR_EXIT("poll");
		}
		if(nready == 0)//超时，现在timeout设置为NULL，故不可能发生
			continue; 
		
		if(client[0].revents & POLLIN) {
			peerlen = sizeof(peeraddr);//必须设置一个初始值 
			conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen);
			if(conn == -1)
				ERR_EXIT("accept"); 
			//保存到某个空闲位置 
			for(i = 0; i < 2048; ++i) {
				if(client[i].fd < 0) {
					client[i].fd = conn;
					if(i > maxi)
						maxi = i;
					break;
				}		
			} 
			if(i == 2048) {
				fprintf(stderr, "too mang clients\n");
				exit(EXIT_FAILURE);
			}
			printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
			
			client[i].events = POLLIN; 
			
			if(--nready <= 0)//检测到的事件已经处理完了，应继续监听，没必要处理下面的代码 
				continue;	
		}
		
		//已连接套接口产生了数据
		for(i = 1; i <= maxi; ++i) {
			conn = client[i].fd;
			if(conn == -1)
				continue;
			if(client[i].events & POLLIN) {
				char recvbuf[1024] = {0};
				int ret = readline(conn, recvbuf, 1024);
				if(ret == -1)
					ERR_EXIT("readline");
				if(ret == 0) {
					printf("client close\n");
					client[i].fd = -1;
					close(conn);
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

### epoll函数实现单进程下多个客户端并发

**epollsrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <signal.h>
#include <fcntl.h>
#include <sys/wait.h>
#include <sys/epoll.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>
#include <vector>
#include <algorithm>

typedef std::vector<struct epoll_event> EventList;

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void activate_nonblock(int fd) {
	int ret;
	int flags = fcntl(fd, F_GETFL);
	if(flags == -1)
		ERR_EXIT("fcntl");
		
	flags |= O_NONBLOCK;
	ret = fcntl(fd, F_SETFL, flags);
	if(ret == -1) 
		ERR_EXIT("fcntl");
}

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

void handle_sigpipe(int sig) {
	printf("recv a sig = %d\n", sig);
}

int main() {
	int count = 0;
	signal(SIGPIPE, handle_sigpipe);
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
	
	std::vector<int> clients;
	int epollfd;
	//创建一个epoll实例，EPOLL_CLOEXEC表示当进程被替换时文件描述符会关闭
	epollfd = epoll_create1(EPOLL_CLOEXEC); 
	
	/*
	typedef union epoll_data {
		void         *ptr;
		int          fd;
		__uint32_t   u32;
		__uint64_t   u64;
	} epoll_data_t;//大小为8个字节 
	
	struct epoll_event {
		__uint32_t   events;//Epoll事件 
		epoll_data_t data;//用户数据变量 ，数据类型为共用体 
	};
	*/
	
	struct epoll_event event;
	event.data.fd = listenfd;
	event.events = EPOLLIN | EPOLLET;//EPOLLET表示边沿方式触发 
	//将感兴趣的文件描述符加入epoll进行管理 
	epoll_ctl(epollfd, EPOLL_CTL_ADD, listenfd, &event);
	
	//连接
	EventList events(16);	
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int conn;
	int i; 
	 
	int nready;//检测到的事件个数 
	while(1) {
		nready = epoll_wait(epollfd, &*events.begin(), static_cast<int>(events.size()), -1);
		if(nready == -1) {
			if(errno == EINTR)//被信号中断
				continue;
			ERR_EXIT("poll_wait");
		}
		if(nready == 0)//超时，现在timeout设置为NULL，故不可能发生
			continue; 
			
		if((size_t)nready == events.size())
			events.resize(events.size() * 2);
		
		//遍历返回的事件
		for(i = 0; i < nready; ++i) {
			if(events[i].data.fd == listenfd) {
				peerlen = sizeof(peeraddr);//必须设置一个初始值 
				conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen);
				if(conn == -1)
					ERR_EXIT("accept"); 

				printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
				printf("count = %d\n", ++count);
				clients.push_back(conn);
				
				activate_nonblock(conn);
				
				event.data.fd = conn;
				event.events = EPOLLIN | EPOLLET;
				epoll_ctl(epollfd, EPOLL_CTL_ADD, conn, &event);				
			}
			else if(events[i].events & EPOLLIN) {
				conn = events[i].data.fd;
				if(conn < 0)
					continue;

				char recvbuf[1024] = {0};
				int ret = readline(conn, recvbuf, 1024);
				if(ret == -1)
					ERR_EXIT("readline");
				if(ret == 0) {
					printf("client close\n");
					close(conn);
					event = events[i];
					epoll_ctl(epollfd, EPOLL_CTL_DEL, conn, &event);
					clients.erase(std::remove(clients.begin(), clients.end(), conn), clients.end());
				} 
				
				fputs(recvbuf, stdout);
				writen(conn, recvbuf, strlen(recvbuf));
			}
		} 
	}
	return 0;
} 
```

## UDP客户/服务器模型

**echosrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>//sockaddr_in
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void echo_srv(int sock) {
	char recvbuf[1024] = {0};
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	int n;//接收到的字节数 
	while(1) {
		peerlen = sizeof(peeraddr);
		memset(recvbuf, 0, sizeof(recvbuf));
		n = recvfrom(sock, recvbuf, sizeof(recvbuf), 0, (struct sockaddr*)&peeraddr, &peerlen);
		if(n == -1) {
			if(errno == EINTR)
				continue;
			ERR_EXIT("recvfrom");
		}
		else if(n > 0) {//回射回去 
			fputs(recvbuf, stdout);
			sendto(sock, recvbuf, n, 0, (struct sockaddr*)&peeraddr, peerlen);
		}
	}
	close(sock);
}

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0)//SOCK_DGRAM代表UDP
		ERR_EXIT("socket");
	//初始化地址
	struct  sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	
	//绑定
	if(bind(sock, (struct sockaddr*)& servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	echo_srv(sock);
	
	return 0;
}
```

**echocli.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>//sockaddr_in
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

void echo_cli(int sock) {
	struct  sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	char sendbuf[1024] = {0};
	char recvbuf[1024] = {0};
	while(fgets(sendbuf, sizeof(sendbuf), stdin) != NULL) {
		sendto(sock, sendbuf, strlen(sendbuf), 0, (struct sockaddr*)&servaddr, sizeof(servaddr));
		recvfrom(sock, recvbuf, sizeof(recvbuf), 0, NULL, NULL);
		fputs(recvbuf, stdout);
		memset(sendbuf, 0, sizeof(sendbuf));
		memset(recvbuf, 0, sizeof(recvbuf));
	}
		
	close(sock);
}

int main() {
	//创建套接字 
	int sock;
	if((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0)//SOCK_DGRAM代表UDP
		ERR_EXIT("socket");
		
	echo_cli(sock);
	
	return 0;
}
```

### UDP聊天室实现

**chatsrv.cpp**

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

#include "pub.h"

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

USER_LIST client_list;//聊天室成员列表 

void do_login(MESSAGE& msg, int sock, struct sockaddr_in *cliaddr);
void do_logout(MESSAGE& msg, int sock, struct sockaddr_in *cliaddr);
void do_sendlist(int sock, struct sockaddr_in *cliaddr);

void chat_srv(int sock) {
	struct sockaddr_in cliaddr;
	socklen_t clilen;
	int n;
	MESSAGE msg;
	while(1) {
		memset(&msg, 0, sizeof(msg));
		clilen = sizeof(cliaddr);
		n = recvfrom(sock, &msg, sizeof(msg), 0, (struct sockaddr*)&cliaddr,&clilen);
		if(n < 0) {
			if(errno == EINTR)
				continue;
			ERR_EXIT("recvfrom");
		}
		
		int cmd = ntohl(msg.cmd);
		switch(cmd) {
		case C2S_LOGIN:
			do_login(msg, sock, &cliaddr);
			break;
		case C2S_LOGOUT:
			do_logout(msg, sock, &cliaddr);
			break;
		case C2S_ONLINE_USER:
			do_sendlist(sock, &cliaddr);
			break;
		default:
			break;
		}
	}
} 

void do_login(MESSAGE& msg, int sock, struct sockaddr_in *cliaddr) {
	USER_INFO user;
	strcpy(user.username, msg.body);
	user.ip = cliaddr->sin_addr.s_addr;
	user.port = cliaddr->sin_port;
	
	//查找用户
	USER_LIST::iterator it;
	for(it = client_list.begin(); it != client_list.end(); ++it) 
		if(strcmp(it->username, msg.body) == 0)
			break;
	
	if(it == client_list.end()) {//没找到用户 
		printf("has a user login : %s <-> %s:%d\n", msg.body, inet_ntoa(cliaddr->sin_addr), cliaddr->sin_port);
		client_list.push_back(user); 
		
		//登录成功应答
		MESSAGE reply_msg;
		memset(&reply_msg, 0, sizeof(reply_msg));
		reply_msg.cmd = htonl(S2C_LOGIN_OK);
		sendto(sock, &reply_msg, sizeof(msg), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
		
		int count = htonl((int)client_list.size());
		//发送在线人数
		sendto(sock, &count, sizeof(int), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
		
		//发送在线列表 
		printf("sending user list informatio to:%s <-> %s:%d\n",msg.body, inet_ntoa(cliaddr->sin_addr), cliaddr->sin_port);
	 	for(it = client_list.begin(); it != client_list.end(); ++it) 
			sendto(sock, &*it, sizeof(USER_INFO), 0, (struct sockaddr*)cliaddr,sizeof(*cliaddr));
		
		//向其他用户通知有新用户登录
		for(it = client_list.begin(); it != client_list.end(); ++it) {
			if(strcmp(it->username, msg.body) == 0)
				continue;
			
			struct sockaddr_in peeraddr;
			memset(&peeraddr, 0, sizeof(peeraddr));
			peeraddr.sin_family = AF_INET;
			peeraddr.sin_port = it->port;
			peeraddr.sin_addr.s_addr = it->ip;
			
			msg.cmd = htonl(S2C_SOMEONE_LOGIN);
			memcpy(msg.body, &user, sizeof(user));
			
			if(sendto(sock, &msg, sizeof(msg), 0, (struct sockaddr*)&peeraddr, sizeof(peeraddr)) < 0)
				ERR_EXIT("sendto");
		} 
	}
	else {//找到用户 
		printf("user %s has already logined\n", msg.body);
		
		MESSAGE reply_msg;
		memset(&reply_msg, 0, sizeof(reply_msg));
		reply_msg.cmd = htonl(S2C_ALREADY_LOGINED);
		sendto(sock, &reply_msg, sizeof(reply_msg), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
	}
}

void do_logout(MESSAGE& msg, int sock, struct sockaddr_in *cliaddr) {
	printf("has a user logout : %s <-> %s:%d\n", msg.body, inet_ntoa(cliaddr->sin_addr), ntohs(cliaddr->sin_port));
	
	USER_LIST::iterator it;
	for(it = client_list.begin(); it != client_list.end(); ++it) {
		if(strcmp(it->username, msg.body) == 0) {
			it = client_list.erase(it); 
			continue;
		} 
			
		struct sockaddr_in peeraddr;
		memset(&peeraddr, 0, sizeof(peeraddr));
		peeraddr.sin_family      = AF_INET;
		peeraddr.sin_port        = it->port;
		peeraddr.sin_addr.s_addr = it->ip;
		
		msg.cmd = htonl(S2C_SOMEONE_LOGOUT);
		
		if(sendto(sock, &msg, sizeof(msg), 0, (struct sockaddr*)&peeraddr, sizeof(peeraddr)) < 0) 
			ERR_EXIT("sendto");
	}
}

void do_sendlist(int sock, struct sockaddr_in *cliaddr) {
	MESSAGE msg;
	msg.cmd = htonl(S2C_ONLINE_USER);
	sendto(sock, (const char*)&msg, sizeof(msg), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
	
	int count = htonl((int)client_list.size());
	//发送在线用户数
	sendto(sock, (const char*)&count, sizeof(int), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
	//发送在线用户列表
	for(USER_LIST::iterator it = client_list.begin(); it != client_list.end(); ++it) {
		sendto(sock,&*it, sizeof(USER_INFO), 0, (struct sockaddr*)cliaddr, sizeof(*cliaddr));
	} 
}

int main() {
	int sock;
	if((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0)//SOCK_DGRAM代表UDP
		ERR_EXIT("socket");
		
	struct  sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	
	if(bind(sock, (struct sockaddr*)&servaddr, sizeof(servaddr)) < 0)
		ERR_EXIT("bind");
		
	chat_srv(sock);
	
	return 0;
} 
```

**chatcli.cpp**

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

#include "pub.h"

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

char username[16];

USER_LIST client_list;//聊天室成员列表

void do_someone_login(MESSAGE& msg);
void do_someone_logout(MESSAGE& msg);
void do_getlist();
void do_chat();

void parse_cmd(char* cmdline, int sock, struct sockaddr_in *servaddr);
bool sendmsgto(int sock, char* username, char* msg);

void parse_cmd(char* cmdline, int sock, struct sockaddr_in *servaddr) {
	char cmd[10] = {0};
	char *p;
	p = strchr(cmdline, ' ');
	if(p != NULL)
		*p = '\0';
		
	strcpy(cmd, cmdline);
	
	if(strcmp(cmd, "exit") == 0) {
		MESSAGE msg;
		memset(&msg, 0, sizeof(msg));
		msg.cmd = htonl(C2S_LOGOUT);
		strcpy(msg.body, username);
		
		if(sendto(sock, &msg, sizeof(msg), 0, (struct sockaddr*)servaddr, sizeof(*servaddr)) < 0)
			ERR_EXIT("sendto");
		
		printf("user %s has logout server\n", username);
		exit(EXIT_SUCCESS);
	}
	else if(strcmp(cmd, "send") == 0) {
		char peername[16] = {0};
		char msg[MSG_LEN] = {0};
		
		/* send  user  msg */
		/*       p     p2  */
		while(*p++ == ' ') ;
		char *p2;
		p2 = strchr(p, ' ');
		if(p2 == NULL) {
			printf("bad command\n");
			printf("\nCommands are:\n");
			printf("send username msg\n");
			printf("list\n");
			printf("exit\n");
			printf("\n");
			return;
		}
		*p2 = '\0';
		strcpy(peername, p);
		
		while(*p2++ == ' ') ;
		strcpy(msg, p2);
		sendmsgto(sock, peername, msg);
	}
	else if(strcmp(cmd, "list") == 0) {
		MESSAGE msg;
		memset(&msg, 0, sizeof(msg));
		msg.cmd = htonl(C2S_ONLINE_USER);
		
		if(sendto(sock, &msg, sizeof(msg), 0, (struct sockaddr*)servaddr, sizeof(*servaddr)) < 0)
			ERR_EXIT("sendto");
	}
	else {
		printf("bad command\n");
		printf("\nCommands are:\n");
		printf("send username msg\n");
		printf("list\n");
		printf("exit\n");
		printf("\n");
	}
} 

bool sendmsgto(int sock, char* name, char* msg) {
	if(strcmp(name, username) == 0) {
		printf("can't send message to self\n");
		return false;
	}
	
	USER_LIST::iterator it;
	for(it = client_list.begin(); it != client_list.end(); ++it) {
		if(strcmp(it->username, name) == 0)
			break;
	}
	
	if(it == client_list.end()) {
		printf("user %s has not logined server\n", name);
		return false;
	}
	
	MESSAGE m;
	memset(&m, 0, sizeof(m));
	m.cmd = htonl(C2C_CHAT);
	
	CHAT_MSG cm;
	strcpy(cm.username, username);
	strcpy(cm.msg, msg);
	
	memcpy(m.body, &cm, sizeof(cm));
	
	struct sockaddr_in peeraddr;
	memset(&peeraddr, 0, sizeof(peeraddr));
	peeraddr.sin_family      = AF_INET;
	peeraddr.sin_addr.s_addr = it->ip;
	peeraddr.sin_port        = it->port;
	
	in_addr tmp;
	tmp.s_addr = it->ip;
	
	printf("sending message [%s] to user [%s] <-> %s:%d\n", msg, name, inet_ntoa(tmp), it->port);
	
	sendto(sock, (const char*)&m, sizeof(m), 0, (struct sockaddr*)&peeraddr, sizeof(peeraddr));
	return true;
}

void do_getlist(int sock) {
	int count;
	recvfrom(sock, &count, sizeof(int), 0, NULL, NULL);
	printf("has %d users logined server\n", ntohl(count));
	client_list.clear();
	
	int n = ntohl(count);
	for(int i = 0; i < n; ++i) {
		USER_INFO user;
		recvfrom(sock, &user, sizeof(USER_INFO), 0, NULL, NULL);
		in_addr tmp;
		tmp.s_addr = user.ip;
		
		printf("%s <-> %s:%d\n", user.username, inet_ntoa(tmp), ntohs(user.port));
	}
}

void do_someone_login(MESSAGE& msg) {
	USER_INFO *user = (USER_INFO*)msg.body;
	in_addr tmp;
	tmp.s_addr = user->ip;
	printf("%s <-> %s:%d has logined server\n", user->username, inet_ntoa(tmp), user->port);
	client_list.push_back(*user);
} 

void do_someone_logout(MESSAGE& msg) {
	USER_LIST::iterator it;
	for(it = client_list.begin(); it != client_list.end(); ++it) {
		if(strcmp(it->username, msg.body) == 0)
			break;
	}
	
	if(it != client_list.end())
		client_list.erase(it);
		
	printf("user %s has logout server\n", msg.body);
}

void do_chat(const MESSAGE& msg) {
	CHAT_MSG *cm = (CHAT_MSG*)msg.body;
	printf("recv a msg [%s] from [%s]\n", cm->msg, cm->username);
}

void chat_cli(int sock) {
	struct  sockaddr_in servaddr;
	memset(&servaddr, 0, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_port = htons(5188);
	servaddr.sin_addr.s_addr = inet_addr("127.0.0.1");
	
	struct sockaddr_in peeraddr;
	socklen_t peerlen;
	
	MESSAGE msg;
	while(1) {
		memset(username, 0, sizeof(username));
		printf("please input your name:");
		fflush(stdout);
		scanf("%s", username);
		
		memset(&msg, 0, sizeof(msg));
		msg.cmd = htonl(C2S_LOGIN);
		strcpy(msg.body, username);
		
		sendto(sock, &msg, sizeof(msg), 0, (struct sockaddr*)&servaddr, sizeof(servaddr));
		
		memset(&msg, 0, sizeof(msg));
		recvfrom(sock, &msg, sizeof(msg), 0, NULL, NULL);
		int cmd = ntohl(msg.cmd);
		if(cmd == S2C_ALREADY_LOGINED) 
			printf("user %s already logined server, please use another username\n", username);
		else if(cmd == S2C_LOGIN_OK) {
			printf("user %s already server\n", username);
			break;
		}
	}
	int count;
	recvfrom(sock, &count, sizeof(int), 0, NULL, NULL);
	
	int n = ntohl(count);
	printf("has %d user logined server\n", n);
	
	for(int i = 0; i < n; ++i) {
		USER_INFO user;
		recvfrom(sock, &user, sizeof(USER_INFO), 0, NULL, NULL);
		client_list.push_back(user);
		in_addr tmp;
		tmp.s_addr = user.ip;
		
		printf("%d %s <-> %s:%d\n", i, user.username, inet_ntoa(tmp), ntohs(user.port));
	}
	
	printf("\nCommands are:\n");
	printf("send username msg\n");
	printf("list\n");
	printf("exit\n");
	printf("\n");
	
	fd_set rset;
	FD_ZERO(&rset);
	int nready;
	while(1) {
		FD_SET(STDIN_FILENO, &rset);
		FD_SET(sock, &rset);
		nready = select(sock + 1, &rset, NULL, NULL, NULL);
		if(nready == -1)
			ERR_EXIT("select");
		if(nready == 0)
			continue;
		if(FD_ISSET(sock, &rset)) {
			peerlen = sizeof(peeraddr);
			memset(&msg, 0, sizeof(msg));
			recvfrom(sock, &msg, sizeof(msg), 0, (struct sockaddr*)&peeraddr, &peerlen);
			int cmd = ntohl(msg.cmd);
			switch(cmd) {
			case S2C_SOMEONE_LOGIN:
				do_someone_login(msg);
				break;
			case S2C_SOMEONE_LOGOUT:
				do_someone_logout(msg);
				break;
			case S2C_ONLINE_USER:
				do_getlist(sock);
				break;
			case C2C_CHAT:
				do_chat(msg);
				break;
			defalut:
				break;	
			}
		}
		if(FD_ISSET(STDIN_FILENO, &rset)) {
			char cmdline[100] = {0};
			if(fgets(cmdline, sizeof(cmdline), stdin) == NULL)
				break;
				
			if(cmdline[0] == '\n')
				continue;
			cmdline[strlen(cmdline) - 1] = '\0';
			parse_cmd(cmdline, sock, &servaddr);
		}
	}
	
	memset(&msg, 0, sizeof(msg));
	msg.cmd = htonl(C2S_LOGOUT);
	strcpy(msg.body, username);
	
	sendto(sock, (const char*)&msg, sizeof(msg), 0, (struct sockaddr*)&servaddr, sizeof(servaddr));
	close(sock);
}

int main() {
	int sock;
	if((sock = socket(PF_INET, SOCK_DGRAM, 0)) < 0)
		ERR_EXIT("socket");
	
	chat_cli(sock);
	 
	return 0;
}
```

## 多线程

**thread.cpp**

**编译语句为：g++ thread.cpp -o thread -lpthread**

```c++
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <pthread.h>
#include <stdlib.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)

void* thread_routine(void *arg) {
	for(int i = 0; i < 20; ++i) {
		printf("B");
		fflush(stdout);
		usleep(20);
		if(i == 3)
			pthread_exit(const_cast<char*>("ABC"));
	}

	return 0;
}

int main() {
	pthread_t tid;
	int ret;
	if((ret = pthread_create(&tid, NULL, thread_routine, NULL)) != 0) {
		fprintf(stderr, "pthread_create:%s\n", strerror(ret));
		exit(EXIT_FAILURE);
	}

	for(int i = 0; i < 20; ++i) {
		printf("A");
		fflush(stdout);
		usleep(20);
	}

	void *value;
	if((ret = pthread_join(tid, &value) != 0)) {//等待其他线程完成 
		fprintf(stderr, "pthread_join:%s\n", strerror(ret));
		exit(EXIT_FAILURE);
	}

	printf("\nreturn msg = %s\n", (char*)value);

	return 0;
}
```

### 多线程下的回射客户/服务器模型

**echosrv.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <pthread.h>
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

void* thread_routine(void* arg) {
	pthread_detach(pthread_self());
	int conn = (int)arg;
	do_service(conn);
	printf("exiting thread……\n");
	return NULL;
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
	
	while(1) {
		if((conn = accept(listenfd, (struct sockaddr*)&peeraddr, &peerlen)) < 0)
			ERR_EXIT("accept"); 
		printf("ip=%s port=%d\n", inet_ntoa(peeraddr.sin_addr), ntohs(peeraddr.sin_port));
		
		pthread_t tid;
		int ret;
		int *p = malloc(sizeof(int));
		*p = conn;
		if((ret = pthread_create(&tid, NULL, thread_routine, p)) != 0) {
			fprintf(stderr, "pthread_create:%s\n", strerror(ret));
			exit(EXIT_FAILURE);
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

### 信号量+互斥锁实现生产者-消费者模型

**pctest.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <pthread.h> 
#include <semaphore.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>

#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)
	
#define CONSUMERS_COUNT 1
#define PRODUCERS_COUNT 5
#define BUFFSIZE 10

int g_buffer[BUFFSIZE];

unsigned short in = 0;//从0号位置开始存放 
unsigned short out = 0;//从0号位置开始取走
unsigned short produce_id = 0;
unsigned short consumer_id = 0;

sem_t g_sem_full;
sem_t g_sem_empty;
pthread_mutex_t g_mutex;

pthread_t g_thread[CONSUMERS_COUNT + PRODUCERS_COUNT];

void* consume(void *arg) {
	int num = (long)arg;
	int i;
	while(1) {
		printf("%d wait buffer not empty\n", num);
		sem_wait(&g_sem_empty);
		pthread_mutex_lock(&g_mutex);
		for(i = 0; i < BUFFSIZE; ++i) {
			printf("%02d ", i);
			if(g_buffer[i] == -1)
				printf("%s", "null");
			else
				printf("%d", g_buffer[i]);
			if(i == out)
				printf("\t<--consume");
			printf("\n");
		}
		int consume_id = g_buffer[out];
		printf("%d begin consume product %d\n", num, consume_id);
		g_buffer[out] = -1;	
		out = (out + 1) % BUFFSIZE;
		printf("%d end consume product %d\n", num, consume_id);
		pthread_mutex_unlock(&g_mutex);
		sem_post(&g_sem_full);	
		sleep(5);	
	}

	return NULL;
}

void* produce(void *arg) {
	int num = (long)arg;
	int i;
	while(1) {
		printf("%d wait buffer not full\n", num);
		sem_wait(&g_sem_full);
		pthread_mutex_lock(&g_mutex);
		for(i = 0; i < BUFFSIZE; ++i) {
			printf("%02d ", i);
			if(g_buffer[i] == -1)
				printf("%s", "null");
			else
				printf("%d", g_buffer[i]);
			if(i == in)
				printf("\t<--produce");
			printf("\n");
		}
		
		printf("%d begin produce product %d\n", num, produce_id);
		g_buffer[in] = produce_id;	
		in = (in + 1) % BUFFSIZE;
		printf("%d end produce product %d\n", num, produce_id++);
		pthread_mutex_unlock(&g_mutex);
		sem_post(&g_sem_empty);	
		sleep(1);	
	}

	return NULL;
}

int main() {
	int i;
	for(i = 0; i < BUFFSIZE; ++i)
		g_buffer[i] = -1;
	sem_init(&g_sem_full, 0, BUFFSIZE);
	sem_init(&g_sem_empty, 0, 0);
	
	pthread_mutex_init(&g_mutex, NULL);
	
	for(i = 0; i < CONSUMERS_COUNT; ++i)
		pthread_create(&g_thread[i], NULL, consume, (void*)(long)i);

	for(i = 0; i < PRODUCERS_COUNT; ++i)
		pthread_create(&g_thread[CONSUMERS_COUNT + i], NULL, produce, (void*)(long)i);	
		
	for(i = 0; i < CONSUMERS_COUNT + PRODUCERS_COUNT; ++i)
		pthread_join(g_thread[i], NULL);	
		
	sem_destroy(&g_sem_full);
	sem_destroy(&g_sem_empty);
	pthread_mutex_destroy(&g_mutex);

	return 0;
}
```

### 条件变量+互斥锁实现生产者-消费者模型

**pctest.cpp**

```c++
#include <unistd.h>
#include <sys/types.h>
#include <pthread.h> 
#include <semaphore.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>
#include <string.h>


#define ERR_EXIT(m) \
	do { \
		perror(m); \
		exit(EXIT_FAILURE); \
	} while(0)
	
#define CONSUMERS_COUNT 2
#define PRODUCERS_COUNT 1

pthread_mutex_t g_mutex;
pthread_cond_t g_cond;

pthread_t g_thread[CONSUMERS_COUNT + PRODUCERS_COUNT];

int nready = 0;

void* consume(void *arg) {
	int num = (long)arg;
	while(1) {
		pthread_mutex_lock(&g_mutex);
		while(nready == 0) {//条件不满足则等待 
			//解锁，等待其他线程改变nready的值 
			printf("consumer %d begin wait a condition...\n", num);
			pthread_cond_wait(&g_cond, &g_mutex);
		}
		printf("consumer %d end wait a condition...\n", num);
        printf("consumer %d begin consume product...\n", num);
		--nready;
		pthread_mutex_unlock(&g_mutex);
		sleep(1);
	}

	return NULL;
}

void* produce(void *arg) {
	int num = (long)arg;

	while(1) {
		pthread_mutex_lock(&g_mutex);
        printf("producer %d begin produce product...\n", num);
		++nready;
        printf("producer %d end produce product...\n", num);
		printf("producer %d signal...\n", num);
		pthread_cond_signal(&g_cond);
		pthread_mutex_unlock(&g_mutex);
		sleep(5);
	}

	return NULL;
}

int main() {
	int i;
	
	pthread_mutex_init(&g_mutex, NULL);
	pthread_cond_init(&g_cond, NULL);
	

	for(i = 0; i < CONSUMERS_COUNT; ++i)
		pthread_create(&g_thread[i], NULL, consume, (void*)(long)i);
		
	sleep(1); 

	for(i = 0; i < PRODUCERS_COUNT; ++i)
		pthread_create(&g_thread[CONSUMERS_COUNT + i], NULL, produce, (void*)(long)i);	
		
	for(i = 0; i < CONSUMERS_COUNT + PRODUCERS_COUNT; ++i)
		pthread_join(g_thread[i], NULL);	
		
	pthread_mutex_destroy(&g_mutex);
	pthread_cond_destroy(&g_cond);

	return 0;
}
```

## 简单的线程池实现

**condition.h**

```c++
#ifndef _CONDITION_H_
#define _CONDITION_H_

#include <pthread.h>

typedef struct condition {
	pthread_mutex_t pmutex;
	pthread_cond_t pcond;
} condition_t;

int condition_init(condition_t *cond);
int condition_lock(condition_t *cond);
int condition_unlock(condition_t *cond);
int condition_wait(condition_t *cond);
int condition_timedwait(condition_t *cond, const struct timespec *abstime);
int condition_signal(condition_t *cond);
int condition_broadcast(condition_t *cond);
int condition_destroy(condition_t *cond);

#endif /* _CONDITION_H_ */
```

**condition.cpp**

```c++
#include "condition.h"

int condition_init(condition_t *cond) {
	int status;
	if((status = pthread_mutex_init(&cond->pmutex, NULL)))
		return status;
	
	if((status = pthread_cond_init(&cond->pcond, NULL)))
		return status;
		
	return 0;
}

int condition_lock(condition_t *cond) {
	return pthread_mutex_lock(&cond->pmutex);
}

int condition_unlock(condition_t *cond) {
	return pthread_mutex_unlock(&cond->pmutex);
}

int condition_wait(condition_t *cond) {
	return pthread_cond_wait(&cond->pcond, &cond->pmutex);
}

int condition_timedwait(condition_t *cond, const struct timespec *abstime) {
	return pthread_cond_timedwait(&cond->pcond, &cond->pmutex, abstime);
}

int condition_signal(condition_t *cond) {
	return pthread_cond_signal(&cond->pcond);
}

int condition_broadcast(condition_t *cond) {
	return pthread_cond_broadcast(&cond->pcond);
}
int condition_destroy(condition_t *cond) {
	int status;
	if((status = pthread_mutex_destroy(&cond->pmutex)))
		return status;
		
	if((status = pthread_cond_destroy(&cond->pcond)))
		return status;
		
	return 0;
}
```

**threadpool.h**

```c++
#ifndef _THREAD_POOL_H_
#define _THREAD_POOL_H_

#include "condition.h"

typedef struct task {
	void *(*run)(void *arg); //任务回调函数
	void *arg;               //回调函数参数
	struct task *next; 
} task_t; 

typedef struct threadpool {
	condition_t ready; //任务准备就绪或者线程池销毁通知
	task_t *first;     //任务队列头指针
	task_t *last;      //任务队列尾指针
	int counter;       //线程池中当前线程数
	int idle;          //线程池中当前正在等待任务的线程数
	int max_treads;    //线程池中最大允许的线程数
	int quit;          //销毁线程池的时候置1 
} threadpool_t;

//初始化线程池 
void threadpool_init(threadpool_t *pool, int threads);
//往线程池中添加任务
void threadpool_add_task(threadpool_t *pool, void *(*run)(void *arg), void *arg);
//销毁线程池
void threadpool_destroy(threadpool_t *pool);

#endif /* _THREAD_POOL_H_ */ 
```

**threadpool.cpp**

```c++
#include "threadpool.h"
#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <errno.h>
#include <time.h>

void *thread_routine(void *arg) {
	struct timespec abstime;
	int timeout;
	printf("thread 0x%x is starting\n", (int)pthread_self());
	threadpool_t *pool = (threadpool_t *)arg;
	while(1) {
		timeout = 0;
		condition_lock(&pool->ready);
		pool->idle++;
		//等待队列有任务到来或者线程池销毁通知
		while(pool->first == NULL && !pool->quit) {
			printf("thread 0x%x is waiting\n", (int)pthread_self());
			clock_gettime(CLOCK_REALTIME, &abstime);
			abstime.tv_sec += 2;
			int status = condition_timedwait(&pool->ready, &abstime);
			if(status == ETIMEDOUT) {
				printf("thread 0x%x is wait timed out\n", (int)pthread_self());
				timeout = 1;
				break;
			}
		} 
		
		//等待到条件，处于工作状态 
		pool->idle--;
		
		//等待到任务 
		if(pool->first != NULL) {
			//从队头取出任务 
			task_t *t = pool->first; 
			pool->first = t->next;
			//执行任务需要一定时间，所以要先解锁，以便生产者进程
			//能够往队列中添加新任务，其他消费者进程能够进入等待任务 
			condition_unlock(&pool->ready);
			t->run(t->arg);
			free(t);
			condition_lock(&pool->ready);
		} 
		//如果等待到线程池销毁通知，且任务都执行完毕 
		if(pool->quit && pool->first == NULL) {
			pool->counter--;
			if(pool->counter == 0)
				condition_signal(&pool->ready);
			//跳出循环之前要基带解锁 
			condition_unlock(&pool->ready);
			break;
		} 
		
		if(timeout && pool->first == NULL) {
			pool->counter--;
			//跳出循环之前要基带解锁 
			condition_unlock(&pool->ready);
			break;	
		}
		condition_unlock(&pool->ready);
	}
	
	printf("thread 0x%x is exiting\n", (int)pthread_self());
	return NULL;
}

//初始化线程池 
void threadpool_init(threadpool_t *pool, int threads) {
	//对线程池中的各个字段初始化 
	condition_init(&pool->ready);
	pool->first = NULL;
	pool->last = NULL;
	pool->counter = 0;
	pool->idle = 0;
	pool->max_treads = threads;
	pool->quit = 0;
} 
 
//往线程池中添加任务
void threadpool_add_task(threadpool_t *pool, void *(*run)(void *arg), void *arg) {
	//生成新任务 
	task_t *newtask = (task_t *)malloc(sizeof(task_t));
	newtask->run = run;
	newtask->arg = arg;
	newtask->next = NULL;
	
	condition_lock(&pool->ready);
	//将任务添加到队列
	if(pool->first == NULL)
		pool->first = newtask;
	else
	 	pool->last->next = newtask;
	pool->last = newtask;
	
	//如果有等待线程，则唤醒其中一个 
	if(pool->idle > 0) {
		condition_signal(&pool->ready);
	}
	else if(pool->counter < pool->max_treads) {
		//没有等待线程，并且当前线程数不超过最大线程数，则创建一个新线程
		pthread_t tid;
		pthread_create(&tid, NULL, thread_routine, pool);
		pool->counter++;
	}
	
	condition_unlock(&pool->ready);
}
 
//销毁线程池
void threadpool_destroy(threadpool_t *pool) {
	if(pool->quit) 
		return; 
		
	condition_lock(&pool->ready);	
	pool->quit = 1;	
	if(pool->counter > 0) {
		if(pool->idle > 0) 
			condition_broadcast(&pool->ready);
			
		//处于执行任务状态中的线程，不会收到广播 
		//线程池需要等待执行任务状态中的线程全部退出 	
		while(pool->counter > 0) 
			condition_wait(&pool->ready);	
	}
	condition_unlock(&pool->ready);
	condition_destroy(&pool->ready);
} 
```

**main.cpp**

```c++
#include "threadpool.h"
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>

void* mytask(void *arg) {
	printf("thread 0x%x is working on task %d\n", (int)pthread_self(), *(int*)arg);
	sleep(1);
	free(arg);
	return NULL;
}

int main() {
	threadpool_t pool;
	threadpool_init(&pool, 3);
	
	int i;
	for(i = 0; i < 10; ++i) {
		int *arg = (int*)malloc(sizeof(int));
		*arg = i;
		threadpool_add_task(&pool, mytask, arg);		
	}

	threadpool_destroy(&pool);
	return 0;
} 
```

**Makefile**

```c++
OBJS = main.o threadpool.o condition.o
main: $(OBJS)
	g++ $(OBJS) -o main -lpthread -lrt
main.o: main.cpp threadpool.h condition.h
	g++ -c main.cpp -o main.o
threadpool.o: threadpool.cpp threadpool.h condition.h
	g++ -c threadpool.cpp -o threadpool.o
condition.o: condition.cpp condition.h
	g++ -c condition.cpp -o condition.o
clean:
	rm -rf *.o main
```



