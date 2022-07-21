```c
#include <stdio.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h> /* superset of previous */
#include <strings.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <string.h>

#define R 1
#define L 2
#define Q 3
#define H 4

typedef struct MSG{
    int type;
    char name[30];
    char text[256];
}msg_t;

//函数声明
int do_register(int sockfd, msg_t *msg);
int do_login(int sockfd, msg_t *msg);
int do_qurey(int sockfd, msg_t *msg);
int do_history(int sockfd, msg_t *msg);

int main(int argc,const char * argv[])
{
    int sockfd;
    int input = 0; //用户输入的选项值
    msg_t msg;

    //参数校验
    if(argc != 3){
        fprintf(stderr,"Usage:<%s><IP><port>",argv[0]);
        exit(-1);
    }

    //创建流式套接字
    if(-1 == (sockfd = socket(AF_INET,SOCK_STREAM,0))){
        perror("fail to socket");
        exit(-1);
    }

    //填充服务器信息结构体
    struct sockaddr_in serveraddr;
    bzero(&serveraddr,sizeof(serveraddr));
    serveraddr.sin_family = AF_INET;
    serveraddr.sin_addr.s_addr = inet_addr(argv[1]);
    serveraddr.sin_port = htons(atoi(argv[2]));

    socklen_t serveraddr_len = sizeof(serveraddr);

    //与服务器建立连接
    if(-1 == connect(sockfd,(struct sockaddr*)&serveraddr,serveraddr_len)){
        perror("fail to connect");
        exit(-1);
    }

    //与服务器交互
    while(1){
        printf("******************************************\n");
        printf("** 1.register       2.login       3.quit**\n");
        printf("******************************************\n");
        printf("please choose:");
        scanf("%d",&input);
        //getchar(); //吃掉缓冲区的'\n'
        switch(input){
            case 1:                                     
                do_register(sockfd,&msg);
                break;
            case 2:
                if(1 == do_login(sockfd,&msg))
                    goto NEXT;
                break;
            case 3:
                close(sockfd);
                exit(0);
                break;
        }
    }
NEXT:
    while(1){
        printf("******************************************\n");
        printf("** 1.qurey        2.history       3.quit**\n");
        printf("******************************************\n");
        printf("please choose:");
        scanf("%d",&input);
        getchar();
        switch(input){
            case 1:
                do_qurey(sockfd,&msg);
                break;
            case 2:
                do_history(sockfd,&msg);
                break;
            case 3:
                close(sockfd);
                exit(0);
                break;
        }
    }
    return 0;
}

//注册
int do_register(int sockfd, msg_t *msg){
    msg->type = R;

    printf("input register name:");
    scanf("%s",msg->name);
    getchar();

    printf("input regidter password:");
    scanf("%s",msg->text);
    getchar();
    
    //向服务器发送注册的用户信息
    if(-1 == send(sockfd,msg,sizeof(msg_t),0)){
        perror("fail to send ");
        return -1;
    }

    //接收服务器的注册反馈信息
    memset(msg->text,0,sizeof(msg->text));
    if(-1 == recv(sockfd,msg,sizeof(msg_t),0)){
        perror("fail to recv");
        return -1;
    }

    if(0 == strcmp(msg->text,"register success")){
        printf("[%s]\n",msg->text);
        return 0;
    }
    printf("[%s]\n",msg->text);

}

//登录
int do_login(int sockfd, msg_t *msg){
    msg->type = L;
    
    //printf("input login name:");
    scanf("%s",msg->name);
    getchar();

    printf("input login pass:");
    scanf("%s",msg->text);
    getchar();

    //向服务器发送登录用户名和密码
    if(-1 == send(sockfd,msg,sizeof(msg_t),0)){
        perror("fail to send ");
        return -1;
    }

    //接收服务器的登录反馈信息
    memset(msg->text,0,sizeof(msg->text));
    if(-1 == recv(sockfd,msg,sizeof(msg_t),0)){
        perror("fail to recv ");
        return -1;
    }
    if(0 == strcmp(msg->text,"login success")){
        printf("[%s]\n",msg->text);
        return 1;//成功
    }
    printf("[%s]\n",msg->text);
}

//查询
int do_qurey(int sockfd, msg_t *msg){
    msg->type = Q;
    while(1){
        memset(msg->text,0,sizeof(msg->text));
        printf("input word to search(输入#退出查询):");
        scanf("%s",msg->text);

        if(0 == strcmp(msg->text,"#")){
            break;
        }

        if(-1 == send(sockfd,msg,sizeof(msg_t),0)){
            perror("fail to send");
            return -1;
        }

        //接收查询到的结果
        memset(msg->text,0,sizeof(msg->text));
        if(-1 == recv(sockfd,msg,sizeof(msg_t),0)){
            perror("fail to recv");
            return -1;
        }
        printf("%s",msg->text);
    }
    return 0;
}

//历史
int do_history(int sockfd, msg_t *msg){
    msg->type = H;
    printf("fsgd\n");
    if(-1 == send(sockfd,msg,sizeof(msg_t),0)){
        perror("fail to send");
        return -1;
    }

    while(1){
        memset(msg->text,0,sizeof(msg->text));

        if(-1 == recv(sockfd,msg,sizeof(msg_t),0)){
            perror("fail to recv");
            return -1;
        }

        if(0 == strncmp(msg->text,"over",5)){
            break;
        }

        printf("%s\n",msg->text);
    }

    return 0;
}

```
