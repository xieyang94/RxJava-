# Rxjava2-Completable

**只发射一条完成通知，或者一条异常通知，不能发射数据，其中完成通知与异常通知只能发射一个**
```
Completable
    .create(new CompletableOnSubscribe() {
        @Override
        public void subscribe(CompletableEmitter emitter) throws Exception {
            System.out.println("subscribe:" + emitter);
            emitter.onComplete();
        }
    })
    .subscribe(new CompletableObserver() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe:" + d);
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError:");
        }
    });
```
```
结果打印：
onSubscribe:null
subscribe:null
onComplete
```
```
Completable
    .fromRunnable(new Runnable() {
        @Override
        public void run() {
            System.out.println("run");
        }
    })
    .subscribe(new CompletableObserver() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe:" + d);
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError:");
        }
    });
```
```
结果打印：
onSubscribe:RunnableDisposable(disposed=false, EmptyRunnable)
run
onComplete
```
```
Completable
    .complete()
    .subscribe(new CompletableObserver() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe:" + d);
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError:");
        }
    });
```
```
结果打印:
onSubscribe:INSTANCE
onComplete
```
调用也是很简单，看看源码：
### create
```
public static Completable create(CompletableOnSubscribe source) {
    ObjectHelper.requireNonNull(source, "source is null");
    return RxJavaPlugins.onAssembly(new CompletableCreate(source));
}
```
返回了一个CompletableCreate
```
public CompletableCreate(CompletableOnSubscribe source) {
    this.source = source;
}
```
对事件源做了保存
然后开始订阅subscribe：
```
public final void subscribe(CompletableObserver s) {
    ...
    subscribeActual(s);
    ...
}
```
也就是上一步中的被观察CompletableCreate中的subscribeActual方法
```
protected void subscribeActual(CompletableObserver s) {
    Emitter parent = new Emitter(s);
    s.onSubscribe(parent);

    try {
        source.subscribe(parent);
    } catch (Throwable ex) {
        Exceptions.throwIfFatal(ex);
        parent.onError(ex);
    }
}
```
对传入的观察者做了new Emitter(s)
```
Emitter(CompletableObserver actual) {
    this.actual = actual;
}
```
然后调用了CompletableObserver的onSubscribe方法，然后执行了source.subscribe(parent);
这个source就是上一步中的被观察者CompletableCreate的构造中传入的CompletableOnSubscribe，就是调用了他的subscribe方法，然后就开始发射我们定义的事件，一个onComplete事件，再来看看这个onComplete是怎么定义的，还是由Emitter发射的
```
public void onComplete() {
    if (get() != DisposableHelper.DISPOSED) {
        Disposable d = getAndSet(DisposableHelper.DISPOSED);
        if (d != DisposableHelper.DISPOSED) {
            try {
                actual.onComplete();
            } finally {
                if (d != null) {
                    d.dispose();
                }
            }
        }
    }
}
```
先判断是否已经断流，然后调用传入的CompletableObserver的onComplete方法，最后设置为断流d.dispose()，所以可以发射多次事件，但是只能接受一次事件；

### fromRunnable
```
public static Completable fromRunnable(final Runnable run) {
    ObjectHelper.requireNonNull(run, "run is null");
    return RxJavaPlugins.onAssembly(new CompletableFromRunnable(run));
}
```
返回一个CompletableFromRunnable根据惯例，最终调用他的subscribeActual方法
```
protected void subscribeActual(CompletableObserver s) {
    Disposable d = Disposables.empty();
    s.onSubscribe(d);
    try {
        runnable.run();
    } catch (Throwable e) {
        Exceptions.throwIfFatal(e);
        if (!d.isDisposed()) {
            s.onError(e);
        }
        return;
    }
    if (!d.isDisposed()) {
        s.onComplete();
    }
}
```
执行一个自定义的Runnable，如果还没有断流则还执行以下s.onComplete()，因为这是一个同步过程，所以总run方法执行完成了才判断是否断流来判断是否执行s.onComplete()。
### complete
```
public static Completable complete() {
    return RxJavaPlugins.onAssembly(CompletableEmpty.INSTANCE);
}
```
```
public final class CompletableEmpty extends Completable {
    public static final Completable INSTANCE = new CompletableEmpty();

    private CompletableEmpty() {
    }

    @Override
    public void subscribeActual(CompletableObserver s) {
        EmptyDisposable.complete(s);
    }
}
```
```
public static void complete(CompletableObserver s) {
    s.onSubscribe(INSTANCE);
    s.onComplete();
}
```
可以看出这个比较简单，几乎一气呵成，就是简单直接的调用了CompletableObserver的onSubscribe和onComplete方法，和Maybe的empty同出一辙。

### 其他

第一个demo中返回结果打印了两个null，不是说引用为null，而是值为null

```
结果打印：
onSubscribe:null
subscribe:null
onComplete
```

![TIM截图20190603124241.png-6.3kB][1]

因为Emitter是实现了Disposable接口，所以他们俩实际上是同一个东西
```
Emitter parent = new Emitter(s);
s.onSubscribe(parent);
```
以上就可以看出他们是同一个东西；
```
static final class Emitter
    extends AtomicReference<Disposable>
    implements CompletableEmitter, Disposable 
```
可以看出Emitter继承自AtomicReference，这个value值就是AtomicReference中的value，那他是在何时赋值的呢？
在onComplete和tryOnError中有这么一行代码

```
Disposable d = getAndSet(DisposableHelper.DISPOSED);
```
```
public final V getAndSet(V newValue) {
    return (V)U.getAndSetObject(this, VALUE, newValue);
}
```
所以我们在onComplete中断点看一下

![TIM截图20190603125042.png-9.1kB][2]

可以看出value已经有值了。

---
[1]: http://static.zybuluo.com/xiey/47yisp7rqvvx4227l790e26r/TIM%E6%88%AA%E5%9B%BE20190603124241.png
[2]: http://static.zybuluo.com/xiey/4f9nrjbblq43f1oxq783ccw1/TIM%E6%88%AA%E5%9B%BE20190603125042.png