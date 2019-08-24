# RxJava实践指南


<style>
img{
    width: 30%;
	padding-left: 5%;
}
</style>

初次接触RxJava的同学，可以翻阅相关文章：[给 Android 开发者的 RxJava 详解](http://gank.io/post/560e15be2dca930e00da1083)。抛物线的大神在RxJava1的基础上详细介绍了RxJava的工作原理，是很好的入门基础。

本文是基于RxJava2的基础上，介绍工作中实用性很强操作符，方便后期快速查阅。

## 1. Observables
### 1.1 Observable<T>
基础被观察者，能够发射0或n个数据，并以成功或错误事件终止。

### 1.2 Flowable<T>
 `Observable`的支持`Backpressure`版本，可以控制数据源发射的速度。

### 1.3 Single<T>
1. 只有 `onSuccess` 和 `onError` 事件。
2. `onSuccess`相当于合并了`onNext`、`onComplete`，只发送一次数据便结束。
3. 可以通过`toXXX`方法转换成`Observable`、`Flowable`、`Completable`、`Maybe`。

### 1.4 Completable
1. 只有 `onComplete` 和 `onError` 事件
2. `onComplete`不会发送数据，只是一个时机的回调。
3. 通过`fromXXX`操作符可以创建一个`Completable`。
4. 没有`map`、`flatMap`等操作符，它的操作符比起 `Observable/Flowable` 要少得多。
5. 经常会结合`andThen`操作符。
6. `andThen`有多个重载的方法，正好对应了五种被观察者的类型。
7. 可以通过`toXXX`方法转换成`Observable`、`Flowable`、`Single`、`Maybe`。

```java
# 使用fromAction创建对象，使用andThen执行下边任务。
Completable.fromAction(new Action() {code ...})
        .andThen(Observable.range(1, 10))
        .subscribe(new Consumer<Integer>() {code ...});
        
# andThen多重载方法
Completable       andThen(CompletableSource next)
<T> Maybe<T>      andThen(MaybeSource<T> next)
<T> Observable<T> andThen(ObservableSource<T> next)
<T> Flowable<T>   andThen(Publisher<T> next)
<T> Single<T>     andThen(SingleSource<T> next)
```

### 1.5 Maybe<T>
1. 能够发射0或者1个数据，要么成功，要么失败。
2. 可以通过`toXXX`方法转换成`Observable`、`Flowable`、`Single`。

## 2. 操作符
#### 2.1 创建操作
用于创建Observable的操作符

**just( )**
将一个或多个对象转换成发射这个或这些对象的一个Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/just.png)

**from( )**
将一个Iterable, 一个Future, 或者一个数组[] 转换成一个Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/from.png)

**repeat( )**
创建一个重复发射指定数据or数据序列的Observable。
它不是创建一个Observable，而是接收到onComplete()会触发重订阅，重复发射原始Observable的数据序列，这个序列可以是无限的，或者通过repeat(n)指定重复次数。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/repeat.o.png)

**repeatWhen( )**
创建一个重复发射指定数据或数据序列的Observable，它依赖于另一个Observable发射的数据。
它不是缓存和重放原始Observable的数据序列，而是有条件的重新订阅和发射原来的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/repeatWhen.f.png)

**create( )**
使用一个函数从头创建一个Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/create.c.png)

**defer( )**
一直等待，直到有观察者订阅它，然后它使用Observable工厂方法生成一个新的Observable。
尽管每个订阅者都以为自己订阅的是同一个Observable，事实上每个订阅者获取的是它们自己的单独的数据序列。
等待直到最后时刻（即订阅发生时）才生成Observable，可以确保Observable包含最新的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/defer.png)

**range( )**
创建一个发射指定范围的整数序列的Observable
使用：range(start，len) 
它接受两个参数，一个是范围的起始值，一个是范围的数据的数目。
如果你将第二个参数设为0，将导致Observable不发射任何数据（如果设置为负数，会抛异常）
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/range.png)

**interval( )**
创建一个按照给定的时间间隔发射整数序列的Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/interval.png)

**timer( )**
创建一个在给定的延时之后发射单个数据的Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/timer.png)

**empty( )**
创建一个不发射任何数据但是正常终止的Observable

**error( )**
创建一个什么都不做直接通知错误的Observable

**never( )**
创建一个不发射任何数据，也不终止的Observable

#### 2.2 变换操作
这些操作符可用于对Observable发射的数据进行变换，详细解释可以看每个操作符的文档

**map()**
Map操作符对原始Observable发射的每一项数据应用一个你选择的函数，然后返回一个发射这些结果的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/map.png)

**cast()**
cast操作符将原始Observable发射的每一项数据都强制转换为一个指定的类型，然后再发射数据，它是map的一个特殊版本。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/cast.png)

**flatMap()**
FlatMap将一个发射数据的Observable变换为多个Observables，然后将它们发射的数据合并后放进一个单独的Observable
FlatMap操作符使用一个指定的函数对原始Observable发射的每一项数据执行变换操作，这个函数返回一个本身也发射数据的Observable，然后FlatMap合并这些Observables发射的数据，最后将合并后的结果当做它自己的数据序列发射。
当你有一个这样的Observable：它发射一个数据序列，这些数据本身包含Observable成员或者可以变换为Observable，因此你可以创建一个新的Observable发射这些次级Observable发射的数据的完整集合。
FlatMap对这些Observables发射的数据做的是合并(merge)操作，因此它们可能是交错的。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/mergeMap.png)

**concatMap()**
类似于最简单版本的flatMap，但是它按次序连接而不是合并那些生成的Observables，然后产生自己的数据序列。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/concatMap.png)

**scan()** 
对Observable发射的每一项数据应用一个函数，然后按顺序依次发射每一个值
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/scan.c.png)
Scan操作符对原始Observable发射的第一项数据应用一个函数，然后将那个函数的结果作为自己的第一项数据发射。它将函数的结果同第二项数据一起填充给这个函数来产生它自己的第二项数据。它持续进行这个过程来产生剩余的数据序列。这个操作符在某些情况下被叫做accumulator。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/scan.png)

**buffer()**
定期收集Observable的数据放进一个数据包裹，然后发射这些数据包裹，而不是一次发射一个值。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/buffer3.png)

#### 2.3 过滤操作
这些操作符用于从Observable发射的数据中进行选择
**filter()**
Filter操作符使用你指定的一个谓词函数测试数据项，只有通过测试的数据才会被发射。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/filter.c.png)
**ofType()**
ofType是filter操作符的一个特殊形式。它过滤一个Observable只返回指定类型的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/ofClass.png)

**first()**
只发射第一项（或者满足某个条件的第一项）数据
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/first.png)
满足某个条件的第一项
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/firstN.png)

**last()**
只发射最后一项（或者满足某个条件的最后一项）数据
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/last.png)
满足某个条件的最后一项
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/last.p.png)

**take()**
Take操作符让你可以修改Observable的行为，只返回前面的N项数据，然后发射完成通知，忽略剩余的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/take.png)
take的这个变体接受一个时长而不是数量参数。它会丢发射Observable开始的那段时间发射的数据，时长和时间单位通过参数指定。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/take.t.png)

**takeFirst()**
takeFirst与first类似。
如果原始Observable没有发射任何满足条件的数据，first会抛出一个NoSuchElementException，takeFist会返回一个空的Observable（不调用onNext()但是会调用onCompleted）。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/first.takeFirst.png)

**takeLast()**
收集原始Observable发射的任何数据项，等发送完成后，只发射最后的N项数据。会有延迟。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/takeLast.c.png)
有个变体接受一个时长而不是数量参数。它会发射在原始Observable的生命周期内最后一段时间内发射的数据。时长和时间单位通过参数指定。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/takeLast.t.png)

**skip()**
抑制Observable发射的前N项数据，只保留之后的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/skip.png)
skip的有个变体接受一个时长而不是数量参数。它会丢弃原始Observable开始的那段时间发射的数据，时长和时间单位通过参数指定。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/skip.t.png)

**skipLast()**
抑制Observable发射的后N项数据，你可以忽略Observable'发射的后N项数据，只保留前面的数据。
注意：这个机制是这样实现的：延迟原始Observable发射的任何数据项，直到它发射了N项数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/skipLast.png)
有一个skipLast变体接受一个时长而不是数量参数。它会丢弃在原始Observable的生命周期内最后一段时间内发射的数据。时长和时间单位通过参数指定。
注意：这个机制是这样实现的：延迟原始Observable发射的任何数据项，直到自这次发射之后过了给定的时长。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/skipLast.t.png)

**elementAt()**
ElementAt操作符获取原始Observable发射的数据序列指定索引位置的数据项，然后当做自己的唯一数据发射。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/elementAt.png)

**sample()、throttleLast()**
Sample操作符定时查看一个Observable，然后发射自上次采样以来它最近发射的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/sample.png)

**throttleFirst()**
Sample操作符定时查看一个Observable，然后发射在那段时间内的第一项数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/throttleFirst.png)

**debounce()、throttleWithTimeout()**
只有当Observable在指定的时间后还没有发射数据时，才发射一个数据
Debounce操作符会过滤掉发射速率过快的数据项。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/debounce.c.png)


**timeout()**
对原始Observable的一个镜像，如果过了一个指定的时长仍没有发射数据，它会发一个错误通知
**timeout(long,TimeUnit))**
timeout的变体，第一个变体接受一个时长参数，每当原始Observable发射了一项数据，timeout就启动一个计时器，如果计时器超过了指定指定的时长而原始Observable没有发射另一项数据，timeout就抛出TimeoutException，以一个错误通知终止Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/timeout.1.png)
**timeout(long,TimeUnit,Observable))**
timeout的变体，timeout在超时时会切换到使用一个你指定的备用的Observable，而不是发错误通知。它也默认在computation调度器上执行。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/timeout.2.png)

**distinct()**
过滤掉重复数据，只允许还没有发射过的数据项通过。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/distinct.png)
**distinct(Func1)**
distinct操作符有一个变体接受一个函数。
这个函数根据原始Observable发射的数据项产生一个Key，然后，比较这些Key而不是数据本身，来判定两个数据是否是不同的。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/distinct.key.png)

**distinctUntilChanged()**
过滤掉连续重复的数据,它只判定一个数据和它的直接前驱是否是不同的。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/distinctUntilChanged.png)
**distinctUntilChanged(Func1)**
distinctUntilChanged的变体，接收一个函数，根据一个函数产生的Key判定两个相邻的数据项是不是不同的。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/distinctUntilChanged.key.png)

**ignoreElements()**
IgnoreElements操作符抑制原始Observable发射的所有数据，只允许它的终止通知（onError或onCompleted）通过。
如果你不关心一个Observable发射的数据，但是希望在它完成时或遇到错误终止时收到通知，你可以对Observable使用ignoreElements操作符，它会确保永远不会调用观察者的onNext()方法。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/ignoreElements.c.png)

#### 2.4 组合操作
组合操作符用于将多个Observable组合成一个单一的Observable

**startWith()**
Observable在发射数据之前先发射一个指定的数据序列。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/startWith.png)

**startWith(Observable)**
你也可以传递一个Observable给startWith，它会将那个Observable的发射物插在原始Observable发射的数据序列之前，然后把这个当做自己的发射物集合。这可以看作是Concat的反转。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/startWith.o.png)

**merge()**
使用Merge操作符你可以将多个Observables的输出合并，就好像它们是一个单个的Observable一样。
Merge可能会让合并的Observables发射的数据交错
任何一个原始Observable的onError通知会被立即传递给观察者，而且会终止合并后的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/merge.png)

**mergeDelayError()**
合并多个Observables，让没有错误的Observable都完成后再发射错误通知
它会保留onError通知直到合并后的Observable所有的数据发射完成，在那时它才会把onError传递给观察者。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/mergeDelayError.C.png)

**zip()**
通过一个函数将多个Observables的发射物结合到一起，基于这个函数的结果为每个结合体发射单个数据项。
Zip操作符返回一个Obversable，它使用这个函数按顺序结合两个或多个Observables发射的数据项，然后它发射这个函数返回的结果。它按照严格的顺序应用这个函数。它只发射与发射数据项最少的那个Observable一样多的数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/zip.c.png)

**combineLatest()**
当两个Observables中的任何一个发射了数据时，使用一个函数结合每个Observable发射的最近数据项，并且基于这个函数的结果发射数据。
CombineLatest操作符行为类似于zip，但是只有当原始的Observable中的每一个都发射了一条数据时zip才发射数据。CombineLatest则在原始的Observable中任意一个发射了数据时发射一条数据。当原始Observables的任何一个发射了一条数据时，CombineLatest使用一个函数结合它们最近发射的数据，然后发射这个函数的返回值。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/combineLatest.c.png)

join() 
无论何时，如果一个Observable发射了一个数据项，只要在另一个Observable发射的数据项定义的时间窗口内，就将两个Observable发射的数据合并发射
Join操作符结合两个Observable发射的数据，基于时间窗口（你定义的针对每条数据特定的原则）选择待集合的数据项。你将这些时间窗口实现为一些Observables，它们的生命周期从任何一条Observable发射的每一条数据开始。当这个定义时间窗口的Observable发射了一条数据或者完成时，与这条数据关联的窗口也会关闭。只要这条数据的窗口是打开的，它将继续结合其它Observable发射的任何数据项。你定义一个用于结合数据的函数。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/join.c.png)

#### 2.5 错误处理
很多操作符可用于对Observable发射的onError通知做出响应或者从错误中恢复，例如，你可以：

- 吞掉这个错误，切换到一个备用的Observable继续发射数据
- 吞掉这个错误然后发射默认值
- 吞掉这个错误并立即尝试重启这个Observable
- 吞掉这个错误，在一些回退间隔后重启这个Observable

**onErrorResumeNext( )**
让Observable遇到错误时发射一个特殊的项并且正常终止。
onErrorResumeNext方法返回一个镜像原有Observable行为的新Observable，后者会忽略前者的onError调用，不会将错误传递给观察者，作为替代，它会开始镜像另一个，备用的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/onErrorReturn.png)

**onErrorReturn( )**
让Observable在遇到错误时开始发射第二个Observable的数据序列。
onErrorReturn方法返回一个镜像原有Observable行为的新Observable，后者会忽略前者的onError调用，不会将错误传递给观察者，作为替代，它会发发射一个特殊的项并调用观察者的onCompleted方法。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/onErrorResumeNext.png)

**onExceptionResumeNext( )**
让Observable在遇到错误时继续发射后面的数据项。
和onErrorResumeNext类似，onExceptionResumeNext方法返回一个镜像原有Observable行为的新Observable，也使用一个备用的Observable，不同的是，如果onError收到的Throwable不是一个Exception，它会将错误传递给观察者的onError方法，不会使用备用的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/onExceptionResumeNextViaObservable.png)

**retry( )**
Retry操作符不会将原始Observable的onError通知传递给观察者，它会订阅这个Observable，再给它一次机会无错误地完成它的数据序列。Retry总是传递onNext通知给观察者，由于重新订阅，可能会造成数据项重复，如上图所示。
无论收到多少次onError通知，无参数版本的retry都会继续订阅并发射原始Observable。
接受单个count参数的retry会最多重新订阅指定的次数，如果次数超了，它不会尝试再次订阅，它会把最新的一个onError通知传递给它的观察者。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/retry.C.png)

**retryWhen( )**
retryWhen和retry类似，区别是，retryWhen将onError中的Throwable传递给一个函数，这个函数产生另一个Observable，retryWhen观察它的结果再决定是不是要重新订阅原始的Observable。如果这个Observable发射了一项数据，它就重新订阅，如果这个Observable发射的是onError通知，它就将这个通知传递给观察者然后终止。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/retryWhen.f.png)

#### 2.6 辅助操作
一组用于处理Observable的操作符

**timestamp( )**
给Observable发射的每个数据项添加一个时间戳
RxJava中的实现为timestamp，它将一个发射T类型数据的Observable转换为一个发射类型为Timestamped<T>的数据的Observable，每一项都包含数据的原始发射时间。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/timestamp.c.png)

**replay( )**
保证所有的观察者收到相同的数据序列，即使它们在Observable开始发射数据之后才订阅
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/replay.c.png)
可连接的Observable (connectable Observable)与普通的Observable差不多，不过它并不会在被订阅时开始发射数据，而是直到使用了Connect操作符时才会开始。用这种方法，你可以在任何时候让一个Observable开始发射数据。
如果在将一个Observable转换为可连接的Observable之前对它使用Replay操作符，产生的这个可连接Observable将总是发射完整的数据序列给任何未来的观察者，即使那些观察者在这个Observable开始给其它观察者发射数据之后才订阅。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/replay.png)

**observeOn( )**
指定一个观察者在哪个调度器上观察这个Observable
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/observeOn.c.png)

**subscribeOn( )**
指定Observable执行任务的调度器
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/subscribeOn.c.png)

doOnEach( )
注册一个动作，对Observable发射的每个数据项使用
doOnEach操作符让你可以注册一个回调，它产生的Observable每发射一项数据就会调用它一次。你可以以Action的形式传递参数给它，这个Action接受一个onNext的变体Notification作为它的唯一参数，你也可以传递一个Observable给doOnEach，这个Observable的onNext会被调用，就好像它订阅了原始的Observable一样。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/doOnEach.png)

**doOnNext()**
doOnNext操作符类似于doOnEach(Action1)，但是它的Action不是接受一个Notification参数，而是接受发射的数据项。

**doOnCompleted( )**
注册一个动作，对正常完成的Observable使用

**doOnError( )**
注册一个动作，对发生错误的Observable使用

**doOnTerminate( )**
注册一个动作，对完成的Observable使用，无论是否发生错误

**doOnSubscribe( )**
注册一个动作，在观察者订阅时使用

**doOnUnsubscribe( )**
注册一个动作，在观察者取消订阅时使用

**finallyDo( )**
注册一个动作，在Observable完成时使用

**delay( )**
Delay操作符让原始Observable在发射每项数据之前都暂停一段指定的时间段。效果是Observable发射的数据项在时间上向前整体平移了一个增量。
delay接受一个定义时长的参数（包括数量和单位）。每当原始Observable发射一项数据，delay就启动一个定时器，当定时器过了给定的时间段时，delay返回的Observable发射相同的数据项。
注意：delay不会平移onError通知，它会立即将这个通知传递给订阅者，同时丢弃任何待发射的onNext通知。然而它会平移一个onCompleted通知。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/delay.png)

**timeInterval( )**
将一个发射数据的Observable转换为发射那些数据发射时间间隔的Observable
操作符拦截原始Observable发射的数据项，替换为发射表示相邻发射物时间间隔的对象。
将原始Observable转换为另一个Observable，后者发射一个标志替换前者的数据项，这个标志表示前者的两个连续发射物之间流逝的时间长度。新的Observable的第一个发射物表示的是在观察者订阅原始Observable到原始Observable发射它的第一项数据之间流逝的时间长度。不存在与原始Observable发射最后一项数据和发射onCompleted通知之间时长对应的发射物。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/timeInterval.c.png)

#### 2.7 布尔操作
这些操作符可用于单个或多个数据项，也可用于Observable
**all( )**
判断是否所有的数据项都满足某个条件
传递一个谓词函数给All操作符，这个函数接受原始Observable发射的数据，根据计算返回一个布尔值。All返回一个只发射一个单个布尔值的Observable，如果原始Observable正常终止并且每一项数据都满足条件，就返回true；如果原始Observable的任何一项数据不满足条件就返回False。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/all.c.png)

**contains( )**
判断Observable是否会发射一个指定的值
给Contains传一个指定的值，如果原始Observable发射了那个值，它返回的Observable将发射true，否则发射false。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/contains.c.png)

**exists()** 
它通过一个谓词函数测试原始Observable发射的数据，只要任何一项满足条件就返回一个发射true的Observable，否则返回一个发射false的Observable。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/exists.png)

**isEmpty( )**
判定原始Observable是否没有发射任何数据。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/isEmpty.png)

**sequenceEqual( )**
判断两个Observables发射的序列是否相等
传递两个Observable给SequenceEqual操作符，它会比较两个Observable的发射物，如果两个序列是相同的（相同的数据，相同的顺序，相同的终止状态），它就发射true，否则发射false。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/sequenceEqual.c.png)


#### 2.8 算术和聚合操作
算术、聚合操作符用于对整个序列执行算法操作或其它操作，由于这些操作必须等待数据发射完成（通常也必须缓存这些数据），**它们对于非常长或者无限的序列来说是危险的，不推荐使用**。

**Average**
计算原始Observable发射数字的平均值并发射它
Average操作符操作符一个发射数字的Observable，并发射单个值：原始Observable发射的数字序列的平均值。
这个操作符不包含在RxJava核心模块中，它属于不同的rxjava-math模块。
它被实现为四个操作符：averageDouble, averageFloat, averageInteger, averageLong。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/average.c.png)

**Min**
Min操作符操作一个发射数值的Observable并发射单个值：最小的那个值。
RxJava中，min属于rxjava-math模块。
min接受一个可选参数，用于比较两项数据的大小，如果最小值的数据超过一项，min会发射原始Observable最近发射的那一项。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/min.c.png)

**Max**
Max操作符操作一个发射数值的Observable并发射单个值：最大的那个值。
RxJava中，max属于rxjava-math模块。
max接受一个可选参数，用于比较两项数据的大小，如果最大值的数据超过一项，max会发射原始Observable最近发射的那一项。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/max.c.png)

**Count**
计算原始Observable发射物的数量，然后只发射这个值
Count操作符将一个Observable转换成一个发射单个值的Observable，这个值表示原始Observable发射的数据的数量。
如果原始Observable发生错误终止，Count不发射数据而是直接传递错误通知。如果原始Observable永远不终止，Count既不会发射数据也不会终止。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/count.c.png)

**Sum**
计算Observable发射的数值的和并发射这个和
Sum操作符操作一个发射数值的Observable，仅发射单个值：原始Observable所有数值的和。
RxJava的实现是sumDouble, sumFloat, sumInteger, sumLong，它们不是RxJava核心模块的一部分，属于rxjava-math模块。
你可以使用一个函数，计算Observable每一项数据的函数返回值的和。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/sum.c.png)

**reduce()**
对序列使用reduce()函数并发射最终的结果
按顺序对Observable发射的每项数据应用一个函数并发射最终的值
Reduce操作符对原始Observable发射数据的第一项应用一个函数，然后再将这个函数的返回值与第二项数据一起传递给函数，以此类推，持续这个过程知道原始Observable发射它的最后一项数据并终止，此时Reduce返回的Observable发射这个函数返回的最终值。
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/reduce.c.png)


#### 2.9 转换操作

**To()：toFuture(), toIterable(), toList() , toMap() , toMultiMap(), toSortedList()**
将Observable转换为另一个对象或数据结构
![](https://mcxiaoke.gitbooks.io/rxdocs/content/images/operators/to.c.png)

**nest()**
将一个Observable转换为一个发射这个Observable的Observable。

**Blocking**
阻塞Observable的操作符

#### 2.10 字符串操作
StringObservable 类包含一些用于处理字符串序列和流的特殊操作符，如下：

**byLine()**
将一个字符串的Observable转换为一个行序列的Observable，这个Observable将原来的序列当做流处理，然后按换行符分割

**decode()**
将一个多字节的字符流转换为一个Observable，它按字符边界发射字节数组

**encode()**
对一个发射字符串的Observable执行变换操作，变换后的Observable发射一个在原始字符串中表示多字节字符边界的字节数组

**from()**
将一个字符流或者Reader转换为一个发射字节数组或者字符串的Observable

**join()**
将一个发射字符串序列的Observable转换为一个发射单个字符串的Observable，后者用一个指定的字符串连接所有的字符串

**split()**
将一个发射字符串的Observable转换为另一个发射字符串的Observable，后者使用一个指定的正则表达式边界分割前者发射的所有字符串

**stringConcat()**
将一个发射字符串序列的Observable转换为一个发射单个字符串的Observable，后者连接前者发射的所有字符串

#### 2.11 连接操作
一些有精确可控的订阅行为的特殊Observable

**Connect**
指示一个可连接的Observable开始发射数据给订阅者

**Publish**
将一个普通的Observable转换为可连接的

**RefCount**
使一个可连接的Observable表现得像一个普通的Observable

**Replay**
确保所有的观察者收到同样的数据序列，即使他们在Observable开始发射数据之后才订阅

## 3. 操作符决策树
实际使用中几种主要的需求

- 直接创建一个Observable（创建操作）
- 组合多个Observable（组合操作）
- 对Observable发射的数据执行变换操作（变换操作）
- 从Observable发射的数据中取特定的值（过滤操作）
- 转发Observable的部分值（条件/布尔/过滤操作）
- 对Observable发射的数据序列求值（算术/聚合操作）


## 4. 参考文章

参考：[ReactiveX/RxJava文档中文版](https://mcxiaoke.gitbooks.io/rxdocs/content/)
参考：[全部操作符列表](https://mcxiaoke.gitbooks.io/rxdocs/content/All-Operators-List.html)
