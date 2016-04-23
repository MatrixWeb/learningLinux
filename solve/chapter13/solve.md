# 13-1
a)

BUF_SIZE|real|user|sys
--------|----|----|---
1|0.061|0.017|0.043
4|0.027|0.008|0.020
8|0.018|0.005|0.013
64|0.006|0.002|0.004
128|0.005|0.002|0.003
256|0.004|0.001|0.003
512|0.004|0.001|0.003

b)

BUF_SIZE|real|user|sys
-----|----|----|----
1|11.400|0.034|1.199
4|2.882|0.010|0.316
8|1.610|0.006|0.162
64|0.200|0.002|0.027
256|0.067|0.002|0.012
512|0.043|0.002|0.009
1024|0.028|0.002|0.006
2048|0.017|0.001|0.004

以上单位都是s。可以看出read,write系统调用在操作瓷盘文件时不会直接发起磁盘访问，而是仅仅在用户缓冲区和内核缓冲区高速缓存之间复制数据。

# 13-3
fflush(fp):强制将stdio输出流中的数据刷新到内核缓冲区。
fsync(fileno(fp)):将使缓冲数据和与打开文件描述符fd相关的所有元数据都刷新到磁盘上。

# 13-4
因为i/o系统调用会直接讲数据传递到内核缓冲区高速缓存，而stdio库函数会等用户空间的流缓冲区填满，再调用write讲其传递到内核缓冲区高速缓存。所以将标准输出重定向后。代码的输出会不一样。

# 13-5
```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<fcntl.h>
#ifndef BUF_SIZE
#define BUF_SIZE 1024
#endif
int main(int argc,char *argv[])
{
	int count=10;
	char buf[BUF_SIZE];
	if(argc>3)
	{
		int ch;
		if((ch=getopt(argc,argv,"n:"))!=-1)
		{
			switch(ch)
			{
				case 'n':
					count=atoi(optarg);
				break;
			}
		}
	}
	char *filename=argv[argc-1];
	int fd;
	int sum=0;
	fd=open(filename,O_RDONLY);
	if(fd==-1)
	  printf("open file error!");
	lseek(fd,0,SEEK_END);
	lseek(fd,-count-1,SEEK_CUR);
	int len,k;
	int c=0;
	while((len=read(fd,buf,count+1))!=0)
	{
		int j;
		for(j=len-1;j>=0;j--)
		{
			if(buf[j]=='\n')
			{
				c++;
				if(c==count+1)
				{
					k=len-j-1;break;
				}
			}
		}
		if(c==count+1)
		  break;
		sum=sum+len;
		lseek(fd,-count-1-len,SEEK_CUR);
	}
	lseek(fd,-sum-k,SEEK_END);
	while((len=read(fd,buf,BUF_SIZE))!=0)
	{
		for(int k=0;k<len;k++)
		  printf("%c",buf[k]);
	}
	close(fd);
	return 0;
}


```
这题做的我简直想砸电脑。各种搞不清位置。其实就是按step去找最后第几行的位置。真是日狗了！！草
