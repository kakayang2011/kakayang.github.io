---
layout:     post
title:      深入理解AsyncTask
subtitle:   AsyncTask的实现原理
date:       2018-07-06
author:     BY
header-img: img/post-bg-ios9-web.jpg
catalog: true
tags:
    - Android 知识总结
    - 多线程
    - 源码解析
---
# 前言

>目前我们在项目中的异步操作大多是使用RxJava来实现。但是仍然有必要理解AsyncTask的实现原理及优缺点。

# 常见操作方法介绍

#### 方法
1. execute
2. executeOnExecutor
3. cancel

#### 回调

1. onPreExecute：调用execute方法后立即执行，一般用于显示进度条。在UI线程执行
2. doInBackground：用于做耗时操作，在子线程执行
3. onProgressUpdate：用于在耗时操作过程中改变进度条，由publishProgress方法触发，在UI线程中执行
4. onPostExecute：用于显示耗时操作的结果，关闭进度条，在UI线程中执行
5. onCancelled：在用户调用AsyncTask对象的cancel方法后触发，一般用于关闭进度条，在UI线程中调用
#### 操作思想
AsyncTask是对Handler和线程池的良好封装

#### 源码解析
execute方法
```
executeOnExecutor(sDefaultExecutor, params)
```
executeOnExecutor方法：调用了onPreExecute方法，设置了mWorker的参数，然后通过线程池exec执行了mFuture。
```
        if (mStatus != Status.PENDING) {
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

        onPreExecute();

        mWorker.mParams = params;
        exec.execute(mFuture);
```


mWorker：调用了doInBackground方法，并传入了参数
```
        mWorker = new WorkerRunnable<Params, Result>() {
            public Result call() throws Exception {
                mTaskInvoked.set(true);
                Result result = null;
                try {
                    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                    //noinspection unchecked
                    result = doInBackground(mParams);
                    Binder.flushPendingCommands();
                } catch (Throwable tr) {
                    mCancelled.set(true);
                    throw tr;
                } finally {
                    postResult(result);
                }
                return result;
            }
        };
```
mFuture：在done方法中调用了postResultIfNotInvoked方法
```
       mFuture = new FutureTask<Result>(mWorker) {
            @Override
            protected void done() {
                try {
                    postResultIfNotInvoked(get());
                } catch (InterruptedException e) {
                    android.util.Log.w(LOG_TAG, e);
                } catch (ExecutionException e) {
                    throw new RuntimeException("An error occurred while executing doInBackground()",
                            e.getCause());
                } catch (CancellationException e) {
                    postResultIfNotInvoked(null);
                }
            }
        };
```
postResult方法：将MESSAGE_POST_RESULT传递给Handler
```
    private void postResultIfNotInvoked(Result result) {
        final boolean wasTaskInvoked = mTaskInvoked.get();
        if (!wasTaskInvoked) {
            postResult(result);
        }
    }

    private Result postResult(Result result) {
            @SuppressWarnings("unchecked")
            Message message = getHandler().obtainMessage(MESSAGE_POST_RESULT,
                    new AsyncTaskResult<Result>(this, result));
            message.sendToTarget();
            return result;
    }
```
Handler：分别调用onProgressUpdate方法和onPostExecute方法
```
private static class InternalHandler extends Handler {
        public InternalHandler(Looper looper) {
            super(looper);
        }

        @SuppressWarnings({"unchecked", "RawUseOfParameterizedType"})
        @Override
        public void handleMessage(Message msg) {
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
线程池
```
    private static final ThreadFactory sThreadFactory = new ThreadFactory() {
        private final AtomicInteger mCount = new AtomicInteger(1);

        public Thread newThread(Runnable r) {
            return new Thread(r, "AsyncTask #" + mCount.getAndIncrement());
        }
    };

    private static final BlockingQueue<Runnable> sPoolWorkQueue =
            new LinkedBlockingQueue<Runnable>(128);

    /**
     * An {@link Executor} that can be used to execute tasks in parallel.
     */
    public static final Executor THREAD_POOL_EXECUTOR;

    static {
        ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
                CORE_POOL_SIZE, MAXIMUM_POOL_SIZE, KEEP_ALIVE_SECONDS, TimeUnit.SECONDS,
                sPoolWorkQueue, sThreadFactory);
        threadPoolExecutor.allowCoreThreadTimeOut(true);
        THREAD_POOL_EXECUTOR = threadPoolExecutor;
    }

    /**
     * An {@link Executor} that executes tasks one at a time in serial
     * order.  This serialization is global to a particular process.
     */
    public static final Executor SERIAL_EXECUTOR = new SerialExecutor();

    private static class SerialExecutor implements Executor {
        final ArrayDeque<Runnable> mTasks = new ArrayDeque<Runnable>();
        Runnable mActive;

        public synchronized void execute(final Runnable r) {
            mTasks.offer(new Runnable() {
                public void run() {
                    try {
                        r.run();
                    } finally {
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
                THREAD_POOL_EXECUTOR.execute(mActive);
            }
        }
    }
```
#### 注意点
1. 只能在UI线程中创建AsyncTask对象和调用execute方法，并且execute方法只能调用一次
2. Android 3.0之前，AsyncTask是并行执行的。而在Android 3.0之后，AsyncTask是串行执行的
3. AsyncTask容易造成内存泄露。因此必须在Activity销毁前调用cancel方法。并且需要注意，如果在doInBackground方法中有一些不可中断的操作，比如BitmapFactory.decodeStream()，那么就无法真正的cancel任务，操作仍将继续。
4. 如果AsyncTask并未声明为静态内部类，那么他就会持有Activity的引用，也会造成内存泄露

