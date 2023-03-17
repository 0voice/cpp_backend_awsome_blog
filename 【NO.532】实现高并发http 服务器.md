# 【NO.532】实现高并发http 服务器

(实现高并发http 服务器)



# 1.需求分析

实现一个http 服务器项目，服务器启动后监听80端口的tcp 连接，当用户通过任意一款浏览器（IE、火狐和腾讯浏览器等）访问我们的http服务器，http服务器会查找用户访问的html页面是否存在，如果存在则通过http 协议响应客户端的请求，把页面返回给浏览器，浏览器显示html页面；如果页面不存在，则按照http 协议的规定，通知浏览器此页面不存在（404 NOT FOUND）

# 2.何为Html 页面

html，全称Hypertext Markup Language，也就是“超文本链接标示语言”。HTML文本是由 HTML命令组成的描述性文本，HTML 命令可以说明文字、 图形、动画、声音、表格、链接等。 即平常上网所看到的的网页。

```html
<html lang=\"zh-CN\">



<head>



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">



<title>This is a test</title>



</head>



<body>



<div align=center height=\"500px\" >



<br/><br/><br/>



<h2>大家好</h2><br/><br/>



<form action="commit" method="post">



尊姓大名: <input type="text" name="name" />



<br/>芳龄几何: <input type="password" name="age" />



<br/><br/><br/><input type="submit" value="提交" />



<input type="reset" value="重置" />



</form>



</div>



</body>



</html>
```

# 3.何为http 协议

HTTP协议是Hyper Text Transfer Protocol(超文本传输协议)的缩写,是用于从万维网(WWW:World Wide Web )服务器传输超文本到本地浏览器的传送协议。
请求格式：
**客户端请求**
客户端发送一个HTTP请求到服务器的请求消息包括以下格式：请求行（request line）、请求头部（header）、空行和请求数据四个部分组成，下图给出了请求报文的一般格式。

![在这里插入图片描述](https://img-blog.csdnimg.cn/43ca25e5af6a4ee292e4d09f54b8088b.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



**服务端响应**
服务器响应客户端的HTTP响应也由四个部分组成，分别是：状态行、消息报头、空行和响应正文。

![在这里插入图片描述](https://img-blog.csdnimg.cn/3692a9e4bbe145649c5c5e51a7022574.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)



# 4.实现Mini型http 服务器



![在这里插入图片描述](https://img-blog.csdnimg.cn/cee19b52ccc2450ca95b37dfbd420e5f.png)



## 4.1 接收http请求

实现按行读取请求头部

```cpp
//返回值： -1 表示读取出错， 等于0表示读到一个空行， 大于0 表示成功读取一行



int get_line(int sock, char *buf, int size){



    int count = 0;



    char ch = '\0';



    int len = 0;



    



    



    while( (count<size - 1) && ch!='\n'){



        len = read(sock, &ch, 1);



        



        if(len == 1){



            if(ch == '\r'){



                continue;



            }else if(ch == '\n'){



                //buf[count] = '\0';



                break;



            }



            



            //这里处理一般的字符



            buf[count] = ch;



            count++;



            



        }else if( len == -1 ){//读取出错



            perror("read failed");



            count = -1;



            break;



        }else {// read 返回0,客户端关闭sock 连接.



            fprintf(stderr, "client close.\n");



            count = -1;



            break;



        }



    }



    



    if(count >= 0) buf[count] = '\0';



    



    return count;



}
```

如果碰到两个连续的回车换行，即，意味着请求头部结束

```cpp
// 1.读取请求行



void do_http_request(int client_sock)



{



    int len = 0;



    char buf[256];



    char method[64];



    char url[256];



    char path[256];



 



    /*读取客户端发送的http 请求*/



 



    // 1.读取请求行



    len = get_line(client_sock, buf, sizeof(buf));



 



    if (len > 0)



    { //读到了请求行



        int i = 0, j = 0;



        while (!isspace(buf[j]) && (i < sizeof(method) - 1))



        {



            method[i] = buf[j];



            i++;



            j++;



        }



 



        method[i] = '\0';



        if (debug)



            printf("request method: %s\n", method);



 



        if (strncasecmp(method, "GET", i) == 0)



        { //只处理get请求



            if (debug)



                printf("method = GET\n");



 



            //获取url



            while (isspace(buf[j++]))



                ; //跳过白空格



            i = 0;



 



            while (!isspace(buf[j]) && (i < sizeof(url) - 1))



            {



                url[i] = buf[j];



                i++;



                j++;



            }



 



            url[i] = '\0';



 



            if (debug)



                printf("url: %s\n", url);



 



            //继续读取http 头部



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



 



            //***定位服务器本地的html文件***



 



            //处理url 中的?



            {



                char *pos = strchr(url, '?');



                if (pos)



                {



                    *pos = '\0';



                    printf("real url: %s\n", url);



                }



            }



 



            sprintf(path, "./html_docs/%s", url);



            if (debug)



                printf("path: %s\n", path);



 



            //执行http 响应



        }



        else



        {



            //非get请求, 读取http 头部，并响应客户端 501     Method Not Implemented



            fprintf(stderr, "warning! other request [%s]\n", method);



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



            // unimplemented(client_sock);   //在响应时再实现



        }



    }



    else



    {   //请求格式有问题，出错处理



        // bad_request(client_sock);   //在响应时再实现



    }



}
```

## 4.2 解析请求

### 4.2.1 响应http 请求

```cpp
void do_http_request(int client_sock)



{



    int len = 0;



    char buf[256];



    char method[16];



    char url[256];



 



    /*读取客户端发送的http请求*/



 



    // 1.读取请求行



    len = get_line(client_sock, buf, sizeof(buf));



 



    if (len > 0)



    {



        int i, j;



        while (!isspace(buf[j]) && (i < sizeof(method) - 1))



        {



            method[i] = buf[j];



            i++;



            j++;



        }



        method[i] = '\0';



 



        //判断方法是否合法



 



        if (strncasecmp(method, "GET", i) == 0)



        { // GET 方法



            printf("requst = %s\n", method);



 



            //获取url



            while (isspace(buf[++j]))



                ;



            i = -1;



            while (!isspace(buf[j]) && (i < sizeof(url) - 1))



            {



                url[i] = buf[j];



                i++;



                j++;



            }



            url[i] = '\0';



 



            printf("url: %s\n", url);



 



            //读取http 头部，不做任何处理



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                printf("read line: %s\n", buf);



            } while (len > 0);



            do_http_response(client_sock);



        }



        else



        {



            printf("other requst = %s\n", method);



            //读取http 头部，不做任何处理



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                printf("read line: %s\n", buf);



            } while (len > 0);



        }



    }



    else



    { //出错的处理



    }



}



void do_http_response(int client_sock)



{



    const char *main_header = "HTTP/1.0 200 OK\r\nServer: Martin Server\r\nContent-Type: text/html\r\nConnection: Close\r\n";



 



    const char *welcome_content = "\



<html lang=\"zh-CN\">\n\



<head>\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\n\



<title>This is a test</title>\n\



</head>\n\



<body>\n\



<div align=center height=\"500px\" >\n\



<br/><br/><br/>\n\



<h2>大家好</h2><br/><br/>\n\



<form action=\"commit\" method=\"post\">\n\



尊姓大名: <input type=\"text\" name=\"name\" />\n\



<br/>芳龄几何: <input type=\"password\" name=\"age\" />\n\



<br/><br/><br/><input type=\"submit\" value=\"提交\" />\n\



<input type=\"reset\" value=\"重置\" />\n\



</form>\n\



</div>\n\



</body>\n\



</html>";



 



    char send_buf[64];



    int wc_len = strlen(welcome_content);



    int len = write(client_sock, main_header, strlen(main_header));



 



    if (debug)



        fprintf(stdout, "... do_http_response...\n");



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, main_header);



 



    len = snprintf(send_buf, 64, "Content-Length: %d\r\n\r\n", wc_len);



    len = write(client_sock, send_buf, len);



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, send_buf);



 



    len = write(client_sock, welcome_content, wc_len);



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, welcome_content);



}
```

## 4.3 读取文件

### 4.3.1 **文件概念简介**



![在这里插入图片描述](https://img-blog.csdnimg.cn/333a29aa6b824c079aafe3d702137a5c.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_20,color_FFFFFF,t_70,g_se,x_16)


inode - "索引节点",储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。每个inode都有一个号码，操作系统用inode号码来识别不同的文件。ls -i 查看inode 号



## 4.4 stat函数

作用：返回文件的状态信息
\#include <sys/types.h>
\#include <sys/stat.h>
\#include <unistd.h>

```javascript
   int stat(const char *path, struct stat *buf);



   int fstat(int fd, struct stat *buf);



   int lstat(const char *path, struct stat *buf);
```

path:
文件的路径
buf:
传入的保存文件状态的指针，用于保存文件的状态
返回值：
成功返回0，失败返回-1，设置errno

struct stat {
dev_t st_dev; /* ID of device containing file */
ino_t st_ino; /* inode number */
mode_t st_mode; /* S_ISREG(st_mode) 是一个普通文件 S_ISDIR(st_mode) 是一个目录*/

```javascript
           nlink_t   st_nlink;   /* number of hard links */



           uid_t     st_uid;     /* user ID of owner */



           gid_t     st_gid;     /* group ID of owner */



           dev_t     st_rdev;    /* device ID (if special file) */



           off_t     st_size;    /* total size, in bytes */



           blksize_t st_blksize; /* blocksize for filesystem I/O */



           blkcnt_t  st_blocks;  /* number of 512B blocks allocated */



           time_t    st_atime;   /* time of last access */



           time_t    st_mtime;   /* time of last modification */



           time_t    st_ctime;   /* time of last status change */



       };
#include <stdio.h>



#include <unistd.h>



#include <sys/types.h>



#include <sys/socket.h>



#include <string.h>



#include <ctype.h>



#include <arpa/inet.h>



#include <errno.h>



#include <sys/stat.h>



#include <unistd.h>



 



#define SERVER_PORT 80



 



static int debug = 1;



 



int get_line(int sock, char *buf, int size);



void do_http_request(int client_sock);



void do_http_response(int client_sock, const char *path);



int headers(int client_sock, FILE *resource);



void cat(int client_sock, FILE *resource);



 



void not_found(int client_sock);



void inner_error(int client_sock);



 



int main(void)



{



 



    int sock; //代表信箱



    struct sockaddr_in server_addr;



 



    // 1.美女创建信箱



    sock = socket(AF_INET, SOCK_STREAM, 0);



 



    // 2.清空标签，写上地址和端口号



    bzero(&server_addr, sizeof(server_addr));



 



    server_addr.sin_family = AF_INET;                //选择协议族IPV4



    server_addr.sin_addr.s_addr = htonl(INADDR_ANY); //监听本地所有IP地址



    server_addr.sin_port = htons(SERVER_PORT);       //绑定端口号



 



    //实现标签贴到收信得信箱上



    bind(sock, (struct sockaddr *)&server_addr, sizeof(server_addr));



 



    //把信箱挂置到传达室，这样，就可以接收信件了



    listen(sock, 128);



 



    //万事俱备，只等来信



    printf("等待客户端的连接\n");



 



    int done = 1;



 



    while (done)



    {



        struct sockaddr_in client;



        int client_sock, len, i;



        char client_ip[64];



        char buf[256];



 



        socklen_t client_addr_len;



        client_addr_len = sizeof(client);



        client_sock = accept(sock, (struct sockaddr *)&client, &client_addr_len);



 



        //打印客服端IP地址和端口号



        printf("client ip: %s\t port : %d\n",



               inet_ntop(AF_INET, &client.sin_addr.s_addr, client_ip, sizeof(client_ip)),



               ntohs(client.sin_port));



        /*处理http 请求,读取客户端发送的数据*/



        do_http_request(client_sock);



        close(client_sock);



    }



    close(sock);



    return 0;



}



 



void do_http_request(int client_sock)



{



    int len = 0;



    char buf[256];



    char method[64];



    char url[256];



    char path[256];



 



    struct stat st;



 



    /*读取客户端发送的http 请求*/



 



    // 1.读取请求行



    len = get_line(client_sock, buf, sizeof(buf));



 



    if (len > 0)



    { //读到了请求行



        int i = 0, j = 0;



        while (!isspace(buf[j]) && (i < sizeof(method) - 1))



        {



            method[i] = buf[j];



            i++;



            j++;



        }



 



        method[i] = '\0';



        if (debug)



            printf("request method: %s\n", method);



 



        if (strncasecmp(method, "GET", i) == 0)



        { //只处理get请求



            if (debug)



                printf("method = GET\n");



 



            //获取url



            while (isspace(buf[j++]))



                ; //跳过白空格



            i = 0;



 



            while (!isspace(buf[j]) && (i < sizeof(url) - 1))



            {



                url[i] = buf[j];



                i++;



                j++;



            }



 



            url[i] = '\0';



 



            if (debug)



                printf("url: %s\n", url);



 



            //继续读取http 头部



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



 



            //***定位服务器本地的html文件***



 



            //处理url 中的?



            {



                char *pos = strchr(url, '?');



                if (pos)



                {



                    *pos = '\0';



                    printf("real url: %s\n", url);



                }



            }



 



            sprintf(path, "./html_docs/%s", url);



            if (debug)



                printf("path: %s\n", path);



 



            //执行http 响应



            //判断文件是否存在，如果存在就响应200 OK，同时发送相应的html 文件,如果不存在，就响应 404 NOT FOUND.



            if (stat(path, &st) == -1)



            { //文件不存在或是出错



                fprintf(stderr, "stat %s failed. reason: %s\n", path, strerror(errno));



                not_found(client_sock);



            }



            else



            { //文件存在



 



                if (S_ISDIR(st.st_mode))



                {



                    strcat(path, "/index.html");



                }



 



                do_http_response(client_sock, path);



            }



        }



        else



        { //非get请求, 读取http 头部，并响应客户端 501     Method Not Implemented



            fprintf(stderr, "warning! other request [%s]\n", method);



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



 



            // unimplemented(client_sock);   //在响应时再实现



        }



    }



    else



    {   //请求格式有问题，出错处理



        // bad_request(client_sock);   //在响应时再实现



    }



}



 



void do_http_response(int client_sock, const char *path)



{



    int ret = 0;



    FILE *resource = NULL;



 



    resource = fopen(path, "r");



 



    if (resource == NULL)



    {



        not_found(client_sock);



        return;



    }



 



    // 1.发送http 头部



    ret = headers(client_sock, resource);



 



    // 2.发送http body .



    if (!ret)



    {



        cat(client_sock, resource);



    }



 



    fclose(resource);



}



 



/****************************



 *返回关于响应文件信息的http 头部



 *输入：



 *     client_sock - 客服端socket 句柄



 *     resource    - 文件的句柄



 *返回值： 成功返回0 ，失败返回-1



 ******************************/



int headers(int client_sock, FILE *resource)



{



    struct stat st;



    int fileid = 0;



    char tmp[64];



    char buf[1024] = {0};



    strcpy(buf, "HTTP/1.0 200 OK\r\n");



    strcat(buf, "Server: Martin Server\r\n");



    strcat(buf, "Content-Type: text/html\r\n");



    strcat(buf, "Connection: Close\r\n");



 



    fileid = fileno(resource);



 



    if (fstat(fileid, &st) == -1)



    {



        inner_error(client_sock);



        return -1;



    }



 



    snprintf(tmp, 64, "Content-Length: %ld\r\n\r\n", st.st_size);



    strcat(buf, tmp);



 



    if (debug)



        fprintf(stdout, "header: %s\n", buf);



 



    if (send(client_sock, buf, strlen(buf), 0) < 0)



    {



        fprintf(stderr, "send failed. data: %s, reason: %s\n", buf, strerror(errno));



        return -1;



    }



 



    return 0;



}



 



/****************************



 *说明：实现将html文件的内容按行



        读取并送给客户端



 ****************************/



void cat(int client_sock, FILE *resource)



{



    char buf[1024];



 



    fgets(buf, sizeof(buf), resource);



 



    while (!feof(resource))



    {



        int len = write(client_sock, buf, strlen(buf));



 



        if (len < 0)



        { //发送body 的过程中出现问题,怎么办？1.重试？ 2.



            fprintf(stderr, "send body error. reason: %s\n", strerror(errno));



            break;



        }



 



        if (debug)



            fprintf(stdout, "%s", buf);



        fgets(buf, sizeof(buf), resource);



    }



}



 



void do_http_response1(int client_sock)



{



    const char *main_header = "HTTP/1.0 200 OK\r\nServer: Martin Server\r\nContent-Type: text/html\r\nConnection: Close\r\n";



 



    const char *welcome_content = "\



<html lang=\"zh-CN\">\n\



<head>\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\n\



<title>This is a test</title>\n\



</head>\n\



<body>\n\



<div align=center height=\"500px\" >\n\



<br/><br/><br/>\n\



<h2>大家好</h2><br/><br/>\n\



<form action=\"commit\" method=\"post\">\n\



尊姓大名: <input type=\"text\" name=\"name\" />\n\



<br/>芳龄几何: <input type=\"password\" name=\"age\" />\n\



<br/><br/><br/><input type=\"submit\" value=\"提交\" />\n\



<input type=\"reset\" value=\"重置\" />\n\



</form>\n\



</div>\n\



</body>\n\



</html>";



 



    // 1. 发送main_header



    int len = write(client_sock, main_header, strlen(main_header));



 



    if (debug)



        fprintf(stdout, "... do_http_response...\n");



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, main_header);



 



    // 2. 生成Content-Length



    char send_buf[64];



    int wc_len = strlen(welcome_content);



    len = snprintf(send_buf, 64, "Content-Length: %d\r\n\r\n", wc_len);



    len = write(client_sock, send_buf, len);



 



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, send_buf);



 



    len = write(client_sock, welcome_content, wc_len);



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, welcome_content);



}



 



//返回值： -1 表示读取出错， 等于0表示读到一个空行， 大于0 表示成功读取一行



int get_line(int sock, char *buf, int size)



{



    int count = 0;



    char ch = '\0';



    int len = 0;



 



    while ((count < size - 1) && ch != '\n')



    {



        len = read(sock, &ch, 1);



 



        if (len == 1)



        {



            if (ch == '\r')



            {



                continue;



            }



            else if (ch == '\n')



            {



                // buf[count] = '\0';



                break;



            }



 



            //这里处理一般的字符



            buf[count] = ch;



            count++;



        }



        else if (len == -1)



        { //读取出错



            perror("read failed");



            count = -1;



            break;



        }



        else



        { // read 返回0,客户端关闭sock 连接.



            fprintf(stderr, "client close.\n");



            count = -1;



            break;



        }



    }



 



    if (count >= 0)



        buf[count] = '\0';



 



    return count;



}



 



void not_found(int client_sock)



{



    const char *reply = "HTTP/1.0 404 NOT FOUND\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML lang=\"zh-CN\">\r\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\r\n\



<HEAD>\r\n\



<TITLE>NOT FOUND</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>文件不存在！\r\n\



    <P>The server could not fulfill your request because the resource specified is unavailable or nonexistent.\r\n\



</BODY>\r\n\



</HTML>";



 



    int len = write(client_sock, reply, strlen(reply));



    if (debug)



        fprintf(stdout, reply);



 



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}



 



void inner_error(int client_sock)



{



    const char *reply = "HTTP/1.0 500 Internal Sever Error\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML lang=\"zh-CN\">\r\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\r\n\



<HEAD>\r\n\



<TITLE>Inner Error</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>服务器内部出错.\r\n\



</BODY>\r\n\



</HTML>";



 



    int len = write(client_sock, reply, strlen(reply));



    if (debug)



        fprintf(stdout, reply);



 



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}
```

## 4.5 并发处理

### 4.5.1 并发概述

通俗的并发通常是指同时能并行的处理多个任务。

![在这里插入图片描述](https://img-blog.csdnimg.cn/8de35c3069ad4f1cb429dbce1900c0c1.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBA5oiR5Lmf6KaB5b2T5piP5ZCb,size_18,color_FFFFFF,t_70,g_se,x_16)



### 4.5.2 并发

同时拥有两个或者多个线程，如果程序在单核处理器上运行，多个线程将交替的换入或者换出内存，这些线程是同时“存在”的。
每个线程都处于执行过程中的某个状态，如果运行在多核处理器上，此时，程序中的每个线程都将分配到一个处理器核上，因此可以同时运行。

### 4.5.3 高并发

高并发是互联网分布式系统架构设计中必须考虑的因素之一，它通常是指，通过设计保证系统能够 同时并行处理 很多请求

### 4.5.4 pthread_create函数

创建一个新线程，并行的执行任务。
\#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start_routine) (void *), void *arg);
返回值：成功：0； 失败：错误号。
参数：
pthread_t：当前Linux中可理解为：typedef unsigned long int pthread_t;
参数1：传出参数，保存系统为我们分配好的线程ID
参数2：通常传NULL，表示使用线程默认属性。若想使用具体属性也可以修改该参数。
参数3：函数指针，指向线程主函数(线程体)，该函数运行结束，则线程结束。
参数4：线程主函数执行期间所使用的参数。
在一个线程中调用pthread_create()创建新的线程后，当前线程从pthread_create()返回继续往下执行，而新的线程所执行的代码由我们传给pthread_create的函数指针start_routine决定。start_routine函数接收一个参数，是通过pthread_create的arg参数传递给它的，该参数的类型为void *，这个指针按什么类型解释由调用者自己定义。start_routine的返回值类型也是void *，这个指针的含义同样由调用者自己定义。start_routine返回时，这个线程就退出了，其它线程可以调用pthread_join得到start_routine的返回值，以后再详细介绍pthread_join。
pthread_create成功返回后，新创建的线程的id被填写到thread参数所指向的内存单元。
attr参数表示线程属性，本节不深入讨论线程属性，所有代码例子都传NULL给attr参数，表示线程属性取缺省值。

### 4.5.5 并发回声服务器改造

```cpp
#include <stdio.h>



#include <unistd.h>



#include <sys/types.h>



#include <sys/socket.h>



#include <string.h>



#include <ctype.h>



#include <arpa/inet.h>



#include <pthread.h>



#include <stdlib.h>



#include <errno.h>



 



#define SERVER_PORT 666



 



void *thread_handleRequest(void *arg);



 



int main(void)



{



 



    int sock; //代表信箱



    struct sockaddr_in server_addr;



 



    // 1.美女创建信箱



    sock = socket(AF_INET, SOCK_STREAM, 0);



 



    // 2.清空标签，写上地址和端口号



    bzero(&server_addr, sizeof(server_addr));



 



    server_addr.sin_family = AF_INET;                //选择协议族IPV4



    server_addr.sin_addr.s_addr = htonl(INADDR_ANY); //监听本地所有IP地址



    server_addr.sin_port = htons(SERVER_PORT);       //绑定端口号



 



    //实现标签贴到收信得信箱上



    bind(sock, (struct sockaddr *)&server_addr, sizeof(server_addr));



 



    //把信箱挂置到传达室，这样，就可以接收信件了



    listen(sock, 128);



 



    //万事俱备，只等来信



    printf("等待客户端的连接\n");



 



    int done = 1;



 



    while (done)



    {



        struct sockaddr_in client;



        int client_sock, len;



        char client_ip[64];



        // char buf[256];



        // int  i=0;



 



        socklen_t client_addr_len;



        client_addr_len = sizeof(client);



        client_sock = accept(sock, (struct sockaddr *)&client, &client_addr_len);



 



        //打印客服端IP地址和端口号



        printf("client ip: %s\t port : %d\n",



               inet_ntop(AF_INET, &client.sin_addr.s_addr, client_ip, sizeof(client_ip)),



               ntohs(client.sin_port));



 



        //启动线程，完成和和客户端的交互



        {



            pthread_t tid;



            int *ptr_int = NULL;



            int err = 0;



 



            ptr_int = (int *)malloc(sizeof(int));



            *ptr_int = client_sock;



 



            if (err = pthread_create(&tid, NULL, thread_handleRequest, (void *)ptr_int))



            {



                fprintf(stderr, "Can't create thread, reason: %s\n", strerror(errno));



                if (ptr_int)



                    free(ptr_int);



            }



        }



 



        /*读取客户端发送的数据*/



        /*len = read(client_sock, buf, sizeof(buf)-1);



        buf[len] = '\0';



        printf("receive[%d]: %s\n", len, buf);







        //转换成大写



        for(i=0; i<len; i++){



            //if(buf[i]>='a' && buf[i]<='z'){



            //    buf[i] = buf[i] - 32;



            //}



            buf[i] = toupper(buf[i]);



        }











        len = write(client_sock, buf, len);







        printf("finished. len: %d\n", len);



        close(client_sock);



        */



    }



    close(sock);



    return 0;



}



 



void *thread_handleRequest(void *arg)



{



    int client_sock = *(int *)arg;



    char buf[256];



    int i = 0;



 



    int len = read(client_sock, buf, sizeof(buf) - 1);



    buf[len] = '\0';



    printf("receive[%d]: %s\n", len, buf);



 



    //转换成大写



    for (i = 0; i < len; i++)



    {



        // if(buf[i]>='a' && buf[i]<='z'){



        //     buf[i] = buf[i] - 32;



        // }



        buf[i] = toupper(buf[i]);



    }



 



    len = write(client_sock, buf, len);



 



    printf("finished. len: %d\n", len);



    close(client_sock);



 



    if (arg)



        free(arg);



}
```

### 4.5.6 并发Mini http 服务器改造

```cpp
#include <stdio.h>



#include <unistd.h>



#include <sys/types.h>



#include <sys/socket.h>



#include <string.h>



#include <ctype.h>



#include <arpa/inet.h>



#include <errno.h>



#include <sys/stat.h>



#include <unistd.h>



#include <pthread.h>



 



#define SERVER_PORT 80



 



static int debug = 1;



 



int get_line(int sock, char *buf, int size);



void *do_http_request(void *client_sock);



void do_http_response(int client_sock, const char *path);



int headers(int client_sock, FILE *resource);



void cat(int client_sock, FILE *resource);



 



void not_found(int client_sock);     // 404



void unimplemented(int client_sock); // 500



void bad_request(int client_sock);   // 400



void inner_error(int client_sock);



 



int main(void)



{



 



    int sock; //代表信箱



    struct sockaddr_in server_addr;



 



    // 1.美女创建信箱



    sock = socket(AF_INET, SOCK_STREAM, 0);



 



    // 2.清空标签，写上地址和端口号



    bzero(&server_addr, sizeof(server_addr));



 



    server_addr.sin_family = AF_INET;                //选择协议族IPV4



    server_addr.sin_addr.s_addr = htonl(INADDR_ANY); //监听本地所有IP地址



    server_addr.sin_port = htons(SERVER_PORT);       //绑定端口号



 



    //实现标签贴到收信得信箱上



    bind(sock, (struct sockaddr *)&server_addr, sizeof(server_addr));



 



    //把信箱挂置到传达室，这样，就可以接收信件了



    listen(sock, 128);



 



    //万事俱备，只等来信



    printf("等待客户端的连接\n");



 



    int done = 1;



 



    while (done)



    {



        struct sockaddr_in client;



        int client_sock, len, i;



        char client_ip[64];



        char buf[256];



        pthread_t id;



        int *pclient_sock = NULL;



 



        socklen_t client_addr_len;



        client_addr_len = sizeof(client);



        client_sock = accept(sock, (struct sockaddr *)&client, &client_addr_len);



 



        //打印客服端IP地址和端口号



        printf("client ip: %s\t port : %d\n",



               inet_ntop(AF_INET, &client.sin_addr.s_addr, client_ip, sizeof(client_ip)),



               ntohs(client.sin_port));



 



        /*处理http 请求,读取客户端发送的数据*/



        // do_http_request(client_sock);



 



        //启动线程处理http 请求



        pclient_sock = (int *)malloc(sizeof(int));



        *pclient_sock = client_sock;



 



        pthread_create(&id, NULL, do_http_request, (void *)pclient_sock);



 



        // close(client_sock);



    }



    close(sock);



    return 0;



}



 



void *do_http_request(void *pclient_sock)



{



    int len = 0;



    char buf[256];



    char method[64];



    char url[256];



    char path[256];



    int client_sock = *(int *)pclient_sock;



 



    struct stat st;



 



    /*读取客户端发送的http 请求*/



 



    // 1.读取请求行



    len = get_line(client_sock, buf, sizeof(buf));



 



    if (len > 0)



    { //读到了请求行



        int i = 0, j = 0;



        while (!isspace(buf[j]) && (i < sizeof(method) - 1))



        {



            method[i] = buf[j];



            i++;



            j++;



        }



 



        method[i] = '\0';



        if (debug)



            printf("request method: %s\n", method);



 



        if (strncasecmp(method, "GET", i) == 0)



        { //只处理get请求



            if (debug)



                printf("method = GET\n");



 



            //获取url



            while (isspace(buf[j++]))



                ; //跳过白空格



            i = 0;



 



            while (!isspace(buf[j]) && (i < sizeof(url) - 1))



            {



                url[i] = buf[j];



                i++;



                j++;



            }



 



            url[i] = '\0';



 



            if (debug)



                printf("url: %s\n", url);



 



            //继续读取http 头部



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



 



            //***定位服务器本地的html文件***



 



            //处理url 中的?



            {



                char *pos = strchr(url, '?');



                if (pos)



                {



                    *pos = '\0';



                    printf("real url: %s\n", url);



                }



            }



 



            sprintf(path, "./html_docs/%s", url);



            if (debug)



                printf("path: %s\n", path);



 



            //执行http 响应



            //判断文件是否存在，如果存在就响应200 OK，同时发送相应的html 文件,如果不存在，就响应 404 NOT FOUND.



            if (stat(path, &st) == -1)



            { //文件不存在或是出错



                fprintf(stderr, "stat %s failed. reason: %s\n", path, strerror(errno));



                not_found(client_sock);



            }



            else



            { //文件存在



 



                if (S_ISDIR(st.st_mode))



                {



                    strcat(path, "/index.html");



                }



 



                do_http_response(client_sock, path);



            }



        }



        else



        { //非get请求, 读取http 头部，并响应客户端 501     Method Not Implemented



            fprintf(stderr, "warning! other request [%s]\n", method);



            do



            {



                len = get_line(client_sock, buf, sizeof(buf));



                if (debug)



                    printf("read: %s\n", buf);



 



            } while (len > 0);



 



            unimplemented(client_sock); //请求未实现



        }



    }



    else



    {                             //请求格式有问题，出错处理



        bad_request(client_sock); //在响应时再实现



    }



 



    close(client_sock);



    if (pclient_sock)



        free(pclient_sock); //释放动态分配的内存



 



    return NULL;



}



 



void do_http_response(int client_sock, const char *path)



{



    int ret = 0;



    FILE *resource = NULL;



 



    resource = fopen(path, "r");



 



    if (resource == NULL)



    {



        not_found(client_sock);



        return;



    }



 



    // 1.发送http 头部



    ret = headers(client_sock, resource);



 



    // 2.发送http body .



    if (!ret)



    {



        cat(client_sock, resource);



    }



 



    fclose(resource);



}



 



/****************************



 *返回关于响应文件信息的http 头部



 *输入：



 *     client_sock - 客服端socket 句柄



 *     resource    - 文件的句柄



 *返回值： 成功返回0 ，失败返回-1



 ******************************/



int headers(int client_sock, FILE *resource)



{



    struct stat st;



    int fileid = 0;



    char tmp[64];



    char buf[1024] = {0};



    strcpy(buf, "HTTP/1.0 200 OK\r\n");



    strcat(buf, "Server: Martin Server\r\n");



    strcat(buf, "Content-Type: text/html\r\n");



    strcat(buf, "Connection: Close\r\n");



 



    fileid = fileno(resource);



 



    if (fstat(fileid, &st) == -1)



    {



        inner_error(client_sock);



        return -1;



    }



 



    snprintf(tmp, 64, "Content-Length: %ld\r\n\r\n", st.st_size);



    strcat(buf, tmp);



 



    if (debug)



        fprintf(stdout, "header: %s\n", buf);



 



    if (send(client_sock, buf, strlen(buf), 0) < 0)



    {



        fprintf(stderr, "send failed. data: %s, reason: %s\n", buf, strerror(errno));



        return -1;



    }



 



    return 0;



}



 



/****************************



 *说明：实现将html文件的内容按行



        读取并送给客户端



 ****************************/



void cat(int client_sock, FILE *resource)



{



    char buf[1024];



 



    fgets(buf, sizeof(buf), resource);



 



    while (!feof(resource))



    {



        int len = write(client_sock, buf, strlen(buf));



 



        if (len < 0)



        { //发送body 的过程中出现问题,怎么办？1.重试？ 2.



            fprintf(stderr, "send body error. reason: %s\n", strerror(errno));



            break;



        }



 



        if (debug)



            fprintf(stdout, "%s", buf);



        fgets(buf, sizeof(buf), resource);



    }



}



 



void do_http_response1(int client_sock)



{



    const char *main_header = "HTTP/1.0 200 OK\r\nServer: Martin Server\r\nContent-Type: text/html\r\nConnection: Close\r\n";



 



    const char *welcome_content = "\



<html lang=\"zh-CN\">\n\



<head>\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\n\



<title>This is a test</title>\n\



</head>\n\



<body>\n\



<div align=center height=\"500px\" >\n\



<br/><br/><br/>\n\



<h2>大家好</h2><br/><br/>\n\



<form action=\"commit\" method=\"post\">\n\



尊姓大名: <input type=\"text\" name=\"name\" />\n\



<br/>芳龄几何: <input type=\"password\" name=\"age\" />\n\



<br/><br/><br/><input type=\"submit\" value=\"提交\" />\n\



<input type=\"reset\" value=\"重置\" />\n\



</form>\n\



</div>\n\



</body>\n\



</html>";



 



    // 1. 发送main_header



    int len = write(client_sock, main_header, strlen(main_header));



 



    if (debug)



        fprintf(stdout, "... do_http_response...\n");



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, main_header);



 



    // 2. 生成Content-Length



    char send_buf[64];



    int wc_len = strlen(welcome_content);



    len = snprintf(send_buf, 64, "Content-Length: %d\r\n\r\n", wc_len);



    len = write(client_sock, send_buf, len);



 



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, send_buf);



 



    len = write(client_sock, welcome_content, wc_len);



    if (debug)



        fprintf(stdout, "write[%d]: %s", len, welcome_content);



}



 



//返回值： -1 表示读取出错， 等于0表示读到一个空行， 大于0 表示成功读取一行



int get_line(int sock, char *buf, int size)



{



    int count = 0;



    char ch = '\0';



    int len = 0;



 



    while ((count < size - 1) && ch != '\n')



    {



        len = read(sock, &ch, 1);



 



        if (len == 1)



        {



            if (ch == '\r')



            {



                continue;



            }



            else if (ch == '\n')



            {



                // buf[count] = '\0';



                break;



            }



 



            //这里处理一般的字符



            buf[count] = ch;



            count++;



        }



        else if (len == -1)



        { //读取出错



            perror("read failed");



            count = -1;



            break;



        }



        else



        { // read 返回0,客户端关闭sock 连接.



            fprintf(stderr, "client close.\n");



            count = -1;



            break;



        }



    }



 



    if (count >= 0)



        buf[count] = '\0';



 



    return count;



}



 



void not_found(int client_sock)



{



    const char *reply = "HTTP/1.0 404 NOT FOUND\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML lang=\"zh-CN\">\r\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\r\n\



<HEAD>\r\n\



<TITLE>NOT FOUND</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>文件不存在！\r\n\



    <P>The server could not fulfill your request because the resource specified is unavailable or nonexistent.\r\n\



</BODY>\r\n\



</HTML>";



 



    int len = write(client_sock, reply, strlen(reply));



    if (debug)



        fprintf(stdout, reply);



 



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}



 



void unimplemented(int client_sock)



{



    const char *reply = "HTTP/1.0 501 Method Not Implemented\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML>\r\n\



<HEAD>\r\n\



<TITLE>Method Not Implemented</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>HTTP request method not supported.\r\n\



</BODY>\r\n\



</HTML>";



 



    int len = write(client_sock, reply, strlen(reply));



    if (debug)



        fprintf(stdout, reply);



 



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}



 



void bad_request(client_sock)



{



    const char *reply = "HTTP/1.0 400 BAD REQUEST\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML>\r\n\



<HEAD>\r\n\



<TITLE>BAD REQUEST</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>Your browser sent a bad request！\r\n\



</BODY>\r\n\



</HTML>";



 



    int len = write(client_sock, reply, strlen(reply));



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}



 



void inner_error(int client_sock)



{



    const char *reply = "HTTP/1.0 500 Internal Sever Error\r\n\



Content-Type: text/html\r\n\



\r\n\



<HTML lang=\"zh-CN\">\r\n\



<meta content=\"text/html; charset=utf-8\" http-equiv=\"Content-Type\">\r\n\



<HEAD>\r\n\



<TITLE>Inner Error</TITLE>\r\n\



</HEAD>\r\n\



<BODY>\r\n\



    <P>服务器内部出错.\r\n\



</BODY>\r\n\



</HTML>";



 


    int len = write(client_sock, reply, strlen(reply));



    if (debug)



        fprintf(stdout, reply);



 



    if (len <= 0)



    {



        fprintf(stderr, "send reply failed. reason: %s\n", strerror(errno));



    }



}
```

原文作者：[我也要当昏君](https://blog.csdn.net/qq_46118239)

原文链接：https://bbs.csdn.net/topics/605742285