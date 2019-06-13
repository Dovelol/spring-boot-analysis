# Spring Boot Request Analysis （2.1.4.RELEASE|Tomcat）

## Acceptor类的`run`方法

```java
/**
 * 此方法是Runnable的实现类，所以也就是作为一个任务在线程中执行。当客户端发起一个http请求时，一系列操作
 * 后就会进入这个方法，由socket = endpoint.serverSocketAccept();接收连接，然后传递给名称为Poller
 * 的线程去侦测I/O事件，我的理解就是，Poller线程会一直select，选出内核将数据从网卡拷贝到内核空间的
 * channel，然后交给名称为Catalina-exec的线程去处理。
 */
@Override
public void run() {
	// 创建errorDelay变量，值为0，用处是当获取连接异常的时候线程休眠的时间。
    int errorDelay = 0;

    // Loop until we receive a shutdown command
    // 循环直到接收到关闭的命令
    while (endpoint.isRunning()) {

        // Loop if endpoint is paused
        // 循环判断endpoint是否停顿，为啥会停顿目前不知道。
        while (endpoint.isPaused() && endpoint.isRunning()) {
            // 修改Acceptor的state为paused
            state = AcceptorState.PAUSED;
            try {
                // 线程休眠50毫秒
                Thread.sleep(50);
            } catch (InterruptedException e) {
                // Ignore
            }
        }
		// endpoint已经关闭。
        if (!endpoint.isRunning()) {
            // 退出循环。
            break;
        }
        // 修改Acceptor的state为running。
        state = AcceptorState.RUNNING;

        try {
            //if we have reached max connections, wait
            // 如果设置的maxConnections=-1，那么不受限制，否则创建一个LimitLatch来控制是否达到最大连接数，如果达到就等待释放后唤醒。
            endpoint.countUpOrAwaitConnection();

            // Endpoint might have been paused while waiting for latch
            // If that is the case, don't accept new connections
            // 判断如果endpoint暂停的话就不要新建连接。
            if (endpoint.isPaused()) {
                continue;
            }
			// 创建socket变量。
            U socket = null;
            try {
                // Accept the next incoming connection from the server
                // socket
                // 其实是调用serverSock.accept()来获取新建的连接，如果没有新的连接会一直阻塞在这。
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                // 出现异常会释放连接的锁。
                endpoint.countDownConnection();
                // endpoint还在运行。
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    // 线程休眠，首次异常不休眠，然后50ms，100ms，200ms，最大是1600ms。
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    // 如果endpoint已经关闭，那就退出循环。
                    break;
                }
            }
            // Successful accept, reset the error delay
            // 如果成功获取到新连接，那么将errorDelay重置为0。
            errorDelay = 0;

            // Configure the socket
            // 判断endpoint正常运行并且没有暂停。
            if (endpoint.isRunning() && !endpoint.isPaused()) {
                // setSocketOptions() will hand the socket off to
                // an appropriate processor if successful
                // #1 
                if (!endpoint.setSocketOptions(socket)) {
                    endpoint.closeSocket(socket);
                }
            } else {
                endpoint.destroySocket(socket);
            }
        } catch (Throwable t) {
            ExceptionUtils.handleThrowable(t);
            String msg = sm.getString("endpoint.accept.fail");
            // APR specific.
            // Could push this down but not sure it is worth the trouble.
            if (t instanceof Error) {
                Error e = (Error) t;
                if (e.getError() == 233) {
                    // Not an error on HP-UX so log as a warning
                    // so it can be filtered out on that platform
                    // See bug 50273
                    log.warn(msg, t);
                } else {
                    log.error(msg, t);
                }
            } else {
                log.error(msg, t);
            }
        }
    }
    state = AcceptorState.ENDED;
}

// #1 NioEndpoint#setSocketOptions；
@Override
protected boolean setSocketOptions(SocketChannel socket) {
    // Process the connection
    try {
        //disable blocking, APR style, we are gonna be polling it
        // 设置SocketChannel为阻塞模式。
        socket.configureBlocking(false);
        // 获取socket对象。
        Socket sock = socket.socket();
        //设置socket配置对象。
        socketProperties.setProperties(sock);
		// 从缓存的nioChannels栈队列（LIFO）中获取一个channel对象。
        NioChannel channel = nioChannels.pop();
        // 如果channel对象为空。
        if (channel == null) {
            // 创建SocketBufferHandler类型的对象。
            SocketBufferHandler bufhandler = new SocketBufferHandler(
                socketProperties.getAppReadBufSize(),
                socketProperties.getAppWriteBufSize(),
                socketProperties.getDirectBuffer());
            // 如果使用到ssl安全套接层。
            if (isSSLEnabled()) {
                // 创建SecureNioChannel对象。
                channel = new SecureNioChannel(socket, bufhandler, selectorPool, this);
            } else {
                // 创建NioChannel对象。
                channel = new NioChannel(socket, bufhandler);
            }
        } else {
            // 利用缓存中的channel，设置SocketChannel属性。
            channel.setIOChannel(socket);
            // 重置channel的buffer。
            channel.reset();
        }
        // #1-1 获取一个Poller对象，然后将NioChannel注册到Poller对象上。
        getPoller0().register(channel);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        try {
            log.error(sm.getString("endpoint.socketOptionsError"), t);
        } catch (Throwable tt) {
            ExceptionUtils.handleThrowable(tt);
        }
        // Tell to close the socket
        return false;
    }
    return true;
}

// #1-1 NioEndpoint#register
public void register(final NioChannel socket) {
    //
    socket.setPoller(this);
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    socket.setSocketWrapper(ka);
    ka.setPoller(this);
    ka.setReadTimeout(getConnectionTimeout());
    ka.setWriteTimeout(getConnectionTimeout());
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    ka.setSecure(isSSLEnabled());
    PollerEvent r = eventCache.pop();
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    else r.reset(socket,ka,OP_REGISTER);
    addEvent(r);
}

```





