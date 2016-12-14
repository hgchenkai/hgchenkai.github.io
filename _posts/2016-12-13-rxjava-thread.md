---
layout: post
title:  "RxJava（RxAndroid）线程切换机制"
desc: "RxJava（RxAndroid）线程切换机制"
keywords: "rxjava"
date: 2016-12-13
categories: [Java]
tags: [Rxjava,操作符]
icon: icon-java
---
　　自从项目中使用RxJava以来，可以很方便的切换线程。至于是怎么实现的，一直没有深入的研究过！本篇文章就是分析RxJava的线程模型。
　　
　　**RxJava基本使用**

　　先上一个平时使用RxJava切换线程的例子：
　　

```java
Observable observable = Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("hello");
                subscriber.onCompleted();
            }
        });
        Subscriber subscriber = new Subscriber<String>() {
            @Override
            public void onCompleted() {
                //TODO
            }

            @Override
            public void onError(Throwable e) {
                //TODO
            }

            @Override
            public void onNext(String o) {
                Log.e("RxJava", o);
            }
        };
        observable
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(subscriber);
```
　　如果结合Lamda表达式，代码会非常的简洁，为了更直观的分析代码，就没有使用Lamda。通过subscribeOn(Schedulers.io())，
        observeOn(AndroidSchedulers.mainThread())轻松的实现线程的切换。非常的简单。下面一步一步分析：

　　**RxJava主要类**

　　如果对RxJava有了解的话，都知道它实际是用观察者模式实现的，我们这里只是简单的介绍一下主要的类。

　　<img src="http://img.blog.csdn.net/20160912182211379" width="300" height="200">

　　Observable和Subscriber大家已经很熟悉了，分别是被观察者和观察者。在使用RxJava过程中我们一般都是三步，第一步创建Observable，第二部创建Subscriber，第三步通过subscribe（）方法达到订阅的目的。为了搞清楚线程切换的实现，必须先搞清楚RxJava内部调用的流程！

　　在上边代码中，Observable.create()对应于第一步，会产生一个Observable。那么是怎么创建的呢？而create（）方法的源码：
　　

```java
public static <T> Observable<T> create(OnSubscribe<T> f) {
        return new Observable<T>(hook.onCreate(f));
    }
```
　　
　　create（）会接收一个OnSubscribe实例，OnSubscribe继承自Action1，有一个call（）回调方法。在subscribe（）方法执行时会被调用。这里的hook.onCreate()其实其他的什么也没有做就是你传进去什么就返回什么。最后可以看到产生了一个Observable的实例。

　　接着看一下`subscribe(subscriber)` , subscribe（）方法会产生订阅行为，接收一个Subscriber实例，现在有了被观察者和观察者，那么我们就可以分析订阅行为了。看subscribe（）方法的最终实现：
　　

```java
 static <T> Subscription subscribe(Subscriber<? super T> subscriber, Observable<T> observable) {
     // validate and proceed
        if (subscriber == null) {
            throw new IllegalArgumentException("subscriber can not be null");
        }
        if (observable.onSubscribe == null) {
            throw new IllegalStateException("onSubscribe function can not be null.");
            /*
             * the subscribe function can also be overridden but generally that's not the appropriate approach
             * so I won't mention that in the exception
             */
        }
        
        // new Subscriber so onStart it
        subscriber.onStart();
        
        /*
         * See https://github.com/ReactiveX/RxJava/issues/216 for discussion on "Guideline 6.4: Protect calls
         * to user code from within an Observer"
         */
        // if not already wrapped
        if (!(subscriber instanceof SafeSubscriber)) {
            // assign to `observer` so we return the protected version
            subscriber = new SafeSubscriber<T>(subscriber);
        }

        // The code below is exactly the same an unsafeSubscribe but not used because it would 
        // add a significant depth to already huge call stacks.
        try {
            // allow the hook to intercept and/or decorate
            hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
            return hook.onSubscribeReturn(subscriber);
        } catch (Throwable e) {
            // special handling for certain Throwable/Error/Exception types
            Exceptions.throwIfFatal(e);
            // in case the subscriber can't listen to exceptions anymore
            if (subscriber.isUnsubscribed()) {
                RxJavaPluginUtils.handleException(hook.onSubscribeError(e));
            } else {
                // if an unhandled error occurs executing the onSubscribe we will propagate it
                try {
                    subscriber.onError(hook.onSubscribeError(e));
                } catch (Throwable e2) {
                    Exceptions.throwIfFatal(e2);
                    // if this happens it means the onError itself failed (perhaps an invalid function implementation)
                    // so we are unable to propagate the error correctly and will just throw
                    RuntimeException r = new OnErrorFailedException("Error occurred attempting to subscribe [" + e.getMessage() + "] and then again while trying to pass to onError.", e2);
                    // TODO could the hook be the cause of the error in the on error handling.
                    hook.onSubscribeError(r);
                    // TODO why aren't we throwing the hook's return value.
                    throw r;
                }
            }
            return Subscriptions.unsubscribed();
        }
    }
```
　　一般情况下会执行第31行，
```java
hook.onSubscribeStart(observable, observable.onSubscribe).call(subscriber);
```
　　hook.onSubscribeStart()什么也没有做，将observable.onSubscribe直接返回，那么这句话就可以简化成

　　`observable.onSubscribe.call(subscriber);`

　　如果我们没有使用任何的操作符，那么这里的observable.onSubscribe就是我们在create（）方法中传入的onSubscribe实例。就会回调其call（ subscriber）方法。在call（）方法中，我们一般又会根据获得的subscriber引用，去调用相应的onNext（）和onComplete（）方法。这就是调用的基本流程！

　　如果我们使用了例如map这样的操作符，那么基本的流程大致是一样的，只不过是将Observable实例进行相应的变化后，向下传递。最终执行subscribe操作的是最后一个Observable,所以，每个变换后的Observable都会持有上一个Observable 中OnSubscribe对象的引用。

　　目前还是没有分析subscribeOn()和observeOn(),接下来就看看他们内部怎么实现的吧！

　　subscribeOn()源码：
　　

```java
 public final Observable<T> subscribeOn(Scheduler scheduler) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return create(new OperatorSubscribeOn<T>(this, scheduler));
    }
```
　　正如前边所说，这里会产生新的Observable，并且持有上一个Observable的OnSubscribe（我们的例子中就是我们在create（）方法中传入的）的引用。我们继续看OperatorSubscribeOn这个类:
　　

```java
public final class OperatorSubscribeOn<T> implements OnSubscribe<T> {

    final Scheduler scheduler;
    final Observable<T> source;

    public OperatorSubscribeOn(Observable<T> source, Scheduler scheduler) {
        this.scheduler = scheduler;
        this.source = source;
    }

    @Override
    public void call(final Subscriber<? super T> subscriber) {
        final Worker inner = scheduler.createWorker();
        subscriber.add(inner);
        
        inner.schedule(new Action0() {
            @Override
            public void call() {
                final Thread t = Thread.currentThread();
                
                Subscriber<T> s = new Subscriber<T>(subscriber) {
                    @Override
                    public void onNext(T t) {
                        subscriber.onNext(t);
                    }
                    
                    @Override
                    public void onError(Throwable e) {
                        try {
                            subscriber.onError(e);
                        } finally {
                            inner.unsubscribe();
                        }
                    }
                    
                    @Override
                    public void onCompleted() {
                        try {
                            subscriber.onCompleted();
                        } finally {
                            inner.unsubscribe();
                        }
                    }
                    
                    @Override
                    public void setProducer(final Producer p) {
                        subscriber.setProducer(new Producer() {
                            @Override
                            public void request(final long n) {
                                if (t == Thread.currentThread()) {
                                    p.request(n);
                                } else {
                                    inner.schedule(new Action0() {
                                        @Override
                                        public void call() {
                                            p.request(n);
                                        }
                                    });
                                }
                            }
                        });
                    }
                };
                
                source.unsafeSubscribe(s);
            }
        });
    }
}
```
　　该类实现了OnSubscribe接口，并且实现了call（）方法，也就意味着，我们之前所分析的在subscribe（）时，会一步一步去执行observable.onSubscribe.call(),对于SubscribeOn来说他的call（）方法的真正实现就是在这里。那么还要注意一点的是这个call（）的回调时机，这里要和ObserveOn（）做比较，稍后再分析。回到这里的call方法，首先通过外部传入的scheduler创建Worker - inner对象，接着在inner中执行了一段代码，Action0中call（）方法这段代码就在worker线程中执行了，也就是此刻线程进行了切换。

　　注意最后一句代码source.unsafeSubscribe(s)就是将当前的Observable与上一个Observable通过onSubscribe关联起来。那么如果上一个Observable也是一个subscribeOn（）产生的那么会出现什么情况？很显然最终会切换到上一个subscribeOn指定的线程中。例如：
　　

```java
  btn_map.setOnClickListener(v -> getObservable()
                .map(integer -> null)
                .subscribeOn(Schedulers.computation())
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(integer1 -> Log.e("map", integer1 + "")));
```
　　map的转换实际上会发生在computation线程而不是io线程，换句话说就是设置多个subscribeOn时，实际上只会切换到第一个subscribeOn指定的线程。这一点很重要！！！
　　
　　到现在我们已经分析了subscribe（）订阅后，一直到回调到我们在create（）方法中传入的OnSubscribe的call（）方法，subscribeOn（）方法是在这个过程中产生作用的。那么ObserveOn呢？看源码：
　　

```java
 public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        if (this instanceof ScalarSynchronousObservable) {
            return ((ScalarSynchronousObservable<T>)this).scalarScheduleOn(scheduler);
        }
        return lift(new OperatorObserveOn<T>(scheduler, delayError, bufferSize));
    }
```
　　这里经过了一次lift变化，这是个啥玩意呢？
　　

```java
public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
        return new Observable<R>(new OnSubscribeLift<T, R>(onSubscribe, operator));
    }
```
　　实际上也是包装了一层Observable。明白了这一点，在回到observeOn()源码中的OperatorObserveOn。
　　

```java
public final class OperatorObserveOn<T> implements Operator<T, T> {

  ...省略...

    /**
     * @param scheduler the scheduler to use
     * @param delayError delay errors until all normal events are emitted in the other thread?
     * @param bufferSize for the buffer feeding the Scheduler workers, defaults to {@code RxRingBuffer.MAX} if <= 0
     */
    public OperatorObserveOn(Scheduler scheduler, boolean delayError, int bufferSize) {
        this.scheduler = scheduler;
        this.delayError = delayError;
        this.bufferSize = (bufferSize > 0) ? bufferSize : RxRingBuffer.SIZE;
    }

    @Override
    public Subscriber<? super T> call(Subscriber<? super T> child) {
        if (scheduler instanceof ImmediateScheduler) {
            // avoid overhead, execute directly
            return child;
        } else if (scheduler instanceof TrampolineScheduler) {
            // avoid overhead, execute directly
            return child;
        } else {
            ObserveOnSubscriber<T> parent = new ObserveOnSubscriber<T>(scheduler, child, delayError, bufferSize);
            parent.init();
            return parent;
        }
    }
    
    ...省略...

    /** Observe through individual queue per observer. */
    private static final class ObserveOnSubscriber<T> extends Subscriber<T> implements Action0 {
       
       ...省略...

        @Override
        public void onNext(final T t) {
            if (isUnsubscribed() || finished) {
                return;
            }
            if (!queue.offer(on.next(t))) {
                onError(new MissingBackpressureException());
                return;
            }
            schedule();
        }

        @Override
        public void onCompleted() {
            if (isUnsubscribed() || finished) {
                return;
            }
            finished = true;
            schedule();
        }

        @Override
        public void onError(final Throwable e) {
            if (isUnsubscribed() || finished) {
                RxJavaPlugins.getInstance().getErrorHandler().handleError(e);
                return;
            }
            error = e;
            finished = true;
            schedule();
        } 
        
         protected void schedule() {
            if (counter.getAndIncrement() == 0) {
                recursiveScheduler.schedule(this);
            }
        }

        ...省略...
    }
}
```
　　安装前文的分析，先看他的call方法，返回了个ObserveOnSubscriber实例，我们需要关注这个实例的onNext方法，很简单，只是执行了schedule（）方法，该方法中的recursiveScheduler是在构造方法中根据我们设置的schedule创建的Scheduler.Worker 。

```java
this.recursiveScheduler = scheduler.createWorker();
```
　　线程再次切换了，并且这次是在OnNext（）方法中切换的，注意是在OnNext（）方法中切换，和subscribeOn()在call中切换是有区别的。这样observeOn设置的线程会影响其后面的流程，直到出现下一次observeOn或者结束。

　　这样最基本的线程切换已经搞清楚了，现在我们来分析一下加入例如map这样的操作符的过程：
　　

```java
 observable
        .map(str->str+"Rx")
        .map(str1->str1+"Java")
        .subscribeOn(Schedulers.io())
        .observeOn(AndroidSchedulers.mainThread())
        .subscribe(subscriber);
```

　　根据上边的分析，上一张流程图（不要在意，图很挫）：

　　<img src="http://img.blog.csdn.net/20160914155616630" width="450" height="250">

　　总的来说，RxJava的处理顺序像一条流水线，这不仅仅是代码写起来像一条链上，逻辑上也是如此，也就是说，当你切换流水的流向（线程），整条链都改变了方向，并不会进行分流。理解了这一点，对RxJava的线程切换也就不会感到困难了。

　　
