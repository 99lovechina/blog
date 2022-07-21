```c
#include  <stdio.h>
#include  <sys/types.h>  /* See NOTES */
#include  <sys/socket.h>
#include  <netinet/in.h>
#include  <netinet/ip.h>  /* superset of previous */
#include  <strings.h>
#include  <signal.h>
#include  <sqlite3.h>
#include  <stdlib.h>
#include  <sys/wait.h>
#include  <unistd.h>
#include  <arpa/inet.h>
#include  <string.h>
#include  <time.h>
  
#define  DATEBASE  "my.db"
#define  R  1
#define  L  2
#define  Q  3
#define  H  4

typedef  struct  MSG{
	int  type;
	char  name[30];
	char  text[256];
}msg_t;

void  handle(int  signo){
	if(SIGCHLD == signo){
	wait(NULL);
	}
}

//函数声明
int  create_date_init(sqlite3 ** db);
int  do_register(int  acfd,msg_t *msg,sqlite3 * db);
int  do_login(int  acfd,msg_t *msg,sqlite3 * db);
int  do_qurey(int  acfd,msg_t *msg,sqlite3 * db);
int  do_history(int  acfd,msg_t *msg,sqlite3 * db);
static  int  do_record(char *word,sqlite3 *db,char *name);

int  main(int  argc,const  char * argv[])
{ 
  int  sockfd;
  int  acfd;
  pid_t  pid;
  int  ret_recv = 0;
  msg_t  msg;
  sqlite3 *db;
  
//参数校验
if(argc != 3){
	fprintf(stderr,"Usage:<%s><IP><port>",argv[0]);
	exit(-1);
}

//创建数据库，并初始化
create_date_init(&db);

//创建流式套接字
if(-1 == (sockfd = socket(AF_INET,SOCK_STREAM,0))){
	perror("fail to socket");
	exit(-1);
}

  

//填充服务器信息结构体
struct  sockaddr_in  serveraddr;
bzero(&serveraddr,sizeof(serveraddr));
serveraddr.sin_family = AF_INET;
serveraddr.sin_addr.s_addr = inet_addr(argv[1]);
serveraddr.sin_port = htons(atoi(argv[2]));
socklen_t  serveraddr_len = sizeof(serveraddr);

//绑定套接字 与 服务器信息
if(-1 == bind(sockfd,(struct  sockaddr*)&serveraddr,serveraddr_len)){
	perror("fail to bind");
	exit(-1);
}

//设置监听
if(-1 == listen(sockfd,5)){
	perror("fail to listen");
	exit(-1);
}

//定义网络结构体保存客户端的信息
struct  sockaddr_in  clientaddr;
bzero(&clientaddr,sizeof(clientaddr));
socklen_t  clientaddr_len = sizeof(clientaddr);

//子进程与客户端进行交互，父进程用户处理不同客户端的连接。
while(1){

if(-1 == (acfd = accept(sockfd,(structsockaddr*)&clientaddr,&clientaddr_len))){
	perror("fail to accept ");
	exit(-1);
}
pid = fork();
if(0 == pid){
	close(sockfd);
	while(1){
		if(-1 == (ret_recv = recv(acfd,&msg,sizeof(msg),0))){
		perror("fail to recv");
		break;
		}else  if(0 == ret_recv){
			printf("客户端断开连接....\n");
			break;
		}

		printf("%d\n",msg.type);
		printf("%s\n",msg.name);
		switch(msg.type){
			case  R:
				do_register(acfd,&msg,db);
				break;
			case  L:
				do_login(acfd,&msg,db);
				break;
			case  Q:
				do_qurey(acfd,&msg,db);
				break;
			case  H:
				do_history(acfd,&msg,db);
				break;
		}
	}
		exit(-1);

}else  if(pid > 0){
	close(acfd);
	signal(SIGCHLD,handle);
}else{
	perror("fail to fork");
	exit(-1);
	}
}
return  0;
}

//创建数据库，并初始化
int  create_date_init(sqlite3 **db){
	char  sqstr[128] = {0};
	 
	//打开数据库
	if(SQLITE_OK != sqlite3_open(DATEBASE,db)){
		printf("errmsg:[%s]\n",sqlite3_errmsg(*db));
		return -1;
	}
	printf("创建数据库成功\n");

	//创建usr表
	sprintf(sqstr,"CREATE TABLE IF NOT EXISTS usr(name CHAR PRIMARY KEY,pass CHAR)");
	if(SQLITE_OK != sqlite3_exec(*db,sqstr,NULL,NULL,NULL)){
		printf("errmsg:[%s]\n",sqlite3_errmsg(*db));
		return -1;
	}
	printf("创建usr表成功\n");

	//创建record表
	sprintf(sqstr,"CREATE TABLE IF NOT EXISTS record(name CHAR,date CHAR,word CHAR)");
	if(SQLITE_OK != sqlite3_exec(*db,sqstr,NULL,NULL,NULL)){
		printf("errmsg:[%s]\n",sqlite3_errmsg(*db));
		return -1;
	}
	printf("创建record表成功\n"); 
	return  0;
}

  

//注册
int  do_register(int  acfd,msg_t *msg,sqlite3 * db){
	char  sqstr[400] = {0};
	sprintf(sqstr,"INSERT INTO usr VALUES('%s','%s')",msg->name,msg->text);
	printf("[%s]\n",sqstr); 
	if(SQLITE_OK != sqlite3_exec(db,sqstr,NULL,NULL,NULL)){
		strcpy(msg->text,"usr alreadly exist");
	}else{
		strcpy(msg->text,"register success");
	}
	if(-1 == send(acfd,msg,sizeof(msg_t),0)){
		perror("fail to send");
		return -1;
	}
	return  0;
}

  

//登录
int  do_login(int  acfd,msg_t *msg,sqlite3 * db){
	char  sqstr[400] = {0};
	char **resultp = NULL;
	int  row = 0;
	int  column = 0;
	sprintf(sqstr,"SELECT * FROM usr WHERE name='%s' and pass='%s'",msg->name,msg->text);
	printf("[%s]\n",sqstr);
	if(SQLITE_OK != sqlite3_get_table(db,sqstr,&resultp,&row,&column,NULL)){
		printf("errmag:[%s]\n",sqlite3_errmsg(db));
		return -1;
	}

	//根据返回的行判断：行 = 1：表示用户存在，且密码用户名正确
	//行 = 0：密码或用户名错误
	if(1 == row){
		strcpy(msg->text,"login success");
	}else  if(0 == row){
		strcpy(msg->text,"usrname or password error");
	}
//将信息反馈给客户端
	if(-1 ==send(acfd,msg,sizeof(msg_t),0)){
		perror("fail to send ");
		return -1;
	}
	return  0;
}

//查询
int  do_qurey(int  acfd,msg_t *msg,sqlite3 * db){
	FILE *fp;
	int  flag = 0;
	char  word[30] = {0};
	printf("%s",msg->text);
	int  word_len = strlen(msg->text);
	char  buff[300] = {0};
	char *p = NULL; 
	strcpy(word,msg->text);//保存一份查询的单词，插入历史表 
	if(NULL == (fp = fopen("./dict.txt","r"))){
		perror("fopen error");
		return -1;
	}
	while(NULL != fgets(buff,sizeof(buff),fp)){
		if(0 == strncmp(buff,msg->text,word_len) && buff[word_len] == ' '){
			p = buff + word_len;
			while(*p == ' '){
				p++;
			}
			flag = 1;
			break;
		}
	}

	if(1 == flag){
		strcpy(msg->text,p);
		//插入记录
		do_record(word,db,msg->name);
	}else  if(0 == flag){
		strcpy(msg->text,"not found");
	}
	if(-1 == send(acfd,msg,sizeof(msg_t),0)){
		perror("fail to send 7777777");
		return -1;
	}
	return  0;
}

  

//历史
int  callback(void*arg, int  column,char **f_value, char **f_name){
	msg_t  msg;
	int  fd = *(int*)arg;  
	sprintf(msg.text,"%s:%s",f_value[1],f_value[2]);
	if(-1 == send(fd,&msg,sizeof(msg),0)){
		perror("send error");
		return -1;
	}
	return  0;
}  
int  do_history(int  acfd,msg_t *msg,sqlite3 * db){
	char  sqstr[128] = {0};  
	sprintf(sqstr,"SELECT * FROM record WHERE name='%s'",msg->name);
	printf("%s\n",sqstr);
	if(SQLITE_OK != sqlite3_exec(db,sqstr,callback,(void*)&acfd,NULL)){
		printf("errmsg:[%s]\n",sqlite3_errmsg(db));
		return -1;
	}
	strcpy(msg->text,"over");
	if(-1 == send(acfd,msg,sizeof(msg_t),0)){
		perror("send error");
		return -1;
	}
	return  0;
}

  

static  int  do_record(char *word,sqlite3 *db,char *name){
	time_t  t;
	struct  tm * tp = NULL;
	char  date[128] = {0};
	char  sqstr[128] = {0}; 
	time(&t);
	tp = localtime(&t);
	sprintf(date,"%04d-%02d-%02d  %02d:%02d:%02d",tp->tm_year+1900,tp->tm_mon+1,
	tp->tm_mday,tp->tm_hour,tp->tm_min,tp->tm_sec);
	printf("date[%s]\n",date);
	sprintf(sqstr,"INSERT INTO record VALUES('%s','%s','%s')",name,date,word);
	printf("[%s]",sqstr); 
	if(SQLITE_OK != sqlite3_exec(db,sqstr,NULL,NULL,NULL)){
		printf("errmsg:[%s]\n",sqlite3_errmsg(db));
		return -1;
	}
	return  0;
}
```
