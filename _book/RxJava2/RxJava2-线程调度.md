# RxJava2-线程调度

RxJava2中的线程调度主要用到subscribeOn/observeOn

从一个使用出发
```
Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> emitter) {
                        emitter.onNext(22245);
//                        emitter.onComplete();
//                        emitter.onError(null);
                        emitter.onNext(2);
                        emitter.onNext(2);

                        Log.i("ccer", "================currentThread name: " + Thread.currentThread().getName());
                    }
                })
                .map(new Function<Integer, String>() {
                    @Override
                    public String apply(Integer integer) throws Exception {
                        return "" + integer;
                    }
                })
                .map(new Function<String, String>() {
                    @Override
                    public String apply(String s) throws Exception {
                        return "我是前缀" + s;
                    }
                })
                .subscribeOn(Schedulers.io())
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(new Observer<String>() {

                    private Disposable mDisposable;

                    @Override
                    public void onSubscribe(Disposable d) {
                        mDisposable = d;
                        Log.i("ccer", "========================onSubscribe" + d);
                    }

                    @Override
                    public void onNext(String s) {
                        Log.i("ccer", "========================onNext" + s);
                        mDisposable.dispose();
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.i("ccer", "========================onError" + e.getMessage());
                    }

                    @Override
                    public void onComplete() {
                        Log.i("ccer", "========================onComplete");
                    }
                });
```

这个例子没有什么实用性，主要是为了说明流程。

先说一些类名，因为很多Rxjava2中有很多名字不太好记，所以先提前说一下 ，这样接下来看不至于混淆

RxJava2的链式使用就如同一个事件流，从上往下流，当然这是代码表象上看到的。

* Observable：被观察者；这个是事件流的起点。他是一个抽象类，是大部分事件流的鼻祖。（还有Flowable呢，还有我不知道的呢，还有等等呢）
* ObservableSource：接口，这个里面就一个订阅方法subscribe，用来订阅的，Observable实现了这个接口
* ObservableOnSubscribe：事件回调；接口，接口方法是subscribe，这个名字起得很让人疑惑，和订阅方法一个名字，只是参数不同
* ObservableEmitter：发射器接口，继承自Emitter；
* ObservableCreate：Observable对应的被观察者
* ObservableSubscribeOn：subscribeOn对应的被观察者
* ObservableObserveOn：observeOn对应的被观察者
* CreateEmitter：Observable对应的观察者
* SubscribeOnObserver：subscribeOn对应的观察者
* ObserveOnObserver：observeOn对应的观察者

>可以仔细看看，上面的名字很有规律的，基本上被观察者都是以Observable开头，然后后面对应着操作符名称，而观察者基本上都是以操作符开头以Observer结尾

这样接下来看源码就不会很混乱了

```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public static <T> Observable<T> create(ObservableOnSubscribe<T> source){
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
}
```
```
public ObservableCreate(ObservableOnSubscribe<T> source) {
    this.source = source;
}
```
```
public abstract class Observable<T> implements ObservableSource<T> {
    ...
}
```
```
@NonNull
public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
    Function<? super Observable, ? extends Observable> f = onObservableAssembly;
    if (f != null) {
        return apply(f, source);
    }
    return source;
}
```
create操作符中传入了一个ObservableOnSubscribe参数，因为Observable也实现了这个接口，所以也可以说他就是一个被观察者；对于一些陌生的代码我们先忽略，例如：RxJavaPlugins.onAssembly；
在create中就是创建了一个ObservableCreate，然后将当前的被观察者ObservableOnSubscribe（事件源头）传进去了，这时在ObservableCreate中持有了ObservableOnSubscribe（事件源头）的引用--source，然后将ObservableCreate返回。


```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.NONE)
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
    ObjectHelper.requireNonNull(mapper, "mapper is null");
    return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
}
```
```
public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
    super(source);
    this.function = function;
}
```
map中创建了一个ObservableMap，同时持有上一步中的被观察者(this)，也就是上一步中返回的ObservableCreate，并持有自己传进来的Function，然后将ObservableMap返回。

再来一次map，则又创建了一个ObservableMap，同时持有上一步中的被观察者(this)，也就是上一步中返回的ObservableMap，并持有自己传进来的Function，然后将ObservableMap返回。

```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> subscribeOn(Scheduler scheduler) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    return RxJavaPlugins.onAssembly(new ObservableSubscribeOn<T>(this, scheduler));
}
```
```
public ObservableSubscribeOn(ObservableSource<T> source, Scheduler scheduler) {
    super(source);
    this.scheduler = scheduler;
}
```
在subscribeOn中创建了一个ObservableSubscribeOn，同时持有上一步中的被观察者（this），也就是上一步中返回的ObservableMap，并持有传进来的Scheduler，然后将ObservableSubscribeOn返回。

再来一次subscribeOn；在subscribeOn中创建了一个ObservableSubscribeOn，同时持有上一步中的被观察者（this），也就是上一步中返回的ObservableSubscribeOn，并持有传进来的Scheduler，然后将ObservableSubscribeOn返回。

```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler) {
    return observeOn(scheduler, false, bufferSize());
}
```
```
@CheckReturnValue
@SchedulerSupport(SchedulerSupport.CUSTOM)
public final Observable<T> observeOn(Scheduler scheduler, boolean delayError, int bufferSize) {
    ObjectHelper.requireNonNull(scheduler, "scheduler is null");
    ObjectHelper.verifyPositive(bufferSize, "bufferSize");
    return RxJavaPlugins.onAssembly(new ObservableObserveOn<T>(this, scheduler, delayError, bufferSize));
}
```
```
public ObservableObserveOn(ObservableSource<T> source, Scheduler scheduler, boolean delayError, int bufferSize) {
    super(source);
    this.scheduler = scheduler;
    this.delayError = delayError;
    this.bufferSize = bufferSize;
}
```
在observeOn中创建了一个ObservableObserveOn，同时持有上一步中的被观察者（this），也就是上一步中返回的ObservableSubscribeOn，并持有传进来的Scheduler，同时将ObservableObserveOn返回。

到目前为止，一直都比较风平浪静，因为是搞事情的订阅还没来；他来了

```
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
    ObjectHelper.requireNonNull(observer, "observer is null");
    try {
        observer = RxJavaPlugins.onSubscribe(this, observer);

        ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");

        subscribeActual(observer);
    } catch (NullPointerException e) { // NOPMD
        throw e;
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        // can't call onError because no way to know if a Disposable has been set or not
        // can't call onSubscribe because the call might have set a Subscription already
        RxJavaPlugins.onError(e);

        NullPointerException npe = new NullPointerException("Actually not, but can't throw other exceptions due to RS");
        npe.initCause(e);
        throw npe;
    }
}
```
这里面就捕捉一行代码
```
@SchedulerSupport(SchedulerSupport.NONE)
@Override
public final void subscribe(Observer<? super T> observer) {
    ...
    subscribeActual(observer);
    ...
}
```
在Observable中，subscribeActual是个抽象方法，但是这个subscribe是由上一步返回的ObservableObserveOn发出的，所以调用的就是他的subscribeActual实现方法
```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    if (scheduler instanceof TrampolineScheduler) {
        source.subscribe(observer);
    } else {
        Scheduler.Worker w = scheduler.createWorker();

        source.subscribe(new ObserveOnObserver<T>(observer, w, delayError, bufferSize));
    }
}
```
```
@NonNull
public static Scheduler trampoline() {
    return TRAMPOLINE;
}
```
```
static {
    ...
    TRAMPOLINE = TrampolineScheduler.instance();
    ...
}
```
```
ObserveOnObserver(Observer<? super T> actual, Scheduler.Worker worker, boolean delayError, int bufferSize) {
    this.actual = actual;
    this.worker = worker;
    this.delayError = delayError;
    this.bufferSize = bufferSize;
}
```
这里的source就是当前被观察者ObservableObserveOn所持有的他的上一步返回的被观察者ObservableSubscribeOn，scheduler是observeOn中传进来的；
先判断这个线程调度，如果是当前线程（Schedulers.trampoline()熟悉吧！当前线程）没有发生任何线程调度直接订阅，否则就去做一个线程调度，当前的案例是调度到主线程（上面的demo），所以就到了那个else中，首先创建了一个Worker线程调度，然后将这个原始Observer和worker传到ObserveOnObserver中持有。

```
@Override
public void subscribeActual(final Observer<? super T> s) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

    s.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
再看看这个订阅方法，也就是ObservableSubscribeOn中的subscribeActual方法,先对上一步传过来的ObserveOnObserver进行包装成了SubscribeOnObserver，然后执行了他的onSubscribe(parent)方法，到目前为止还没有发生任何线程调度，所以这个方法就发生在当前线程。
然后scheduler进行了一个线程调度，在调度后的线程中执行了订阅。

> 在调度后的线程中执行了订阅

这句话很霸道，起码需要一个大篇幅的代码才能说得通，所以先放着，我会在这整个流程结束后再续写这一块是怎么调度的，不然这一块看完流程上又衔接不上了，目前就先知道他的作用就是线程调度就行了。

在订阅代码中又是和上面一样，只是这一次，source是ObservableMap，而MapObserver是在上一层的SubscribeOnObserver上再包一层，然后又调度线程。
上面这一个订阅过程是发生在上一个线程调度后的线程中，所以subscribeOn指定的是他的上一步的线程。

```
@Override
public void subscribeActual(Observer<? super U> t) {
    source.subscribe(new MapObserver<T, U>(t, function));
}
```
这一次的线程调度中执行的是ObservableMap中的订阅方法，这里的source是ObservableMap；只做了订阅，并对上一步中的SubscribeOnObserver进行了再包裹，并传入了自己的Function。

然后又是一次map，做同样处理，只不过这一次的被订阅者是ObservableCreate。
```
@Override
protected void subscribeActual(Observer<? super T> observer) {
    CreateEmitter<T> parent = new CreateEmitter<T>(observer);
    observer.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
将传过来的MapObserver包装成了一个CreateEmitter，然后执行了source.subscribe(parent)，这里的source是ObservableOnSubscribe，到终点了，是用来发射事件的回调。
```
@Override
public void onNext(T t) {
    if (t == null) {
        onError(new NullPointerException("onNext called with null. Null values are generally not allowed in 2.x operators and sources."));
        return;
    }
    if (!isDisposed()) {
        observer.onNext(t);
    }
}
```
这时候发出了OnNext()事件，这个OnNext是由CreateEmitter发出的，他是对MapObserver进行了包装，在他的OnNext方法中对事件流做了是否断流的判断，然后再调用MapObserver的OnNext事件
```
@Override
public void onNext(T t) {
    if (done) {
        return;
    }

    if (sourceMode != NONE) {
        actual.onNext(null);
        return;
    }

    U v;

    try {
        v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
    } catch (Throwable ex) {
        fail(ex);
        return;
    }
    actual.onNext(v);
}
```
精简一下
```
@Override
public void onNext(T t) {
    U v;
    v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value."); 
    actual.onNext(v);
}
```
获取到mapper（就是传进来的Function）的结果，然后作为参数传递给自己包装中的MapObserver中的OnNext，在案例中就是（""+s）；

在包裹中的MapObserver中的OnNext，做的是同样的操作，只不过，这里的MapObserver包裹的是SubscribeOnObserver，在他的OnNext中没有做什么，只是做了传递，传递给了他包裹着的SubscribeOnObserver，同样是传递，然后再OnNext中传递给了ObserveOnObserver，然后再他的OnNext中做事情
```
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
```
```
void schedule() {
    if (getAndIncrement() == 0) {
        worker.schedule(this);
    }
}
```
在这里做了线程调度，因为ObserveOnObserver本身实现了Runnable接口，所以这里就算是把自己传进去了，最终在run方法中做他包裹的Observer的OnNext方法的回调，在案例中，这里的Observer就是原始的Observer，就是最终的Observer了，所以最终Observer运行在上一个observeOn指定的线程中。

到这里整个流程就结束了，再来总结回顾一下：

**案例中，每个操作符都会返回对应的创建的被观察者**

* 事件流从create开始创建事件就是只是创造事件源(被观察者)，他创建了一个被观察者，然后持有一个ObservableOnSubscribe的引用。
* 接着到了第一个map，在这里也是只创建了一个被观察者ObservableMap，然后它持有了上一个ObservableOnSubscribe的引用，并保存了传入的function。
* 接着第二个map同理，创建了一个被观察者ObservableMap，然后它持有了上一个ObservableMap的引用，并保存了传入的function。
* 接着是subscribeOn，他创建了ObservableSubscribeOn对象，并持有了上一个被观察者ObservableMap的引用，并保存了传入的scheduler。
* 接着是subscribeOn，他创建了ObservableSubscribeOn对象，并持有了上一个被观察者ObservableSubscribeOn的引用，并保存了传入的scheduler。
* 接着是observeOn，他创建了ObservableObserveOn，并持有了上一个被观察者ObservableSubscribeOn的引用，并保存了传入的scheduler。
* 接着是subscribe，调用了subscribeActual(observer)方法，后续的基本上都走这个方法。
* 这个subscribe的直接调用者上一步中被观察者，也就是ObservableObserveOn；他在subscribeActual中对observeOn传入的线程对了判断，如果是当前线程则直接订阅，否则做线程调度订阅，当然目前只是把调度机制传入到ObserveOnObserver中，还没有操作。
* 在这个subscribeActual中的source就是ObservableObserveOn所持有的上一步的被观察者也就是ObservableSubscribeOn，所以就是触发了ObservableSubscribeOn中的subscribeActual方法，在这里面触发了onSubscribe();然后进行了线程调度，在调度中线程中又执行了订阅；
* 这一次的订阅中的source是当前被观察者所持有的上一步中的被观察者，就是第一个subscribeOn中创建的ObservableSubscribeOn；
* 调用了ObservableSubscribeOn中的subscribeActual方法，从这里开始，他就运行在了第二个subscribeOn所指定的线程中了；
* 在这里又做了一个线程调度，然后在调度线程中做了订阅，这里的source是他的上一个步中的观察者，也就是ObservableMap；
* 在ObservableMap中的subscribeActual方法中又做了订阅；
* 又往上走，到了第一个map，同样在subscribeActual中做了订阅，但是这里的source是ObservableCreate，在这里面触发了ObservableOnSubscribe的回调subscribe(ObservableEmitter)方法，算是订阅完了，开始发射事件；
* 这个事件是哪里来的呢，每次在执行subscribeActual的时候，都会对底部传上来的Observer进行一次包装，然后将自己想加入的东西加入进去，通过代理模式下发事件；
* 当前案例中就是订阅开始的时候observeOn的subscribeActual中如果做了线程调度就会将原始的Observer包装成ObserveOnObserver，再经过subscribeOn中的subscribeActual方法，又将ObserveOnObserver包装成SubscribeOnObserver；又一次subscribeOn，将SubscribeOnObserver又包裹上一层SubscribeOnObserver；到了map又包装成了MapObserver，同样两次map，包裹又一层MapObserver，最后到了ObservableCreate中的subscribeActual，将上面的MapObserver包裹成了CreateEmitter。
* 上面这个就像一个场景：我们定义一个主题叫做居住，就开始运作；我们在空地上放一张床，这是居住，在床周围搭建了一个茅草屋，床现在在茅草屋里面，这也叫居住；然后我们在周围养了两只鸡，然后围了一圈篱笆，这样也叫居住，然后我们搭建了一个村落，这也叫居住，然后我们搭起了城墙，建成了一座城池，这也叫居住，等等一直到我们成立一个国家，这都叫居住。从一张床到一个国家，我们最终目的都是居住，只是中间被我们增加了很多东西
* 回到代码中，我们在ObservableCreate的subscribeActual中开始发射事件，假设发射了一个OnNext事件，会先调用CreateEmitter的OnNext，在这里做了是否断流判断，然后往下调用了MapObserver的OnNext方法，在这里，获取到传进来的function的结果，将他作为参数传递给下一个ObservableMap的OnNext中，下一个ObservableMap做同理操作，然后将他传递给SubscribeOnObserver的OnNext，然后又将他传递给SubscribeOnObserver的OnNext（案例里两次subscribeOn），然后传递给ObservableObserveOn的OnNext，在OnNext中做了线程调度，然后传递给了最终原始的Observer的OnNext
* 在这中间从下往上开始第一个subscribeOn中的线程调度开始，也就是第二个subscribeOn中的subscribeActual开始，后面的代码都运行在当前线程（是订阅往上走的流程，往上订阅），然后碰到了第一个subscribeOn，线程发生变化，然后之后的线程跟着第一个subscribeOn走（是订阅往上走的流程，往上订阅），一直到订阅结束，然后往下发射事件同理，仍然在这个线程中，一直到最后一个Observer的事件回调发生在observeOn指定的线程中。所以subscribeOn是往上，observeOn是往下。

再来说说之前提到的那个比较霸道的线程切换；

在ObservableSubscribeOn中的subscribeActual中
```
@Override
public void subscribeActual(final Observer<? super T> s) {
    final SubscribeOnObserver<T> parent = new SubscribeOnObserver<T>(s);

    s.onSubscribe(parent);

    parent.setDisposable(scheduler.scheduleDirect(new SubscribeTask(parent)));
}
```
```
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
```
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run) {
    return scheduleDirect(run, 0L, TimeUnit.NANOSECONDS);
}
```
```
@NonNull
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
从上面的代码可以看出，先创建了一个SubscribeTask，他实现了Runnable，并在run方法中指定了订阅操作，接下来就看看这个run是在哪被调度的；
在scheduleDirect方法中
```
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
先创建了一个Worker，通过createWorker()方法，这个createWorker()也是抽象方法，是由传入的Schedulers指定的，案例中我们传入的是Schedulers.io()
```
@NonNull
public static Scheduler io() {
    return RxJavaPlugins.onIoScheduler(IO);
}
```
```
static {
    IO = RxJavaPlugins.initIoScheduler(new IOTask());
}
```
```
static final class IOTask implements Callable<Scheduler> {
    @Override
    public Scheduler call() throws Exception {
        return IoHolder.DEFAULT;
    }
}
```
```
static final class IoHolder {
    static final Scheduler DEFAULT = new IoScheduler();
}
```
```
public IoScheduler() {
    this(WORKER_THREAD_FACTORY);
}
```
经过层层追溯，可以看到，最终调用的应该是IoScheduler的createWorker方法
```
public Worker createWorker() {
    return new EventLoopWorker(pool.get());
}
```
```
EventLoopWorker(CachedWorkerPool pool) {
    this.pool = pool;
    this.tasks = new CompositeDisposable();
    this.threadWorker = pool.get();
}
```
这里创建了一个EventLoopWorker
再回到这个代码，将Worker和我们传进来的这个Runnable包装成了DisposeTask，然后调用了Worker的schedule方法
```
public Disposable scheduleDirect(@NonNull Runnable run, long delay, @NonNull TimeUnit unit) {
    final Worker w = createWorker();

    final Runnable decoratedRun = RxJavaPlugins.onSchedule(run);

    DisposeTask task = new DisposeTask(decoratedRun, w);

    w.schedule(task, delay, unit);

    return task;
}
```
```
public Disposable schedule(@NonNull Runnable action, long delayTime, @NonNull TimeUnit unit) {
    if (tasks.isDisposed()) {
        // don't schedule, we are unsubscribed
        return EmptyDisposable.INSTANCE;
    }

    return threadWorker.scheduleActual(action, delayTime, unit, tasks);
}
```
```
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
在这里再精简一下
```
if (delayTime <= 0) {
    f = executor.submit((Callable<Object>)sr);
} else {
    f = executor.schedule((Callable<Object>)sr, delayTime, unit);
}
```
```
private final ScheduledExecutorService executor;

public NewThreadWorker(ThreadFactory threadFactory) {
    executor = SchedulerPoolFactory.create(threadFactory);
}
```
在这里做到了线程的调度，其中太就不去追究了，到这里基本上就完成了线程的调度。