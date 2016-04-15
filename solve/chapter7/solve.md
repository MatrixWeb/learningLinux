# 7-1

这里我简单说下，sbr会根据页的大小周期性地分配内存快，然后在链表中查找适合的内存快返回给调用者。示例代码很简单：

```c
#include<stdio.h>
#include<stdlib.h>
#define MAX_ALLOCS 1000000
int main(int argc,char *argv[])
{
	char *ptr[MAX_ALLOCS];
	int freeStep,freeMin,freeMax,blockSize,numAllocs,j;
	printf("\n");
	numAllocs=atoi(argv[1]);
	blockSize=atoi(argv[2]);
	freeStep=(argc>3)?atoi(argv[3]):1;
	freeMin=(argc>4)?atoi(argv[4]):1;
	freeMax=(argc>5)?atoi(argv[5]):numAllocs;
	printf("Initial program break:     0x%10x\n",sbrk(0));
	printf("Allocating %d*%d bytes\n",numAllocs,blockSize);
	for(j=0;j<numAllocs;j++)
	{
		ptr[j]=malloc(blockSize);
		printf("current malloc break: 0x%10x\n",sbrk(0));
	}
	printf("Program break now:    0x%10x\n",sbrk(0));
	printf("Free blocks from %d to %d in steps of %d\n",freeMin,freeMax,freeStep);
	for(j=freeMin-1;j<freeMax;j+=freeStep)
	  free(ptr[j]);
	printf("After free(),program break is :0x%10x\n",sbrk(0));
	return 0;
}

```

运行结果：<br>
![7-1](https://github.com/MatrixWeb/learningLinux/blob/gh-pages/solve/chapter7/7-1.png)

# 7-2
这里我简单实现了书上的malloc 和free 的函数。主要思想就是链表结合sbrk系统调用。
参考：[这里](http://blog.codinglabs.org/articles/a-malloc-tutorial.html)

```c
#define _BSD_SOURCE
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#define BLOCK_SIZE 40
typedef struct s_block *t_block;
typedef struct s_block{
	size_t size;
	t_block prev;
	t_block next;
	int free;
	int padding;
	void *ptr;
	char data[1];
}s_block;
void *first_block=NULL;
t_block find_block(t_block *last,size_t size)
{
	t_block b=first_block;
	while(b&&!(b->free&&b->size>=size))
	{
		*last=b;
		b=b->next;
	}
	return b;
}
t_block extend_block(t_block last,size_t s)
{
	t_block b;
	b=sbrk(0);
	if(sbrk(BLOCK_SIZE+s)==(void *)-1)
	  return NULL;
	b->size=s;
	b->next=NULL;
	if(last){
		last->next=b;
		b->prev=last;
	}
	b->free=0;
	return b;
}
void split_block(t_block b,size_t s)
{
	t_block newone;
	newone=(t_block)((b->data)+s);
	newone->size=b->size-s-BLOCK_SIZE;
	newone->next=b->next;
	newone->prev=b;
	b->size=s;
	b->next=newone;
	newone->free=1;
}
size_t align8(size_t s)
{
	if(s&7==0)
	  return s;
	size_t a=((s>>3)+1)<<3;
	return a;
}
void *my_malloc(size_t size)
{
	t_block b,last;
	size_t s=align8(size);
	if(first_block){
		last=first_block;
		b=find_block(&last,s);
		if(b){
			if((b->size-s)>=(BLOCK_SIZE+8))
			  split_block(b,s);
			b->free=0;
		}else
		{
			b=extend_block(last,s);

			if(!b)
			  return NULL;
		}
	}else{
		b=extend_block(NULL,s);
		if(!b)
		  return b;
		first_block=b;
	}
	if(b)
	  b->ptr=b->data;
	return b->data;
}
t_block get_block(void *p)
{
	char *tmp=p;
	tmp=tmp-BLOCK_SIZE;
	p=tmp;
	return p;
}
int valid_addr(void *p)
{
	if(first_block){
		if(p>first_block&&p<sbrk(0))
		  return p==(get_block(p))->ptr;
	}
	return 0;
}
t_block fusion(t_block b)
{
	if(b->next&&b->next->free)
	{
		b->size=b->next->size+BLOCK_SIZE;
		b->next=b->next->next;
		if(b->next)
		  b->next->prev=b;
	}
	return b;
}
void my_free(void *p)
{
	t_block b;
	if(valid_addr(p)){
		b=get_block(p);
		b->free=1;
		if(b->prev&&b->prev->free)
		  b=fusion(b->prev);
		if(b->next)
		  fusion(b);
		else{
			if(b->prev)
				b->prev->next=NULL;
			else
				first_block=NULL;
			brk(b);
		}
	}
}
void print_block(){
	t_block p=first_block;
		printf("=====\n");
	while(p){
		printf("size:%d\n",(int)(p->size));
		printf("free:%d\n",p->free);
		p=p->next;
	}
		printf("-------\n");
}
int main(int argc,char *argv[]){
	char *test=my_malloc(60);
	for(int i=0;i<5;i++)
		test[i]='a';
	test[5]='\0';
	printf("test:%s\n",test);
	print_block();
	char *happy=my_malloc(8);
	print_block();
	my_free(test);
	print_block();
	char *nice=my_malloc(5);
	print_block();
	return 0;
}

```
