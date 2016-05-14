前言：这章的解答脱了好久。主要最近组里赶项目还没赶完。每晚回去很累，看一会百年孤独就睡了。信号这块我五一回去就在看了，但是现在才去解答课本习题。

# 20-1
这题不难，完全的改写课本上的例子
```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
static int sigCnt[NSIG];
static volatile sig_atomic_t gotSigint=0;
void printSigset(FILE *of,const char *prefix,const sigset_t *sigset)
{
	int sig,cnt;
	cnt=0;
	for(sig=1;sig<NSIG;sig++)
	{
		if(sigismember(sigset,sig))
		{
			cnt++;
			fprintf(of,"%s%d (%s)\n",prefix,sig,strsignal(sig));
		}
	}
	if(cnt==0)
	  fprintf(of,"%s<empty signal set>\n",prefix);
}
static void handler(int sig)
{
	if(sig==SIGINT)
	  gotSigint=1;
	else
	  sigCnt[sig]++;
}
int main(int argc,char *argv[])
{
	int n,numSecs;
	sigset_t pendingMask,blockingMask,emptyMask;
	printf("%s :PID is %ld\n",argv[0],(long)getpid());
	struct sigaction sa;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags=0;
	sa.sa_handler=handler;
	for(n=1;n<NSIG;n++)
	  sigaction(n,&sa,NULL);
	if(argc>1)
	{
		numSecs=atoi(argv[1]);
		sigfillset(&blockingMask);
		sigprocmask(SIG_SETMASK,&blockingMask,NULL);
		printf("%s:Sleeping for %d seconds\n",argv[0],numSecs);
		sleep(numSecs);
		sigpending(&pendingMask);
		printf("%s:pengding signals ard:\n",argv[0]);
		printSigset(stdout,"\t\t",&pendingMask);
		sigemptyset(&emptyMask);
		sigprocmask(SIG_SETMASK,&emptyMask,NULL);
	}
	while(!gotSigint)
	  continue;
	for(n=1;n<NSIG;n++)
	  if(sigCnt[n]!=0)
		printf("%s: signal %d caught %d time %s\n",argv[0],n,sigCnt[n],(sigCnt[n]==1?"":"s"));
	return 0;
}

```

# 20-2
试了下解答给出的例子，就可以了
```c
#define _GNU_SOURCE
#include<signal.h>
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>
#include<string.h>
void printSigset(FILE *of,const char *prefix,const sigset_t *sigset)
{
	int sig,cnt;
	cnt=0;
	for(sig=1;sig<NSIG;sig++)
	{
		if(sigismember(sigset,sig))
		{
			cnt++;
			fprintf(of,"%s%d (%s)\n",prefix,sig,strsignal(sig));
		}
	}
	if(cnt==0)
	  fprintf(of,"%s<empty signal set>\n",prefix);
}
static void handler(int sig)
{
	printf("Caught signal %d(%s)\n",sig,strsignal(sig));
}
int main(int argc,char *argv[])
{
	sigset_t pending,blocked;
	const int numsec=5;
	printf("setting up handler for SIGINT\n");
	struct sigaction sa;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags=0;
	sa.sa_handler=handler;
	sigaction(SIGINT,&sa,NULL);
	sigemptyset(&blocked);
	sigaddset(&blocked,SIGINT);
	sigprocmask(SIG_SETMASK,&blocked,NULL);
	printf("BLOGCKING SIGINT fro %d secs\n",numsec);
	sleep(numsec);
	sigpending(&pending);
	printf("pendng signals are:\n");
	printSigset(stdout,"\t\t",&pending);
	sleep(2);
	printf("ignoring SIGINT\n");
	signal(SIGINT,SIG_IGN);
	sigpending(&pending);
	if(sigismember(&pending,SIGINT)){
		printf("SIGINT now is pending!");
	}else{
		printf("pendng signals are:\n");
		printSigset(stdout,"\t\t",&pending);
	}
	sleep(2);
	printf("unblocking SIGINT\n");
	sigemptyset(&blocked);
	sigprocmask(SIG_SETMASK,&blocked,NULL);
	return 0;
}

```

# 20-3
用个简单例子改写下，这里我说明下：

　SA_RESETHAND: 在调用信号处理器之前会去使用默认的处理方式

　SA_NODEFER: 就是信号处理的过程可以再次被这个信号打断

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
	sleep(5);
	printf("Ouch2!\n");
}
int main(int argc,char *argv[])
{
	struct sigaction sa;
	sigemptyset(&sa.sa_mask);
	sa.sa_flags=SA_NODEFER;
	sa.sa_handler=handler;
	sigaction(SIGINT,&sa,NULL);
	sleep(15);
	return 0;
}

 ```

# 20-4

```c
#include <stdio.h>
#include <signal.h>
int
siginterrupt(int sig, int flag)
{
    int status;
    struct sigaction act;

    status = sigaction(sig, NULL, &act);
    if (status == -1)
        return -1;

    if (flag)
        act.sa_flags &= ~SA_RESTART;
    else
        act.sa_flags |= SA_RESTART;

    return sigaction(sig, &act, NULL);
}
```
