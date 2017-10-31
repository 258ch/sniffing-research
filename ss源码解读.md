# shadowsocks 源码解读

> 来源：

> +   <https://www.ioiogoo.cn/2016/12/22/shadowsocks%E6%BA%90%E7%A0%81%E8%A7%A3%E8%AF%BB%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%9A%E5%9F%BA%E6%9C%AC%E6%B5%81%E7%A8%8B/>
> +   <https://www.ioiogoo.cn/2016/12/22/shadowsocks%e6%ba%90%e7%a0%81%e8%a7%a3%e8%af%bb%ef%bc%88%e4%ba%8c%ef%bc%89%ef%bc%9ass-server%e5%b7%a5%e4%bd%9c%e6%b5%81%e7%a8%8b/>

shadowsocks作者已经被约谈了，所以关于它的介绍就省略了，懂的人自然懂。

shadowsocks分为客户端和服务端，服务端运行在能浏览墙外网页的服务器，客户端运行在需要翻墙的主机，目前支持Linux、Windows、Android、IOS等主流操作系统。

## shadowsocks运行原理

![](https://www.ioiogoo.cn/wp-content/uploads/2016/12/whats-shadowsocks.png)

运行原理很简单，SS Local和SS Server分别运行在客户端和服务端，SS Local会有一个socket监听本地端口（默认是1080），并会提供 socks v5 协议接口与本地进程建立TCP连接。当本地进程与SS Local建立连接后，发送一个包含目标服务的地址和端口号的请求包给SS Local，即图中1)过程。SS Local 接收到请求包之后，与SS Server建立连接并发送加密后的请求包，即图中2)过程，SS Server接受请求包并解密，请求目标服务网址返回数据，即图中3)、4)过程。之后将返回的数据加密后传回给SS Local，即图中5)过程，再传给本地进程，即图中6)过程。这就是一整个shadowsocks的工作流程。

很多人会疑惑，为什么正常的请求经过GFW之后会被墙，shadowsocks同样会经过GFW是怎么不被墙的呢？
通常情况下一个正常的请求经过GFW之后，GFW通过分析流量特征达到干扰的目的，但是SS经过GFW的数据是加密后的正常TCP包，没有明显特征码所以无法对通讯数据解密，因此不会受到干扰。

## shadowsocks源码

shadowsocks有Python版和Go版，这里只介绍Python版。

```
# shadowsocks源码结构
|...
├── shadowsocks
│   ├── asyncdns.py
│   ├── common.py
│   ├── crypto
│   │   ├── ....
│   ├── daemon.py
│   ├── encrypt.py
│   ├── eventloop.py
│   ├── __init__.py
│   ├── local.py
│   ├── lru_cache.py
│   ├── manager.py
│   ├── server.py
│   ├── shell.py
│   ├── tcprelay.py
│   └── udprelay.py
|...

```

其中主要包括 eventloop.py、tcprelay.py、udprelay.py、asyncdns.py模块。这里主要介绍TCP模式。

程序的入口可以从server.py开始看，主要工作流程是：先读取配置项，对需要监听的端口分别建立tcprelay.TCPRelay和udprelay.UDPRelay，TCPRelay和UDPRelay类会创建一个socket对端口进行监听。然后将这两个对象加入到eventloop.EventLoop里。然后启动eventloop.EventLoop循环，当eventloop.EventLoop接收到事件时，会调用相应的handle_event进行处理。

大致运行流程就是这样，后面还会针对各个模块的具体流程进行详细地分析。

> 参考资料:
> * [写给非专业人士看的 Shadowsocks 简介](http://vc2tea.com/whats-shadowsocks/)
> * [shadowsocks源码分析：ssserver](http://huiliu.github.io/2016/03/19/shadowsocks.html)


[上一节](http://www.jianshu.com/p/61bad43e12a8)中写到shadowsocks由SS Local和SS Server组成，因为工作原理相似，且SS Server方便调试跟踪，所以以SS Server来分析其中具体流程。

运行`ssserver -p 10001 -k pass -m aes-256-cfb`，运行SS Server

*   监听本地10001端口，密码pass，加密方式aes-256-cfb

在server.py的`main()`下设置断点方便调试。

```
def main():
# ....省略配置处理相关代码
    tcp_servers = []
    udp_servers = [] 
    dns_resolver = asyncdns.DNSResolver()
    port_password = config['port_password']
    del config['port_password']
    for port, password in port_password.items():
        a_config = config.copy()
        a_config['server_port'] = int(port)
        a_config['password'] = password
        logging.info("starting server at %s:%d" %
                     (a_config['server'], int(port)))
        实例化一个tcprelay.TCPRelay类放入到tcp_servers
        tcp_servers.append(tcprelay.TCPRelay(a_config, dns_resolver, False))
        udp_servers.append(udprelay.UDPRelay(a_config, dns_resolver, False))
```

程序一开始先实例化一个TCPRelay类，我们看看TCPRelay在初始化的时候干了什么。

```
def __init__(self, config, dns_resolver, is_local, stat_callback=None):
        # .....省略
        # 判断是SS Local还是SS Server
        if is_local:
            listen_addr = config['local_address']
            listen_port = config['local_port']
        else:
            listen_addr = config['server']
            listen_port = config['server_port']
        self._listen_port = listen_port

        addrs = socket.getaddrinfo(listen_addr, listen_port, 0,
                                   socket.SOCK_STREAM, socket.SOL_TCP)
        if len(addrs) == 0:
            raise Exception("can't get addrinfo for %s:%d" %
                            (listen_addr, listen_port))
        af, socktype, proto, canonname, sa = addrs[0]
        # 创建一个socket
        server_socket = socket.socket(af, socktype, proto)
        server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        # 绑定到监听的端口
        server_socket.bind(sa)
        server_socket.setblocking(False)
        if config['fast_open']:
            try:
                server_socket.setsockopt(socket.SOL_TCP, 23, 5)
            except socket.error:
                logging.error('warning: fast open is not available')
                self._config['fast_open'] = False
        # 监听端口
        server_socket.listen(1024)
        self._server_socket = server_socket
```

初始化的时候根据配置项新建一个socket并绑定到端口进行监听，之后进入到`run_server()`函数

```
 def run_server():
        # ......
        try:
            # 实例化一个EventLoop类事件循环
            # 之后将上面初始化的dns、tcp和udp类加入循环中
            loop = eventloop.EventLoop()
            dns_resolver.add_to_loop(loop)
            list(map(lambda s: s.add_to_loop(loop), tcp_servers + udp_servers))

            daemon.set_user(config.get('user', None))
            loop.run()
        # ...
```

再来看看EventLoop初始化干了什么

```
class EventLoop(object):
    def __init__(self):
        # 根据操作系统选择不同的方法，将select、poll都包装成了epoll的用法
        if hasattr(select, 'epoll'):
            self._impl = select.epoll()
            model = 'epoll'
        elif hasattr(select, 'kqueue'):
            self._impl = KqueueLoop()
            model = 'kqueue'
        elif hasattr(select, 'select'):
            self._impl = SelectLoop()
            model = 'select'
        else:
            raise Exception('can not find any available functions in select '
                            'package')
        # 这个很重要，建立一个以fd为key，(f,handler)为值的字典，之后会根据这个映射分发事件到各自的handler
        self._fdmap = {}  # (f, handler)
        self._last_time = time.time()
        self._periodic_callbacks = []
        self._stopping = False
        logging.debug('using event model: %s', model)
```

然后是`add_to_loop()`函数

```
 def add_to_loop(self, loop):
        if self._eventloop:
            raise Exception('already add to loop')
        if self._closed:
            raise Exception('already closed')
        self._eventloop = loop
        # 加入到事件循环中，并监听socket的可读和错误事件
        self._eventloop.add(self._server_socket,
                            eventloop.POLL_IN | eventloop.POLL_ERR, self)
        self._eventloop.add_periodic(self.handle_periodic)
```

在调用`self._eventloop.add()`函数时，会在EventLoop类中建立一个映射

```
 def add(self, f, mode, handler):
        fd = f.fileno()
        # 这个很重要
        self._fdmap[fd] = (f, handler)
        self._impl.register(fd, mode)
```

到这边总结下程序做的工作，初始化一个TCPRelay类监听本地端口，将它加入事件循环，监听socket的可读和错误事件，事件循环中有一个映射，将fd（文件描述符）映射到对应的（socket,handler），以便之后检测到有请求发生的时候，将请求分发给各自的handler处理，事件循环只管监听事件。

接下来进入到核心的`loop.run()`函数中。

```
 def run(self):
        events = []
        while not self._stopping:
            asap = False
            try:
                # 当有新的请求发送到监听的本地端口时，TCPRelay可读
                events = self.poll(TIMEOUT_PRECISION)
            # ......
            for sock, fd, event in events:
                # 查询fd对应的handler，这个handler是最开始初始化的TCPRelay
                handler = self._fdmap.get(fd, None)
                if handler is not None:
                    handler = handler[1]
                    try:
                        直接交给handler处理事件
                        handler.handle_event(sock, fd, event)
                    # ......
```

然后事件循环继续监听事件。
继续看`handler.handle_event()`函数怎么处理请求的。

```
 def handle_event(self, sock, fd, event):
        # 这边注意下
        if sock == self._server_socket:
            try:
                logging.debug('accept')
                # 接受到新请求之后会生成一个conn
                conn = self._server_socket.accept()
                TCPRelayHandler(self, self._fd_to_handlers,
                                self._eventloop, conn[0], self._config,
                                self._dns_resolver, self._is_local)
        else:
            if sock:
                handler = self._fd_to_handlers.get(fd, None)
                if handler:
                    handler.handle_event(sock, event)
```

然后交给TCPRelayHandler处理请求
TCPRelayHandler在初始化的时候会将conn这个socket加入到loop循环中监听可读或者错误事件

`if sock == self._server_socket`判断当前socket是客户端第一次请求连接还是客户端连接之后发送的请求。
如果是客户端第一次连接，调用TCPRelayHandler，会将新的socket加入事件循环，并且在TCPRelay._fd_to_handlers添加映射，这个等下再讲用途。
如果是客户端在连接之后发来的请求的话，会在`self._fd_to_handlers.get()`查找handler，也就是客户端第一次连接创建的映射，此时handler是TCPRelayHandler，进入`handle_event()`函数后，这时候如果请求正常的话会进入到`self._on_local_read()`函数

```
 def _on_local_read(self):
        # ......
        data = None
        try:
            # 读取请求内容
            data = self._local_sock.recv(BUF_SIZE)
        # ......
        if not is_local:
            # 将请求解密
            data = self._encryptor.decrypt(data)
        # ......
            self._handle_stage_addr(data)
```

之后就是TCPRelayHandler在负责处理数据了。

所以处理的流程大致理清了，我画了一张流程图，更好地理清思路。

![](http://upload-images.jianshu.io/upload_images/3175287-5648410ac3584682.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 总结

shadowsocks原理并不难，主要是程序一直处在EventLoop的循环中，需要在循环里面观察各个模块的运行情况，而且SS Server和SS Local共用同一个TCPRelay，所以就增加了代码的复杂度，只要耐心看下去，摸清楚哪个模块干了些什么事之后就简单了。