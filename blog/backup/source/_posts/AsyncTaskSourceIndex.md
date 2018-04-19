---
title: AsyncTask源码解析
date: 2018-04-19 22:04:28
tags: android
---



> 这是AsyncTask源码的简单分析，主要基于《Android开发艺术探索》一书的内容。



AsyncTask是Android中多线程处理方式之一（其余为1.HandlerThread、2.IntentService以及普通的线程Thread）。

AsyncTask本质是线程池和Handler的包装类，适合实时更新后台任务进度的工作，特别耗时的工作应当交给线程池处理。

AsyncTask常用方法：

* onPreExecute()
* doInBackground()
* onProgressUpdate()
* onPostExecute()

AsyncTask有一下限制：

1. AsyncTask对象必须在主线程（UI线程，下同）创建
2. AsyncTask的execute()必须在主线程调用，且只能被调用一次
3. 不能**直接调用**其4种常用方法（见上）



# 使用

* 继承自AsyncTask，重写对应方法。（注意如果需要更新进度，要在doInBackground()方法中调用publishProgress()方法）
* 在**UI线程** 实例化AsyncTask对象，并调用其execute()方法，传入参数开始执行。

# 概述

**在execute(params)执行后，将参数params传入mWorker.call()方法**

{调用doInBackground(params)后台执行任务，同时通过postResult()方法发送执行结果，由InternalHandler.handleMessage()判断该执行finish()还是onProgressUpdate()}

{将mWorker传入mFuture中作为其callable在runAndReset()方法中执行c.call()方法。}

**通过exec.execute(mFuture)将其压入SerialExecutor线程池中排队，并在THREAD_POOL_EXECUTOR.execute(mActive)真正执行。**

# 代码分析

创建对象（代码有节略，下同）

```java
// Creates a new asynchronous task. This constructor must be invoked on the UI thread.
//注意这里的要求，必须在ui线程
public AsyncTask(@Nullable Looper callbackLooper) {
    mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
        ? getMainHandler()//此处创建InternalHandler用于在UI线程处理消息
        : new Handler(callbackLooper);

    mWorker = new WorkerRunnable<Params, Result>() {
        public Result call() throws Exception {
            try {
                result = doInBackground(mParams);//注意这里会调用doInBackground()方法，后台线程
            } catch (Throwable tr) {
                mCancelled.set(true);
                throw tr;
            } finally {
                postResult(result);//此处发送msg到mHandler那里接受处理
            }
            return result;
        }
    };

    mFuture = new FutureTask<Result>(mWorker) {//这里将mWorker传了进去
        @Override
        protected void done() {
        }
    };
}
```

再看FutureTask

```java
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable; //将mWorker当做了他的callable
    this.state = NEW;       // ensure visibility of callable
}

public void run() {
        try {
            Callable<V> c = callable;
            if (c != null && state == NEW) {
                 result = c.call(); //会在这里回调mWorker的call()方法，即前文所说的doInBackgroud()之类的方法
            }
        } finally {
        }
    }
```

在主线程调用execute()方法

```java
@MainThread
public final AsyncTask<Params, Progress, Result> execute(Params... params) {
    return executeOnExecutor(sDefaultExecutor, params);
}
```

这里调用方法如下：

```java
@MainThread
public final AsyncTask<Params, Progress, Result> executeOnExecutor(Executor exec,
        Params... params) {
    if (mStatus != Status.PENDING) { //此处限制execute()只能被执行一次
        switch (mStatus) {
            case RUNNING:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task is already running.");
            case FINISHED:
                throw new IllegalStateException("Cannot execute task:"
                        + " the task has already been executed "
                        + "(a task can be executed only once)");
        }
    }

    mStatus = Status.RUNNING;

    onPreExecute(); //开始前准备工作

    mWorker.mParams = params; //将参数传入mWorker，并一并传入mFuture中
    exec.execute(mFuture);//将准备好参数、执行时间的mFuture排队放入串行线程池中，等待执行

    return this;
}
```

这里调用了常用方法之一onPreExecute();

mWorker和mFuture的关系前文已经描述了，在看一下exec.execute(mFuture)执行了什么：

exec是execute()传入的，对应于sDefaultExecutor，再查下去

```java
    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
public static final Executor SERIAL_EXECUTOR = new SerialExecutor();
private static volatile Executor sDefaultExecutor = SERIAL_EXECUTOR;
```

再看看SerialExecutor这个线程池

```java
//SerialExecutor主要的作用是将这些线程放到线程池中，并按照串行的顺序依次调用
private static class SerialExecutor implements Executor {
    final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
    Runnable mActive;

    public synchronized void execute(final Runnable r) {
        //offer() Inserts the specified element at the end of this deque.
        //将r插入到线程池中
        mTasks.offer(new Runnable() {
            public void run() {
                try {
                    r.run();
                } finally {
                    //等到当前的执行完了，就调用下一个
                    scheduleNext();
                }
            }
        });
        if (mActive == null) {
            scheduleNext();
        }
    }

    protected synchronized void scheduleNext() {
        if ((mActive = mTasks.poll()) != null) {
            THREAD_POOL_EXECUTOR.execute(mActive);//在这里面才是真正的执行线程的内容
        }
    }
}
```

再仔细看一下THREAD_POOL_EXECUTOR

```java
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
        CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
        sPoolWorkQueue, sThreadFactory);
threadPoolExecutor.allowCoreThreadTimeOut(true);
THREAD_POOL_EXECUTOR = threadPoolExecutor;

 //Executes the given task sometime in the future.  The task
 //may execute in a new thread or in an existing pooled thread.
ThreadPoolExector.executr()
```

以上介绍了线程和线程池部分的内容，接下来看一下在主线程和后台线程之间是如何依靠handler机制来传递消息的。

关于构造函数，由于我们开发者只能接触到AsyncTask()这个构造函数，所以`mHandler=getMainHandler()`

```java
public AsyncTask() {
    this((Looper) null);
}
//@hide，普通开发者不可见
public AsyncTask(@Nullable Looper callbackLooper) {
        mHandler = callbackLooper == null || callbackLooper == Looper.getMainLooper()
            ? getMainHandler()
            : new Handler(callbackLooper);
            ......
}
    private static Handler getMainHandler() {
        synchronized (AsyncTask.class) {
            if (sHandler == null) {
                sHandler = new InternalHandler(Looper.getMainLooper());
            }
            return sHandler;//sHandler是一个类变量，取的是主线程的looper,所以限制了AsyncTask只能在主线程实例化
        }
    }
```

再看一下InternalHandler类

```java
private static class InternalHandler extends Handler {
    public InternalHandler(Looper looper) {
        super(looper);
    }

    @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
    @Override
    public void handleMessage(Message msg) { //在这里处理后台线程发过来的消息，UI线程
        AsyncTaskResult<?> result = (AsyncTaskResult<?>) msg.obj;
        switch (msg.what) {
            case MESSAGE_POST_RESULT:
                // There is only one result
                result.mTask.finish(result.mData[0]);
                break;
            case MESSAGE_POST_PROGRESS:
                result.mTask.onProgressUpdate(result.mData);
                break;
        }
    }
}
```

例如，在doInBackground()方法中可以使用publishProgress()在后台更新进度，即是使用了handler发送消息。

```java
@WorkerThread
protected final void publishProgress(Progress... values) {
    if (!isCancelled()) {
        getHandler().obtainMessage(MESSAGE_POST_PROGRESS,
                new AsyncTaskResult<Progress>(this, values)).sendToTarget();
    }
}
```

最后，AsyncTask的finish()

```java
private void finish(Result result) {
    //可见，最后会根据情况调用onCancelled()或者onPostExecute()
    if (isCancelled()) {
        onCancelled(result);
    } else {
        onPostExecute(result);
    }
    mStatus = Status.FINISHED;
}
```