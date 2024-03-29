## 前言


Rxjava是NetFlix出品的Java框架， 官方描述为 a library for composing asynchronous and event-based programs using observable sequences for the Java VM，翻译过来就是“使用可观察序列组成的一个异步地、基于事件的响应式编程框架”。一个典型的使用示范如下：

```java

        Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                String s = "1234";
                //执行耗时任务
                emitter.onNext(s);
            }
        }).map(new Function<String, Integer>() {
            @Override
            public Integer apply(String s) throws Exception {
                return Integer.parseInt(s);
            }
        }).subscribeOn(Schedulers.io())
          .observeOn(AndroidSchedulers.mainThread())
          .subscribe();

```

本文要讲的主要内容是Rxjava的核心思路，利用一张图并结合源码分析Rxjava的实现原理，至于使用以及其比较深入的内容，比如不常用的操作符，背压等，读者可以自行学习。另外提一句，本文采用的Rxjava版本是2.2.3，Rxjava最新版本是3.x.x，感兴趣的可以自行阅读，但相信其最核心的原理是不会变化的。

## 正题

先放出本文最重要的图：

![Rxjava原理图](https://imgconvert.csdnimg.cn/aHR0cHM6Ly9pbWcyMDE4LmNuYmxvZ3MuY29tL2Jsb2cvOTAzNDUxLzIwMTkxMC85MDM0NTEtMjAxOTEwMjQxMjQzMDcyMzktNjY5Mzc2MjQ2LnBuZw?x-oss-process=image/format,png)

Rxjava的核心思路被总结在了图中，本文分为两部分，第一部分讲图中的三条流和事件传递，第二部分讲线程切换的原理，下面进入正题。

### 流式构建和事件传递

在讲之前，先提一点，在Rxjava中，有Observable和Observer这两个核心的概念，但是它们在发生订阅时，跟普通的观察者模式写法不太一样，因为常识来讲，应该是观察者去订阅(subscribe)被观察者，但是Rxjava为了其基于事件的流式编程，只能反着来，observable去订阅observer，所以在rxjava中，subscribe可以理解“注入”观察者。

首先我们看上面的图片，先简单解释一下：图中方形的框代表的是Observable，因为它代表节点，所以用Ni表示，圆形框代表的是观察者Observer，用Oi标识，后面加括号的意思是Oi持有其下游Observer的引用，左侧代表上游，右侧代表下游。图片里有三条有方向的彩色粗线，代表三个不同的流，这三个流是我们为了分析问题而抽象出来的的，代表从构建到订阅整个事件的流向，按照时间顺序从上到下依次流过，它们的含义分别是：

1. 从左往右的构建流：用来构建整个事件序列，这个流表征了整个链路的构建过程，相当于构造方法。
2. 从右往左的订阅流：当最终订阅(subscribe方法)这个行为发生的时候，每个节点从右向左依次执行订阅行为。
3. 从左往右的观察者回调流：当事件发生以后，会通过这个流依次通知给各个观察者。

我们依次分析这三条流：

#### 构建流

在使用Rxjava时，其流式构建流程是很大的特色，避免了传统回调的繁琐。怎么实现的呢？使用过Rxjava的读者应该都知道，Rxjava的每一步构建过程api都是相同的，这是因为每一步的函数返回结果都是一个Observable，Observable提供了Rxjava所有的功能。那么Obsevable在Rxjava中到底扮演一个什么角色呢？事实上，其官方定义就已经告诉我们答案了，前言里官方定义中有这样一段：“using Observable sequences”，所以说，Obsevable就是构建流的组件，我们可以看成一个个节点，这些节点串起来组成整个链路。Observable这个类实现了一个接口:ObservableSource,这个接口只有一个方法:subscribe(observer)，也就是说，所有的Obsevable节点都具有订阅这个功能，这个功能很重要，是订阅流的关键，待会会讲。总结一下：

***在我们编写Rxjava代码时，每一步操作都会生成一个新的Observable节点(没错，包括ObserveOn和SubscribeOn线程变换操作)，并将新生成的Observable返回，直到最后一步执行subscribe方法***

无论是构建的第一步 create方法，还是observeOn,subscribeOn变换线程方法，还是各种操作符比如map,flatMap等，都会生成对应的Observable，每个Observble中要实现一个最重要的方法就是subscribe，我们看其实现:

```java
    public final void subscribe(Observer<? super T> observer) {
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);
            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
            RxJavaPlugins.onError(e);
            throw npe;
        }
    }
```

这里提一点，大家看源码时遇到RxJavaPlugins时直接略过看里面的代码就好了，它是hook用的，不影响主要流程。所以上面代码其实只有一行有用：

`subscribeActual(observer);` 

也就是说，每个节点在执行subscribe时，其实就是在调用该节点的subscribeActual方法，这个方法是抽象的，每个节点的实现都不一样。我们举个栗子，拿ObseverOn这个操作生成的ObservableSubscribeOn瞧瞧：

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;
    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }
    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);
        observer.onSubscribe(parent);
        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
  //xxx省略
}
```

其中其父类继承Observable，所以它是一个Observble。

整个过程有点像builder模式，不同之处是它是生成了新的节点，而builder模式返回的自身。如果你读过okHttp的源码，okHttp中拦截器跟这里有些相似，okHttp中会构建多个Chain节点，然后用相应的Intercepter去处理Chain。

我们理解了编写Rxjava代码的过程其实就是构建一个一个Observable节点的过程，接下来我们看第二条流。

#### 订阅流

构建过程只是通过构造函数将一些配置传给了各个节点，实际还没有执行任何代码，只有最后一步才真正的执行订阅行为。当最后一个节点调用subscribe方法时，是构建流向订阅流变化的转折点，我们以图中为例:最后一个节点是N5，N5节点是最后一个flatmap操作符方法产生的，也就是说，最后是调用这个节点的subscribe方法，这个方法最终也是会调用到subscribeActual方法中去，我们看其源码：

```java
public final class ObservableFlatMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends ObservableSource<? extends U>> mapper;
    final boolean delayErrors;
    final int maxConcurrency;
    final int bufferSize;

    public ObservableFlatMap(ObservableSource<T> source,
            Function<? super T, ? extends ObservableSource<? extends U>> mapper,
            boolean delayErrors, int maxConcurrency, int bufferSize) {
        super(source);
        this.mapper = mapper;
        this.delayErrors = delayErrors;
        this.maxConcurrency = maxConcurrency;
        this.bufferSize = bufferSize;
    }
    @Override
    public void subscribeActual(Observer<? super U> t) {
        if (ObservableScalarXMap.tryScalarXMapSubscribe(source, t, mapper)) {
            return;
        }
        source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));
    }

    static final class MergeObserver<T, U> extends AtomicInteger implements Disposable, Observer<T> {
        final Observer<? super U> downstream;
        final Function<? super T, ? extends ObservableSource<? extends U>> mapper;
    }

```

刚才我们分析了，N5节点是Observable节点，其subscribe方法最后调用的是subscribeActual方法，我们看上面代码中它的这个方法：前面的判断语句跳过，第二行：

` source.subscribe(new MergeObserver<T, U>(t, mapper, delayErrors, maxConcurrency, bufferSize));`

这行代码需要注意两点：

1. 生成了一个新的Observer，请注意其构造函数中第一个参数t，保存到了downstream这个“下游”变量中，这个t从哪儿传进来的呢？对于N5节点来说，这个t就是我们代码中最后一步编写的Observer，比如我们常用的网络请求返回后的回调。也就是说，这个新生成的Observer包含了它的“下游”观察者的引用，在图片中对应最右边的圆形框O1(observer)。
2. 执行订阅行为，这里的source是该节点构造函数传入的source，通过源码得知其实就是N5节点的上一个节点N4，因此，这里的订阅行为本质上是**让当前节点的上一个节点订阅当前节点新生成的Observer**。

到这里，我们分析了最后一个节点执行subscribe方法的过程，事实上，每个节点的执行流程都是类似的(subscribeOn节点有些特殊，等会线程调度会将)，也就是说，N5会调用N4的subscribe方法，而在N4的subscribe方法中，又去调用了N3的subscribe....一直到N0会调用source的subscribe方法。总结下来就是：

***从最后一个N5节点的订阅行为开始，依次执行前面各个节点真正的订阅方法。在每个节点的订阅方法中，都会生成一个新的Observer，这个Observer会包含“下游”的Observer，这样当每个节点都执行完订阅(subscribeActual)后，也就生成了一串Observer，它们通过downstream,upstream引用连接。***

以上就是订阅流的发生过程，简单讲就是下游节点调用上游节点的subscribeActual方法，从而形成了一个调用链。

#### 观察者回调流

当订阅流执行到最后，也就是第一个节点N0时，我们看发生了什么，首先看看N0节点怎么建立的：

```java
   public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```

生成了ObservableCreate实例，我们看这个类（简化）:

```java
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;
    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

    @Override
    protected void subscribeActual(Observer<? super T> observer) {
        CreateEmitter<T> parent = new CreateEmitter<T>(observer);
        observer.onSubscribe(parent);
        source.subscribe(parent);
    }
}
```

所以订阅流的最终会掉到上面的subscrbeActual方法，它其实还是和其他节点一样，最主要的还是执行了

`source.subscribe(parent)`

这行代码，那么这个节点的source是什么呢？它就是我们事件的源头啊！

```java
   Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> emitter) throws Exception {
                String s = "1234";
                //执行耗时任务
                emitter.onNext(s);
            }
        })
```

上面代码直接拿的开头的例子，这个source是一个ObservableOnSubscribe，看它的subscribe方法里，这里很重要，这个函数里面其实是订阅流和观察者流的转折点，也就是流在这儿“转向了”。这里，这个事件源没有像节点那样，调用上一个节点的订阅方法，而是调用了其参数的emitter的onNext方法，这个emitter对应N0节点的什么呢？看代码知道，时CreateEmitter这个类，我们看这个类里面

```java
    static final class CreateEmitter<T> extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
      
        final Observer<? super T> observer;
      
        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }
        @Override
        public void onNext(T t) {
            if (!isDisposed()) {
                observer.onNext(t);
            }
        }
      //省略
    }
```

看它的onNext方法，执行的是

`observer.onNext(t)`

observer是谁？构造函数传进来的，也就是N0节点subscribeActual方法中的observer，这个observer是谁呢？仔细回想一下，前面订阅流的时候不就是一次订阅上一个节点生成的Observer吗，所以这个observer就是前一个节点N1生成的Observer，我们看N1节点，是一个Map，对应的Observable节点里的Observer源码如下:

```java
    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;
      
        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
            super(actual);
            this.mapper = mapper;
        }

        @Override
        public void onNext(T t) {
            if (done) {
                return;
            }
            if (sourceMode != NONE) {
                downstream.onNext(null);
                return;
            }
            U v;
            try {
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
           downstream.onNext(v);
          //省略后续
           
```

名为MapObserver，看它的onNext方法，忽略前面两个判断语句，核心就两句，一个是mapper.apply(t)，另一个就是downstream.onNext(v)。也就是说，这个mapObserver干了两件事，一个是把上个节点返回的数据进行一次map变换，另一个就是将map后的结果传递给下游，下游是什么呢？看了订阅流的读者自然知道，就是N2节点的Observer，对应图中O4,依次类推，我们知道了，事件发生以后，通过各个节点的Observer事件源被层层处理并传递给下游，一直到最后一个观察者执行完毕，整个事件处理完成。

至此，我们三个流分析完毕，接下来，我们开始分析线程调度是怎么实现的。

### 线程调度

观察仔细的读者可能已经看到了，图中N2节点左侧的所有节点和右侧的节点颜色不同，我为什么要这样画呢？其实里面的玄机就是线程调度，接下来我们分别看subscribeOn和observeOn的线程切换玄机吧。

#### SubscribeOn

在订阅流发生的的时候，大多数节点都是直接调用上一个节点的subscribe方法，实现虽有差别，但大同小异。唯一有个最大的不同就是subscribeOn这个节点，我们看源码：

```java
public final class ObservableSubscribeOn<T> extends AbstractObservableWithUpstream<T, T> {
    final Scheduler scheduler;

    public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
        super(source);
        this.scheduler = scheduler;
    }

    @Override
    public void subscribeActual(final Observer<? super T> observer) {
        final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(observer);

        observer.onSubscribe(parent);

        parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
    }
```

普通的节点执行时，大多只是简单的执行source.subscribe(observer)，但是这个不一样。先看第二行，它调用了观察者的onSubscribe方法，熟悉Rxjava的人知道，我们在自定义Observer的时候，里面有这个回调，其发生时机就在此刻。我们接着看最后一行,忽略parent.setDisposable这个逻辑，我们直接看参数里面的东西。

`scheduler.scheduleDirect(new SubscribeTask(parent))`

看看干了什么：

```java
    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run) {
        return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
    }
```

继续：

```java
    @NonNull
    public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
        final Worker w = createWorker();
        final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        DisposeTask task = new DisposeTask(decoratedRun, w);
        w.schedule(task, delay, unit);
        return task;
    }
```

创建了一个worker，一个runnable，然后将二者封装到一个DisposeTask中，最后用worker执行这个task，那么这个worker是什么呢？

```java
    @NonNull
    public abstract Worker createWorker();
```

createworker是一个抽象方法，所以需要去找Scheduler的子类，我们回想一下rxjava的使用，如果在子线程中执行，我们一般设置调度器为Schedulers.io(),我们看这个子类的实现:

在IOSchedluer类中:

```java
        @Override
        public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
            if (tasks.isDisposed()) {
                // don't schedule, we are unsubscribed
                return EmptyDisposable.INSTANCE;
            }
            return threadWorker.scheduleActual(action, delayTime, unit, tasks);
        }
```

继续:

```java
    @NonNull
    public ScheduledRunnable scheduleActual(final Runnable run, long delayTime, @NonNull TimeUnit unit, @Nullable DisposableContainer parent) {
        Runnable decoratedRun = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable sr = new ScheduledRunnable(decoratedRun, parent);
        if (parent != null) {
            if (!parent.add(sr)) {
                return sr;
            }
        }
        Future<?> f;
        try {
            if (delayTime <= 0) {
                f = executor.submit((Callable<Object>)sr);
            } else {
                f = executor.schedule((Callable<Object>)sr, delayTime, unit);
            }
            sr.setFuture(f);
        } catch (RejectedExecutionException ex) {
            if (parent != null) {
                parent.remove(sr);
            }
            RxJavaPlugins.onError(ex);
        }
        return sr;
    }
```

这里的executor就是一个ExecutorService，熟悉线程池的读者应该知道，这里的submit方法，就是将callable丢到线程池中去执行任务了。

我们回到主线

`scheduler.scheduleDirect(new SubscribeTask(parent))`

对于io线程的调度器来说，上面的代码就是将new SubscribeTask(parent)丢到线程池中执行，我们看参数里面的SubscribeTask：

```java
    final class SubscribeTask implements Runnable {
        private final SubscribeOnObserver<T> parent;

        SubscribeTask(SubscribeOnObserver<T> parent) {
            this.parent = parent;
        }

        @Override
        public void run() {
            source.subscribe(parent);
        }
    }
```

看run方法：source.subscribe(parent)，这里的parent跟普通节点一样，仍然是本节点生成的新的Observer，对于本节点来说，是一个SubscribeOnObserver。因此，我们就知道了，对于subscribeOn这个节点，它跟普通的节点不同之处在于：

***SubscribeOn节点在订阅的时候，将它的上游节点的订阅行为，以runnable的形式扔给了一个线程池(对于IO调度器来说)，也就是说，当订阅流流到SubscribeOn节点时，线程发生了切换，之后流向的节点都在切换后的线程中执行。***

分析到这里，我们就知道了subscribeOn的线程切换原理了，原来是在订阅流中塞了一个线程变化操作。我们再看图中的颜色问题，为什么这个节点上游的节点都是红色的呢？因为当订阅流流过这个节点后，后面的节点只是单纯的传递给上游节点而已，无论是普通的操作符，还是ObserveOn节点，都是简单的传递给上游，没有做线程切换(注意，ObserveOn是在观察者流中做的线程切换，待会会讲)。

**我们再思考一个问题，如果上游还有别的subscribeOn，会发生什么？**

我们假设N1节点的map修改程subscribeOn(AndroidScheduler.Main)，也就是说，切换到主线程。我们还是从N2节点开始分析，刚才说到最后会执行到SubscribeTask里的Run方法，注意此时source.subscribe(parent)发生在子线程中，接下来，回调用N1节点的subscribe，N1节点回调用scheduler.scheduleDirect(new SubscribeTask(parent))，方法，此时，因为线程调度器是主线程的，我们看它的代码：

```java
    private static final class MainHolder {
        static final Scheduler DEFAULT
            = new HandlerScheduler(new Handler(Looper.getMainLooper()), false);
    }
```

看看这个HandlerScheduler的方法:

```java
    @Override
    public Disposable scheduleDirect(Runnable run, long delay, TimeUnit unit) {
        run = RxJavaPlugins.onSchedule(run);
        ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);
        handler.postDelayed(scheduled, unit.toMillis(delay));
        return scheduled;
    }
```

熟悉Android Handler机制的读者应该很清楚，这里会把N1节点上游的操作，通过Handler机制，扔给主线程操作，虽然这一步是在N2节点的子线程中执行的，但是它之前的事件仍然会在主线程中执行。因此我们有以下结论：

***subscribeOn节点影响它前面的节点的线程，如果前面还有多个subscribeOn节点，最终只有第一个，也就是最上游的那个节点生效***

接下来我们分析observeOn

#### ObserveOn

前面的subscribeOn线程切换是在订阅流中发生的，接下来的ObserveOn比较简单，它发生在第三条流-观察者回调流中，我们看ObserveOn节点的源码:

```java
    static final class ObserveOnObserver<T> extends BasicIntQueueDisposable<T>
    implements Observer<T>, Runnable {
    //简化
       @Override
        public void onNext(T t) {
            if (done) {
                return;
            }

            if (sourceMode != QueueDisposable.ASYNC) {
                queue.offer(t);
            }
            schedule();
        }
      }
```

在前面的观察者流分析时，我们知道，观察者流是通过onNext()方法传递的，我们看最后一行，schedule()，线程切换，所以这个ObserveOn节点其实没干啥事，就是切换线程了，而且是在onNext回调中切换的。我们进到这个方法：

```java
       void schedule() {
            if (getAndIncrement() == 0) {
                worker.schedule(this);
            }
        }
```

worker是这个节点订阅时指定的 scheduler.createWorker()， 以主线程观察为例:

```java
  public Disposable schedule(Runnable run, long delay, TimeUnit unit) {
     
            run = RxJavaPlugins.onSchedule(run);
            ScheduledRunnable scheduled = new ScheduledRunnable(handler, run);
            Message message = Message.obtain(handler, scheduled);
            message.obj = this; // Used as token for batch disposal of this worker's runnables.
            if (async) {
                message.setAsynchronous(true);
            }
            handler.sendMessageDelayed(message, unit.toMillis(delay));
            // Re-check disposed state for removing in case we were racing a call to dispose().
            if (disposed) {
                handler.removeCallbacks(scheduled);
                return Disposables.disposed();
            }
            return scheduled;
        }
```

同样，通过Handler机制，将runnable扔给主线程执行,runnable是谁呢，是this，this就是这个ObserveOnObserver,我们看它的run方法：

```java
        @Override
        public void run() {
            if (outputFused) {
                drainFused();
            } else {
                drainNormal();
            }
        }
```

继续看drainNormal

```java
void drainNorml() {
    //简化
  final Observer<? super T> a = downstream;
  T v;
  v = q.poll();
  a.onNext(v);
}
```

抓重点，还是把上游的处理结果扔给下游。也就是说observeOn会将它下游的onNext操作扔给它切换的线程中，因此ObserveOn影响的是它的下游，所以我们途中observeOn后面的颜色都是蓝的。

同样我们思考，如果有多个observeOn会发生什么？很简单，思路同subscribeOn，每个ObserveOn只会影响它下游一直到下一个obseveOn节点的线程，也就是分段的。



## 总结

到此为止我们就讲完了全部内容，包括三条流的原理和线程切换的原理，至于Rxjava的其他功能和原理，限于篇幅，本文不会讲解，感兴趣的读者自行阅读源码。本文主要为读者提供了理解Rxjava的思路，真正要去理解它，还是要多看源码。

在我看来，Rxjava有点像观察者模式和责任链模式的结合变种，普通的观察者模式一般是被观察者通知多个观察者，而Rxjava则是被观察者通知第一个Obsever,接下来Observer依次通知其他节点的Observer，将观察者模式进行了一种类似链式的变换，每个节点又会执行它不同的“职责”，非常巧妙，事件在Observable链条上进行传递，事件结果通过Observer链条进行回调，这或许就是Rxjava的精髓所在.

------



## 后记，关于flatmap

关于flatmap这个操作符，读者可能会用到，但理解起来又比较难，我们通过本文，其实就很容易从源码中理解这个操作符的含义。这里我顺便给大家解释一下，还是看图：
![](https://img2018.cnblogs.com/blog/903451/201910/903451-20191025145151382-1992261292.png)

上图简化了整个事件流向，我们对事件源进行了flatmap操作，flatmap在订阅流的时候跟其他的操作符基本一致，但是在观察者回调流中却很不一样，它在回调流中做了以下内容:
flatmap将上游传过来的数据进行了一次变换，变成了一个Observable，如何变的是由开发者自定义的，比如图中下面三个竖着的三个Observable节点流，这条流跟上面的四个Observable节点本质上是一样的。***flatmap这个节点的Obsever将上游的数据转化成了一个新的Observable流，然后执行这条新的流，当这条新的流走完时，会接着原来的观察者流继续走下去。也就是说，flatMap这个操作符将一条新的Observable节点流“插入”到原始的观察者回调流上去了。***
那图中的橘黄色和紫色的虚线是什么意思呢？
实际上它是flatmap的一种特殊情况，当新插入的流的事件源有多个的时候，这是会产生分流，每个流都会执行一遍下游的原始节点。我们拿下面这个例子来看：
```java
        String[] mainArmy = {"第一大队", "第二大队", "第三大队"};
        Observable.fromArray(mainArmy)
                .flatMap(new Function<String, ObservableSource<String>>() {
                    @Override
                    public ObservableSource<String> apply(String s) throws Exception {
                        String[] littleArmy = {s + "的第一小队", s + "的第二小队", s + "的第三小队"};
                        return Observable.fromArray(littleArmy);
                    }
                }).subscribe(new Consumer<String>() {
            @Override
            public void accept(String little) throws Exception {
                System.out.println(little);
            }
        });
```
这个代码运行结果是
```
第一大队的第一小队
第一大队的第二小队
第一大队的第三小队
第二大队的第一小队
第二大队的第二小队
第二大队的第三小队
第三大队的第一小队
第三大队的第二小队
第三大队的第三小队
```
这个例子也许很好的体现了flatmap这个操作符的意义，把list铺平展开，而且防止了繁琐的嵌套循环。但是，虽然flatmap很擅长处理这种问题(我不知道这个操作符是不是为了解决这个问题而设计出来的)，但flatmap的功能却远不仅如此，它的本质是在合并Obsevable流，可以做很多事情，比如我们网络请求的“连环请求”，举个例子，首先通过书本的Id获取出版商名字，然后拿到出版商名字后获取出版社信息。
```java
api.getBookPublisherNameById("01102").flatmap(new Function<String, ObservableSource<PublisherInfo>>() {
        @Override
   			public ObservableSource<PublisherInfo> apply(String s) throws Exception {
         		return api.getPublisherInfoByName(s);
        }
   	}).subscribe(new COnsume<PublisherInfo>() {
                 @Override
            public void accept(PublisherInfo little) throws Exception {
                //获取到出版社信息
            }
})
```

看完这里，flatmap是不是也蛮好理解的～