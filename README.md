## NtyCo项目分析：

项目简要说明：NtyCo是github上一位作者用纯C版本的协程实现。

---

先来看一下项目结构：

- core：存放实现协程的核心代码
- deps
- htdoc：一些文档
- sample：实现样例
- websocket

主要分析core目录下的内容。**分析一个项目最好的办法就是从样例出发**

---

挑选sample/nty_server.c为样例入手，分析NtyCo的实现原理。

该样例的作用就是实现一个服务器端接受请求并响应的这么一个server。在分析这个文件之前，我想先分享以下在这个项目之前我对实现服务器端的server的2种实现方式。

1.同步的方式

```c
func(){
    fds = epoll_wait();
    
    for(fd in fds){
        recv(fd);
        write(fd);
    }
}
```

2.IO异步方式(注意这里的IO异步并不是Linux中所说的异步IO[AIO]),这里说的异步是指处理业务不在同一个流程里面，例如下面这一个样例。

```c
//run in another thread
handle(fd){
    recv(fd);
    write(fd);s
}

func(){
    fds = epoll_wait();
    
    for(fd in fds){
        handle(fd);
    }
}
```

但是以上两种方式各有优缺点：

1.同步方式响应速度太慢了，但是以同步的方式不那么容易出现问题，比如出现多个线程读写同一个fd的情况（虽然可以通过add/del fd的方式避免这种情况）。

2.异步的方式响应速度比较快，但是编写容易出现多线程读写同一个fd情况。

为了结合两者的优点，协程诞生了，它的响应速度比较快，并且也用同步味道

下面展示sample/nty_server.c中的main函数

```c
int main(int argc, char *argv[]) {
	nty_coroutine *co = NULL;

	int i = 0;
	unsigned short base_port = 9096;
	for (i = 0;i < 100;i ++) {
		unsigned short *port = calloc(1, sizeof(unsigned short));
		*port = base_port + i;
		nty_coroutine_create(&co, server, port); ////////no run
	}

	nty_schedule_run(); //run

	return 0;
}
```

**分析**：从函数名大概可以推断出该程序就是创建了一个server为主要工作流程的协程后，调用了协程调度器开始工作。

首先我们要明确一下研究这段代码时的角度，我们站在一个线程的角度，在Linux中就是一个task_struct结构体表示的调度单位的角度去分析这个程序。

nty_coroutine_create()这个函数的作用只是创建了一些协程的数据来描述server，它依然是在main这个线程里面，即使是到了nty_schedule_run()这个函数进行协程的调度，应依然是处在一个Linux中表示的一个线程当中，即对于Linux内核当中这些东西都是在一个调度单位当中，只是你在自己的一个线程里面实现了自己的调度策略去实现协程（其实就是一段函数而已）的调度。

看到这里应该抛出几个问题就是：

1. 协程调度器的触发机制是什么，这个很关键，因为只有理解了触发机制，才能明白这个协程是怎么调度的？
2. 既然是在一个线程里面，那么一个协程阻塞了不就都阻塞了吗，调度器还有什么用，怎么实现协程切换啊？之所以会有这个疑问是因为对比Linux内核中。我们假设只有一个cpu，有多个线程跑在这个cpu上，但是有一个线程阻塞了，我们知道，这种情况别的线程是不会阻塞的，这是因为内核的调度器的触发机制中有一个是时钟中断，所以当时钟中断到达的时候，调度器就能够执行，自然能够切换到其他线程中去执行。这也是第一个问题比较重要的理由。

针对以上两个问题，先给出我在看源码的时候的一个简单的解释：

1. 触发调度这个协程调度策略的机制就是通过主动调用resume/yield/switch这些切换协程的函数
2. 如果一个协程真的阻塞了，针对这个项目来说那确实会导致所有协程都阻塞，因为调度器没法执行。（之所以说是这个项目，是因为我觉得这是由调度策略触发机制影响的，即如果有一个协程调度策略触发机制可以在协程阻塞的情况下发生，例如内核或者别的线程参与进来等让调度器得以执行，那么这种情况下即使一个协程阻塞了，其他协程也能够得到执行）

以上都是我的看法，我们进入到代码里面一探虚实。

我们继续看nty_coroutine_create(&co , server , port)创建的协程：server为主要流程

```c
 void server(void *arg) {

	unsigned short port = *(unsigned short *)arg;
	free(arg);

	int fd = nty_socket(AF_INET, SOCK_STREAM, 0);
	if (fd < 0) return ;

	struct sockaddr_in local, remote;
	local.sin_family = AF_INET;
	local.sin_port = htons(port);
	local.sin_addr.s_addr = INADDR_ANY;
	bind(fd, (struct sockaddr*)&local, sizeof(struct sockaddr_in));

	listen(fd, 20);
	printf("listen port : %d\n", port);

	
	struct timeval tv_begin;
	gettimeofday(&tv_begin, NULL);

	while (1) {
		socklen_t len = sizeof(struct sockaddr_in);
		int cli_fd = nty_accept(fd, (struct sockaddr*)&remote, &len);
		if (cli_fd % 1000 == 999) {

			struct timeval tv_cur;
			memcpy(&tv_cur, &tv_begin, sizeof(struct timeval));
			
			gettimeofday(&tv_begin, NULL);
			int time_used = TIME_SUB_MS(tv_begin, tv_cur);
			
			printf("client fd : %d, time_used: %d\n", cli_fd, time_used);
		}
		printf("new client comming\n");

		nty_coroutine *read_co;
		nty_coroutine_create(&read_co, server_reader, &cli_fd);
	}
}
```

我们看到这个不就是我们所熟悉的server的内容，只不过accept()改成了nty_accrpt()，然后创建一个协程去处理它。这个其实就是达到了一个比较优的策略：每一个fd由一个协程处理（对应的是每一个fd被一个线程处理，我们知道，这样的效率在多核中是非常高的，可以做到业务处理的真正并行）

我们知道accept()是阻塞的，那么这里面的nty_accept()是怎么做的呢。可以先带着猜想去看：因为accept()是阻塞的，还记得我们所思考的角度吗：一个线程的视角，那么意味这nty_accept()里面一定不可能直接调用accept()，一定是通过某种方式出发了协程调度器的调度机制从而跳到调度器的流程里面去了。我们一探究竟。

```c
int nty_accept(int fd, struct sockaddr *addr, socklen_t *len) {
  int sockfd = -1;
  int timeout = 1;
  nty_coroutine *co = nty_coroutine_get_sched()->curr_thread;

  while (1) {
    struct pollfd fds;
    fds.fd = fd;
    fds.events = POLLIN | POLLERR | POLLHUP;

    //这里面肯定会调用调度器，指导fds有消息过来才唤醒协程
    nty_poll_inner(&fds, 1, timeout);

    //到这里的时候应该已经接收了
    sockfd = accept(fd, addr, len);
    if (sockfd < 0) {
      if (errno == EAGAIN) {
        continue;
      } else if (errno == ECONNABORTED) {
        printf("accept : ECONNABORTED\n");

      } else if (errno == EMFILE || errno == ENFILE) {
        printf("accept : EMFILE || ENFILE\n");
      }
      return -1;
    } else {
      break;
    }
  }

  int ret = fcntl(sockfd, F_SETFL, O_NONBLOCK);
  if (ret == -1) {
    close(sockfd);
    return -1;
  }
  int reuse = 1;
  setsockopt(fd, SOL_SOCKET, SO_REUSEADDR, (char *)&reuse, sizeof(reuse));

  return sockfd;
}
```

可以看到在accept()之前调用了nty_poll_inner()这个函数。从中我们可以大体猜测在sockfd = accept(fd , addr , len)这条语句调用之前，fd已经有数据或者说有事件到来了,不然就阻塞了呀。所以一定是nty_poll_inner()捣的鬼。我们继续深入看一下nty_poll_inner里面到底做了什么？
```c
static int nty_poll_inner(struct pollfd *fds, nfds_t nfds, int timeout) {

  if (timeout == 0)
  {
    return poll(fds, nfds, timeout);
  }
  if (timeout < 0)
  {
    timeout = INT_MAX;
  }

  //正常情况应该是走这里
  nty_schedule *sched = nty_coroutine_get_sched();
  if (sched == NULL) {
    printf("scheduler not exit!\n");
    return -1;
  }

  nty_coroutine *co = sched->curr_thread;

  int i = 0;
  for (i = 0;i < nfds;i ++) {

    struct epoll_event ev;
    //得到events
    ev.events = nty_pollevent_2epoll(fds[i].events);
    ev.data.fd = fds[i].fd;
    //把这些fd加入到sched的poller里面去，以此来让调度器去判断有没有信息到来
    epoll_ctl(sched->poller_fd, EPOLL_CTL_ADD, fds[i].fd, &ev);

    co->events = fds[i].events;
    nty_schedule_sched_wait(co, fds[i].fd, fds[i].events, timeout);
  }

  //之前的动作都是做一些准备操作，这里才是真正的让出处理流程，转到调度协程去
  nty_coroutine_yield(co);

  for (i = 0;i < nfds;i ++) {

    struct epoll_event ev;
    ev.events = nty_pollevent_2epoll(fds[i].events);
    ev.data.fd = fds[i].fd;
    epoll_ctl(sched->poller_fd, EPOLL_CTL_DEL, fds[i].fd, &ev);

    //从wait等待队列中移除
    nty_schedule_desched_wait(fds[i].fd);
  }

  return nfds;
}

```

我们可以看到其中的一个关键函数nty_coroutine_yield()，这个函数的作用就是用来让出“处理器”，这里其实就是让出在一个线程中的处理流程，转到其它处理流程当中去（这里是跳转到调度器)。这也是这个项目中协程调度器的触发机制-以主动的方式去调用。

那么问题也就来了：
1. 跳转过去之后本流程的服务怎么办，我本来是要监听事件到来然后处理的？
争对这个问题，我们把焦点放在fds上，跟踪它是怎么被处理的。
```c
for(){
	...
	epoll_ctl(sched->poller_fd , EPOLL_CTL_ADD , fds[i].fd ...);
	...
}
`````

关键的地方在这个epoll_ctl(),没错，这里把fd传给了调度器里面拿到poller，即这些fd的事件由调度器去发现。这样似乎就清晰很多了，各个协程把需要等待的事件对应的fd发送给调度器的poller，由它去管理所有的事件，当接收到事件的时候，由调度器决定唤醒哪一个协程，从而切换上下文，跳转到处理流程中去。

所以再来看nty_poll_inner()函数，在nty_coroutine_yield()之前的操作就是做一些为了跳转到调度器流程的准备操作：
1. 把关系的事件发送给调度器
2. nty_schedule_sched_wait()将自己放入相应的队列当中，在本项目中有三个队列：
	1. 等待队列
	2. 就绪队列
	3. 睡眠队列

当调度器的poller监听到时间到来，就会调度本流程，继续从nty_coroutin_yield()往下处理，来到一下片段：
```c
for(){
	...
	nty_schedule_desched_wait();

}

return nfds;
`````

这个也很简单，就是把自己从睡眠或者等待队列中删除（由nty_schedule_desched_wait()来做），把事件从调度器的poller中删除等。最后把事件返回。看到这里，是不是有一种感觉：nty_poll_inner()的功能其实就像epoll_wait()一样，只不过把fds发送给了调度器，由调度器统一管理监听事件，等事件到来之后唤醒该协程，继续往下处理。

回到调用nty_poll_inner()的流程继续看：nty_accept()
```c
int nty_accept(){
	...
	nty_poll_inner();

	sockfd = accept(fd ,...);
	...

	return sockfd;
}
`````

没错，可以自信的调用accept了，因为你知道事件已经到来了，就等着你accept()呢。:smile:不是你在等事件到来了，是事件在等你去处理了哦！

到这里，也证明了我们之前对nty_accept()的猜想。nty_accept()的使命也干完了，它的工作其实也是类似与accept()的，

然后再回到调用nty_accept()的server()函数去看，毕竟做事要有始有终，形成闭环嘛！

```c
void server(){
	...
	while(1){
		...
		int cli_fd = nty_accept();
		...
		nty_coroutin_create(&read_co , server_reader , &cli_fd);
	}
}
`````
没错，在得到cli_fd之后，又创建了一个协程去处理这个fd对应的事件。又回到我们一开始分析创建协程的地方

最后我们回到nty_server.c/main()当中：
```c
int main(){
	...
	for(){
		nty_cortoutine_create();
	}

	nty_schedule_run();

	return 0;
}
`````

最后调用了nty_schedule_run()执行调度器，main从此也可以称为一个协程了，里面跑着nty_schedule_run();

