# 15-1
a) 无法访问，反正都是permission deny!
b) 在一个可读但无执行权限的目录下，无法列出其文件名。无论文件本身的权限如何，也不能防伪起内容。
c) 对文件可读，可写，可删除。父目录：rwx,进程:rw-。重命名也一样。若重命名操作的目标文件存在，对该文件具有写读权限可将其覆盖。对于 sticky　的目录，只有文件所有者才能对文件重命名和删除。

# 15-2
必然不改变，stat命令只是读文件的元数据信息，并不对文件造成修改或是访问。所以时间戳都是不改变的。

# 15-3
这题不可解，原因是我系统glibc库太低，没有timespec结构。

# 15-4
很简单，就是对比当前有效用户id 和组id，然后去找权限。
```c
#include<stdio.h>
#include<unistd.h>
#include<fcntl.h>
int my_access(const char *pathname,int mode)
{
	uid_t rid,eid,sid;
	gid_t rgid,egid,sgid;
	mode_t fmode;
	struct stat sb;
	getresuid(&rid,&eid,&sid);
	getresgid(&rgid,&egid,&sgid);
	stat(pathname,&sb);
	switch(mode)
	{
		case F_OK:
			return access(pathname,F_OK);
			break;
		case R_OK:
			if(eid==sb.st_uid)
			  return sb.st_mode&S_IRUSR;
			if(egid==sb.st_gid)
			  return sb.st_mode&S_IRGRP;
			return sb.st_mode&S_IROTH;
			break;
		case W_OK:
			if(eid==sb.st_uid)
			  return sb.st_mode&S_IWUSR;
			if(egid==sb.st_gid)
			  return sb.st_mode&S_IWGRP;
			return sb.st_mode&S_IWOTH;
			break;
		case X_OK:
			if(eid==sb.st_uid)
			  return sb.st_mode&S_IXUSR;
			if(egid==sb.st_gid)
			  return sb.st_mode&S_IXGRP;
			return sb.st_mode&S_IXOTH;
			break;
	}
	return 0;
}
int main(int argc,char *argv[])
{
	int ret=my_access(argv[1],R_OK);
	int ret2=my_access(argv[1],W_OK);
	int ret3=my_access(argv[1],X_OK);
	printf("R:%d,W:%d,X:%d\n",ret,ret2,ret3);
	return 0;
}

```

# 15-5
很简单。首先利用系统调用创建一个文件。open的时候mode参数给他所有的权限777，然后去获取创建后的实际权限 a，对a取反得到的是当前的umask，然后调用
```c
u=umask(~a)
```
得到的是当前umask的拷贝又不去改变进程当前的umask

# 15-6
说多都是泪，这题if一句后面多了一个分号，害我调试了半天。
直接上代码；
```c
#include<stdio.h>
#include<sys/stat.h>
#include<sys/types.h>
#include<unistd.h>
#include<time.h>
void process(char *filename)
{
	struct stat sb;
	mode_t mode;
	stat(filename,&sb);
	switch(sb.st_mode & S_IFMT)
	{
		case S_IFREG:
			if(sb.st_mode&S_IXUSR)
			{
				printf("%s\n",filename);
			mode=(sb.st_mode|S_IXUSR|S_IXGRP|S_IXOTH);
			chmod(filename,mode);
			}
			break;
		case S_IFDIR:
			mode=(sb.st_mode|S_IXUSR|S_IXGRP|S_IXOTH);
			chmod(filename,mode);
		break;
	}
}
int main(int argc,char *argv[])
{
	int i=1;
	for(;i<argc;i++)
	{
		process(argv[i]);
	}
}

```

# 15-7
差点以为这题又是一题不可解问题。但是翻翻书发现居然是先打开文件，再去设置i标志位，这是不科学的。因为一旦你比如设置了i,然后后面再去打开设置的话都会是error，因为i是不可变更的。所以会有问题。我实现了一个简单版的，是有问题的。有兴趣的去看源码
```c
#include <sys/types.h>
#include <dirent.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#ifdef HAVE_ERRNO_H
#include <errno.h>
#endif
#include <sys/param.h>
#include <sys/stat.h>
#include<linux/fs.h>


static int add;
static int rem;
static int set;
static int set_version;

static unsigned long version;

static int recursive;
static int verbose;
static int silent;

static unsigned long af;
static unsigned long rf;
static unsigned long sf;

#ifdef _LFS64_LARGEFILE
#define LSTAT		lstat64
#define STRUCT_STAT	struct stat64
#else
#define LSTAT		lstat
#define STRUCT_STAT	struct stat
#endif
static void usage(void)
{
	printf("hehe");
		exit(1);
}

struct flags_char {
	unsigned long	flag;
	char 		optchar;
};

static const struct flags_char flags_array[] = {
	{ FS_NOATIME_FL, 'A' },
	{ FS_SYNC_FL, 'S' },
	{ FS_DIRSYNC_FL, 'D' },
	{ FS_APPEND_FL, 'a' },
	{ FS_COMPR_FL, 'c' },
	{ FS_NODUMP_FL, 'd' },
	{ FS_IMMUTABLE_FL, 'i' },
	{ FS_JOURNAL_DATA_FL, 'j' },
	{ FS_SECRM_FL, 's' },
	{ FS_UNRM_FL, 'u' },
	{ FS_NOTAIL_FL, 't' },
	{ FS_TOPDIR_FL, 'T' },
	{ FS_NOCOW_FL, 'C' },
	{ 0, 0 }
};

static unsigned long get_flag(char c)
{
	const struct flags_char *fp;

	for (fp = flags_array; fp->flag != 0; fp++) {
		if (fp->optchar == c)
			return fp->flag;
	}
	return 0;
}


static int decode_arg (int * i, int argc, char ** argv)
{
	char * p;
	char * tmp;
	unsigned long fl;

	switch (argv[*i][0])
	{
	case '-':
		for (p = &argv[*i][1]; *p; p++) {
			if ((fl = get_flag(*p)) == 0)
				usage();
			rf |= fl;
			rem = 1;
		}
		break;
	case '+':
		add = 1;
		for (p = &argv[*i][1]; *p; p++) {
			if ((fl = get_flag(*p)) == 0)
				usage();
			af |= fl;
		}
		break;
	case '=':
		set = 1;
		for (p = &argv[*i][1]; *p; p++) {
			if ((fl = get_flag(*p)) == 0)
				usage();
			sf |= fl;
		}
		break;
	default:
		return EOF;
		break;
	}
	return 1;
}


static int change_attributes(const char* filename)
{
	unsigned long flags;
	int fd=open(filename,O_RDWR);
	if(ioctl(fd,FS_IOC_GETFLAGS,&flags)==-1)
	{
		printf("error!\n");
		exit(1);
	}
		if (rem)
			flags &= ~rf;
		if (add)
			flags |= af;
	if(ioctl(fd,FS_IOC_SETFLAGS,&flags)==-1)
	{
		printf("error!\n");
		exit(1);
	}
	close(fd);
		return 0;
}

int main (int argc, char ** argv)
{
	int retval=0;
	int i=1,j,err;
	decode_arg(&i,argc,argv);
	i++;
	for (j = i; j < argc; j++) {
		 err = change_attributes (argv[j]);
		if (err)
		retval = 1;
		}
	    exit(retval);
}

```
