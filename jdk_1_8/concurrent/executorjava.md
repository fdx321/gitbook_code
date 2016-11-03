# Executor.java

### **介绍**

```java
public interface Executor {
    void execute(Runnable command);
}

```

一个用来执行task的接口，该task必须实现Runnable接口。通过Executor，可以避免下面这种显示的线程调用。

```java
new Thread(new(RunnableTask())).start();

```

取而代之的是下面这种使用方式：

```java
executor.execute(new RunnableTask1());

```

### **使用方式**

1. Executor接口没有强制要求执行的任务是异步的，可以这样用：

  ```java
  class DirectExecutor implements Executor {
  public void execute(Runnable r) {
  r.run();
  }
  }

  ```

2. 但更典型的用法是在另一个新开的线程中执行任务，达到异步的效果：

  ```java
  class ThreadPerTaskExecutor implements Executor {
  public void execute(Runnable r) {
  new Thread(r).start();
  }
  }

  ```

3. 还有一些更复杂的用法，在executor中控制任务什么时候已怎样的方式被执行


