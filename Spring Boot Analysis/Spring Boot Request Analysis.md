# Spring Boot Request Analysis （2.1.4.RELEASE|Tomcat|NIO）

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
            // 退 出循环。
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
                // #1 创建一个NioChannel对象处理连接。
                if (!endpoint.setSocketOptions(socket)) {
                    // 出现异常就关闭连接。
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
    // 设置Poller对象。
    socket.setPoller(this);
    // 创建一个包装类对象，根据不同的I/O模型有3种wrapper，作用是read请求信息。
    NioSocketWrapper ka = new NioSocketWrapper(socket, NioEndpoint.this);
    // socket设置wrapper对象。
    socket.setSocketWrapper(ka);
    // wrapper设置对应的poller对象，这样相当于把poller绑在了wrapper上。
    ka.setPoller(this);
    // 设置read超时时间。
    ka.setReadTimeout(getConnectionTimeout());
    // 设置write的超时时间。
    ka.setWriteTimeout(getConnectionTimeout());
    // 设置最大的长连接个数。
    ka.setKeepAliveLeft(NioEndpoint.this.getMaxKeepAliveRequests());
    // 设置是否启用ssl。
    ka.setSecure(isSSLEnabled());
    // 从SynchronizedStack<PollerEvent>从取出一个PollerEvent对象，目的是做了一个缓存，这样减少了创建对象的开销，每个Poller对象维护一个events队列。
    PollerEvent r = eventCache.pop();
    // 设置监听的事件类型为OP_READ。
    ka.interestOps(SelectionKey.OP_READ);//this is what OP_REGISTER turns into.
    // 如果没有从缓存中取到PollerEvent对象，那么就创建一个。
    if ( r==null) r = new PollerEvent(socket,ka,OP_REGISTER);
    // 如果pop获取到了PollerEvent对象，那么就重置里面的属性，这样才可以重新使用。
    else r.reset(socket,ka,OP_REGISTER);
    // 添加到SynchronizedQueue<PollerEvent>中去，
    addEvent(r);
}
```



## Poller类的`run`方法

```java
@Override
public void run() {
    // Loop until destroy() is called
    while (true) {
		// 设置hasEvents标识属性。
        boolean hasEvents = false;

        try {
            // 如果没有关闭。
            if (!close) {
                // #1 从队列中获取一个PollerEvent对象，表明有新的连接待处理。
                hasEvents = events();
                // 说明同时有好几个PollerEvent添加到队列中。
                if (wakeupCounter.getAndSet(-1) > 0) {
                    //if we are here, means we have other stuff to do
                    //do a non blocking select
                    // 非阻塞的select，如果没有channel准备好，那就返回0。
                    keyCount = selector.selectNow();
                } else {
                    // 阻塞的select，等待时间为selectorTimeout。
                    keyCount = selector.select(selectorTimeout);
                }
                // 设置唤醒数量为0。
                wakeupCounter.set(0);
            }
            // 如果关闭的话。
            if (close) {
                // 调用events关闭当前连接。
                events();
                // 关闭连接的后续处理。
                timeout(0, false);
                try {
                    // 关闭选择器。
                    selector.close();
                } catch (IOException ioe) {
                    log.error(sm.getString("endpoint.nio.selectorCloseFail"), ioe);
                }
                break;
            }
        } catch (Throwable x) {
            ExceptionUtils.handleThrowable(x);
            log.error(sm.getString("endpoint.nio.selectorLoopError"), x);
            continue;
        }
        //either we timed out or we woke up, process events first
        // 如果keyCount等于0，hasEvents和events()返回值做异或操作后赋值给hasEvents。
        if ( keyCount == 0 ) hasEvents = (hasEvents | events());
		// 获取selector中已经准备好的数据。
        Iterator<SelectionKey> iterator =
            keyCount > 0 ? selector.selectedKeys().iterator() : null;
        // Walk through the collection of ready keys and dispatch
        // any active event.
        // 如果iterator不为空且有值。
        while (iterator != null && iterator.hasNext()) {
            // 获取SelectionKey对象。
            SelectionKey sk = iterator.next();
            // 从SelectionKey中获取注册的NioSocketWrapper对象。
            NioSocketWrapper attachment = (NioSocketWrapper)sk.attachment();
            // Attachment may be null if another thread has called
            // cancelledKey()
            // 如果NioSocketWrapper对象为空。
            if (attachment == null) {
                // 从迭代器中删除。
                iterator.remove();
            } else {
                // 从迭代器中删除。
                iterator.remove();
                // #2 处理请求。
                processKey(sk, attachment);
            }
        }//while

        //process timeouts
        timeout(keyCount,hasEvents);
    }//while

    getStopLatch().countDown();
}

// #1 NioEndpoint#PollerEvent#events
public boolean events() {
    // 设置返回结果为false。
    boolean result = false;
	// 创建PollerEvent变量。
    PollerEvent pe = null;
    // 循环条件，events中有
    for (int i = 0, size = events.size(); i < size && (pe = events.poll()) != null; i++ ) {
        // 设置返回结果为true。
        result = true;
        try {
            // #1-1 调用PollerEvent对象的run方法。
            pe.run();
            // 重置当前PollerEvent对象。
            pe.reset();
            // 如果正在运行并且没有暂停。
            if (running && !paused) {
                // 把PollerEvent对象重新添加到eventCache中，以便下个请求使用。
                eventCache.push(pe);
            }
        } catch ( Throwable x ) {
            log.error(sm.getString("endpoint.nio.pollerEventError"), x);
        }
    }
	// 返回结果。
    return result;
}

// #1-1 NioEndpoint#PollerEvent#run
@Override
public void run() {
    // 如果interestOps等于OP_REGISTER，这个值在Poller.register的时候设置的就是OP_REGISTER
    if (interestOps == OP_REGISTER) {
        try {
            // 获取NioChannel中的SocketChannel类型的对象，然后调用register方法，目前个人理解是将SocketChannel类型的对象和socketWrapper一起绑定到selector上，这样selector就可以获取到对应的channel。
            socket.getIOChannel().register(
                socket.getPoller().getSelector(), SelectionKey.OP_READ, socketWrapper);
        } catch (Exception x) {
            log.error(sm.getString("endpoint.nio.registerFail"), x);
        }
    } else {
        // 从NioChannel中获取SelectionKey。
        final SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());
        try {
            // 如果key是空的。
            if (key == null) {
                // The key was cancelled (e.g. due to socket closure)
                // and removed from the selector while it was being
                // processed. Count down the connections at this point
                // since it won't have been counted down when the socket
                // closed.
                // 获取到NioEndpoint对象，然后进行LimitLatch的减1操作。
                socket.socketWrapper.getEndpoint().countDownConnection();
                // 设置close对象为true。
                ((NioSocketWrapper) socket.socketWrapper).closed = true;
            } else {
                // 从SelectionKey中获取NioSocketWrapper对象。
                final NioSocketWrapper socketWrapper = (NioSocketWrapper) key.attachment();
                // 如果socketWrapper不是空。
                if (socketWrapper != null) {
                    //we are registering the key to start with, reset the fairness counter.
                    // 异或操作。
                    int ops = key.interestOps() | interestOps;
                    // 设置新的interestOps属性的值。
                    socketWrapper.interestOps(ops);
                    // SelectionKey设置新的感兴趣的操作。
                    key.interestOps(ops);
                } else {
                    // 关闭连接，并且回收processor对象。
                    socket.getPoller().cancelledKey(key);
                }
            }
        } catch (CancelledKeyException ckx) {
            try {
                // 关闭连接，并且回收processor对象。
                socket.getPoller().cancelledKey(key);
            } catch (Exception ignore) {}
        }
    }
}

// #2 NioEndpoint#Poller#processKey
protected void processKey(SelectionKey sk, NioSocketWrapper attachment) {
    try {
        // 连接是否关闭。
        if ( close ) {
            // 取消操作。
            cancelledKey(sk);
        } else if ( sk.isValid() && attachment != null ) {
            // 如果SelectionKey是可读或者可写的。
            if (sk.isReadable() || sk.isWritable() ) {
                // 获取sendfileData的值，判断是否不为空。
                if ( attachment.getSendfileData() != null ) {
                    // 没看懂。
                    processSendfile(sk,attachment, false);
                } else {
                    // 没看懂是干啥的，设置了一下sk和attachment的interestOps值。
                    unreg(sk, attachment, sk.readyOps());
                    // 创建closeSocket变量。
                    boolean closeSocket = false;
                    // Read goes before write
                    // 如果SelectionKey是可读的。
                    if (sk.isReadable()) {
                        // 创建或者获取缓存的SocketProcessorBase类对象，然后用catalina-exec线程池去执行任务，也就是说这个时候把任务交给线程池去完成整个后续的请求处理。
                        if (!processSocket(attachment, SocketEvent.OPEN_READ, true)) {
                            // 出现异常，设置closeSocket为true。
                            closeSocket = true;
                        }
                    }
                    // 如果closeSocket不为false并且sk是可写的。
                    if (!closeSocket && sk.isWritable()) {
                        // 创建或者获取缓存的SocketProcessorBase类对象，然后用catalina-exec线程池去执行任务。
                        if (!processSocket(attachment, SocketEvent.OPEN_WRITE, true)) {
                            // 出现异常，设置closeSocket为true。
                            closeSocket = true;
                        }
                    }
                    if (closeSocket) {
                        cancelledKey(sk);
                    }
                }
            }
        } else {
            //invalid key
            cancelledKey(sk);
        }
    } catch ( CancelledKeyException ckx ) {
        cancelledKey(sk);
    } catch (Throwable t) {
        ExceptionUtils.handleThrowable(t);
        log.error(sm.getString("endpoint.nio.keyProcessingError"), t);
    }
}

// SocketProcessorBase#run 这个就是exec线程执行的任务。
@Override
public final void run() {
    synchronized (socketWrapper) {
        // It is possible that processing may be triggered for read and
        // write at the same time. The sync above makes sure that processing
        // does not occur in parallel. The test below ensures that if the
        // first event to be processed results in the socket being closed,
        // the subsequent events are not processed.
        // 判断socket是否关闭
        if (socketWrapper.isClosed()) {
            // 直接返回
            return;
        }
        // 调用不同Endpoint的doRun方法。
        doRun();
    }
}

@Override
protected void doRun() {
    // 从wrapper中获取到NioChannel。
    NioChannel socket = socketWrapper.getSocket();
    // 1.获取SocketChannel；2.从SocketChannel中遍历找到对应的selector和Poller中selector一样的SelectionKey。每个Channel向Selector注册时,都将会创建一个selectionKey
    SelectionKey key = socket.getIOChannel().keyFor(socket.getPoller().getSelector());

    try {
        // 握手变量？？
        int handshake = -1;

        try {
            // 如果key不为空。
            if (key != null) {
                // 
                if (socket.isHandshakeComplete()) {
                    // No TLS handshaking required. Let the handler
                    // process this socket / event combination.
                    handshake = 0;
                } else if (event == SocketEvent.STOP || event == SocketEvent.DISCONNECT ||
                           event == SocketEvent.ERROR) {
                    // Unable to complete the TLS handshake. Treat it as
                    // if the handshake failed.
                    handshake = -1;
                } else {
                    handshake = socket.handshake(key.isReadable(), key.isWritable());
                    // The handshake process reads/writes from/to the
                    // socket. status may therefore be OPEN_WRITE once
                    // the handshake completes. However, the handshake
                    // happens when the socket is opened so the status
                    // must always be OPEN_READ after it completes. It
                    // is OK to always set this as it is only used if
                    // the handshake completes.
                    event = SocketEvent.OPEN_READ;
                }
            }
        } catch (IOException x) {
            handshake = -1;
            if (log.isDebugEnabled()) log.debug("Error during SSL handshake",x);
        } catch (CancelledKeyException ckx) {
            handshake = -1;
        }
        if (handshake == 0) {
            SocketState state = SocketState.OPEN;
            // Process the request from this socket
            if (event == null) {
                state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
            } else {
                state = getHandler().process(socketWrapper, event);
            }
            if (state == SocketState.CLOSED) {
                close(socket, key);
            }
        } else if (handshake == -1 ) {
            close(socket, key);
        } else if (handshake == SelectionKey.OP_READ){
            socketWrapper.registerReadInterest();
        } else if (handshake == SelectionKey.OP_WRITE){
            socketWrapper.registerWriteInterest();
        }
    } catch (CancelledKeyException cx) {
        socket.getPoller().cancelledKey(key);
    } catch (VirtualMachineError vme) {
        ExceptionUtils.handleThrowable(vme);
    } catch (Throwable t) {
        log.error(sm.getString("endpoint.processing.fail"), t);
        socket.getPoller().cancelledKey(key);
    } finally {
        socketWrapper = null;
        event = null;
        //return to cache
        if (running && !paused) {
            processorCache.push(this);
        }
    }
}
}


```



