# Tiny-Web-fuwuqi
 It is used in the design of Tiny Web server.
#ifndef __CSAPP_H__  
#define __CSAPP_H__  
#include <stdio.h>  
#include <stdlib.h>  
#include <unistd.h>  
#include <string.h>  
#include <ctype.h>  
#include <setjmp.h>  
#include <signal.h>  
#include <sys/time.h>  
#include <sys/types.h>  
#include <sys/stat.h>  
#include <fcntl.h>  
#include <sys/mman.h>  
#include <errno.h>  
#include <math.h>  
#include <semaphore.h>  
#include <sys/socket.h>  
#include <netdb.h>  
#include <netinet/in.h>  
#include <arpa/inet.h>
//以上的头文件按说都是在”csapp.h”中，由于电脑中没有这个头文件，就把他们拆成了以下头文件 
#define DEF_MODE   S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH //文件权限定义 
#define DEF_UMASK  S_IWGRP|S_IWOTH  //文件权限定义 
typedef struct sockaddr SA;  
#define RIO_BUFSIZE 8192  
typedef struct {  
    int rio_fd;                /* 内部缓存区的描述符 */  
    int rio_cnt;               /* 内部缓存区剩下还未读的字节数 */  
    char *rio_bufptr;          /* 指向内部缓存区中下一个未读字节 */  
    char rio_buf[RIO_BUFSIZE]; /* 内部缓存区 */  
} rio_t; //定义一个叫rio_t的结构体 
extern char **environ;   
#define MAXLINE  8192  /* 每行最大字符数 */  
#define MAXBUF   8192  /* I/O缓存区的最大容量 */  
#define LISTENQ  1024  /* 监听的第二个参数 */  
ssize_t rio_writen(int fd,void *usrbuf,size_t n);  
void rio_readinitb(rio_t *rp,int fd);  //将程序的内部缓存区与描述符相关联。
ssize_t rio_readlineb(rio_t *rp,void *usrbuf,size_t maxlen);  
/*从内部缓存区读出一个文本行至buf中，以null字符来结束这个文本行。当然，每行最大的字符数量不能超过MAXLINE。*/
int open_clientfd(char *hostname, int portno);  
int open_listenfd(int portno);  
#endif   
void doit(int fd);
void read_requesthdrs(rio_t *rp);  //读并忽略请求报头
int parse_uri(char *uri, char *filename, char *cgiargs);//解析uri，得文件名存入filename中，参数存入cgiargs中。
void serve_static(int fd, char *filename, int filesize);//提供静态服务。
void get_filetype(char *filename, char *filetype);
void serve_dynamic(int fd, char *cause, char *cgiargs);//提供动态服务。
void clienterror(int fd, char *cause, char *errnum, char *shortmsg, char *longmsg);
void doit(int fd)
{
	int is_static;
	struct stat sbuf;
	char buf[MAXLINE],method[MAXLINE],uri[MAXLINE],version[MAXLINE];
	char filename[MAXLINE],cgiargs[MAXLINE];
	rio_t rio;
	rio_readinitb(&rio,fd);
	rio_readlineb(&rio,buf,MAXLINE);
	sscanf(buf,"%s %s %s",method,uri,version);
	if(strcasecmp(method,"GET")) {
		clienterror(fd,method,"501","Not Implemented","Tiny does not implement this method");
		return;
	}//状态消息：未实现；描述：服务器不支持请求的方法。
	read_requesthdrs(&rio);//读并忽略请求报头 
	is_static = parse_uri(uri,filename,cgiargs);
	if(stat(filename,&sbuf) < 0) {
		clienterror(fd,filename, "404", "Not found","Tiny coundn't find this file");
		return;
	}//状态消息：未发现；描述：不能找到所请求的文件 
	if(is_static) {
		if(!(S_ISREG(sbuf.st_mode)) || !(S_IRUSR & sbuf.st_mode)) {
			clienterror(fd,filename, "403", "Forbidden","Tiny coundn't read the file");
			return;
		}
		serve_static(fd,filename,sbuf.st_size);
	}
	else {
		if(!(S_ISREG(sbuf.st_mode)) || !(S_IXUSR & sbuf.st_mode)) {
			clienterror(fd,filename, "403", "Forbidden","Tiny coundn't run the CGI program");
			return;
		}
		serve_dynamic(fd,filename,cgiargs);
	}//状态消息 ：禁止； 描述：无权访问所请求的文件 
}
/*
   从doit函数中可知，我们的Tiny Web服务器只支持“GET”方法，其他方法请求的话则会发送一条错误消息，主程序返回
，并等待下一个请求。否则，我们读并忽略请求报头。（其实，我们在请求服务时，直接不用写请求报头即可，写上只是
为了符合HTTP协议标准）。
   然后，我们将uri解析为一个文件名和一个可能为空的CGI参数，并且设置一个标志位，表明请求的是静态内容还是动态
内容。通过stat函数判断文件是否存在。
   最后，如果请求的是静态内容，我们需要检验它是否是一个普通文件，并且可读。条件通过，则我们服务器向客服端发送
静态内容；相似的，如果请求的是动态内容，我就核实该文件是否是可执行文件，如果是则执行该文件，并提供动态功能。
*/
void clienterror(int fd, char *cause, char *errnum, char *shortmsg, char *longmsg)
{
	char buf[MAXLINE],body[MAXBUF];
	sprintf(body,"<html><title>Tiny Error</title>");
	sprintf(body,"%s<body bgcolor=""ffffff"">\r\n",body);
	sprintf(body,"%s%s: %s\r\n",body,errnum,shortmsg);
	sprintf(body,"%s<p>%s: %s\r\n",body,longmsg,cause);
	sprintf(body,"%s<hr><em>The Web server</em>\r\n",body);
	sprintf(buf,"HTTP/1.0 %s %s\r\n",errnum,longmsg);
	rio_writen(fd,buf,strlen(buf));
	sprintf(buf,"Content-type: text/html\r\n");
	rio_writen(fd,buf,strlen(buf));
	sprintf(buf,"sContent-length: %d\r\n\r\n",(int)strlen(body));
	rio_writen(fd,buf,strlen(buf));
	rio_writen(fd,body,strlen(body));
}
void read_requesthdrs(rio_t *rp)//读并忽略请求报头
{
	char buf[MAXLINE];
	rio_readlineb(rp,buf,MAXLINE);
	while(strcmp(buf,"\r\n")) {
		rio_readlineb(rp,buf,MAXLINE);
		printf("%s", buf);
	}
	return;
}
/*
Tiny不需要请求报头中的任何信息，这个函数就是来跳过这些请求报头的,读这些请求报头，直到空行，然后返回。
*/
int parse_uri(char *uri, char *filename,char *cgiargs)
{
	char *ptr;
 
	if(!strstr(uri,"cgi-bin")) {
		strcpy(cgiargs,"");
		strcpy(filename,".");
		strcat(filename,uri);
        if(uri[strlen(uri)-1] == '/') {
        	strcat(filename,"home.html");
        }
        return 1;
	}
	else {
		ptr = index(uri,'?');
		if(ptr) {
			strcpy(cgiargs,ptr+1);
			*ptr = '\0';
		}
		else {
			strcpy(cgiargs,"");
		}
		strcpy(filename,".");
		strcat(filename,uri);
		return 0;
	}
}
/*
根据uri中是否含有cgi-bin来判断请求的是静态内容还是动态内容。如果没有cgi-bin，则说明请求的是静态内容。那么
，我们需把cgiargs置NULL，然后获得文件名，如果我们请求的uri最后为 “/”，则自动添加上tiny。比如说，我请求
的是“/”,则返回的文件名为“./tiny”,而我们请求“/hybphoto。jpg”,则返回的文件名为“./hybphoto。jpg”。
*/
void serve_static(int fd, char *filename, int filesize)
{
	int srcfd;
	char *srcp,filetype[MAXLINE],buf[MAXBUF];
	get_filetype(filename,filetype);
	sprintf(buf,"HTTP/1.0 200 OK\r\n");
	sprintf(buf,"%sServer:Tiny Web Server\r\n",buf);
	sprintf(buf,"%sContent-length:%d\r\n",buf,filesize);
	sprintf(buf,"%sContent-type:%s\r\n\r\n",buf,filetype);
	rio_writen(fd,buf,strlen(buf));
	srcfd = open(filename,O_RDONLY,0);
	srcp = mmap(0,filesize, PROT_READ, MAP_PRIVATE,srcfd,0);
	close(srcfd);
	rio_writen(fd,srcp,filesize);
	munmap(srcp,filesize);
}
/*打开文件名为filename的文件，把它映射到一个虚拟存储器空间，将文件的前filesize字节映射到从地址srcp开始的虚拟存储区域。关闭文件描述符srcfd，把虚拟存储区的数据写入fd描述符，最后释放虚拟存储器区域。*/
void get_filetype(char *filename, char *filetype)
{
	if(strstr(filename,".html"))
		strcpy(filetype,"text/html");
	else if(strstr(filename,".gif"))
		strcpy(filetype,"image/gif");
	else if(strstr(filename,".jpg"))
		strcpy(filetype,"image/jpg");
	else 
		strcpy(filetype,"text/plain");
}
void serve_dynamic(int fd, char *filename, char *cgiargs)
{
	char buf[MAXLINE],*emptylist[] = {NULL};
	sprintf(buf,"HTTP/1.0 200 OK\r\n");
	rio_writen(fd,buf,strlen(buf));
	sprintf(buf,"Server:Tiny Web Server\r\n");
	rio_writen(fd,buf,strlen(buf));
	if(fork() == 0) {
		setenv("QUERY_STRING",cgiargs,1);
		dup2(fd,STDOUT_FILENO);
		execve(filename,emptylist,environ);
	}
	wait(NULL);
} 
ssize_t rio_writen(int fd, void *usrbuf, size_t n) 
{
    size_t nleft = n;
    ssize_t nwritten;
    char *bufp = usrbuf;
    while (nleft > 0) {
		if ((nwritten = write(fd, bufp, nleft)) <= 0) {
	    	if (errno == EINTR)  
				nwritten = 0;   
	    	else
				return -1;      
		}
		nleft -= nwritten;
		bufp += nwritten;
    }
    return n;
}
static ssize_t rio_read(rio_t *rp, char *usrbuf, size_t n)
{
    int cnt;
    while (rp->rio_cnt <= 0) {  /* 如果缓存区空，则重新填充 */
		rp->rio_cnt = read(rp->rio_fd, rp->rio_buf, sizeof(rp->rio_buf));
		if (rp->rio_cnt < 0) {
	    	if (errno != EINTR) 
				return -1;
			}
		else if (rp->rio_cnt == 0)  /* EOF */
			return 0;
		else 
	   		rp->rio_bufptr = rp->rio_buf; /* 重新设置缓存区指针 */
    }
    /* 从内部缓存区拷贝 min(n, rp->rio_cnt) 个字节到usrbuf*/
    cnt = n;          
    if (rp->rio_cnt < n)   
		cnt = rp->rio_cnt;
    memcpy(usrbuf, rp->rio_bufptr, cnt);
    rp->rio_bufptr += cnt;
    rp->rio_cnt -= cnt;
    return cnt;
}
void rio_readinitb(rio_t *rp, int fd) 
{
    rp->rio_fd = fd;  
    rp->rio_cnt = 0;  
    rp->rio_bufptr = rp->rio_buf;
}
ssize_t rio_readlineb(rio_t *rp, void *usrbuf, size_t maxlen) 
{
    int n, rc;
    char c, *bufp = usrbuf;
    for (n = 1; n < maxlen; n++) { 
		if ((rc = rio_read(rp, &c, 1)) == 1) {
	    	*bufp++ = c;
	    	if (c == '\n')
				break;
		}else if (rc == 0) {
	    	if (n == 1)
				return 0; /* EOF, no data read */
	   		else
				break;    /* EOF, some data was read */
		} else
	   		return -1;	  /* error */
    }
    *bufp = 0;
    return n;
}
int open_clientfd(char *hostname, int port) 
{
    int clientfd;
    struct hostent *hp;
    struct sockaddr_in serveraddr;
    if ((clientfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
		return -1; 
    if ((hp = gethostbyname(hostname)) == NULL)
		return -2; 
    bzero((char *) &serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    bcopy((char *)hp->h_addr_list[0], 
		  (char *)&serveraddr.sin_addr.s_addr, hp->h_length);
    serveraddr.sin_port = htons(port);
    if (connect(clientfd, (SA *) &serveraddr, sizeof(serveraddr)) < 0)
		return -1;
    return clientfd;
}
int open_listenfd(int port) 
{
    int listenfd, optval=1;
    struct sockaddr_in serveraddr;
    /* 创建一个套接字描述符 */
    if ((listenfd = socket(AF_INET, SOCK_STREAM, 0)) < 0)
	return -1;
    /* Eliminates "Address already in use" error from bind. */
    if (setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, 
		   (const void *)&optval , sizeof(int)) < 0)
	return -1;
    /* Listenfd will be an endpoint for all requests to port
       on any IP address for this host */
    bzero((char *) &serveraddr, sizeof(serveraddr));
    serveraddr.sin_family = AF_INET; 
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY); 
    serveraddr.sin_port = htons((unsigned short)port); 
    if (bind(listenfd, (SA *)&serveraddr, sizeof(serveraddr)) < 0)
		return -1;
    /* Make it a listening socket ready to accept connection requests */
    if (listen(listenfd, LISTENQ) < 0)
		return -1;
    return listenfd;
}
int main(int argc, char const *argv[])
{
	int listenfd, connfd, port, clientlen;
	struct sockaddr_in clientaddr;
 
	if(argc != 2) {
		fprintf(stderr, "usage: %s\n", argv[0]);
		exit(1);
	}	
	port = atoi(argv[1]);
	listenfd = open_listenfd(port);
	while(1) {
		clientlen = sizeof(clientaddr);
		connfd = accept(listenfd,(SA *)&clientaddr,&clientlen);
		doit(connfd);
		close(connfd);
	}
}
/*
    Tiny是一个迭代服务器，监听在命令行中确定的端口上的连接请求。在通过open_listenedfd函数打开一个监听套接字以后，Tiny执行典型的无限服务循环，反复地接受一个连接(accept)请求，执行事务(doit)，最后关闭连接描述符(close)
*/
