---
layout: post
title:  "RxJava操作符之Transforming Observables"
desc: "RxJava操作符之Transforming Observables"
keywords: "rxjava"
date: 2016-12-13
categories: [Java]
tags: [Rxjava,操作符]
icon: icon-java
---

　在[RxJava操作符之Creating Observables](rxjava-creating-observables.html)  我们学会了怎么创建Observable，但是我们项目中往往会遇到很多复杂的情况，需要我们对数据进行过滤和转化，以得到我们想要的结果。这篇文章我们主要是学习怎么转化数据：

　　**Buffer**

　　**FlatMap**

　　**GroupBy**

　　**Map**

　　**Scan**

　　**Window**

　　以上几个操作符就是专门用来处理数据转化功能的，接下来一个一个的来介绍吧。为了使代码看起来优雅一些，本篇开始都将使用lamda表达式。

　　**Buffer**

　　Buffer操作符就是将数据依据设置的大小做一下缓存，然后将缓存的数据作为一个集合发射出去。如下图所示，第一张示例图中我们指定buffer的大小为3，收集到3个数据后就发射出去，第二张图中我们加入了一个skip参数用来指定每次发射一个集合需要跳过几个数据，图中如何指定count为2，skip为3，就会每3个数据发射一个包含两个数据的集合，如果count==skip的话，我们就会发现其等效于第一种情况了。

　　<img src="http://img.blog.csdn.net/20160709103302027" width="400">
　
　　<img src="http://img.blog.csdn.net/20160709103347559" width="400">
　
　　在代码中使用也是一目了然的，对第一种情况
　　

```java
 private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
    btn_buffer.setOnClickListener(v -> createObservable()
                .buffer(3)
                .subscribe(o -> Log.e("buffer", o + "")));
```
　　会得到以下结果：
```
buffer: [1, 2,3]
buffer: [4, 5,6]
buffer: [7, 8,9]
buffer: [10]
```
　　第二种情况只需要在buffer操作符多设置一个skip就好了：
　　

```java
 private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
    btn_buffer.setOnClickListener(v -> createObservable()
                .buffer(2, 3)
                .subscribe(o -> Log.e("buffer", o + "")));
```
　　会得到以下结果：
```
buffer: [1, 2]
buffer: [4, 5]
buffer: [7, 8]
buffer: [10]
```
　　Buffer操作符就是将数据依据设置的大小做一下缓存，然后将缓存的数据作为一个集合发射出去，Buffer不仅设置数量规则，还可以设置时间：
　　

```java
private Observable bufferTimeObserver() {
        return Observable.interval(1, TimeUnit.SECONDS).buffer(3, TimeUnit.SECONDS).observeOn(AndroidSchedulers.mainThread());
    }
```
　　在上次我们讲过Interval根据设置的时间发射一个数据，本例中是每秒发射一个数据，而buffer(3, TimeUnit.SECONDS)，是设置每3秒将缓存的数据发射出去。
　　
　　**Map**

　　Map操作符会将数据**直接** 进行转换，其和稍后要讲的flatmap功能类似，但又有不同。先看map的原理图：
　　　<img src="http://img.blog.csdn.net/20160709110529728" width=400>
　　从图中可以看出，通过map操作符，设置数据转换规则，将数据直接进行转换。或许下图能更清楚的数目map的作用：
　　　<img src="http://img.blog.csdn.net/20160709110711496" width=400>
　　其用法也是非常简单的，下面我们就来验证一下是否能得到想要的效果：

```java
 private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
btn_map.setOnClickListener(v -> getObservable()
                .map(integer -> integer *10)
                .subscribe(integer1 -> Log.e("map", integer1 + "")));
```

```
map: 10
map: 20
map: 30
map: 40
map: 50
map: 60
map: 70
map: 80
map: 90
map: 100
``` 
　　将所有发出的数据都乘以了10，显然得到了我们想要的数据。下面接着介绍更加强大的flatmap
　　
　　**Flatmap**　

　　FlatMap在项目中使用频率很高的操作符，可以将要原数据（我们看做A类型的**Observable**）根据你想要的规则进行转化成B类型**Observable**后再发射出去。其原理就是将这个Observable转化为多个以原Observable发射的数据作为源数据的Observable，然后再将这多个Observable发射的数据整合发射出来，需要注意的是最后的顺序可能会**交错地**发射出来，如果需要顺序输出数据可以使用**concatmap**操作符。FlatMapIterable和FlatMap基相同，不同之处为其转化的多个Observable是使用Iterable作为源数据的。
　　　
　　<img src="http://img.blog.csdn.net/20160709111811024" width="400">

　　现在我们将原数据都加上flatmap字符串怎么实现呢？
　　

```java
 btn_flatmap.setOnClickListener(v -> getObservable()
                .flatMap(integer -> Observable.just("flatMap" + integer))
                .subscribe(s -> {
                    Log.e("flatmap", s);
                }));
                 private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
```
　　注意和map操作符的区别，flatmap是需要将原始Observable转换成目标Observable的，而map直接转化数据
　　以下是程序运行的结果：
　　

```
flatmap: flatMap1
flatmap: flatMap2
flatmap: flatMap3
flatmap: flatMap4
flatmap: flatMap5
flatmap: flatMap6
flatmap: flatMap7
flatmap: flatMap8
flatmap: flatMap9
flatmap: flatMap10
```
　　flatmap是需要好好掌握的，是一个功能强大的操作符。
　　
　　**GroupBy**

　　  GroupBy操作符将原始Observable发射的数据按照设置的key来拆分成一些小的Observable，然后这些小的Observable分别发射其所包含的的数据，类似数据库查询时groupBy。在使用中，我们需要提供一个生成key的规则，所有key相同的数据会分到同一个小的Observable中。

　　  　<img src="http://img.blog.csdn.net/20160709112759951" width="400">
　
　　  我们使用GroupBy把数据分成奇数和偶数，并且输出偶数：

```
 btn_groupby.setOnClickListener(v -> getObservable()
                .groupBy(integer -> integer % 2 == 0)
                .subscribe(booleanIntegerGroupedObservable -> {
                    if (booleanIntegerGroupedObservable.getKey()) {
                        booleanIntegerGroupedObservable.subscribe(integer -> Log.e("groupby"
                                , integer + ""));
                    }

                }));
                private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
```
　　运行结果如下，很明显得到了想要的结果：
```
groupby: 2
groupby: 4
groupby: 6
groupby: 8
groupby: 10
```
　　**Scan**

　　Scan操作符对一个序列的数据如{1，2，3，4}应用一个函数a*b，并将这个函数的结果1+2=3发射出去作为下个数据3应用这个函数时候的第一个参数使用,也就是3+3=6，有点类似于递归操作。通过scan我们可以很容易的实现斐波拉契数列。

　　　<img src="http://img.blog.csdn.net/20160709114415582" width="400">
　
　　　<img src="http://img.blog.csdn.net/20160709114441640" width="400">


```
btn_scan.setOnClickListener(v -> getObservable()
                .scan((integer, integer2) -> integer +integer2)
                .subscribe(integer1 -> Log.e("scan", "" + integer1)));
                private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
```
　　运行结果如下，
```
scan: 1
scan: 3
scan: 6
scan: 10
scan: 15
scan: 21
scan: 28
scan: 36
scan: 45
scan: 55
```
　　**Window**

　　Window操作符功能和我们前面讲过的buffer类似，不同之处在于Window发射的是一些小的Observable对象，由这些小的Observable对象来发射内部包含的数据。同buffer一样，window不仅可以通过数目来分组还可以通过时间等规则来分组

　　　<img src="http://img.blog.csdn.net/20160709115416024" width="400">
　　
　　我们将{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}分成两个Observable每个包含5个数据，看看window能否做到
```
btn_window.setOnClickListener(v -> getObservable()
                .window(5)
                .subscribe(integerObservable->{
                   integerObservable
                           .subscribe(integer -> Log.e("window"+integerObservable,integer+""));
                }));
                private Observable<Integer> getObservable() {
        return Observable.from(new Integer[]{1, 2, 3, 4, 5, 6, 7, 8, 9, 10});
    }
```
　　得到以下结果

```
windowrx.subjects.UnicastSubject@875d781: 1
windowrx.subjects.UnicastSubject@875d781: 2
windowrx.subjects.UnicastSubject@875d781: 3
windowrx.subjects.UnicastSubject@875d781: 4
windowrx.subjects.UnicastSubject@875d781: 5
windowrx.subjects.UnicastSubject@9c4026: 6
windowrx.subjects.UnicastSubject@9c4026: 7
windowrx.subjects.UnicastSubject@9c4026: 8
windowrx.subjects.UnicastSubject@9c4026: 9
windowrx.subjects.UnicastSubject@9c4026: 10
```

　　可以清晰看到确实有两个Observable，rx.subjects.UnicastSubject@875d781和rx.subjects.UnicastSubject@9c4026，每个Observable包含5个数据。也是验证上边的结论。
　　
　　**总结**：Transforming操作符，在Rxjava中经常使用，并且功能强大，所以一定要熟练的掌握转换操作符。这样才能灵活使用Rxjava。

　　