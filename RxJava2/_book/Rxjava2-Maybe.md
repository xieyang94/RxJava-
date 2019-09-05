# Rxjava2-Maybe

**Maybe:可以发射一条单一的数据以及一条异常通知或者一条完成通知，需要注意的是，异常通知和完成通知只能选择其中一个，发射数据只能在完成通知或者异常通知之前，否则发送数据无效**
```
Maybe.just(10).subscribe(new MaybeObserver<Integer>() {
    @Override
    public void onSubscribe(Disposable d) {
        if (d != null) {
            System.out.println("onSubscribe:" + d.isDisposed());
        }
    }

    @Override
    public void onSuccess(Integer integer) {
        System.out.println("onSuccess:" + integer);
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e);
    }

    @Override
    public void onComplete() {
        System.out.println("onComplete");
    }
        });
```
```
打印结果：
onSubscribe:true
onSuccess:10
```
```
Maybe.empty().subscribe(new MaybeObserver<Object>() {
    @Override
    public void onSubscribe(Disposable d) {
        System.out.println("onSubscribe:" + d);
    }

    @Override
    public void onSuccess(Object o) {
        System.out.println("onSuccess:" + o);
    }

    @Override
    public void onError(Throwable e) {
        System.out.println("onError:" + e);
    }

    @Override
    public void onComplete() {
        System.out.println("onComplete");
    }
});
```
```
结果打印：
onSubscribe:INSTANCE
onComplete
```
看看代码的调用比较简单，这里就直接去看源码：
```
public static <T> Maybe<T> just(T item) {
    ObjectHelper.requireNonNull(item, "item is null");
    return RxJavaPlugins.onAssembly(new MaybeJust<T>(item));
}
```
Maybe只能使用单参数的just，然后在just中返回了一个MaybeJust
```
public MaybeJust(T value) {
    this.value = value;
}
```
将just传入的值进行了保存
然后在订阅方法subscribe中,传入了MaybeObserver

```
public final void subscribe(MaybeObserver<? super T> observer) {
    ...
    subscribeActual(observer);
    ...
}
```
还是一如既往的调用subscribeActual(observer);
也就是上一个被观察者MaybeJust的subscribeActual方法
```
@Override
protected void subscribeActual(MaybeObserver<? super T> observer) {
    observer.onSubscribe(Disposables.disposed());
    observer.onSuccess(value);
}
```
```
public interface MaybeObserver<T> {
    void onSubscribe(@NonNull Disposable d);
    void onSuccess(@NonNull T t);
    void onError(@NonNull Throwable e);
    void onComplete();
}
```
这里面的代码真的是很少，MaybeObserver甚至是都没有被包装，直接就调用了；先调用了他的onSubscribe方法
```
public static Disposable disposed() {
    return EmptyDisposable.INSTANCE;
}
```
```
public boolean isDisposed() {
    return this == INSTANCE;
}
```
返回一个空的单例，看名字就知道，是个空EmptyDisposable，而且isDisposed()方法也很明显，绝对的true；
接着看subscribeActual方法，然后调用了onSuccess方法将结果转发，就没了。
源码就这么多。。。

再看看empty()方法
```
public static <T> Maybe<T> empty() {
    return RxJavaPlugins.onAssembly((Maybe<T>)MaybeEmpty.INSTANCE);
}
```
```
public final void subscribe(MaybeObserver<? super T> observer) {
    ...
    subscribeActual(observer);
    ...
```
```
public final class MaybeEmpty extends Maybe<Object> implements ScalarCallable<Object> {

    public static final MaybeEmpty INSTANCE = new MaybeEmpty();

    @Override
    protected void subscribeActual(MaybeObserver<? super Object> observer) {
        EmptyDisposable.complete(observer);
    }

    @Override
    public Object call() {
        return null; // nulls of ScalarCallable are considered empty sources
    }
}
```
```
public static void complete(MaybeObserver<?> s) {
    s.onSubscribe(INSTANCE);
    s.onComplete();
}
```
可以看出，源码更少，甚至觉得有点耍我们，就是调用了onSubscribe方法和onComplete方法，也没有别的操作。