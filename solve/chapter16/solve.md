# 16-1
这题送分题
```c
#include<stdio.h>
#include<unistd.h>
#include<sys/xattr.h>
#include<string.h>
int main(int argc,char *argv[])
{
	char *value=argv[3];
	setxattr(argv[1],argv[2],argv[3],strlen(value),0);
	return 0;
}

```
