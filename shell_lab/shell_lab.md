# 深入理解计算机操作系统 shell lab

这里记录下本人在做这个lab的时候一些体会和心得

> 前台工作组

默认的，一个子进程和它的父进程同属于一个进程组，而Unix系统提供的大量向进程发送信号的机制，都是基于进程组这个概念的。当我们输入Ctrl + C，内核会发送一个SIGINT信号到前台进程组的每个进程，类似的，输入Ctrl + Z会导致内核发送一个SIGTSTP信号给前台进程组中的每个进程。这儿的“前台进程组”指的是Shell进程所属的进程组。实验中，我们并不期望信号直接作用于Shell进程本身（否则Shell收到SIGINT信号就终止了），而是需要让Shell将信号转发给Shell前台作业中的子进程及其所属进程组中的所有进程。所以，我们不能让子进程和Shell进程同属一个进程组。具体做法是通过使用setpgid函数来改变子进程的进程组，当调用`setpgid(0, 0)时`，内核会创建一个新的进程组，其进程组ID是调用者进程的PID，并且会把调用者进程加入到这个进程组中。

> kill -pid 

实验中，我们是通过`kill(pid_t pid, int sig)`来发送信号，注意到我们并不仅仅是向PID = pid的进程发送信号，kill函数帮助我们实现了这一点：如果pid小于0，kill发送信号sig给进程组|pid|（pid的绝对值）中的每个进程。我们可以意识到，上一点需要注意的地方正是为这一点做铺垫的。

> 信号阻塞

父进程（Shell）fork了一个子进程后，父进程需要将这个进程作为一个job添加到job队列中去（addjob），当子进程终止时，内核会发送一个SIGCHLD信号给父进程，然后在相应的信号处理程序中，把终止的子进程对应的job从job队列中删除（deletejob）。
考虑一种情况：<br/>
如果当父进程fork了一个子进程后，子进程开始进行调度，然后子进程终结。这时候控制权又回到了父进程，父进程开始执行`addjob（）`***（？？？？可是子进程已经结束了，addjob（）就意味着从此多了一条再也无法删除的记录了）**

	Sigprocmask(SIG_BLOCK, &mask, NULL);
通过了对SIGNCHLD信号的阻塞，然后添加job，再解除该信号的阻塞。这样就能保证是addjob（）在deletejob（）前。因为就算发生上诉的情况，由信号是阻塞的，所以shell对SIGNCHLD信号的处理**会被延时到信号解除后** <br/>注意，子进程继承了它们父进程的被阻塞信号集合，所以我们必须在调用execve之前，解除子进程中阻塞的SIGCHLD信号。

> waitpid()的正确使用

在一个shell中会有两种工作组，前台工作组和后台工作组。而shell每次执行前台工作组时必须要`等待执行完成`后才能进行下一个。所以在waitfg（）函数中，有人会直接用
	
    if(waitpid(...) > 0 ) {}
    
来等待前台工作组的回收。但对于后台工作组来说，回收工作是通过向shell发送SIGNCHLD导致触发siganl_handler函数来执行的，而在该函数内，也使用的`waitpid（）` <br/>
	
    • One of the tricky parts of the assignment is deciding on the allocation of work between the waitfg <br>
    and sigchld handler functions. We recommend the following approach:<br>
    	– In waitfg, use a busy loop around the sleep function.<br>
    	– In sigchld handler, use exactly one call to waitpid.
对于一个程序同时出现两个`waitpid`不会导致程序出错，但应该保持唯一性。<br/>
**ps: 在处理waitfg中最好的处理方式应该使用suspend（）**

> 信号阻塞 while 

***信号是不会排队的！！***假设三个子进程同时结束，如果我们使用if的情况，到达顺序为A,B,C<br/>
- A: 信号处理器空闲中，将A进行处理
- B: 信号处理器在处理A，B将阻塞等待A的完成，添加到待处理中
- C：因为该类型的信号已经存在阻塞，所以C会被抛弃
所以我们使用while在每次SIGCHLD程序被调用时，回收尽可能多的僵死子进程

> WNOHANG | WUNTRACED

这是waitpid（）中最后一个参数options的值，waitpid（）的默认行为是`挂起shell进程的执行，直到有子进程终止或被停止`，所以为了避免程序不停的在等待，我们使用WNOHANG | WUNTRACED，该参数的作用是判断当前进程中是否存在已经停止或者终止的进程，如果存在则返回pid，不存在则立即返回.**然后我们在while（）{}体内对是停止或终止的进程做处理**


> 如何捕获程序自身内（子进程）发送的信号

明确一点：***signal_handler是处理外界对shell程序发送的信号***<br/>
那么，对于子进程自己给自己发送SIGINT信号和SIGSTSP信号的唯一改变就是，shell外壳不会主动调用sigint_handler和sigstsp_handler，只会在调用发送信号函数kill()的时候，将进程终止或者停止的时候，调用sigchld_handler回收僵尸进程。<br/>
所以我们这里需要用到`waitpid(,&status,)`的第二个参数status
	
    WIFSIGNALED(status);  	/* process is terminated by a signal */
    WIFSTOPPED(status);		/* process is stoped by a siganl */
    
通过判断两个条件式来判断***子进程/外部***是否发送了信号，如果发送了，做出相应的处理。





