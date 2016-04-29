# 18-1
使用ls -i可以看到编译后的二进制文件具有不同的i-node节点号。这是因为编译器移除了任何与目标可执行文件同名的文件，然后在创建一个同名的新文件。解除对可执行文件的链接是允许的。

# 18-2
问题出在symlink("myfile","../mysql")会在父目录创建一个悬空链接，导致后面修改权限失败。

# 18-3
我是在实现getcwd的基础上实现realpath的。只要进入到目标的path再调用getpwd就可以。
```c
#include<stdio.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<dirent.h>
#include<string.h>
#include<unistd.h>
#include<stdlib.h>
static char abspath[500]="\0";
ino_t get_inode(char *fname)
{
	struct stat info;
	if(stat(fname,&info)==-1){
		fprintf(stderr,"not stat");

	}
	return info.st_ino;
}
void inum_to_name(ino_t inode_to_find,char *namebuf,int buflen)
{
	DIR *dirp;
	struct dirent *direntp;
	dirp=opendir(".");
	if(dirp==NULL)
	{
		printf("error!\n");
		exit(1);
	}
	while((direntp=readdir(dirp))!=NULL)
	  if(direntp->d_ino==inode_to_find)
	  {
		  strncpy(namebuf,direntp->d_name,buflen);
		  namebuf[buflen-1]='\0';
		  closedir(dirp);
		  return ;
	  }
}
void printpathto(ino_t this_inode)
{
	ino_t my_inode;
	char its_name[500];
	if(get_inode("..")!=this_inode)
	{
		chdir("..");
		inum_to_name(this_inode,its_name,500);
		my_inode=get_inode(".");
		printpathto(my_inode);
		strcat(abspath,"/");
		strcat(abspath,its_name);
	}
}
char *myrealpath(char *pathname,char *resolved_path)
{
	struct stat info;
	lstat(pathname,&info);
	char pname[500];
	char filename[500];
	strcpy(pname,pathname);
	for(int i=strlen(pathname);i>=0;i--)
	{
		if(pathname[i]=='/')
		{
			strcpy(filename,pathname+i+1);
			pname[i+1]='\0';
		}
	}
	if (S_ISREG(info.st_mode))
	{
		chdir(pname);
		printpathto(get_inode("."));
		strcpy(resolved_path,abspath);
		strcat(resolved_path,"/");
		strcat(resolved_path,filename);
		return resolved_path;
	}
	if(S_ISLNK(info.st_mode))
	{
		char realf[500];
		ssize_t len=readlink(pathname,realf,500);
		realf[len]='\0';
		chdir(pname);
		char *res=myrealpath(realf,resolved_path);
		return res;
	}
	if(S_ISDIR(info.st_mode))
	{
		chdir(pathname);
		printpathto(get_inode("."));
		strcpy(resolved_path,abspath);
		return resolved_path;
	}
	return NULL;
}
int main(int argc ,char *argv[])
{
	char *fname=argv[1];
	char result[1000];
	myrealpath(fname,result);
	printf("%s\n",result);
	return 0;
}

```

# 18-4
这个简单，就是要自己分配dirent,栈上，堆上都可以

```c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<dirent.h>
#include<string.h>
#include<stddef.h>
static void listFiles(const char *dirpath)
{
	DIR *dirp;
	struct dirent *dp;
	struct dirent *result;
	int isCurrent;
	isCurrent=strcmp(dirpath,".")==0;
	dirp=opendir(dirpath);
	size_t len;
	len=offsetof(struct dirent,d_name)+NAME_MAX+1;
	dp=malloc(len);
	while(1)
	{
		readdir_r(dirp,dp,&result);
		if(result==NULL)
		  break;
		if(strcmp(dp->d_name,".")==0||strcmp(dp->d_name,"..")==0)
		  continue;
		if(!isCurrent)
		  printf("%s/",dirpath);
		printf("%s\n",dp->d_name);
	}
	free(dp);
	closedir(dirp);
}
int main(int argc,char *argv[])
{
	if(argc==1)
	  listFiles(".");
	else
	{
	  for(argv++;*argv;argv++)
		listFiles(*argv);
	}
	return 0;
}

```

# 18-5
这里我用大三时候再linux课上李贵林老师给我们的那本教材的方法

```c
#include<stdio.h>
#include<sys/types.h>
#include<sys/stat.h>
#include<dirent.h>
#include<string.h>
#include<unistd.h>
#include<stdlib.h>
ino_t get_inode(char *fname)
{
	struct stat info;
	if(stat(fname,&info)==-1){
		fprintf(stderr,"not stat");

	}
	return info.st_ino;
}
void inum_to_name(ino_t inode_to_find,char *namebuf,int buflen)
{
	DIR *dirp;
	struct dirent *direntp;
	dirp=opendir(".");
	if(dirp==NULL)
	{
		printf("error!\n");
		exit(1);
	}
	while((direntp=readdir(dirp))!=NULL)
	  if(direntp->d_ino==inode_to_find)
	  {
		  strncpy(namebuf,direntp->d_name,buflen);
		  namebuf[buflen-1]='\0';
		  closedir(dirp);
		  return ;
	  }
}
void printpathto(ino_t this_inode)
{
	ino_t my_inode;
	char its_name[500];
	if(get_inode("..")!=this_inode)
	{
		chdir("..");
		inum_to_name(this_inode,its_name,500);
		my_inode=get_inode(".");
		printpathto(my_inode);
		printf("/%s",its_name);
	}
}
int main()
{
	printpathto(get_inode("."));
	putchar('\n');
	return 0;
}

```

# 18-6
其实这题就是打印顺序不一样，FTW_DEPTH是先去遍历目录里面的内容，再去对目录之行函数的操作。

# 18-7
很简单

```c
#define _XOPEN_SOURCE 600
#include<ftw.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
static int numreg=0,numdir=0,numchr=0,numblk=0,numfifo=0,numsock=0,numlnk=0,numnostable=0;
static int dirTree(const char *pathname,const struct stat *sbuf,int type,struct FTW *ftwb)
{
	if(type==FTW_NS)
	{
		numnostable++;
		return 0;
	}
	switch(sbuf->st_mode&S_IFMT){
		case S_IFREG:
			numreg++;
			break;
		case S_IFDIR:
			numdir++;
			break;
		case S_IFCHR:
			numchr++;
			break;
		case S_IFBLK:
			numblk++;
			break;
		case S_IFLNK:
			numlnk++;
			break;
		case S_IFIFO:
			numfifo++;
			break;
		case S_IFSOCK:
			numsock++;
			break;
	}
	return 0;
}
static void print_s(const char *msg,int num,int sum)
{
	printf("%s:%6d,%6.1f\n",msg,num,(num*100.0/sum));
}
int main(int argc,char *argv[])
{
	if(nftw(argv[1],&dirTree,10,FTW_PHYS)==-1)
	{
		printf("error\n");
		return 1;
	}
	int sum=numreg+numdir+numchr+numblk+numlnk+numfifo+numsock+numnostable;
	if(sum==0)
	  printf("no files\n");
	else{
		print_s("Regualr file:",numreg,sum);
		print_s("Diretory:",numdir,sum);
		print_s("Char divice:",numchr,sum);
		print_s("Block divice:",numblk,sum);
		print_s("symbolic link:",numlnk,sum);
		print_s("FIFO:",numfifo,sum);
		print_s("SOCK:",numsock,sum);
		print_s("no stable:",numnostable,sum);
	}
	return 0;
}



```
# 18-8
自己实现了简单版，但是远远不够的。等我下源码

```c
#define _XOPEN_SOURCE 500
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<sys/stat.h>
#include<dirent.h>
#include<ftw.h>
#include<string.h>
int my_nftw(const char *dirpath,int (*func)(const char *pathname, const struct stat *statbuf, int typeflag, struct FTW *ftwbuf),int nopenfd,int flags)
{
	struct stat info;
	struct FTW ftwinfo;
	stat(dirpath,&info);
	if(S_ISREG(info.st_mode)||S_ISLNK(info.st_mode))
	{
		char filename[500];
		char *pathname=realpath(dirpath,filename);
		int typeflag=FTW_F;
		for(int i=strlen(pathname);i>=0;i--)
		{
			if(pathname[i]=='/')
			{
				ftwinfo.base=i+1;
				break;
			}
		}
		(*func)(pathname,&info,typeflag,&ftwinfo);
	}else if(S_ISDIR(info.st_mode))
	{
		char filename[500];
		int typeflag=FTW_D;
		char *pathname=realpath(dirpath,filename);
		(*func)(pathname ,&info ,typeflag,&ftwinfo);
		DIR *dirp;
		dirp=opendir(dirpath);
		struct dirent *direntp;
		while((direntp=readdir(dirp))!=NULL)
		{
			if(!strcmp(direntp->d_name,".")||!strcmp(direntp->d_name,".."))
				continue;
			char fullpath[500];
			strcpy(fullpath,dirpath);
			strcat(fullpath,"/");
			strcat(fullpath,direntp->d_name);
			my_nftw(fullpath,func,nopenfd,flags);
		}
	closedir(dirp);
	}
	return 0;
}
static int dirTree(const char *pathname,const struct stat *sbuf,int type,struct FTW *ftwb)
{
	char result[500];
	printf("%s\n",realpath(pathname,result));
	return 0;
}
int main(int argc,char *argv[])
{
	my_nftw(argv[1],dirTree,10,FTW_DEPTH);
}

```

# 18-9
使用fchdir更为高效。如果是在循环中反复执行操作，那么当调用fchdir时，可以在循环前调用一次open。调用chdir之所以代价高昂，是因为传递buf参数的内核需要在用户空间和内核空间进行大数据传输。每次调用的时候必须将buf中的路径解析道相应目录的i-node节点上。
