title: 守护进程简单分析
Date: 2013-04-10 9:10
Tags: linux,linux应用,系统编程
Author: lijsf
md_linebreaks: two

守护进程（Daemon）是运行在后台的一种特殊进程。它独立于控制终端并且周期性地执行某种任务或等待处理某些发生的事件。守护进程是一种很有用的进程。Linux的大多数服务就是用守护进程实现的。比如，Internet服务器inetd，Web服务器httpd等。同时，守护进程完成许多系统任务。比如，作业规划进程crond，打印进程lpd等。这些带有d的服务都是守护进程。

## 系统守护进程

系统守护进程分为两种，一种时刻等待某个事件发生，另一种则周期性的运行。前者如httpd，后者如crond。另一方面守护进程在实现时有两种，一种是所谓的standalone守护进程，另一种即super daemon。前者是单独执行一个服务，后者则在某个事件发生时，去启动针对的处理程序。可以看到super daemon是一个管理进程，本身并不提供特定的服务。

Linux系统启动时会自动启动相应的守护进程，使系统正常工作，这些自启动的守护进程主要是由/etc/rc#.d，/etc/init.d这几个目录下的脚本完成的。

系统首先启动init进程，该进程再根据启动级别（0——6）去执行rc#.d目录下的脚本，而rc#.d下的脚本实际上都是指向init.d目录下脚本的链接。在执行rc#.d目录下的脚本时，会根据脚本的名字来决定执行的次序，而rc#.d下的脚本的名字命名规范是；

 + S{number}{name}
 + K{number}{name}

 其中S开始的文件向脚本传递start参数,K开始的文件向脚本传递stop参数,number决定执行的顺序，序号越小则执行越早，并且K开始的脚本先于S开始的脚本。这是因为在从较高的执行级别切换到较低的执行级别时，需要先关掉一部分服务。

## 守护进程编程

守护进程最重要的特性是后台运行，其次，守护进程必须与其运行前的环境隔离开来。这些环境包括未关闭的文件描述符，控制终端，会话和进程组，工作目录以及文件创建掩模等。这些环境通常是守护进程从执行它的父进程（特别是shell）中继承下来的。最后，守护进程的启动方式有其特殊之处。它可以在Linux系统启动时从启动脚本/etc/rc.d中启动，可以由作业规划进程crond启动，还可以由用户终端（通常是shell）执行。
总之，除开这些特殊性以外，守护进程与普通进程基本上没有什么区别。因此，编写守护进程实际上是把一个普通进程按照上述的守护进程的特性改造成为守护进程。

### 守护进程的编程要点

不同Unix环境下守护进程的编程规则并不一致。所幸的是守护进程的编程原则其实都一样，区别在于具体的实现细节不同。这个原则就是要满足守护进程的特性。同时，Linux是基于Syetem V的SVR4并遵循Posix标准，实现起来与BSD4相比更方便。编程要点如下；

1. 在后台运行。
为避免挂起控制终端，需将Daemon放入后台执行。方法是在进程中调用fork，然后使父进程终止，让Daemon在子进程中后台执行。

        :::c
        if(pid=fork()>0)
          exit(0);//结束父进程，子进程继续

2. 脱离控制终端，登录会话和进程组

进程属于一个进程组，进程组号（GID）就是进程组长的进程号（PID）。登录会话可以包含多个进程组。这些进程组共享一个控制终端。这个控制终端通常是创建进程的登录终端。
控制终端，登录会话和进程组通常是从父进程继承下来的。我们的目的就是要摆脱它们，使之不受它们的影响。方法是在第1点的基础上，调用setsid()使进程成为会话组长：

        :::c
        setsid();

说明：当进程是会话组长时setsid()调用失败。但第一点已经保证进程不是会话组长。setsid()调用成功后，进程成为新的会话组长和新的进程组长，并与原来的登录会话和进程组脱离。由于会话过程对控制终端的独占性，进程同时与控制终端脱离。

3. 禁止进程重新打开控制终端

现在，进程已经成为无终端的会话组长。但它可以重新申请打开一个控制终端。可以通过使进程不再成为会话组长来禁止进程重新打开控制终端：

        :::c
        if(pid=fork()>0)
          exit(0);//结束第一子进程，第二子进程继续（第二子进程不再是会话组长）

4. 关闭打开的文件描述符

进程从创建它的父进程那里继承了打开的文件描述符。如不关闭，将会浪费系统资源，造成进程所在的文件系统无法卸下以及引起无法预料的错误。按如下方法关闭它们：

        :::c
        for(i=0;i<NOFILE; close(i);

5. 改变当前工作目录

进程活动时，其工作目录所在的文件系统不能卸下。一般需要将工作目录改变到根目录。对于需要转储核心，写运行日志的进程将工作目录改变到特定目录如/tmpchdir("/")

6. 重设文件创建掩模

进程从创建它的父进程那里继承了文件创建掩模。它可能修改守护进程所创建的文件的存取位。为防止这一点，将文件创建掩模清除：umask(0);

7. 处理SIGCHLD信号

处理SIGCHLD信号并不是必须的。但对于某些进程，特别是服务器进程往往在请求到来时生成子进程处理请求。如果父进程不等待子进程结束，子进程将成为僵尸进程（zombie）从而占用系统资源。如果父进程等待子进程结束，将增加父进程的负担，影响服务器进程的并发性能。在Linux下可以简单地将SIGCHLD信号的操作设为SIG_IGN。

        :::c
        signal(SIGCHLD,SIG_IGN);

这样，内核在子进程结束时不会产生僵尸进程。这一点与BSD4不同，BSD4下必须显式等待子进程结束才能释放僵尸进程。

经过上面的几步处理后，一个进程即满足了守护进程需求，可以长期在后台运行，完成任务。

### 守护进程实例

守护进程实例包括两部分：主程序test.c和初始化程序init.c。主程序每隔一分钟向/tmp目录中的日志test.log报告运行状态。初始化程序中的init_daemon函数负责生成守护进程。

init.c

        :::c
        #include <unistd.h>
        #include <signal.h>
        #include <sys/param.h>
        #include <sys/types.h>
        #include <sys/stat.h>
        void init_daemon(void)
        {
            int pid;
            int i;
            if((pid=fork())>0)
                exit(0);//是父进程，结束父进程
            else if(pid< 0)
                exit(1);//fork失败，退出
            //是第一子进程，后台继续执行
            setsid();//第一子进程成为新的会话组长和进程组长
            //并与控制终端分离
            if(pid=fork())
                exit(0);//是第一子进程，结束第一子进程
            else if(pid< 0)
                exit(1);//fork失败，退出
            //是第二子进程，继续
            //第二子进程不再是会话组长

            for(i=0;i< NOFILE;++i)//关闭打开的文件描述符
                close(i);
            chdir("/tmp");//改变工作目录到/tmp
            umask(0);//重设文件创建掩模
            return;
        }

test.c

        :::c
        #include <stdio.h>
        #include <time.h>

        void init_daemon(void);//守护进程初始化函数

        main()
        {
            FILE *fp;
            time_t t;
            init_daemon();//初始化为Daemon

            while(1)//每隔一分钟向test.log报告运行状态
            {
                sleep(30);//睡眠一分钟
                if((fp=fopen("test.log","a")) >=0)
                {
                    t=time(0);
                    fprintf(fp,"Im here at %s/n",asctime(localtime(&t)) );
                    fclose(fp);
                }
            }
        }


编译：gcc -g -o test init.c test.c
执行：./test
查看进程：ps -ef | grep test

## daemon函数

前文中给出了一个daemon守护进程的实现函数init_daemon()，实际上Linux系统提供了一个函数daemon可以直接完成把进程转为守护进程。函数原型如下：

        :::c
        #include <unistd.h>
        int daemon(int nochdir, int noclose);

1． daemon()函数主要用于希望脱离控制台，以守护进程形式在后台运行的程序。
2． 当nochdir为0时，daemon将更改进程的根目录为root(“/”)。
3． 当noclose为0是，daemon将进程的STDIN, STDOUT, STDERR都重定向到/dev/null。

使用时，直接把业务逻辑程序放在daemon函数后面即可。如下所示：

        int main()
        {
            daemon(1, 1)； //参数根据需求定
            /*  在这里添加你需要在后台做的工作代码  */
        }