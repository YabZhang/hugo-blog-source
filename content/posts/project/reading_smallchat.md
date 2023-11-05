---
title: "smallchat 源码阅读"
date: 2023-11-04T23:35:15+08:00
draft: false
categories:
    - project
    - reading
tags:
    - C
    - chat
---

`smallchat`[^1] 是 `redis` 作者 `antirez` 所写的一个聊天室的小程序；代码短小精悍，很有意思。据说作者以此例向前端朋友展示系统编程的趣味 😄 [^2]~

这里记录下阅读源码所获。

首先从 `main` 开始:


```c
/* The main() function implements the main chat logic:
 * 1. Accept new clients connections if any.
 * 2. Check if any client sent us some new message.
 * 3. Send the message to all the other clients. */
int main(void) {
    initChat();

    while(1) {
        ...
    }
    return 0;
}
```

根据注释，`main` 函数主要做了三件事：
1. 接受新的客户端连接；
2. 检查是否有客户端发送过来新的消息；
3. 发送消息给其他客户端；

代码首先调用了 `initChat` 完成一些初始化工作；然后程序进入到死循环，作为常驻服务，会不停执行消息逻辑。主题结构大体是这样。

---

那么来看下 `initChat` 做了哪些事情：
    
```c
#define MAX_CLIENTS 1000 // This is actually the higher file descriptor.
#define SERVER_PORT 7711


/* This structure represents a connected client. There is very little
 * info about it: the socket descriptor and the nick name, if set, otherwise
 * the first byte of the nickname is set to 0 if not set.
 * The client can set its nickname with /nick <nickname> command. */
struct client {
    int fd;     // Client socket.
    char *nick; // Nickname of the client.
};

/* This global structure encasulates the global state of the chat. */
struct chatState {
    int serversock;     // Listening server socket.
    int numclients;     // Number of connected clients right now.
    int maxclient;      // The greatest 'clients' slot populated.
    struct client *clients[MAX_CLIENTS]; // Clients are set in the corresponding
                                         // slot of their socket descriptor.
};

struct chatState *Chat; // Initialized at startup.


/* Allocate and init the global stuff. */
void initChat(void) {
    Chat = chatMalloc(sizeof(*Chat));
    memset(Chat,0,sizeof(*Chat));
    /* No clients at startup, of course. */
    Chat->maxclient = -1; // 记录当前最大客户端槽数
    Chat->numclients = 0; // 记录当前的客户端连接数

    /* Create our listening socket, bound to the given port. This
     * is where our clients will connect. */
    Chat->serversock = createTCPServer(SERVER_PORT);
    if (Chat->serversock == -1) {
        perror("Creating listening socket");
        exit(1);
    }
}

void *chatMalloc(size_t size) {
    void *ptr = malloc(size);
    if (ptr == NULL) {
        perror("Out of memory");
        exit(1);
    }
    return ptr;
}

```

`initChat` 注释说是负责分配和初始化全局变量。

顺便提一下， `C` 语言中是需要手动做动态内存管理的，具体是需要申请和回收动态堆内存。如果是常驻进程，申请后不回收会导致内存泄露，严重时会导致程序 OOM 被系统杀死退出；程序的内存会在退出后，由系统回收。

此外在申请到动态内存后需要手动清零初始化后使用（否则会有残留的随机值）。
程序里`malloc` 和 `free` 就是用来做动态内存申请和回收函数, `memset` 用来清零初始化。

程序声明了全局变量 `Chat` 来保存聊天室的全局状态，包括：
1. `serversock` 用来保存服务监听连接, 用来处理新的客户端连接请求；
2. `numclients` 和 `maxclient` 用来记录客户端连接数的状态；
3. `client` 指向一个数组，用来保存所有接入的客户端连接（用 `client` 结构记录）；

程序初始化为 `Chat` 申请了内存，并清零和初始化。通过 `createTCPServer` 创建了服务连接，保存在`serversock` 字段。初始化就完成了。

---

好奇 `createTCPServer` 都做了哪些事？属于偏底层的网络编程范畴了，不亏是 `redis` 的作者。

```c
/* Create a TCP socket lisetning to 'port' ready to accept connections. */
int createTCPServer(int port) {
    int s, yes = 1;
    struct sockaddr_in sa;

    if ((s = socket(AF_INET, SOCK_STREAM, 0)) == -1) return -1;
    setsockopt(s, SOL_SOCKET, SO_REUSEADDR, &yes, sizeof(yes)); // Best effort.

    memset(&sa,0,sizeof(sa));
    sa.sin_family = AF_INET;
    sa.sin_port = htons(port);
    sa.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(s,(struct sockaddr*)&sa,sizeof(sa)) == -1 ||
        listen(s, 511) == -1)
    {
        close(s);
        return -1;
    }
    return s;
}

struct sockaddr_in {
    short            sin_family;   // e.g. AF_INET, AF_INET6
    unsigned short   sin_port;     // e.g. htons(3490)
    struct in_addr   sin_addr;     // see struct in_addr, below
    char             sin_zero[8];  // zero this if you want to, padding for memory alignment
};

```

首先声明了变量，`s` 用于保存服务连接的文件描述符，`yes` 用于传递 `socket` 设置的参数选项。

`socket(AF_INET, SOCK_STREAM, 0)` 创建了一个套接字描述符; 
而 `AF_INET` 类型的 `SOCK_STREAM` 流式套接字类型，是指一种以太网协议族的流式协议类型，也就是 `TCP` 协议网络套接字。其他的类型声明可参照 `socket.h`。

`setsockopt` 设置套接字参数选项，`SOL_SOCKET`常量指明为套接字，`SO_REUSEADDR` 选项表示允许重用本地地址和端口，可以在同一端口上启动多个实例监听。这里的用途是在服务关闭后可以快速重启，避免因本地端口占用导致服务启动失败（因`TCP`连接退出会保持你在 `TIME_WAIT` 一段时间，以确认所有包被接收）。

完成地址参数`sa`的初始化后，调用 `bind` 绑定套接字到本地地址; 绑定成功后，使用`listen`标记为服务端套接字，并开始监听连接请求。其中 `listen` 的第二个参数 `511` 表示连接等待队列(`backlog queue`)的最大连接数，具体可参考 `listen` 的文档。

```
NAME
     listen – listen for connections on a socket

SYNOPSIS
     #include <sys/socket.h>

     int
     listen(int socket, int backlog);

DESCRIPTION
     Creation of socket-based connections requires several operations.  First, a socket is created with socket(2).  Next, a willingness to accept incoming connections and a queue limit for incoming connections are specified with listen().  Finally, the connections are accepted with accept(2).  The listen() call
     applies only to sockets of type SOCK_STREAM.

     The backlog parameter defines the maximum length for the queue of pending connections.  If a connection request arrives with the queue full, the client may receive an error with an indication of ECONNREFUSED.  Alternatively, if the underlying protocol supports retransmission, the request may be ignored so
     that retries may succeed.
```

至此，服务端监听套接字创建和初始化完成，可以开始接受客户端连接请求了。整个初始化 `initChat` 就完成了。

---

接下来看 `main` 函数中如何实现“三件事”的。

参照前文，“三件事”分别为：
1. 接受新的客户端连接；
2. 检查是否有客户端发送过来新的消息；
3. 发送消息给其他客户端；

为了方便地处理网络IO读写一般会做多路复用。基本原理是让系统帮我们监听一组描述符上的读写事件，当有事件发生时，系统会通知我们去处理对应的事件。常见的IO复用处理方式有: `select`, `poll`, `epoll` 等，更多细节推荐《Linux 高性能服务器编程》[^3]。

这里程序中采用 `select` 来实现，但其实 `epoll` 会更高效一些。

```c
while(1) {
        fd_set readfds;
        struct timeval tv;
        int retval;

        FD_ZERO(&readfds);
        /* When we want to be notified by select() that there is
         * activity? If the listening socket has pending clients to accept
         * or if any other client wrote anything. */
        FD_SET(Chat->serversock, &readfds);

        for (int j = 0; j <= Chat->maxclient; j++) {
            if (Chat->clients[j]) FD_SET(j, &readfds);
        }

        /* Set a timeout for select(), see later why this may be useful
         * in the future (not now). */
        tv.tv_sec = 1; // 1 sec timeout
        tv.tv_usec = 0;

        /* Select wants as first argument the maximum file descriptor
         * in use plus one. It can be either one of our clients or the
         * server socket itself. */
        int maxfd = Chat->maxclient;
        if (maxfd < Chat->serversock) maxfd = Chat->serversock;
        retval = select(maxfd+1, &readfds, NULL, NULL, &tv);

        ...

}
```

为了使用`select`, 代码首先声明了 `fd_set` 类型变量 `readfds` 用来保存一组描述符，完成清零(`FD_ZERO(&readfds)`) 后分别把服务套接字和所有客户端套接字注册到 `readfds` 上:

1. `FD_SET(Chat->serversock, &readfds)`
2. `for (int j = 0; j <= Chat->maxclient; j++) { if (Chat->clients[j]) FD_SET(j, &readfds);}`

同时还声明了超时参数`struct timeval tv`，和初始化参数值。
最后调用`select`来监听套接字上的读写事件，直到超时返回。

---

接下来看如何处理新的客户端连接：

```c
/* If the listening socket is "readable", it actually means
    * there are new clients connections pending to accept. */
if (FD_ISSET(Chat->serversock, &readfds)) {
    int fd = acceptClient(Chat->serversock);
    struct client *c = createClient(fd);
    /* Send a welcome message. */
    char *welcome_msg =
        "Welcome to Simple Chat! "
        "Use /nick <nick> to set your nick.\n";
    write(c->fd,welcome_msg,strlen(welcome_msg));
    printf("Connected client fd=%d\n", fd);
}

/* Create a new client bound to 'fd'. This is called when a new client
 * connects. As a side effect updates the global Chat state. */
struct client *createClient(int fd) {
    char nick[32]; // Used to create an initial nick for the user.
    int nicklen = snprintf(nick,sizeof(nick),"user:%d",fd);
    struct client *c = chatMalloc(sizeof(*c));
    socketSetNonBlockNoDelay(fd); // Pretend this will not fail.
    c->fd = fd;
    c->nick = chatMalloc(nicklen+1);
    memcpy(c->nick,nick,nicklen);
    assert(Chat->clients[c->fd] == NULL); // This should be available.
    Chat->clients[c->fd] = c;
    /* We need to update the max client set if needed. */
    if (c->fd > Chat->maxclient) Chat->maxclient = c->fd;
    Chat->numclients++;
    return c;
}

/* Set the specified socket in non-blocking mode, with no delay flag. */
int socketSetNonBlockNoDelay(int fd) {
    int flags, yes = 1;

    /* Set the socket nonblocking.
     * Note that fcntl(2) for F_GETFL and F_SETFL can't be
     * interrupted by a signal. */
    if ((flags = fcntl(fd, F_GETFL)) == -1) return -1;
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) return -1;

    /* This is best-effort. No need to check for errors. */
    setsockopt(fd, IPPROTO_TCP, TCP_NODELAY, &yes, sizeof(yes));
    return 0;
}
```

在 `select` 返回后，通过`FD_ISSET`检查服务套接字是否可读，如果可读则表示有新的客户端连接请求，调用 `acceptClient` 来接受客户端连接。并向新的客户端发送一条欢迎消息。

`createClient` 函数用来创建一个新的客户端连接，传入的参数是客户端套接字描述符，函数会为其创建一个 `client` 结构体，并初始化一些状态参数（如客户端连接数等），最后把新的客户端连接保存到全局变量 `Chat` 中。这里程序采用文件描述符数值作为数组下标来保存客户端连接，这样可以方便的通过文件描述符来查找对应的客户端连接，但是前几位被标准输入输出错误等占用，程序中会先找到当前最大的文件描述符，然后从这个位置开始保存客户端连接。

`socketSetNonBlockNoDelay` 函数用来设置套接字为非阻塞模式。
同时设置 `TCP_NODELAY` 参数，表示禁用 `Nagle` 算法（用于自动拼接小的文本段，以提供网络传输效率），这里是为了尽快送达和避免消息延迟。

---

处理来自客户端的消息:

```c
/* Here for each connected client, check if there are pending
    * data the client sent us. */
char readbuf[256];
for (int j = 0; j <= Chat->maxclient; j++) {
    if (Chat->clients[j] == NULL) continue;
    if (FD_ISSET(j, &readfds)) {
       
       /* Here we just hope that there is a well formed
        * message waiting for us. But it is entirely possible
        * that we read just half a message. In a normal program
        * that is not designed to be that simple, we should try
        * to buffer reads until the end-of-the-line is reached. */
        int nread = read(j,readbuf,sizeof(readbuf)-1);
        ... 
    }
```

这段代码遍历所有客户端连接，检查是否有客户端发送过来的消息。如果有则调用 `read` 读取消息内容; 因为所有的客户端连接都是非阻塞的，所以能够立刻返回结果。但read很可能读取到的是不完整的消息，这里程序简单处理，直接读取一行消息内容(真实的工程中会通过分隔符来判断读取完全，否则会被判定为不完整信息)。

```c
if (nread <= 0) {
    /* Error or short read means that the socket
     * was closed. */
    printf("Disconnected client fd=%d, nick=%s\n",
        j, Chat->clients[j]->nick);
    freeClient(Chat->clients[j]);
} else {
    /* The client sent us a message. We need to
     * relay this message to all the other clients
     * in the chat. */
    struct client *c = Chat->clients[j];
    readbuf[nread] = 0;

    /* If the user message starts with "/", we
     * process it as a client command. So far
     * only the /nick <newnick> command is implemented. */
    if (readbuf[0] == '/') {
        ...
    } else {
    /* Create a message to send everybody (and show
     * on the server console) in the form:
     *   nick> some message. */
    char msg[256];
    int msglen = snprintf(msg, sizeof(msg),
        "%s> %s", c->nick, readbuf);
    printf("%s",msg);

    /* Send it to all the other clients. */
    sendMsgToAllClientsBut(j,msg,msglen);
    }
}
```

这段代码根据`read`返回的结果来进一步处理。首先是判定客户端已关闭（一般需要有专门的客户端关闭指令或者多次心跳保活判定），需要对客户端资源进行释放，以及状态更新。其次是正常读取的客户端消息，这里消息细分为两类，一种是以`/<cmd> content`格式的指令内容；另一种就是全文本消息，需要加上发送人昵称转发给其他所有人。

针对“指令消息”的处理逻辑就是常见的字符串处理：

```c
/* If the user message starts with "/", we
 * process it as a client command. So far
 * only the /nick <newnick> command is implemented. */
if (readbuf[0] == '/') {
    /* Remove any trailing newline. */
    char *p;
    p = strchr(readbuf,'\r'); if (p) *p = 0;
    p = strchr(readbuf,'\n'); if (p) *p = 0;
    /* Check for an argument of the command, after
        * the space. */
    char *arg = strchr(readbuf,' ');
    if (arg) {
        *arg = 0; /* Terminate command name. */
        arg++; /* Argument is 1 byte after the space. */
    }

    if (!strcmp(readbuf,"/nick") && arg) {
        free(c->nick);
        int nicklen = strlen(arg);
        c->nick = chatMalloc(nicklen+1);
        memcpy(c->nick,arg,nicklen+1);
    } else {
        /* Unsupported command. Send an error. */
        char *errmsg = "Unsupported command\n";
        write(c->fd,errmsg,strlen(errmsg));
    }
}
```
根据首位的`/`判定为指令消息后，根据分隔换行标识出消息尾部（并置零）；
然后利用空格区分出指令内容和消息内容，以`0`分隔；
语句`!strcmp(readbuf "/nick) && arg` 判定指令为`/nick`后，把指令内容`arg`指向的字符串拷贝后赋值到客户端的昵称字段中。

而转发消息给其他所有人的逻辑则是遍历所有客户端连接，把消息内容发送给其他所有人：

```c
/* Send the specified string to all connected clients but the one
 * having as socket descriptor 'excluded'. If you want to send something
 * to every client just set excluded to an impossible socket: -1. */
void sendMsgToAllClientsBut(int excluded, char *s, size_t len) {
    for (int j = 0; j <= Chat->maxclient; j++) {
        if (Chat->clients[j] == NULL ||
            Chat->clients[j]->fd == excluded) continue;

        /* Important: we don't do ANY BUFFERING. We just use the kernel
         * socket buffers. If the content does not fit, we don't care.
         * This is needed in order to keep this program simple. */
        write(Chat->clients[j]->fd,s,len);
    }
}
```

具体实现就是遍历所有客户端连接，把消息内容写入到客户端套接字中(这里写入也有可能失败，展示程序处于简单并未处理可能的错误)。其中未赋值的客户端连接 和 指定的排除客户端连接（消息源本人）会被跳过。

至此，整个程序逻辑就完成了。

作为一个展示样例，这端代码展示了如何处理内存管理、网络套接字的使用、多路复用、客户端连接的管理、消息和指令的处理等逻辑。除去注释只有短短两百多行，用来学习和理解服务端编程已经足够了 ———— 麻雀虽小五脏俱全。


[^1]: https://github.com/antirez/smallchat

[^2]: 视频分享 https://www.youtube.com/watch?v=eT02gzeLmF0

[^3]: Linux 高性能服务器编程 https://book.douban.com/subject/24722611/

