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
	// 创建errorDelay变量，值为0。
    int errorDelay = 0;

    // Loop until we receive a shutdown command
    // 循环直到接收到关闭的命令
    while (endpoint.isRunning()) {

        // Loop if endpoint is paused
        while (endpoint.isPaused() && endpoint.isRunning()) {
            state = AcceptorState.PAUSED;
            try {
                Thread.sleep(50);
            } catch (InterruptedException e) {
                // Ignore
            }
        }

        if (!endpoint.isRunning()) {
            break;
        }
        state = AcceptorState.RUNNING;

        try {
            //if we have reached max connections, wait
            endpoint.countUpOrAwaitConnection();

            // Endpoint might have been paused while waiting for latch
            // If that is the case, don't accept new connections
            if (endpoint.isPaused()) {
                continue;
            }

            U socket = null;
            try {
                // Accept the next incoming connection from the server
                // socket
                socket = endpoint.serverSocketAccept();
            } catch (Exception ioe) {
                // We didn't get a socket
                endpoint.countDownConnection();
                if (endpoint.isRunning()) {
                    // Introduce delay if necessary
                    errorDelay = handleExceptionWithDelay(errorDelay);
                    // re-throw
                    throw ioe;
                } else {
                    break;
                }
            }
            // Successful accept, reset the error delay
            errorDelay = 0;

            // Configure the socket
            if (endpoint.isRunning() && !endpoint.isPaused()) {
                // setSocketOptions() will hand the socket off to
                // an appropriate processor if successful
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
```





