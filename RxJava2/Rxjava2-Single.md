# Rxjava2-Single

```
Single
                .create(new SingleOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(SingleEmitter<Integer> emitter) throws Exception {
                        emitter.onSuccess(2);
                        emitter.onSuccess(6);
//                        emitter.onError(new Exception("test exception"));
                    }
                })
                .map(new Function<Integer, Integer>() {
                    @Override
                    public Integer apply(Integer integer) throws Exception {
                        return integer * 3;
                    }
                })
                .subscribeOn(Schedulers.io())
                .observeOn(Schedulers.trampoline())
                .subscribe(new SingleObserver<Integer>() {

                    @Override
                    public void onSubscribe(Disposable d) {

                    }

                    @Override
                    public void onSuccess(Integer integer) {
                        System.out.println("" + integer);
                    }

                    @Override
                    public void onError(Throwable e) {
                        System.out.println("" + e.toString());
                    }
                });
```
从上面就可以看出这就是一个缩减版的Observable；有被观察者（Single），有观察者（SingleObserver），还有订阅关系（subscribe），还有事件流开关（Disposable）

在create里面发射事件时，我们做这么几种情况
```
emitter.onSuccess(2);
emitter.onSuccess(6);
emitter.onError(new Exception("test exception"));
```
```
打印结果://这里的6不好误会为第二个数字6，因为上面用到了map操作符，做了*3操作，2*3=6；
6
```
```
//emitter.onSuccess(2);
//emitter.onSuccess(6);
emitter.onError(new Exception("test exception"));
```
```
打印结果：
java.lang.Exception: test exception
```
```
emitter.onError(new Exception("test exception"));
emitter.onSuccess(2);
emitter.onSuccess(6);
```
```
打印结果：
java.lang.Exception: test exception
```
可以看出：**只发射一条单一的数据，或者一条异常通知，不能发射完成通知，其中数据与通知只能发射一个。**

来看看源码，简化一下使用，去掉操作符
```
Single
                .create(new SingleOnSubscribe<Integer>() {
                    @Override
                    public void subscribe(SingleEmitter<Integer> emitter) throws Exception {
                        emitter.onError(new Exception("test exception"));
                        emitter.onSuccess(2);
                        emitter.onSuccess(6);
                    }
                })
                .subscribe(new SingleObserver<Integer>() {

                    @Override
                    public void onSubscribe(Disposable d) {

                    }

                    @Override
                    public void onSuccess(Integer integer) {
                        System.out.println("" + integer);
                    }

                    @Override
                    public void onError(Throwable e) {
                        System.out.println("" + e.toString());
                    }
                });
```

从create出发：
```
public static <T> Single<T> create(SingleOnSubscribe<T> source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new SingleCreate<T>(source));
}
```
create的参数是SingleOnSubscribe，
```
public interface SingleOnSubscribe<T> {
    void subscribe(@NonNull SingleEmitter<T> emitter) throws Exception;
}
```
subscribe的参数是SingleEmitter
```
public interface SingleEmitter<T> {
    void onSuccess(@NonNull T t);
    void onError(@NonNull Throwable t);
    void setDisposable(@Nullable Disposable s);
    void setCancellable(@Nullable Cancellable c);
    boolean isDisposed();
    boolean tryOnError(@NonNull Throwable t);
}
```
我们将SingleOnSubscribe传入到create中，当他的回调方法subscribe被调用时，就可以在里面发射一些事件，其中事件的类型主要是onSuccess/onError


再说说create，它返回了一个SingleCreate被观察者
```
public SingleCreate(SingleOnSubscribe<T> source) {
    this.source = source;
}
```
这里主要是对传入create中的SingleOnSubscribe做了保存，往下看，我们先不管中间那些操作符，直接看最简单的订阅方法
```
public final void subscribe(SingleObserver<? super T> subscriber) {
    ObjectHelper.requireNonNull(subscriber, "subscriber is null");

    subscriber = RxJavaPlugins.onSubscribe(this, subscriber);

    ObjectHelper.requireNonNull(subscriber, "subscriber returned by the RxJavaPlugins hook is null");

    try {
        subscribeActual(subscriber);
    } catch (NullPointerException ex) {
        throw ex;
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        NullPointerException npe = new NullPointerException("subscribeActual failed");
        npe.initCause(ex);
        throw npe;
    }
}
```
这里和Observable一样，千言万语最后都到了subscribeActual(subscriber)，这个subscribe方法是他的上游中的SingleCreate发出了，所以就对应着SingleCreate的subscribeActualfagnfa
```
protected void subscribeActual(SingleObserver<? super T> s) {
        Emitter<T> parent = new Emitter<T>(s);
    s.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
在这里可以看到，他对创建来的SingleObserver做了包装，成了Emitter，在这里调用了SingleObserver的onSubscribe方法；然后触发了他(SingleCreate)的source的subscribe方法，这里他的source是啥呢，还记得create方法吗，返回了一个SingleCreate，
```
public SingleCreate(SingleOnSubscribe<T> source) {
    this.source = source;
}
```
没错，他的source就是SingleOnSubscribe，然后调用了sourced的subscribe方法，也就是他的subscribe方法，也就是我们要发射的事件；
案例中我们发射了一个onSuccess/onError事件，通过调用SingleEmitter的onSuccess/onError方法；而在SingleCreate中SingleEmitter的实现类是Emitter
```
static final class Emitter<T>
    extends AtomicReference<Disposable>
    implements SingleEmitter<T>, Disposable {

        final SingleObserver<? super T> actual;

        Emitter(SingleObserver<? super T> actual) {
            this.actual = actual;
        }


        private static final long serialVersionUID = -2467358622224974244L;

        @Override
        public void onSuccess(T value) {
            if (get() != DisposableHelper.DISPOSED) {
                Disposable d = getAndSet(DisposableHelper.DISPOSED);
                if (d != DisposableHelper.DISPOSED) {
                    try {
                        if (value == null) {
                            actual.onError(new NullPointerException("onSuccess called with null. Null values are generally not allowed in 2.x operators and sources."));
                        } else {
                            actual.onSuccess(value);
                        }
                    } finally {
                        if (d != null) {
                            d.dispose();
                        }
                    }
                }
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
            if (get() != DisposableHelper.DISPOSED) {
                Disposable d = getAndSet(DisposableHelper.DISPOSED);
                if (d != DisposableHelper.DISPOSED) {
                    try {
                        actual.onError(t);
                    } finally {
                        if (d != null) {
                            d.dispose();
                        }
                    }
                    return true;
                }
            }
            return false;
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
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
    }
```
可以先看看他的构造方法，
```
Emitter(SingleObserver<? super T> actual) {
    this.actual = actual;
}
```
将我们开始传进来的SingleObserver进行了存储
看看onSuccess中方法是怎么操作的
```
public void onSuccess(T value) {
    if (get() != DisposableHelper.DISPOSED) {
        Disposable d = getAndSet(DisposableHelper.DISPOSED);
        if (d != DisposableHelper.DISPOSED) {
            try {
                if (value == null) {
                    actual.onError(new NullPointerException("onSuccess called with null. Null values are generally not allowed in 2.x operators and sources."));
                } else {
                    actual.onSuccess(value);
                }
            } finally {
                if (d != null) {
                    d.dispose();
                }
            }
        }
    }
}
```
获取了当前事件流的开关状态
```
Disposable d = getAndSet(DisposableHelper.DISPOSED);
```
然后判断当前是否已经断流，如果断流则什么都不操作；如果没断流则去判断传入的值，这里可以看出，这里是不允许传入null的，会回调onError，并传入一个NullPointerException异常，否则回调传入的SingleObserver的onSuccess，这里最后看那个finally，在这里它做了断流操作；案例中我们发射多个事件，当第一个事件到达后就直接断流了，所以后续事件虽然发射了但是没有观察者接收。