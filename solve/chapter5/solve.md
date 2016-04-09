# 5-1
:blush: 这题我在ubuntu x86_64下直接用示例程序是可以创建高达10G的大文件。代码如下：

```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
int main(int argc,char *argv[])
{
	int fd;
	off_t off;
	fd=open(argv[1],O_RDWR|O_CREAT,S_IRUSR|S_IWUSR);
	if(fd==-1)
	  fatal("open error");
	off=atoll(argv[2]);
	if(lseek(fd,off,SEEK_SET)==-1)
	  fatal("lseek error");
	if(write(fd,"test",4)==-1)
	  fatal("write error");
	return 0;
}

```
唯一需要注意的是atoll函数，这个函数如果你一般编译的话是会在转化32位以上数字的时候溢出。是不是很诡异？反正我这里是出现了这样的情况。然后我man了一下，里面说编译的时候要加-std=c99的选项才能正常用这个函数，所以我们的编译命令是：

> gcc -std=c99 -D_FILE_OFFSET_BITS=64 5_1.C -O 5_1

# 5-2

:see_no_evil:在末端，因为O_APPEND打开后，是一个原子操作：移动到末端，写数据。这是O_APPEND打开的作用。中间的插入时无效的。

```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
int main(int argc,char *argv[])
{
	int fd;
	off_t off;
	fd=open(argv[1],O_WRONLY|O_APPEND,S_IRUSR|S_IWUSR);
	if(fd==-1)
	  fatal("open error");
	if(lseek(fd,0,SEEK_SET)==-1)
	  fatal("lseek error");
	if(write(fd,"test",4)==-1)
	  fatal("write error");
	return 0;
}

```

# 5-3

:hear_no_evil:其实跟上一题是一样的。这里的大小是f1>=f2。因为O_APPEND是原子操作，所以并行的两个程序最后会把所有字节有序写入。而没有加这个标志的程序会覆盖写。

![5-3](https://github.com/MatrixWeb/learningLinux/blob/gh-pages/solve/chapter5/5-3.png)

示例代码：

```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
int main(int argc,char *argv[])
{
	int fd,i;
	int off=atoi(argv[2]);
	if(argv[argc-1][0]=='x')
	{
		fd=open(argv[1],O_WRONLY|O_CREAT,S_IRUSR|S_IWUSR);
		if(fd==1)
			 fatal("open error");
		if(lseek(fd,0,SEEK_END)==-1)
			fatal("lseek error");
		for(i=0;i<off;i++)
		  write(fd,"x",1);
	}else{
		fd=open(argv[1],O_WRONLY|O_CREAT|O_APPEND,S_IRUSR|S_IWUSR);
		for(i=0;i<off;i++)
		  write(fd,"x",1);
	}
	return 0;
}

```

# 5-4

:speak_no_evil:这题没难度，按他说的就可以写出来：

```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
#include<errno.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
int my_dup(int oldfd)
{
	int oldflag,newfd,newflag;
	oldflag=fcntl(oldfd,F_GETFL);
	if(oldflag==-1)
	  return -1;
	newfd=fcntl(oldfd,F_DUPFD,0);
	newflag=fcntl(newfd,F_GETFL);
	printf("dup:%d\n",newflag);
	return newfd;
}
int my_dup2(int oldfd,int newfd)
{
	int newfd2,newflag,oldflag;
	if(oldfd==newfd)
	{
		oldflag=fcntl(oldfd,F_GETFL);
		if(oldflag==-1)
		{
			errno=EBADF;
			return -1;
		}
	}
	newflag=fcntl(newfd,F_GETFL);
	if(newflag!=-1)
	  close(newfd);
	newfd2=fcntl(oldfd,F_DUPFD,newfd);
	newflag=fcntl(newfd2,F_GETFL);
	printf("dup2:%d\n",newflag);
	return newfd2;
}
int main(int argc,char *argv[])
{
	int fd,fdflag,newfd,newfd2;
	off_t off;
	fd=open(argv[1],O_RDONLY|O_CREAT,S_IRUSR|S_IWUSR);
	fdflag=fcntl(fd,F_GETFL);
	printf("open file flag:%d\n",fdflag);
	newfd=my_dup(fd);
	printf("dup fd:%d\n",newfd);
	newfd2=my_dup2(fd,455);
	printf("du2 fd:%d\n",newfd2);
		return 0;
}

```

# 5-5

:sob:这题想都不用想，必然是共享的。看下面这张图你就会明白：

![5-5-1](https://github.com/MatrixWeb/learningLinux/blob/gh-pages/solve/chapter5/5-5-2.png)

示例代码：
```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
#include<errno.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
int main(int argc,char *argv[])
{
	int fd,fdflag,newfd,newflag;
	off_t off;
	fd=open(argv[1],O_WRONLY|O_CREAT|O_APPEND,S_IRUSR|S_IWUSR);
	newfd=dup(fd);
	fdflag=fcntl(fd,F_GETFL);
	newflag=fcntl(newfd,F_GETFL);
	printf("open file flag:%d\n",fdflag);
	printf("newfd:%d\n",newfd);
	printf("newflag:%d\n",newflag);
	write(fd,"hello",4);
	write(newfd,"world",5);
		return 0;
}
```
运行结果：
![5-1-2](https://github.com/MatrixWeb/learningLinux/blob/gh-pages/solve/chapter5/5-5-1.png)

# 5-6

> :flushed: Giddayworld

# 5-7

:sweat_smile: readv()和writev() 是原子操作的，这里留个小遗憾，没实现原子操作<br>
示例代码:

```c
#include<sys/stat.h>
#include<stdio.h>
#include<fcntl.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/uio.h>
void fatal(const char *info)
{
	fprintf(stderr,"%s",info);
}
ssize_t my_readv(int fd,const struct iovec *iov,int iovcnt)
{
	int i=0;
	ssize_t total=0;
	for(i=0;i<iovcnt;i++)
	{
		read(fd,iov[i].iov_base,iov[i].iov_len);
		total+=iov[i].iov_len;
	}
	return total;
}
ssize_t my_writev(int fd,const struct iovec *iov,int iovcnt)
{
	int i;
	ssize_t total=0;
	for(i=0;i<iovcnt;i++)
	{
		write(fd,iov[i].iov_base,iov[i].iov_len);
		total+=iov[i].iov_len;
	}
	return total;
}
int main(int argc,char *argv[])
{
	int fromfd,tofd;
	ssize_t numRead,numWrite;
	off_t off;
	fromfd=open(argv[1],O_RDWR|O_CREAT,S_IRUSR|S_IWUSR);
	tofd=open(argv[2],O_RDWR|O_CREAT,S_IRUSR|S_IWUSR);
	struct iovec iov[3];
	struct stat myStat;
	int x;
	char str[20];
	iov[0].iov_base=&myStat;
	iov[0].iov_len=sizeof(myStat);
	iov[1].iov_base=&x;
	iov[1].iov_len=sizeof(x);
	iov[2].iov_base=str;
	iov[2].iov_len=20;
	numRead=my_readv(fromfd,iov,3);
	numWrite=my_writev(tofd,iov,3);
	printf("read:%d\nwrite:%d\n",(int)numRead,(int)numWrite);
	return 0;
}

```
