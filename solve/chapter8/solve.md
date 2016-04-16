# 8-1
实验证明，并没有发现同一个数字显示两次的现象。本题的用意应该是想让我意识到getpwnam函数是不可重入的。就是说每次调用这个函数，都会重新写一遍passwd的静态结构，这部分是存在数据区的。但每次printf对参数的处理是会按顺序并将结果村到栈中的。所以不出现题目的现象。
我的代码：
```c
#include<stdio.h>
#include<pwd.h>
int main(int argc,char *argv[])
{
	printf("%ld %ld\n",(long)(getpwnam("sshd")->pw_uid),(long)(getpwnam("matrix")->pw_uid));
	return 0;
}

```

# 8-2
这题就是简单的遍历，查找。
代码：

```c
#include<stdio.h>
#include<pwd.h>
#include<string.h>
struct passwd *my_getpwnam(const char *name)
{
	struct passwd *pwd;
	while((pwd=getpwent())!=NULL)
	{
		if(strcmp(pwd->pw_name,name)==0)
		{
			endpwent();
			return pwd;
		}
	}
	printf("counld not find user name\n");
	endpwent();
	return NULL;
}
int main(int argc,char *argv[])
{
	struct passwd *myp=my_getpwnam("matrix");
	if(myp!=NULL)
	{
		printf("name:%s\n",myp->pw_name);
		printf("uid:%ld\n",(long)(myp->pw_uid));
		printf("gid:%ld\n",(long)(myp->pw_gid));
	}
	return 0;
}

```
