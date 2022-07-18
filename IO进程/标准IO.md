# 一 IO
## 1.1 IO 分类
### 1.1.1 标准IO：
**库函数：缓冲区 + 系统调用**
缓冲区分为行缓冲和全缓冲，对于标准IO，皆从缓冲区进行读写操作，所有要考虑缓冲区的刷新机制。
行缓冲：与终端相关的读写操作，**stdin,stdout**   【注】：stderr属于无缓冲机制
全缓冲：涉及与文件打开的话，就属于全缓冲
无缓冲：**stderr**
### 1.1.2 文件IO：
**系统调用**：从用户空间进入到内核的过程就叫系统调用，系统调用的可移植性差，无缓冲区，效率低
## 1.2 标准IO
### 1.2.1 标准IO的接口函数
**1.文件打开函数：fopen**
		使用fopen打开文件时,如果打开成功，会产生一个FILE 结构体指针，该结构体指针记录了所打开文件的全部消息，后续的文件的读写操作，皆基于此结构体指针进行.
```c
FILE *fopen(const char *pathname, const char *mode);
	@pathname:所打开的文件的路径以及文件的名字
	@mode:以什么权限去打开文件
		r:只读方式打开文件,文件打开时，光标自动定位在文件的开头
		r+:读写的方式打开文件，文件打开时，光标自动定位在文件的开头，进行写写操作时，想写入
		的内容，会从头进行替换原因的内容
		w:只写的方式打开文件，文件存在则清空文件的内容，文件不存在的话就创建，
		光标自动定位在文件的开头
		w+:读写的方式打开文件，文件存在则清空文件的内容，文件不存在就创建文件，
		光标自动定位在文件的开头
		a:以追加的方式打开文件，文件不存在就创建，光标自动定位在文件的末尾
		a+:以读和追加的方式打开文件，文件不存在就创建，若进行读操作，光标定位在文件的开头，
		若进行追加操作，光标定位在文件的末尾。
	函数的返回值：
		成功返回FILE结构体指针，失败返回NULL设置错误码
fopen使用函数示例：
	FILE *fp;
	if(NULL == (fp = fopen(argv[1],"r"))){  //只读方式打开文件，文件不存在或打开失败返回错误信息
		perror("fopen file error");
		return -1;
	}	
```
**2 文件关闭函数fclose**
```c
int fclose(FILE *stream);
	@stream:需要关闭的FILE结构体指针
	函数的返回值：
		成功返回0， 失败返回-1设置错误码
函数使用示例：
	FILE *fp;
	//只读方式打开一个文件
	if(NULL == (fp = fopen(argv[1],"w"))){
		perror("fopen file error");
		return -1;
	}
	//关闭打开的文件
	if(-1 == fclose(fp)){
		perror("fclose error");
		return -1;
	}
```
**3 fgetc/fputc**
```c
int fgetc(FILE *stream);
	@stream:FILE结构体指针：从那个文件中获取一个字符
	返回值：
		成功返回获取的字符的ascii值，失败返回-1 

int fputc(int c, FILE *stream);
	@c:想文件中写如的字符，可以是ascii值，也可以是字符：如‘c’
	@stream:FILE结构体指针：向哪个文件中输入一个字符
返回值：成功返回实际写入的字符的ASCII值，失败返回-1

使用示例：利用fgetc/fputc实现文件的拷贝
#include  <head.h>
int  main(int  argc,const  char * argv[])
{

	FILE *fstr;
	FILE *fdst;
	
	if(argc != 3){
		fprintf(stderr,"Usage:<%s><strfile><dstfile>",argv[0]);
		return -1;
	}
	if(NULL == (fstr = fopen(argv[1],"r"))){
		perror("fopen strfile error");
		return -1;
	}
	if(NULL == (fdst = fopen(argv[2],"w"))){
		perror("fopen dstfile error");
		return -1;
	}
	int  ch = 0;
	while(-1 != (ch = fgetc(fstr))){
		fputc(ch,fdst);
	} 
	return  0;
}
```
## 1.3 文件IO
### 1.3.1 文件IO的接口函数
