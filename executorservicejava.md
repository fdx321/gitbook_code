# ExecutorService**.java**

### **介绍**

这是一种增强型的Executor，提供了接口用于终止任务，提供返回Future的接口，用于异步的控制执行的task.

ExecutorService是能能够被关闭的，一旦被关闭，它就拒绝接受新提交的task. 它提供了两个关闭方法：

* **shutdown\(\)**

  * 在终止之前允许之前提交的任务执行完

* **shutdownNow\(\)**

  * 立即停止，不等待之前任务执行完成


一个不用的ExcutorService应该被停止，避免资源浪费。

**submit** 方法比基本的 **execute\(\)** 提供了更强大的功能，它会创建并返回一个 Future 对象，通过 future, 可以取消任务、异步等待任务执行完成

**invokeAll\(\)** **invokeAny\(\)** 用于批量的执行任务，分别等待全部\/至少一个任务执行完毕后返回结果。

### **使用样例**

一个简单的网络服务端代码样例。

```
class NetworkService implements Runnable {
  private final ServerSocket serverSocket;
  private final ExecutorService pool;

  public NetworkService(int port, int poolSize) throws IOException {
    serverSocket = new ServerSocket(port);
    pool = Executors.newFixedThreadPool(poolSize);
  }

  public void run() { 
    try {
      for (;;) {
        pool.execute(new Handler(serverSocket.accept()));
      }
    } catch (IOException ex) {
      pool.shutdown();
    }
  }
}

class Handler implements Runnable {
  private final Socket socket;
  Handler(Socket socket) { this.socket = socket; }
  public void run() {
    // read and service request on socket
  }
}
```

