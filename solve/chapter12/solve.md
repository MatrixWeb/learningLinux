# 12-1
遍历/proc/pid/status文件就可以
```c
#include<stdio.h>
#include<stdlib.h>
#include<dirent.h>
#include<unistd.h>
#include<string.h>
#include<pwd.h>
#include<grp.h>
int cur_uid;
uid_t userIdFromName(const char *name)
{
	struct passwd *pwd;
	uid_t t;
	char *endptr;
	if(name==NULL||*name=='\0')
	  return -1;
	t=strtol(name,&endptr,10);
	if(*endptr=='\0')
	  return t;
	pwd=getpwnam(name);
	if(pwd==NULL)
	  return -1;
	return pwd->pw_uid;
}
void print_process(const char *filename)
{
	char line[1000];
	int count=0;
	FILE *fp=fopen(filename,"rb");
	if(fp==NULL)
	  return;
		char name[1000];
		char pid[1000];
		int is_print=0;
	while(fgets(line,999,fp)!=NULL)
	{
		char key[100];

		int i;
		count=strlen(line);
		for(i=0;i<count;i++)
		{
			key[i]=line[i];
			if(line[i]==':')
			  break;
		}
		key[i]='\0';
		if(strcmp(key,"Name")==0)
		{
			strcpy(name,line);
		}
		if(strcmp(key,"Pid")==0)
		  strcpy(pid,line);
		if(strcmp(key,"Uid")==0)
		{
			int flag=0;
			char reusr[100];
			int index=0;
			for(int j=i+1;j<count;j++)
			{
				if(('0'<=line[j])&&('9'>=line[j]))
				{
					if(flag==0)
					  flag=1;
					if(flag==1)
						reusr[index++]=line[j];
				}else{
					if(flag==1)
					{
						reusr[index]='\0';
					  break;
					}
				}
			}
			int userid=atoi(reusr);
			if(cur_uid==userid)
			  is_print=1;

		}

	}
		if(is_print){
			printf("======\n");
			printf("%s\n",name);
			printf("%s\n",pid);
			printf("======\n");
		}
	fclose(fp);
}
void process(const char *dirname)
{
	int namelen=20+strlen(dirname);
	char *filename=malloc(namelen);
	strcpy(filename,"/proc/");
	filename[6]='\0';
	strcat(filename,dirname);
	strcat(filename,"/status");
	print_process(filename);
}
int main(int argc,char *argv[])
{
	DIR* dirp=NULL;
	struct dirent *direntp=NULL;
	uid_t uid=userIdFromName(argv[1]);
	cur_uid=(int)uid;
	if((dirp=opendir("/proc/"))==NULL)
	{
		printf("open dir error!\n");
	}else{
		while(direntp=readdir(dirp))
		{
			char *dirname=direntp->d_name;
			if(('0'<dirname[0])&&(dirname[0]<='9'))
			{
				process(dirname);
			}
		}
		closedir(dirp);
	}
}

```

# 12-2
这题稍有难度，不过难度终究是俗物。解决方式搓点没关系。思路就是建树，然后深度优先遍历。

```c
#include<stdio.h>
#include<stdlib.h>
#include<dirent.h>
#include<unistd.h>
#include<string.h>
#include<pwd.h>
#include<grp.h>
struct proc{
	int pid;
	int ppid;
	char name[200];
	int count;
	struct proc *child[1000];
}info[1000];
int bottom=0;
void print_proc(struct proc p)
{
	printf("%d,%d,%s,%d\n",p.ppid,p.pid,p.name,p.count);
}
char *getName(char *line)
{
	char key[100];

	int i,count;
	count=strlen(line);
	for(i=0;i<count;i++)
	{
		key[i]=line[i];
		if(line[i]==':')
		  break;
	}
	key[i]='\0';
	char *result=malloc(count-i+2);
	strcpy(result,line+i+2);
	return result;
}
int getPid(char *line,char *targ)
{
	char key[100];

	int i,count;
	count=strlen(line);
	for(i=0;i<count;i++)
	{
		key[i]=line[i];
		if(line[i]==':')
		  break;
	}
	key[i]='\0';
		char value[100];
		int k=0;
		for(int j=i+1;j<count;j++)
		{
			if(('0'<=line[j])&&('9'>=line[j]))
				value[k++]=line[j];
		}
		value[k]='\0';
		int result=atoi(value);
		return result;

}
void getproc(char *filename)
{
	FILE * fp=fopen(filename,"rb");
	if(fp==NULL)
	  return;
	char line[1000];
	while(fgets(line,1000,fp)!=NULL)
	{
		char key[100];

		int i,count;
		count=strlen(line);
		for(i=0;i<count;i++)
		{
			key[i]=line[i];
			if(line[i]==':')
			break;
		}
		key[i]='\0';
		if(strcmp(key,"Pid")==0)
		{
			int pid=getPid(line,"Pid");
			info[bottom].pid=pid;
		}
		if(strcmp(key,"PPid")==0)
		{
			int ppid=getPid(line,"PPid");
			info[bottom].ppid=ppid;
		}
		if(strcmp(key,"Name")==0)
		{
			char *tmp=getName(line);
			strcpy(info[bottom].name,tmp);
			char *bn=info[bottom].name;
			bn[strlen(bn)-1]='\0';
			free(tmp);
		}
	}
	info[bottom].count=0;
	bottom++;
	fclose(fp);
}
void process(const char *dirname)
{
	int namelen=20+strlen(dirname);
	char *filename=malloc(namelen);
	strcpy(filename,"/proc/");
	filename[6]='\0';
	strcat(filename,dirname);
	strcat(filename,"/status");
	getproc(filename);
	free(filename);
}
void maketree()
{
	for(int i=0;i<bottom;i++)
	{
		for(int j=0;j<bottom;j++)
		{
			if(i!=j)
			{
				if((info[i].ppid!=0)&&(info[i].ppid==info[j].pid))
				{
					info[j].child[info[j].count]=&info[i];
					info[j].count++;
					break;
				}
			}
		}
	}
}
void rescur(char *last,struct proc p)
{
	char *tem=malloc(strlen(last)+strlen(p.name)+3);
	strcpy(tem,last);
	strcat(tem,"-");
	strcat(tem,p.name);
	if(p.count==0)
	{
		printf("%s\n",tem);
		free(tem);
		return;
	}
	else
	{
		for(int j=0;j<p.count;j++)
		  rescur(tem,*p.child[j]);
		free(tem);
	}
}
void print_tree()
{
	int inde=0;
	for(int i=0;i<bottom;i++)
	{
		if(info[i].pid==1)
		{
			inde=i;
			break;
		}
	}
	rescur("",info[inde]);
}
int main(int argc,char *argv[])
{
	DIR* dirp=NULL;
	struct dirent *direntp=NULL;
	if((dirp=opendir("/proc/"))==NULL)
	{
		printf("open dir error!\n");
	}else{
		while((direntp=readdir(dirp))!=NULL)
		{
			char *dirname=direntp->d_name;
			if(('0'<dirname[0])&&(dirname[0]<='9'))
			{
				process(dirname);
			}
		}
		closedir(dirp);
	}
	maketree();
	print_tree();
	return 0;
}

```

# 12-3
这题没难度，就是遍历。
```c
#include<stdio.h>
#include<stdlib.h>
#include<dirent.h>
#include<unistd.h>
#include<string.h>
#include<pwd.h>
#include<grp.h>
int judgeeq(char *path,char *target)
{
	char buf[1000];
	ssize_t result=readlink(path,buf,1000);
	buf[(int)result]='\0';
	if(strcmp(buf,target)==0)
	  return 1;
	return 0;
}
int judgefile(char *filename,char *target)
{
	char abspath[1000];
	strcpy(abspath,filename);
	strcat(abspath,"/fd/");

	DIR* dirp=NULL;
	struct dirent *direntp=NULL;
	if((dirp=opendir(abspath))==NULL)
	{
		printf("open dir error!\n");
	}else{
		while((direntp=readdir(dirp))!=NULL)
		{
			char *dirname=direntp->d_name;
			if(('0'<dirname[0])&&(dirname[0]<='9'))
			{
				char finalpre[500];
				strcpy(finalpre,abspath);
				strcat(finalpre,dirname);
				//printf("%s\n",finalpre);
				if(judgeeq(finalpre,target))
				  return 1;
			}
		}

		closedir(dirp);

	}
	return 0;
}
char *getName(char *line)
{
	char key[100];

	int i,count;
	count=strlen(line);
	for(i=0;i<count;i++)
	{
		key[i]=line[i];
		if(line[i]==':')
		  break;
	}
	key[i]='\0';
	char *result=malloc(count-i+2);
	strcpy(result,line+i+2);
	return result;
}

char* getproc(char *filename)
{
	FILE * fp=fopen(filename,"rb");
	if(fp==NULL)
	  return;
	char line[1000];
	char *tmp;
	while(fgets(line,1000,fp)!=NULL)
	{
		char key[100];

		int i,count;
		count=strlen(line);
		for(i=0;i<count;i++)
		{
			key[i]=line[i];
			if(line[i]==':')
			break;
		}
		key[i]='\0';
		if(strcmp(key,"Name")==0)
		{
			tmp=getName(line);
		}
	}
	fclose(fp);
	return tmp;
}
void process(char *dirname,char *target)
{
	int namelen=20+strlen(dirname);
	char *filename=malloc(namelen);
	strcpy(filename,"/proc/");
	filename[6]='\0';
	strcat(filename,dirname);
	if(judgefile(filename,target))
	{
		strcat(filename,"/status");
		char *procName=getproc(filename);
		printf("%s\n",procName);
		free(procName);
	}
	free(filename);
}
int main(int argc,char *argv[])
{

	DIR* dirp=NULL;
	struct dirent *direntp=NULL;
	//printf("%s\n",argv[1]);
	if((dirp=opendir("/proc/"))==NULL)
	{
		printf("open dir error!\n");
	}else{
		while((direntp=readdir(dirp))!=NULL)
		{
			char *dirname=direntp->d_name;
			if(('0'<dirname[0])&&(dirname[0]<='9'))
			{
				process(dirname,argv[1]);
			}
		}
		closedir(dirp);
	}
	return 0;
}

```

总结：这三题花了我快一周的时间，五天，每天晚上十点半到十二点。有时候遇到一个coredump调了好久。
