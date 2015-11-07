##StatefulRunnable
&#8195;StatefulRunnable使用原子操作的AtomicInteger来标记Runnable状态，StatefulRunnable在对象实例化时为STATE_CREATED状态，在开始运行时会被设置为STATE_STARTED状态，而当Runnable还没有开始运行时，可以调用cancel()来终止任务的进行。
```
  protected static final int STATE_CREATED = 0;
  protected static final int STATE_STARTED = 1;
  protected static final int STATE_CANCELLED = 2;
  protected static final int STATE_FINISHED = 3;
  protected static final int STATE_FAILED = 4;

  protected final AtomicInteger mState;
```
&#8195;**++StatefulRunnable为了更好地实现(任务执行)结果的过程与结果处理之间的解耦，设计了getResult()、onSuccess()、onFailure()、onCancellation()4个不同的方法来进行处理++**。我们从run()方法的实现来看一下这几个方法的使用：
```
  public final void run() {
    if (!mState.compareAndSet(STATE_CREATED, STATE_STARTED)) {
      return;
    }
    T result;
    try {
      result = getResult();
    } catch (Exception e) {
      mState.set(STATE_FAILED);
      onFailure(e);
      return;
    }

    mState.set(STATE_FINISHED);
    try {
      onSuccess(result);
    } finally {
      disposeResult(result);
    }
  }
```
&#8195;**++getResult()由StatefulRunnable的具体实现类来实现，用来封装任务执行过程并返回计算结果。当成功获取任务执行结果时，将调用onSuccess()方法进行处理，若任务失败则调用onFailure()进行处理，disposeResult()用来对结果进行一些后处理工作++**
&#8195;**StatefulRunnable的任务执行框架不仅实现了任务执行与结果处理的解耦，而且通过原子操作的状态设置来实现轻量级任务同步，使得任务只会被执行一次。**