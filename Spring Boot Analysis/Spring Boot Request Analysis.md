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
                // 握手是否完整。
                if (socket.isHandshakeComplete()) {
                    // No TLS handshaking required. Let the handler
                    // process this socket / event combination.
                    // 设置handshake=0。
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
            // 设置state为Open。
            SocketState state = SocketState.OPEN;
            // Process the request from this socket
            // 如果event为空。
            if (event == null) {
                state = getHandler().process(socketWrapper, SocketEvent.OPEN_READ);
            } else {
                // #1 先获取ConnectionHandler，然后调用process方法处理请求。
                state = getHandler().process(socketWrapper, event);
            }
            // 如果state状态是closed。
            if (state == SocketState.CLOSED) {
                // 关闭连接。
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

// #1 AbstractProtocol#ConnectionHandler#process
@Override
public SocketState process(SocketWrapperBase<S> wrapper, SocketEvent status) {
    // 输出log。
    if (getLog().isDebugEnabled()) {
        getLog().debug(sm.getString("abstractConnectionHandler.process",
                                    wrapper.getSocket(), status));
    }
    // 如果wrapper是空。
    if (wrapper == null) {
        // Nothing to do. Socket has been closed.
        // 返回closed状态。
        return SocketState.CLOSED;
    }
	// 获取NioChannel类型的对象。
    S socket = wrapper.getSocket();
	// 从连接Map中获取Processor类型的对象。
    Processor processor = connections.get(socket);
    // 输出log。
    if (getLog().isDebugEnabled()) {
        getLog().debug(sm.getString("abstractConnectionHandler.connectionsGet",
                                    processor, socket));
    }

    // Async timeouts are calculated on a dedicated thread and then
    // dispatched. Because of delays in the dispatch process, the
    // timeout may no longer be required. Check here and avoid
    // unnecessary processing.
    // 如果状态为超市并且processor为空或者processor不是异步的或者processor的checkAsyncTimeoutGeneration为false。
    if (SocketEvent.TIMEOUT == status && (processor == null ||
                                          !processor.isAsync() || !processor.checkAsyncTimeoutGeneration())) {
        // This is effectively a NO-OP
        // 返回open状态。
        return SocketState.OPEN;
    }
	// processor不为空。
    if (processor != null) {
        // Make sure an async timeout doesn't fire
        // 从waitingProcessors集合中删除processor。
        getProtocol().removeWaitingProcessor(processor);
    } else if (status == SocketEvent.DISCONNECT || status == SocketEvent.ERROR) {
        // Nothing to do. Endpoint requested a close and there is no
        // longer a processor associated with this socket.
        // 返回closed状态。
        return SocketState.CLOSED;
    }
	// 设置线程变量true。
    ContainerThreadMarker.set();

    try {
        // 如果processor为空。
        if (processor == null) {
            // 获取negotiatedProtocol的值。
            String negotiatedProtocol = wrapper.getNegotiatedProtocol();
            // 如果negotiatedProtocol不为空。
            if (negotiatedProtocol != null) {
                // 这个地方应该是获取http2.0的协议对象。
                UpgradeProtocol upgradeProtocol =
                    getProtocol().getNegotiatedProtocol(negotiatedProtocol);
                // 如果upgradeProtocol不为空。
                if (upgradeProtocol != null) {
                    // 从upgradeProtocol中获取processor。
                    processor = upgradeProtocol.getProcessor(
                        wrapper, getProtocol().getAdapter());
                } else if (negotiatedProtocol.equals("http/1.1")) {
                    // Explicitly negotiated the default protocol.
                    // Obtain a processor below.
                } else {
                    // TODO:
                    // OpenSSL 1.0.2's ALPN callback doesn't support
                    // failing the handshake with an error if no
                    // protocol can be negotiated. Therefore, we need to
                    // fail the connection here. Once this is fixed,
                    // replace the code below with the commented out
                    // block.
                    if (getLog().isDebugEnabled()) {
                        getLog().debug(sm.getString(
                            "abstractConnectionHandler.negotiatedProcessor.fail",
                            negotiatedProtocol));
                    }
                    return SocketState.CLOSED;
                    /*
                             * To replace the code above once OpenSSL 1.1.0 is
                             * used.
                            // Failed to create processor. This is a bug.
                            throw new IllegalStateException(sm.getString(
                                    "abstractConnectionHandler.negotiatedProcessor.fail",
                                    negotiatedProtocol));
                            */
                }
            }
        }
        // 如果processor为空。
        if (processor == null) {
            // 从recycledProcessors缓存中获取一个processor对象。
            processor = recycledProcessors.pop();
            // 输出log。
            if (getLog().isDebugEnabled()) {
                getLog().debug(sm.getString("abstractConnectionHandler.processorPop",
                                            processor));
            }
        }
        // 如果processor仍旧为空。
        if (processor == null) {
            // 创建一个新的processor对象。
            processor = getProtocol().createProcessor();
            // 设置请求头信息。
            register(processor);
        }
		// ssl相关配置。
        processor.setSslSupport(
            wrapper.getSslSupport(getProtocol().getClientCertProvider()));

        // Associate the processor with the connection
        // socket和processor作为k,v添加到connections集合中。
        connections.put(socket, processor);
		// state为closed。
        SocketState state = SocketState.CLOSED;
        do {
            // #1-1
            state = processor.process(wrapper, status);

            if (state == SocketState.UPGRADING) {
                // Get the HTTP upgrade handler
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                // Retrieve leftover input
                ByteBuffer leftOverInput = processor.getLeftoverInput();
                if (upgradeToken == null) {
                    // Assume direct HTTP/2 connection
                    UpgradeProtocol upgradeProtocol = getProtocol().getUpgradeProtocol("h2c");
                    if (upgradeProtocol != null) {
                        processor = upgradeProtocol.getProcessor(
                            wrapper, getProtocol().getAdapter());
                        wrapper.unRead(leftOverInput);
                        // Associate with the processor with the connection
                        connections.put(socket, processor);
                    } else {
                        if (getLog().isDebugEnabled()) {
                            getLog().debug(sm.getString(
                                "abstractConnectionHandler.negotiatedProcessor.fail",
                                "h2c"));
                        }
                        return SocketState.CLOSED;
                    }
                } else {
                    HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                    // Release the Http11 processor to be re-used
                    release(processor);
                    // Create the upgrade processor
                    processor = getProtocol().createUpgradeProcessor(wrapper, upgradeToken);
                    if (getLog().isDebugEnabled()) {
                        getLog().debug(sm.getString("abstractConnectionHandler.upgradeCreate",
                                                    processor, wrapper));
                    }
                    wrapper.unRead(leftOverInput);
                    // Mark the connection as upgraded
                    wrapper.setUpgraded(true);
                    // Associate with the processor with the connection
                    connections.put(socket, processor);
                    // Initialise the upgrade handler (which may trigger
                    // some IO using the new protocol which is why the lines
                    // above are necessary)
                    // This cast should be safe. If it fails the error
                    // handling for the surrounding try/catch will deal with
                    // it.
                    if (upgradeToken.getInstanceManager() == null) {
                        httpUpgradeHandler.init((WebConnection) processor);
                    } else {
                        ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                        try {
                            httpUpgradeHandler.init((WebConnection) processor);
                        } finally {
                            upgradeToken.getContextBind().unbind(false, oldCL);
                        }
                    }
                }
            }
        } while ( state == SocketState.UPGRADING);

        if (state == SocketState.LONG) {
            // In the middle of processing a request/response. Keep the
            // socket associated with the processor. Exact requirements
            // depend on type of long poll
            longPoll(wrapper, processor);
            if (processor.isAsync()) {
                getProtocol().addWaitingProcessor(processor);
            }
        } else if (state == SocketState.OPEN) {
            // In keep-alive but between requests. OK to recycle
            // processor. Continue to poll for the next request.
            connections.remove(socket);
            release(processor);
            wrapper.registerReadInterest();
        } else if (state == SocketState.SENDFILE) {
            // Sendfile in progress. If it fails, the socket will be
            // closed. If it works, the socket either be added to the
            // poller (or equivalent) to await more data or processed
            // if there are any pipe-lined requests remaining.
        } else if (state == SocketState.UPGRADED) {
            // Don't add sockets back to the poller if this was a
            // non-blocking write otherwise the poller may trigger
            // multiple read events which may lead to thread starvation
            // in the connector. The write() method will add this socket
            // to the poller if necessary.
            if (status != SocketEvent.OPEN_WRITE) {
                longPoll(wrapper, processor);
            }
        } else if (state == SocketState.SUSPENDED) {
            // Don't add sockets back to the poller.
            // The resumeProcessing() method will add this socket
            // to the poller.
        } else {
            // Connection closed. OK to recycle the processor. Upgrade
            // processors are not recycled.
            connections.remove(socket);
            if (processor.isUpgrade()) {
                UpgradeToken upgradeToken = processor.getUpgradeToken();
                HttpUpgradeHandler httpUpgradeHandler = upgradeToken.getHttpUpgradeHandler();
                InstanceManager instanceManager = upgradeToken.getInstanceManager();
                if (instanceManager == null) {
                    httpUpgradeHandler.destroy();
                } else {
                    ClassLoader oldCL = upgradeToken.getContextBind().bind(false, null);
                    try {
                        httpUpgradeHandler.destroy();
                    } finally {
                        try {
                            instanceManager.destroyInstance(httpUpgradeHandler);
                        } catch (Throwable e) {
                            ExceptionUtils.handleThrowable(e);
                            getLog().error(sm.getString("abstractConnectionHandler.error"), e);
                        }
                        upgradeToken.getContextBind().unbind(false, oldCL);
                    }
                }
            } else {
                release(processor);
            }
        }
        return state;
    } catch(java.net.SocketException e) {
        // SocketExceptions are normal
        getLog().debug(sm.getString(
            "abstractConnectionHandler.socketexception.debug"), e);
    } catch (java.io.IOException e) {
        // IOExceptions are normal
        getLog().debug(sm.getString(
            "abstractConnectionHandler.ioexception.debug"), e);
    } catch (ProtocolException e) {
        // Protocol exceptions normally mean the client sent invalid or
        // incomplete data.
        getLog().debug(sm.getString(
            "abstractConnectionHandler.protocolexception.debug"), e);
    }
    // Future developers: if you discover any other
    // rare-but-nonfatal exceptions, catch them here, and log as
    // above.
    catch (Throwable e) {
        ExceptionUtils.handleThrowable(e);
        // any other exception or error is odd. Here we log it
        // with "ERROR" level, so it will show up even on
        // less-than-verbose logs.
        getLog().error(sm.getString("abstractConnectionHandler.error"), e);
    } finally {
        ContainerThreadMarker.clear();
    }

    // Make sure socket/processor is removed from the list of current
    // connections
    connections.remove(socket);
    release(processor);
    return SocketState.CLOSED;
}

// #1-1 AbstractProcessorLight#process
@Override
public SocketState process(SocketWrapperBase<?> socketWrapper, SocketEvent status)
    throws IOException {
	// 设置state为closed。
    SocketState state = SocketState.CLOSED;
    // 创建Iterator类型的变量。
    Iterator<DispatchType> dispatches = null;
    do {
        // 如果dispatches不为空
        if (dispatches != null) {
            // 获取一个DispatchType对象。
            DispatchType nextDispatch = dispatches.next();
            // 
            state = dispatch(nextDispatch.getSocketStatus());
        } else if (status == SocketEvent.DISCONNECT) {
            // Do nothing here, just wait for it to get recycled
        } else if (isAsync() || isUpgrade() || state == SocketState.ASYNC_END) {
            state = dispatch(status);
            if (state == SocketState.OPEN) {
                // There may be pipe-lined data to read. If the data isn't
                // processed now, execution will exit this loop and call
                // release() which will recycle the processor (and input
                // buffer) deleting any pipe-lined data. To avoid this,
                // process it now.
                state = service(socketWrapper);
            }
        } else if (status == SocketEvent.OPEN_WRITE) {
            // Extra write event likely after async, ignore
            state = SocketState.LONG;
        } else if (status == SocketEvent.OPEN_READ){
            // 调用service方法处理请求。包含了解析请求头，数据body，并且最后交给DispatcherServlet去处理请求。
            state = service(socketWrapper);
        } else {
            // Default to closing the socket if the SocketEvent passed in
            // is not consistent with the current state of the Processor
            state = SocketState.CLOSED;
        }

        if (getLog().isDebugEnabled()) {
            getLog().debug("Socket: [" + socketWrapper +
                           "], Status in: [" + status +
                           "], State out: [" + state + "]");
        }

        if (state != SocketState.CLOSED && isAsync()) {
            state = asyncPostProcess();
            if (getLog().isDebugEnabled()) {
                getLog().debug("Socket: [" + socketWrapper +
                               "], State after async post processing: [" + state + "]");
            }
        }

        if (dispatches == null || !dispatches.hasNext()) {
            // Only returns non-null iterator if there are
            // dispatches to process.
            dispatches = getIteratorAndClearDispatches();
        }
    } while (state == SocketState.ASYNC_END ||
             dispatches != null && state != SocketState.CLOSED);

    return state;
}


// FrameworkServlet#service
@Override
protected void service(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
	// 获取请求对应方法的httpMethod。
    HttpMethod httpMethod = HttpMethod.resolve(request.getMethod());
    // 如果method是patch或者是null。
    if (httpMethod == HttpMethod.PATCH || httpMethod == null) {
        processRequest(request, response);
    }
    else {
        // #1
        super.service(request, response);
    }
}

// #1 HttpServlet#service
protected void service(HttpServletRequest req, HttpServletResponse resp)
    throws ServletException, IOException {
	// 获取请求的方法类型。
    String method = req.getMethod();
	// 如果是get方法。
    if (method.equals(METHOD_GET)) {
        // 获取最后修改的时间 lastModified是服务器传给客户端的时间。
        long lastModified = getLastModified(req);
        if (lastModified == -1) {
            // servlet doesn't support if-modified-since, no reason
            // to go through further expensive logic
            // #1-1
            doGet(req, resp);
        } else {
            // 获取最后修改的时间 lastModified是客户端传给服务器的时间。
            long ifModifiedSince;
            try {
                // 从请求头中获取最后修改的时间。
                ifModifiedSince = req.getDateHeader(HEADER_IFMODSINCE);
            } catch (IllegalArgumentException iae) {
                // Invalid date header - proceed as if none was set
                // 出异常就把ifModifiedSince改为-1。
                ifModifiedSince = -1;
            }
            // 如果ifModifiedSince小于lastModified。
            if (ifModifiedSince < (lastModified / 1000 * 1000)) {
                // If the servlet mod time is later, call doGet()
                // Round down to the nearest second for a proper compare
                // A ifModifiedSince of -1 will always be less
                maybeSetLastModified(resp, lastModified);
                doGet(req, resp);
            } else {
                resp.setStatus(HttpServletResponse.SC_NOT_MODIFIED);
            }
        }

    } else if (method.equals(METHOD_HEAD)) {
        long lastModified = getLastModified(req);
        maybeSetLastModified(resp, lastModified);
        doHead(req, resp);

    } else if (method.equals(METHOD_POST)) {
        doPost(req, resp);

    } else if (method.equals(METHOD_PUT)) {
        doPut(req, resp);

    } else if (method.equals(METHOD_DELETE)) {
        doDelete(req, resp);

    } else if (method.equals(METHOD_OPTIONS)) {
        doOptions(req,resp);

    } else if (method.equals(METHOD_TRACE)) {
        doTrace(req,resp);

    } else {
        //
        // Note that this means NO servlet supports whatever
        // method was requested, anywhere on this server.
        //

        String errMsg = lStrings.getString("http.method_not_implemented");
        Object[] errArgs = new Object[1];
        errArgs[0] = method;
        errMsg = MessageFormat.format(errMsg, errArgs);

        resp.sendError(HttpServletResponse.SC_NOT_IMPLEMENTED, errMsg);
    }
}

// #1-1 FrameworkServlet#processRequest
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
    throws ServletException, IOException {
	// 获取系统当前时间。
    long startTime = System.currentTimeMillis();
    // 创建Throwable类型的变量。
    Throwable failureCause = null;
	// 获取本地上下文变量。
    LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
    // 构建本地上下文变量。
    LocaleContext localeContext = buildLocaleContext(request);
	// 获取RequestAttributes类型的对象。
    RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
    // 创建ServletRequestAttributes类型的对象。
    ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);
	// 获取异步的WebAsyncManager对象。
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
    // 注册一个拦截器。
    asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());
	// 将localeContext和requestAttributes添加到对应的ContextHolder中。
    initContextHolders(request, localeContext, requestAttributes);

    try {
        // #1-1-1 
        doService(request, response);
    }
    catch (ServletException | IOException ex) {
        failureCause = ex;
        throw ex;
    }
    catch (Throwable ex) {
        failureCause = ex;
        throw new NestedServletException("Request processing failed", ex);
    }

    finally {
        resetContextHolders(request, previousLocaleContext, previousAttributes);
        if (requestAttributes != null) {
            requestAttributes.requestCompleted();
        }
        logResult(request, response, failureCause, asyncManager);
        publishRequestHandledEvent(request, response, startTime, failureCause);
    }
}

// #1-1-1 DispatcherServlet#doService
protected void doService(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 输出日志。
    logRequest(request);

    // Keep a snapshot of the request attributes in case of an include,
    // to be able to restore the original attributes after the include.
    // 创建attributesSnapshot集合变量。
    Map<String, Object> attributesSnapshot = null;
    // request是否包含key为javax.servlet.include.request_uri的属性。
    if (WebUtils.isIncludeRequest(request)) {
        // 创建新的map集合。
        attributesSnapshot = new HashMap<>();
        // 获取request中所有的属性名。
        Enumeration<?> attrNames = request.getAttributeNames();
        while (attrNames.hasMoreElements()) {
            // 遍历获取每个属性名。
            String attrName = (String) attrNames.nextElement();
            if (this.cleanupAfterInclude || attrName.startsWith(DEFAULT_STRATEGIES_PREFIX)) {
                // 将属性添加到attributesSnapshot集合中。
                attributesSnapshot.put(attrName, request.getAttribute(attrName));
            }
        }
    }

    // Make framework objects available to handlers and view objects.
    // 设置属性org.springframework.web.servlet.DispatcherServlet.CONTEXT=ACSWAC
    request.setAttribute(WEB_APPLICATION_CONTEXT_ATTRIBUTE, getWebApplicationContext());
    // 设置属性org.springframework.web.servlet.DispatcherServlet.LOCALE_RESOLVER=AcceptHeaderLocaleResolver
    request.setAttribute(LOCALE_RESOLVER_ATTRIBUTE, this.localeResolver);
    // 设置属性org.springframework.web.servlet.DispatcherServlet.THEME_RESOLVER=FixedThemeResolver
    request.setAttribute(THEME_RESOLVER_ATTRIBUTE, this.themeResolver);
    // 设置属性org.springframework.web.servlet.DispatcherServlet.THEME_SOURCE=ACSWAC
    request.setAttribute(THEME_SOURCE_ATTRIBUTE, getThemeSource());
	// 如果flashMapManager不为null。
    if (this.flashMapManager != null) {
        // 找出符合要求的FlashMap对象。
        FlashMap inputFlashMap = this.flashMapManager.retrieveAndUpdate(request, response);
        if (inputFlashMap != null) {
            // 设置属性org.springframework.web.servlet.DispatcherServlet.INPUT_FLASH_MAP=inputFlashMap
            request.setAttribute(INPUT_FLASH_MAP_ATTRIBUTE, Collections.unmodifiableMap(inputFlashMap));
        }
        // 设置属性org.springframework.web.servlet.DispatcherServlet.OUTPUT_FLASH_MAP=FlashMap
        request.setAttribute(OUTPUT_FLASH_MAP_ATTRIBUTE, new FlashMap());
        // 设置属性org.springframework.web.servlet.DispatcherServlet.FLASH_MAP_MANAGER=flashMapManager
        request.setAttribute(FLASH_MAP_MANAGER_ATTRIBUTE, this.flashMapManager);
    }

    try {
        // #1-1-1-1
        doDispatch(request, response);
    }
    finally {
        if (!WebAsyncUtils.getAsyncManager(request).isConcurrentHandlingStarted()) {
            // Restore the original attribute snapshot, in case of an include.
            if (attributesSnapshot != null) {
                restoreAttributesAfterInclude(request, attributesSnapshot);
            }
        }
    }
}

// #1-1-1-1 DispatcherServlet#doDispatch
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    // 创建HttpServletRequest类型变量。
    HttpServletRequest processedRequest = request;
    // 创建HandlerExecutionChain类型变量。
    HandlerExecutionChain mappedHandler = null;
    // 创建multipartRequestParsed=false。
    boolean multipartRequestParsed = false;
	// 从request的属性中获取WEB_ASYNC_MANAGER的值。
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        // 创建ModelAndView类型对象。
        ModelAndView mv = null;
        // 创建Exception类型对象。
        Exception dispatchException = null;

        try {
            // 检查request的contentType是否是multipart类型的，如果是，需要处理，如果不是，processedRequest=request。
            processedRequest = checkMultipart(request);
            // 判断是否需要做multipart类型的解析。
            multipartRequestParsed = (processedRequest != request);

            // Determine handler for the current request.
            // 获取请求对应的HandlerExecutionChain。
            mappedHandler = getHandler(processedRequest);
            // 如果对应的mappedHandler为空。
            if (mappedHandler == null) {
                // 那么返回404。
                noHandlerFound(processedRequest, response);
                return;
            }

            // Determine handler adapter for the current request.
            // 获取handlerAdapter对象。
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // Process last-modified header, if supported by the handler.
            // 获取request的method。
            String method = request.getMethod();
            // 判断request是否是GET请求。
            boolean isGet = "GET".equals(method);
            // 如果是GET方法或者HEAD方法。
            if (isGet || "HEAD".equals(method)) {
                // 返回-1。
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                // 是否修改过，不知道是什么操作。
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
			// 执行拦截器的preHandle方法，如果不满足条件就返回。
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // Actually invoke the handler.
            // 反射执行controller的方法。
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
			// 是否异步处理。
            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
			// 设置默认的viewName。
            applyDefaultViewName(processedRequest, mv);
            // 执行所有拦截器的postHandle方法。
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            // 赋值dispatchException。
            dispatchException = ex;
        }
        catch (Throwable err) {
            // As of 4.3, we're processing Errors thrown from handler methods as well,
            // making them available for @ExceptionHandler methods and other scenarios.
            // 创建新的Exception类型对象。
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                               new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}

```



