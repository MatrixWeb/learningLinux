最近比较忙，每次回到家都好晚了，也没有时间和精力去解题了，取而代之的是每晚的锻炼和早睡。

# 22-1

题目描述的没错。可以验证下：

```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
static void handler(int sig)
{
	printf("Ouch!\n");
}
int main(int argc,char *argv[])
{
	struct sigaction sa;
	sigset_t block,pre;
	sigemptyset(&block);
	sigaddset(&block,SIGCONT);
	sigprocmask(SIG_BLOCK,&block,&pre);
	sigemptyset(&sa.sa_mask);
	sa.sa_flags=0;
	sa.sa_handler=handler;
	sigaction(SIGCONT,&sa,NULL);
	printf("begin sleep\n");
	sleep(15);
	sigprocmask(SIG_SETMASK,&pre,NULL);
	return 0;
}

```

# 22-2

实时信号排队，标准信号按调度不规律获取。
发送信号代码：
```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
int main(int argc,char *argv[])
{
	int sig,numSigs,j,sigData;
	union sigval sv;
	printf("%s :PID is %ld,UID is %ld",argv[0],(long)getpid(),(long)getuid());
	sigData=atoi(argv[2]);
	numSigs=atoi(argv[3]);
	for(j=0;j<numSigs;j++)
	{
		sv.sival_int=sigData+j;
		sigqueue(atol(argv[1]),SIGRTMIN,sv);
		kill(atol(argv[1]),SIGINT);
	}
	return 0;
}

```

接收信号代码：
```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
static volatile int handlerSleepTime;
static volatile int sigCnt=0;
static volatile int allDone=0;
static void siginfoHandler(int sig,siginfo_t *si,void *ucontext)
{
	if(sig==SIGTERM){
	  allDone=1;
	  return;
	}
	if(sig==SIGINT)
	  printf("SIGINT handler begin:\n");
	sigCnt++;
	printf("caught signal %d\n",sig);
	printf("	si_signo=%d, si_code=%d(%s), ",si->si_signo,si->si_code,
				(si->si_code==SI_USER)?"SI_USER":(si->si_code==SI_QUEUE)?"SI_QUEUE":"other");
	printf("si_value=%d\n",si->si_value.sival_int);
	printf(" si_pid=%ld,si_uid=%ld\n",(long)si->si_pid,(long)si->si_uid);
	sleep(handlerSleepTime);
}
int main(int argc,char *argv[])
{
	struct sigaction sa;
	int sig;
	sigset_t prevMask,blockMask;
	printf("%s PID is %ld\n",argv[0],(long)getpid());
	handlerSleepTime=atoi(argv[2]);
	sa.sa_sigaction=siginfoHandler;
	sa.sa_flags=SA_SIGINFO;
	sigfillset(&sa.sa_mask);
	for(sig=1;sig<NSIG;sig++)
	  if(sig!=SIGTSTP && sig!=SIGQUIT)
		sigaction(sig,&sa,NULL);
	sigfillset(&blockMask);
	sigdelset(&blockMask,SIGTERM);
	sigprocmask(SIG_SETMASK,&blockMask,&prevMask);
	printf("%s:signals blocked -sleeping %s seconds\n",argv[0],argv[1]);
	sleep(atoi(argv[1]));
	printf("%s:sleep complete\n",argv[0]);
	sigprocmask(SIG_SETMASK,&prevMask,NULL);
	while(!allDone)
	  pause();
	return 0;
}

```


# 22-3
不得不说，sigwaitinfo确实比sigsuspend快很多
```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
#include<sys/time.h>
#define TESTSIG SIGUSR1
static void handler(int sig)
{
}
int main(int argc,char *argv[])
{
	int numSigs,scnt;
	pid_t childPid;
	sigset_t blockedMask,emptyMask;
	struct sigaction sa;
	struct timeval start,end;
	unsigned long timer;
	gettimeofday(&start,NULL);
	numSigs=atoi(argv[1]);
	sigemptyset(&sa.sa_mask);
	sa.sa_flags=0;
	sa.sa_handler=handler;
	sigaction(TESTSIG,&sa,NULL);
	sigemptyset(&blockedMask);
	sigaddset(&blockedMask,TESTSIG);
	sigprocmask(SIG_SETMASK,&blockedMask,NULL);
	sigemptyset(&emptyMask);
	switch(childPid=fork()){
		case 0:
			for(scnt=0;scnt<numSigs;scnt++)
			{
				kill(getppid(),TESTSIG);
				sigwaitinfo(&blockedMask,NULL);
			}
		gettimeofday(&end,NULL);
		timer=1000000*(end.tv_sec-start.tv_sec)+(end.tv_usec-start.tv_usec);
		printf("child process:%ld us\n",timer);
		return 0;
		default:
			for(scnt=0;scnt<numSigs;scnt++)
			{
				sigwaitinfo(&blockedMask,NULL);
				kill(childPid,TESTSIG);
			}
		gettimeofday(&end,NULL);
		timer=1000000*(end.tv_sec-start.tv_sec)+(end.tv_usec-start.tv_usec);
		printf("parent process:%ld us\n",timer);
		return 0;
	}
	return 0;
}

```
