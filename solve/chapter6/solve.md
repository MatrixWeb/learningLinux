# 6-1

这是显然的。因为这些数组在进程中属于未初始化段。在生成二进制时，程序在磁盘上存储。没有必要为未初始化的变量分配内存。相反，可执行文件只需记录未初始化数据段的位置和大小，直到运行时再由程序加载器来分配这一空间。

# 6-2

照理说，这会是一个严重错误。longjmp()调用不能跳转到一个已经返回的函数。因为在这种情况下，longjmp()尝试将栈解开，恢复到一个不存在的栈帧位置，这无疑引起混乱。但是，我写的代码里，longjmp()都会跳转到setjmp()返回。下面是我的代码：

```c
#include<stdio.h>
#include<setjmp.h>
static jmp_buf env;
void f2()
{
	longjmp(env,1);
}
void f1()
{
	int a=4;
	printf("first a=%d\n",a);
	switch (setjmp(env)){
		case 0:
			//f2();
			return;
		break;
		case 1:
			printf("second a=%d\n",a);
		break;
	}

	//f2();
	//printf("second a=%d\n",a);
	return;
}

void f3()
{
	printf("hehe\n");
	f2();
	printf("f3 end\n");
}
void f4()
{
	printf("after jmp\n");
}
int main(int argc,char *argv[])
{
	f1();
	f3();
	//f2();
	//f4();
	return 0;
}

```

# 6-3

这里我做过实验，putenv函数会实现对环境变量的检查。如果是变量的多次定义，则将移除该变量的所有定义。
代码：

```c

#include<stdio.h>
#include<stdlib.h>
#include<string.h>
extern char **environ;
int my_setenv(const char *name,const char *value,int overwrite)
{
	int total_len=strlen(name)+strlen(value)+2;
	int i,j,ret;
	if(overwrite||((overwrite==0)&&(getenv(name)==NULL)))
	{
		char *newstr=malloc(total_len);
		for(i=0;i<strlen(name);i++)
			  newstr[i]=name[i];
			newstr[i++]='=';
		for(j=0;j<strlen(value);j++,i++)
			{
				newstr[i]=value[j];
			}
			newstr[i]='\0';
			ret=putenv(newstr);
		if(ret!=0)
	  return -1;
	}
		return 0;
}
int my_unsetenv(const char *name)
{
	char *tempname=malloc(strlen(name));
	strcpy(tempname,name);
		if(getenv(tempname)!=NULL)
		  putenv(tempname);
		return 0;
}
void envprint()
{
	char **ep;
	for(ep=environ;*ep!=NULL;ep++)
	  puts(*ep);
}
int main(int argc,char *argv[])
{
	my_setenv("hello","world2",0);
	my_setenv("hellogz","world3",1);
	char *buf=malloc(13);
	strcpy(buf,"hello=world2\0");
	char **ep;
	for(ep=environ;*ep!=NULL;ep++);
	ep--;
	free(*ep);
	*ep=buf;
	envprint();
	my_unsetenv("hello");
	//*ep=buf;
	//ep++;
	//*ep=NULL
	my_setenv("hello2","world",1);
	printf("=======");
	envprint();
	return 0;
}

```
