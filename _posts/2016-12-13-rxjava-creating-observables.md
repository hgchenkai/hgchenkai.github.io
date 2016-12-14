---
layout: post
title:  "RxJava操作符之Creating Observables"
desc: "RxJava操作符之Creating Observables"
keywords: "rxjava"
date: 2016-12-13
categories: [Java]
tags: [Rxjava,操作符]
icon: icon-java
---
RxJava 是一个在Java虚拟机上实现的响应式扩展库：提供了基于observable序列实现的异步调用及基于事件编程。 它扩展了观察者模式，支持数据、事件序列并允许你合并序列，无需关心底层的线程处理、同步、线程安全、并发数据结构和非阻塞I/O处理。

　　官网定义：RxJava is a Java VM implementation of Reactive Extensions: a library for composing asynchronous and event-based programs by using observable sequences.

　　如果没有接触过响应式编程，看起来会难以理解。但是，一旦上手之后，你会发觉用起来很爽。在Android中可以使用RxAndroid轻松实现线程切换，轻松写出优雅的代码。不过这都建立在熟练使用的基础上的。

　　本文不是讲解RxJava的原理的，而是RxJava另一重要的内容，功能强大、丰富的操作符。

　　将会以下分类用一系列的文章,介绍这些操作符

　　1.Creating Observables

　　2.Transforming Observables

　　3.Filtering Observables

　　4.Combining Observables

　　5.Error Handling Operators

　　6.Observable Utility Operators

　　7.Conditional and Boolean Operators

　　8.Mathematical and Aggregate Operators

　　9.Connectable Observable Operators

　　10.Backpressure Operators

　　从Create Observables类操作符开始讲起吧，顾名思义这类操作符都是可以得到Observable的。主要包括：

　　1.Create

　　2.Defer

　　3.Empty/Never/Throw

　　4.From

　　5.Interval

　　6.Just

　　7.Range

　　8.Repeat

　　9.Timer

　**create操作符**

　　create操作符是最基本的操作符，可以在合适的时机调用subscriber的onNext，onError，onComplete方法。下图是官方给的create的原理图（本系列所用的原理图都是官方所给的）：

　　<img src="http://img.blog.csdn.net/20160622212041814" width="400" height="200">

　　onNext就是发射数据给Subscriber; onComplete用来通知Subscriber所有的数据都已发射完毕；onError是在发生错误的时候发射一个Throwable对象给Subscriber。这里要注意一下，Observable在需要调用OnComplete方法时，必须通知所有订阅其的Subscriber，之后Observable将不再发射数据，OnError也是同样的。接下来看看具体的代码，为了方便看代码，部分代码暂时没有使用Lamda表达式：
```java
private Observable<String> createObservable() {
        return Observable.create(subscriber -> {
            for (int i = 0; i < 5; i++) {
                int num = new Random().nextInt(10);
                if (!subscriber.isUnsubscribed()) {
                    if (num > 5 && num < 8) {
                        subscriber.onError(new Throwable());
                    } else if (num >= 8) {
                        subscriber.onCompleted();
                    }else {
                        subscriber.onNext(num+"");
                    }
                }
            }
        });
    }
btn_create.setOnClickListener(v1 -> createObservable()
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {
                        Log.e("create", "onComplete");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("create", "onError");
                    }

                    @Override
                    public void onNext(String s) {
                        Log.e("create", s);
                    }
                }));
```

　执行结果

```
create: 4
create: 5
create: 0
create: 3
create: 1
create: 5
create: onError
create: 4
create: 4
create: 1
create: onComplete
``` 
　　以上结果是执行了3次的结果，第一次顺利执行，第二次触发了onError后Observable就停止发射数据了，第三次可以看到触发同样在触发onComplete后就停止发射数据了。

**defer/just**

　　defer操作符只有当有Subscriber来订阅的时候才会创建一个新的Observable对象,也就是说每次订阅都会得到一个刚创建的最新的Observable对象，这可以确保Observable对象里的数据是最新的,看原理图：

　　<img src="http://img.blog.csdn.net/20160622221541792" width="400" height="200">

 　　just操作符将某个对象转化为Observable对象，这些对象可以是一个数字、一个字符串、数组、Iterate，并且将其**一次性**发射出去，是一种非常快捷的创建Observable对象的方法。

<img src="http://img.blog.csdn.net/20160622221846652" width="400" height="200">

　　下面通过代码来认识一下他们之间的区别

```java
Observable<Integer>justObservable=justOperator();
        Observable<Integer>deferObservable=deferOperator();
        Button btn_defer = (Button) findViewById(R.id.btn_defer_oper);
        btn_defer.setOnClickListener(v->deferObservable
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onCompleted() {
//                        Log.e("defer", "onComplete");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("defer", "onError");
                    }

                    @Override
                    public void onNext(Integer i) {
                        Log.e("defer", i+"");
                    }
                }));
        Button btn_just = (Button) findViewById(R.id.btn_just_oper);
        btn_just.setOnClickListener(v->justObservable
                .subscribe(new Subscriber<Integer>() {
                    @Override
                    public void onCompleted() {
//                        Log.e("just", "onComplete");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("just", "onError");
                    }

                    @Override
                    public void onNext(Integer i) {
                        Log.e("just", i+"");
                    }
                }));
private Observable<Integer>deferOperator(){
        return Observable.defer(()->Observable.just(new Random().nextInt(100)));
    }
    private Observable<Integer> justOperator() {
        return Observable.just(new Random().nextInt(100));
    }
```

运行结果：
```
defer: 43
defer: 97
defer: 23
just: 98
just: 98
just: 98
```
　　正如上文所说，defer只有在subscibe时才会生成Observable，以保证是最新的数据，而just无论订阅几次都是用的首次创建的Observable对象。

**from**

　　 from操作符用来将某个对象转化为Observable对象，并且**依次**将其内容发射出去。听起来和just很像，那么它到底和just有什么不一样，这个类似于just，但是just会将这个对象整个发射出去。比如说一个含有10个数字的数组，使用from就会发射10次，每次发射一个数字，而使用just会发射一次来将整个的数组发射出去。

<img src="http://img.blog.csdn.net/20160622223446330" width="400" height="200">

　　代码：
　　

```java
private List<String> dataList=new ArrayList<>();
 private void initData() {
        dataList.add("welcome");
        dataList.add("to");
        dataList.add("Rxjava");
    }
 private Observable<String> fromOperator() {
        return Observable.from(dataList);
    }
Button btn_from = (Button) findViewById(R.id.btn_from_oper);
        btn_from.setOnClickListener(v -> fromOperator()
                .subscribe(new Subscriber<String>() {
                    @Override
                    public void onCompleted() {
                        Log.e("from", "onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("from", "onError");
                    }

                    @Override
                    public void onNext(String s) {
                        Log.e("from", s);
                    }
                }));
```
　　执行结果：
　　

```
from: welcome
from: to
from: Rxjava
from: onCompleted
```

**Empty/Never/Throw**

　　这三个操作符都是很简单的，就拿Empty来说吧，创建一个Observable，不会发射任何的数据，但是会正常的执行OnComplete，也就是说创建了一个Empty的Observable。

　　<img src="http://img.blog.csdn.net/20160622224100192" width="400" height="200">

代码：
　　

```java
private Observable emptyOperator(){
        return Observable.empty();
    }
     Button btn_empty = (Button) findViewById(R.id.btn_empty_oper);
        btn_empty.setOnClickListener(v -> emptyOperator()
                .subscribe(new Subscriber() {
                    @Override
                    public void onCompleted() {
                        Log.e("empty", "onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("empty", "onError");
                    }

                    @Override
                    public void onNext(Object s) {
                        Log.e("empty", s+"");
                    }
                }));
```
　　
　　执行结果：

```
empty: onCompleted
```
　　由于篇幅原因，有兴趣的可以自己去看看其他两个的实现。

**Range**

　　Range操作符根据输入的初始值n和数目m发射一系列大于等于n的m个值。

　　<img src="http://img.blog.csdn.net/20160622224543723" width="400" height="200">

　　具体的使用：

```java
Button btn_range = (Button) findViewById(R.id.btn_range_oper);
        btn_range.setOnClickListener(v -> rangeOperator()
                .subscribe(new Subscriber() {
                    @Override
                    public void onCompleted() {
                        Log.e("range", "onCompleted");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.e("range", "onError");
                    }

                    @Override
                    public void onNext(Object s) {
                        Log.e("range", s+"");
                    }
                }));
                 private Observable rangeOperator(){
        return Observable.range(1,10);
    }
```

执行结果

```
range: 1
range: 2
range: 3
range: 4
range: 5
range: 6
range: 7
range: 8
range: 9
range: 10
range: onCompleted
```

**Interval**

 　　Interval所创建的Observable对象会从0开始，每隔固定的时间发射一个数字。在Android这个对象默认是运行在**computation Scheduler**,所以如果需要在view中显示结果，需要在切换回主线程。

 　　<img src="http://img.blog.csdn.net/20160622225515924" width="400" height="200">

 　　Interval的用法有很多，在Android可以轻松实现计时器的功能，那么在使用时也有很多要注意的地方，除了上边说的要注意线程的问题，还有就是在Activity中使用的时候，需要在合适的时机进行反注册。否则可能会造成内存溢出。
 　　

```java
Observable<Long> observable = interval();

        Subscriber<Long> subscriber = new Subscriber<Long>() {
            @Override
            public void onCompleted() {
                Log.e"onCompleted" );
            }

            @Override
            public void onError(Throwable e) {
                Log.e("onError:" + e.getMessage());
            }

            @Override
            public void onNext(Long i) {
                Log.e("interval:" + i);
            }

        };
        sButton.setOnClickListener(e -> observable.subscribe(subscriber));
        unSButton.setOnClickListener(e -> subscriber.unsubscribe());
        
private Observable<Long> interval() {
        return Observable.interval(1, TimeUnit.SECONDS)
        .observeOn(AndroidSchedulers.mainThread());
    }
```
执行结果：

```
inerval：0
inerval：1
inerval：2
......
```
**Repeat/Timer**

 　　Repeat会重复发射一个Observable对象，并且可以指定其发射的次数。

 　　<img src="http://img.blog.csdn.net/20160622230420234" width="400" height="200">

 　　 Timer会在指定时间后发射一个指定类型的数据，例如如果我们指定的是Long，那么它会发射一个0。同interval一样，其也是运行在computation Scheduler中的。注意切换主线程。

 　　 <img src="http://img.blog.csdn.net/20160622230644172" width="400" height="200">

 　　 这都是非常简单的操作符，代码就不上了。试试就可以很清楚了。创建Observable的操作符常用的就这些了，下次将介绍Transforming Observables转换类的操作符。