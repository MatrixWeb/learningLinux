:blush:<br>
# 4-1
```c
#include<stdio.h>
#include<unistd.h>
#include<fcntl.h>
#define MAX_READ 20
char buffer[MAX_READ+1];
void fatal(const char *errorinfo){
	fprintf(stderr,"%s",errorinfo);
}
void process(const char *filename,int flag)
{
	int fd,numread;
	if(flag==0)
	{
		fd=open(filename,O_WRONLY|O_CREAT|O_TRUNC);
	}else{
		fd=open(filename,O_WRONLY|O_CREAT|O_APPEND);
	}
	if(fd==-1)
	{
		fatal("open file error\n");
	}
	while((numread=read(STDIN_FILENO,buffer,MAX_READ))>0){
		buffer[numread]='\0';
		if(write(STDOUT_FILENO,buffer,numread+1)!=(numread+1)){
			fatal("could not write buffer into stdout\n");
		};
		if(write(fd,buffer,numread)!=numread){
			fatal("could not write buffer into file\n");
		};
	}
	close(fd);
}
void usage()
{
	printf("Usage:./tee -a file or ./tee filei/n");
}
int main(int argc,char *argv[])
{
	int ch;
	if((ch=getopt(argc,argv,"a:"))!=-1)
	{
		switch(ch)
		{
			case 'a':
				process(optarg,1);
				break;
			default:
				usage();
		}
	}
	process(argv[1],0);
	return 0;
}

```
这题没啥难度，就是用几个系统调用，唯一要注意的是，创建文件的时候如果不给文件指定权限，貌似还会系统会随机给。

# 4-2

这题没难度，就直接用示例的代码就可以了:smile:

```c
#include<stdio.h>
#include<fcntl.h>
#include<sys/stat.h>
#include<unistd.h>
#define BUF_SIZE 1024
void fatal(const char *info){
	fprintf(stderr,"%s",info);
}
int main(int argc,char *argv[]){
	int inputfd,outputfd,numread;
	char buf[BUF_SIZE];
	mode_t filePerms;
	filePerms = S_IRUSR|S_IWUSR|S_IRGRP|S_IWGRP|S_IROTH|S_IWOTH;
	inputfd=open(argv[1],O_RDONLY);
	if(inputfd==-1)
	  fatal("open file error");
	outputfd=open(argv[2],O_CREAT|O_WRONLY|O_TRUNC,filePerms);
	while((numread=read(inputfd,buf,BUF_SIZE))>0)
	  if(write(outputfd,buf,numread)!=numread)
		fatal("could not write whole buffer");
	close(inputfd);
	close(outputfd);
}

```


