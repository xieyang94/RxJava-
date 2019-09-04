# RxJava2

[TOC]


### 添加依赖：
```
implementation 'io.reactivex.rxjava2:rxjava:2.1.4'
implementation 'io.reactivex.rxjava2:rxandroid:2.0.2'
```
### 使用：
```
        Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> emitter) {
                        emitter.onNext(22245);
                        emitter.onComplete();
                        Log.i("ccer", "================currentThread name: " + Thread.currentThread().getName());
                    }
                })
                .subscribe(new Observer<Integer>() {
                    @Override
                    public void onSubscribe(Disposable d) {
                        Log.i("ccer", "========================onSubscribe");
                    }

                    @Override
                    public void onNext(Integer integer) {
                        Log.i("ccer", "========================onNext");
                    }

                    @Override
                    public void onError(Throwable e) {
                        Log.i("ccer", "========================onError");
                    }

                    @Override
                    public void onComplete() {
                        Log.i("ccer", "========================onComplete");
                    }
                });
```

```
05-27 10:26:09.754 7192-7192/com.example.administrator.myapplication I/ccer: ========================onSubscribe
05-27 10:26:09.754 7192-7192/com.example.administrator.myapplication I/ccer: ========================onNext
05-27 10:26:09.754 7192-7192/com.example.administrator.myapplication I/ccer: ========================onComplete
05-27 10:26:09.754 7192-7192/com.example.administrator.myapplication I/ccer: ================currentThread name: main
```

### 原理分析
#### create
```
Observable
                .create(new ObservableOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(ObservableEmitter<Integer> emitter) {
                        emitter.onNext(22245);
                        emitter.onComplete();
                        Log.i("ccer", "================currentThread name: " + Thread.currentThread().getName());
                    }
                })
```
首先是这一部分代码，调用了create方法，参数是ObservableOnSubscribe；

#### ObservableOnSubscribe
```
public interface ObservableOnSubscribe<T> {

    /**
     * Called for each Observer that subscribes.
     * @param emitter the safe emitter instance, never null
     * @throws Exception on error
     */
    void subscribe(@NonNull ObservableEmitter<T> emitter) throws Exception;
}
```
ObservableOnSubscribe是一个接口，接口方法是subscribe，里面有一个参数ObservableEmitter

#### ObservableEmitter
```
public interface ObservableEmitter<T> extends Emitter<T> {

    /**
     * Sets a Disposable on this emitter; any previous Disposable
     * or Cancellation will be unsubscribed/cancelled.
     * @param d the disposable, null is allowed
     */
    void setDisposable(@Nullable Disposable d);

    /**
     * Sets a Cancellable on this emitter; any previous Disposable
     * or Cancellation will be unsubscribed/cancelled.
     * @param c the cancellable resource, null is allowed
     */
    void setCancellable(@Nullable Cancellable c);

    /**
     * Returns true if the downstream disposed the sequence.
     * @return true if the downstream disposed the sequence
     */
    boolean isDisposed();

    /**
     * Ensures that calls to onNext, onError and onComplete are properly serialized.
     * @return the serialized ObservableEmitter
     */
    @NonNull
    ObservableEmitter<T> serialize();

    /**
     * Attempts to emit the specified {@code Throwable} error if the downstream
     * hasn't cancelled the sequence or is otherwise terminated, returning false
     * if the emission is not allowed to happen due to lifecycle restrictions.
     * <p>
     * Unlike {@link #onError(Throwable)}, the {@code RxJavaPlugins.onError} is not called
     * if the error could not be delivered.
     * @param t the throwable error to signal if possible
     * @return true if successful, false if the downstream is not able to accept further
     * events
     * @since 2.1.1 - experimental
     */
    @Experimental
    boolean tryOnError(@NonNull Throwable t);
}
```
这个发射器也是一个接口，继承自Emitter，先一追到底，然后反过来看。

#### Emitter
```
public interface Emitter<T> {

    /**
     * Signal a normal value.
     * @param value the value to signal, not null
     */
    void onNext(@NonNull T value);

    /**
     * Signal a Throwable exception.
     * @param error the Throwable to signal, not null
     */
    void onError(@NonNull Throwable error);

    /**
     * Signal a completion.
     */
    void onComplete();
}
```

发射器接口，里面的三个方法是我们常见的三个方法[onNext/onError/onComplete]，一般在被观察者(Observable)中的回调去发射这三个方法（可以看看我们一开始的使用案例）

- void onNext(@NonNull T value)：发射事件，这个可以发射无数次；
- void onError(@NonNull Throwable error)：发射一个错误事件，一旦发射就没有后续发送事件的可能了，这个事件流就断了（就是发射onError之后再发射onNext是不可能了，接收不到）
- void onComplete()：发射一个结束事件；同理onError，而且onError和onComplete是只能发生其中的一个，因为他们都是结束事件流，谁先发射谁就是接收事件的最后一个。

> 发射和接收是不相关的，你可以发射N个事件，但是如果Observer已经关闭了这个流，则不管怎么发送也接收不到了

现在我们追到底了，然后回溯；ObservableEmitter继承自Emitter，在这三个方法的基础上增加了[setDisposable/setCancellable/isDisposed/serialize/tryOnError]这五个方法

- setDisposable：设置一个Disposable
- setCancellable：设置一个Cancelable
- isDisposed：判断是否已经断流
- serialize：序列化
- tryOnError：

再往上追溯我们就回来了create参数那；

回顾一下：
ObservableOnSubscribe就是一个接口，接口方法中的参数是一个发射器，ObservableEmitter，他继承自Emitter，里面定义了一些发射事件，成功失败和设置流开关的一些方法。

再来看看create方法：

```
    @CheckReturnValue
    @SchedulerSupport(SchedulerSupport.NONE)
    public static <T> Observable<T> create(ObservableOnSubscribe<T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```
先是进行null判断，然后做了一个包装处理
#### RxJavaPlugins.onAssembly()
```
    /**
     * Calls the associated hook function.
     * @param <T> the value type
     * @param source the hook's input value
     * @return the value returned by the hook
     */
    @SuppressWarnings({ "rawtypes", "unchecked" })
    @NonNull
    public static <T> Observable<T> onAssembly(@NonNull Observable<T> source) {
        Function<? super Observable, ? extends Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```
先忽略上面的代码，就知道是直接返回这个参数，也就是ObservableCreate

#### ObservableCreate
```
public final class ObservableCreate<T> extends Observable<T> {
    final ObservableOnSubscribe<T> source;

    public ObservableCreate(ObservableOnSubscribe<T> source) {
        this.source = source;
    }

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
    ...
    CreateEmitter
    ...
    SerializedEmitter
    ...
}    
```
就当前的代码调用而言，就是ObservableCreate中有一个成员变量ObservableOnSubscribe，将传递过来的参数存储下来。然后回溯。这样create方法就结束了，至于ObservableOnSubscribe里面的subscribe方法中的调用，因为这是一个回调，所以暂时是不会触发的（你懂的）。

#### Observer
订阅方法subscribe,传进来的了一个Observer（观察者）参数

```
public interface Observer<T> {

    /**
     * Provides the Observer with the means of cancelling (disposing) the
     * connection (channel) with the Observable in both
     * synchronous (from within {@link #onNext(Object)}) and asynchronous manner.
     * @param d the Disposable instance whose {@link Disposable#dispose()} can
     * be called anytime to cancel the connection
     * @since 2.0
     */
    void onSubscribe(@NonNull Disposable d);

    /**
     * Provides the Observer with a new item to observe.
     * <p>
     * The {@link Observable} may call this method 0 or more times.
     * <p>
     * The {@code Observable} will not call this method again after it calls either {@link #onComplete} or
     * {@link #onError}.
     *
     * @param t
     *          the item emitted by the Observable
     */
    void onNext(@NonNull T t);

    /**
     * Notifies the Observer that the {@link Observable} has experienced an error condition.
     * <p>
     * If the {@link Observable} calls this method, it will not thereafter call {@link #onNext} or
     * {@link #onComplete}.
     *
     * @param e
     *          the exception encountered by the Observable
     */
    void onError(@NonNull Throwable e);

    /**
     * Notifies the Observer that the {@link Observable} has finished sending push-based notifications.
     * <p>
     * The {@link Observable} will not call this method if it calls {@link #onError}.
     */
    void onComplete();
}
```
四个方法，是不是和发射器的那三个方法很像，基本就对应着回调，就是代理模式。
void onSubscribe(@NonNull Disposable d);这个方法则是一个用于性能优化的方法，可以用来断流。

回溯看看订阅方法subscribe
#### subscribe
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
一开始是个空检查，中间也有一些空检查和异常处理，总结就两行调用
```
observer = RxJavaPlugins.onSubscribe(this, observer);

subscribeActual(observer);
```

#### RxJavaPlugins.onSubscribe(this, observer)
```
    /**
     * Calls the associated hook function.
     * @param <T> the value type
     * @param source the hook's input value
     * @param observer the observer
     * @return the value returned by the hook
     */
    @SuppressWarnings({ "rawtypes", "unchecked" })
    @NonNull
    public static <T> Observer<? super T> onSubscribe(@NonNull Observable<T> source, @NonNull Observer<? super T> observer) {
        BiFunction<? super Observable, ? super Observer, ? extends Observer> f = onObservableSubscribe;
        if (f != null) {
            return apply(f, source, observer);
        }
        return observer;
    }
```
这个和之前的create方法中的RxJavaPlugins.onAssembly()差不多，忽略一些代码后就直接返回了一个Observer；
然后调用了subscribeActual方法，他是一个抽象方法；我们目前处在Observable中，他的实现类是我们之前在create中RxJavaPlugins.onAssembly()中创建的ObservableCreate对象，我们就去看看这个方法

#### subscribeActual
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


创建一个发射器CreateEmitter
```
static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {


        private static final long serialVersionUID = -3434801548987643227L;

        final Observer<? super T> observer;

        CreateEmitter(Observer<? super T> observer) {
            this.observer = observer;
        }

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

        @Override
        public void onError(Throwable t) {
            if (!tryOnError(t)) {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public boolean tryOnError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                    dispose();
                }
                return true;
            }
            return false;
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                    dispose();
                }
            }
        }

        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }

        @Override
        public void setCancellable(Cancellable c) {
            setDisposable(new CancellableDisposable(c));
        }

        @Override
        public ObservableEmitter<T> serialize() {
            return new SerializedEmitter<T>(this);
        }

        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }

```
继承自AtomicReference（这个我也不清楚），实现了ObservableEmitter和Disposable接口，
```
/**
 * Represents a disposable resource.
 */
public interface Disposable {
    /**
     * Dispose the resource, the operation should be idempotent.
     */
    void dispose();

    /**
     * Returns true if this resource has been disposed.
     * @return true if this resource has been disposed
     */
    boolean isDisposed();
}
```

上面两个接口中的方法，我们应该熟悉；
ObservableEmitter是继承自Emitter实现了三个基本方法并且增加关于事件流和序列化以及Cancelable方法。
CreateEmitter相对于Observer就是一个代理类，在subscribeActual方法中，将我们传进来的Observer通过CreateEmitter进行了代理。

发射器构架好了，然后调用了Observer中的onSubscribe方法，这样可以在发射事件之前拿到事件源，这样就可以随时断流。

然后调用我们一开始create的回调subscribe，在这里面我们进行事件发射，这样整个事件流就发射出去了。
>这里还可以看到，onComplete和onError方法是我们手动调用触发，不会自动触发，只有在发射事件的时候遇到不可抗力(int i=1/0;)时，才会try-catch中调用onError.

回顾一下：
订阅的时候，我们传入一个Observer，并实现了他的四个方法，然后将Observer通过CreateEmitter包装代理，调用Observer的onSubscribe方法，获取到事件源；最后调用subscribe(这个方法名不知道为什么要这么起名字，之前叫做call不是挺好的嘛)方法发射事件。

整个回顾一下：
Observable通过静态方法create创建一个被观察者Observable，是怎么创建的呢，在create中传入了一个ObservableOnSubscribe参数(他是只有一个接口方法subscribe的接口，而且这个接口方法的参数是一个发射器ObservableEmitter，继承自Emitter，有一些常规的方法以及对Observer的代理调用)，在create方法内部创建一个ObservableCreate对象，他是Observable的子类，构造内部对ObservableOnSubscribe进行了保存，然后通过RxJavaPlugins.onAssembly进行返回一个Observable。
然后就是订阅方法，在订阅方法中传入了一个Observer的实现类，并通过ObservableEmitter进行了包装代理，先调用了onSubscribe方法获取到事件源，也就是发射器（CreateEmitter实现了Disposable），最后调用之前在create中传进来的ObservableOnSubscribe的subscribe进行回调发射事件，触发Observer中的方法。