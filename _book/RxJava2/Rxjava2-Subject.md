# Rxjava2-Subject

[TOC]

#### Subject：

```
public abstract class Subject<T> extends Observable<T> implements Observer<T> {
    ...
}
```
Subject 可以同时代表 Observer 和 Observable，允许从数据源中多次发送结果给多个观察者。除了 onSubscribe(), onNext(), onError() 和 onComplete() 之外，所有的方法都是线程安全的。此外，你还可以使用 toSerialized() 方法，也就是转换成串行的，将这些方法设置成线程安全的。

Subject 同时继承了 Observable 和 Observer 两个接口，说明它既是被观察的对象，同时又是观察对象，也就是可以生产、可以消费、也可以自己生产自己消费。

#### 实现类：

| Subject         | 发射行为                                                     |
| --------------- | ------------------------------------------------------------ |
| AsyncSubject    | 不论什么时候订阅只发射最后一个数据而且必须要调用onComplete()才会开始发射数据 |
| BehaviorSubject | 发送订阅之前发射的最后一个数据(如果没有可以预定义一个默认的数据)和订阅之后的全部数据 |
| ReplaySubject   | 不论订阅发生在什么时候，都发射全部数据，可以自定义接收订阅之前的数据的数量和有效时间 |
| PublishSubject  | 发射订阅之后的全部数据                                       |
| UnicastSubject  | 只允许一个 Observer 进行监听，在该 Observer 注册之前会将发射的所有的事件放进一个队列中，并在 Observer 注册的时候一起通知给它 |

#### 实现类Demo：
```
//订阅后全部发送
@Test
public void test14() {
    ReplaySubject<Object> subject = ReplaySubject.create();
    subject.onNext("one");
    subject.onNext("two");
    subject.onNext("three");

    subject.subscribe(new Observer<Object>() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe");
        }

        @Override
        public void onNext(Object o) {
            System.out.println("onNext:" + o);
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }
    });
    subject.onNext("one1");
    subject.onNext("two1");
    subject.onNext("three1");
    subject.onComplete();
}
```
```
结果打印：
onSubscribe
onNext:one
onNext:two
onNext:three
onNext:one1
onNext:two1
onNext:three1
onComplete
```

```
//发送订阅前最后一个数据项+订阅后所有的数据项，没有数据项时会发送一个默认数据；如果有异常则中断发送一个异常
@Test
public void test15() {
    BehaviorSubject<String> subject = BehaviorSubject.createDefault("default");
    subject.onNext("1");
    subject.onNext("2");
    subject.onNext("3");
    //subject.onError(new Exception("11111111111"));
    subject.subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe");
        }

        @Override
        public void onNext(String s) {
            System.out.println("onNext:" + s);
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }
    });

    subject.onNext("4");
    subject.onNext("5");
    subject.onNext("6");
}
```

```
结果打印：
onSubscribe
onNext:3
onNext:4
onNext:5
onNext:6
...
//只发送onError
onSubscribe
onError
```

```
//只发送订阅后的数据项
@Test
public void test16() {
    PublishSubject<String> subject = PublishSubject.create();
    subject.onNext("1");
    subject.onNext("2");
    subject.onNext("3");
    subject.subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe");
        }

        @Override
        public void onNext(String s) {
            System.out.println("onNext:" + s);
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }
    });
    subject.onNext("4");
    subject.onNext("5");
    subject.onNext("6");
}
```

```
结果打印：
onSubscribe
onNext:4
onNext:5
onNext:6
```

```
//只有发射onComplete才发射最后一个数据项，无关订阅
@Test
public void test17() {
    AsyncSubject<String> subject = AsyncSubject.create();
    subject.onNext("1");
    subject.onNext("2");
    subject.onNext("3");
    subject.subscribe(new Observer<String>() {
        @Override
        public void onSubscribe(Disposable d) {
            System.out.println("onSubscribe");
        }

        @Override
        public void onNext(String s) {
            System.out.println("onNext:" + s);
        }

        @Override
        public void onError(Throwable e) {
            System.out.println("onError");
        }

        @Override
        public void onComplete() {
            System.out.println("onComplete");
        }
    });
    subject.onNext("4");
    subject.onNext("5");
    subject.onNext("6");
    subject.onComplete();
}
```

```
结果打印：
onSubscribe
onNext:6
onComplete
```

```
@Test
public void test21(){
    //只允许一个 Observer 进行监听，在该 Observer 注册之前会将发射的所有的事件放进一个队列中，并在 Observer 注册的时候一起通知给它
    UnicastSubject<String> subject=UnicastSubject.create();
    subject.onNext("1");
    subject.onNext("2");
    subject.onNext("3");
    subject.subscribe(new Consumer<String>() {
        @Override
        public void accept(String s) throws Exception {
            System.out.println(s);
        }
    });
    subject.onNext("4");
    subject.onNext("5");
    subject.onNext("6");
    subject.subscribe(new Consumer<String>() {
        @Override
        public void accept(String s) throws Exception {
            System.out.println(s);
        }
    });
    subject.onNext("7");
    subject.onNext("8");
    subject.onNext("9");
}
```
```
操作结果：
1
2
3
4
5
6
io.reactivex.exceptions.OnErrorNotImplementedException: Only a single observer allowed.
```
#### 原理解释：

**ReplaySubject**

```
public static <T> ReplaySubject<T> create() {
    return new ReplaySubject<T>(new UnboundedReplayBuffer<T>(16));
}
```
创建了一个ReplaySubject，内部传入了一个List类型的数据集，初始化大小是16
```
final List<Object> buffer;
UnboundedReplayBuffer(int capacityHint) {
    this.buffer = new ArrayList<Object>(ObjectHelper.verifyPositive(capacityHint, "capacityHint"));
}
```
```
ReplaySubject(ReplayBuffer<T> buffer) {
    this.buffer = buffer;
    this.observers = new AtomicReference<ReplayDisposable<T>[]>(EMPTY);
}
```
然后执行subscribeActual
```
protected void subscribeActual(Observer<? super T> observer) {
    ReplayDisposable<T> rs = new ReplayDisposable<T>(observer, this);
    observer.onSubscribe(rs);

    if (!rs.cancelled) {
        if (add(rs)) {
            if (rs.cancelled) {
                remove(rs);
                return;
            }
        }
        buffer.replay(rs);
    }
}
```

将Observer包装成了ReplayDisposable
```
ReplayDisposable(Observer<? super T> actual, ReplaySubject<T> state) {
    this.actual = actual;
    this.state = state;
}
```
```
//ReplayDisposable
volatile boolean cancelled;
public void dispose() {
    if (!cancelled) {
        cancelled = true;
        state.remove(this);
    }
}
```

可以看到，只有断流之后cancelled才会等于true；
在subscribeActual中，只要没断流都会加入到list中；最后会调用一个replay(rs)方法

```
//还挺长
public void replay(ReplayDisposable<T> rs) {
    //一开始就做了一个判断，这个只有设置才会有这个return
    //在terminate方法中有个observers.getAndSet(TERMINATED)
    //一看这个方法名字就知道是终结者，这里
    if (rs.getAndIncrement() != 0) {
        return;
    }

    int missed = 1;
    //拿到List
    final List<Object> b = buffer;
    //拿到Observer
    final Observer<? super T> a = rs.actual;

    Integer indexObject = (Integer)rs.index;
    int index;
    if (indexObject != null) {
        index = indexObject;
    } else {
        index = 0;
        rs.index = 0;
    }

    
    for (;;) {
        //判断是否已经断流，断流则没有后续
        if (rs.cancelled) {
            rs.index = null;
            return;
        }

        int s = size;
        //循环List
        while (s != index) {
            //判断是否已经断流，断流则没有后续
            if (rs.cancelled) {
                rs.index = null;
                return;
            }

            Object o = b.get(index);
            //这个done会在onComplete/onError中设置为true；对应着这里就是done==true则回调onComplete/onError
            if (done) {
                if (index + 1 == s) {
                    s = size;
                    if (index + 1 == s) {
                        if (NotificationLite.isComplete(o)) {
                            a.onComplete();
                        } else {
                            a.onError(NotificationLite.getError(o));
                        }
                        rs.index = null;
                        rs.cancelled = true;
                        return;
                    }
                }
            }
            //否则就调用OnNext回调
            a.onNext((T)o);
            //++
            index++;
        }

        if (index != size) {
            continue;
        }

        rs.index = index;

        missed = rs.addAndGet(-missed);
        if (missed == 0) {
            break;
        }
    }
}
```
中间只解释了我们想知道的，就是这个方法会对List进行遍历然后发送事件


**BehaviorSubject**

```
public static <T> BehaviorSubject<T> createDefault(T defaultValue) {
    return new BehaviorSubject<T>(defaultValue);
}

public static <T> BehaviorSubject<T> create() {
    return new BehaviorSubject<T>();
}

BehaviorSubject(T defaultValue) {
    this();
    this.value.lazySet(ObjectHelper.requireNonNull(defaultValue, "defaultValue is null"));
}

static final BehaviorDisposable[] EMPTY = new BehaviorDisposable[0];
BehaviorSubject() {
    this.lock = new ReentrantReadWriteLock();
    this.readLock = lock.readLock();
    this.writeLock = lock.writeLock();
    this.subscribers = new AtomicReference<BehaviorDisposable<T>[]>(EMPTY);
    this.value = new AtomicReference<Object>();
    this.terminalEvent = new AtomicReference<Throwable>();
}
```
赋值操作；

```
protected void subscribeActual(Observer<? super T> observer) {
    BehaviorDisposable<T> bs = new BehaviorDisposable<T>(observer, this);
    observer.onSubscribe(bs);
    if (add(bs)) {
        if (bs.cancelled) {
            remove(bs);
        } else {
            bs.emitFirst();
        }
    } else {
        Throwable ex = terminalEvent.get();
        if (ex == ExceptionHelper.TERMINATED) {
            observer.onComplete();
        } else {
            observer.onError(ex);
        }
    }
}
```

```
public void onNext(T t) {
    ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");

    if (terminalEvent.get() != null) {
        return;
    }
    Object o = NotificationLite.next(t);
    setCurrent(o);
    for (BehaviorDisposable<T> bs : subscribers.get()) {
        bs.emitNext(o, index);
    }
}
```
```
void setCurrent(Object o) {
    writeLock.lock();
    try {
        index++;
        value.lazySet(o);
    } finally {
        writeLock.unlock();
    }
}
```
用index做标记,OnNext中的for循环一开始跑不起来，主要是subscribers.get()是个空数组(EMPTY)，当订阅之后，在add方法中subscribers.compareAndSet(a, b)，意思就是如果a没有变化继续for循环，按道理讲应该不会重现这种情况，因为每次订阅中的BehaviorDisposable都是new出来的；否则就是return true；

订阅完后，如果没有断流则调用bs.emitFirst();
经过层层调用，通过index进行定位到最后一个位置，然后调用OnNext
```
emitFirst->test(Object)->NotificationLite.accept(o, actual)->s.onNext((T)o);
```
```
void emitFirst() {
    if (cancelled) {
        return;
    }
    Object o;
    synchronized (this) {
        if (cancelled) {
            return;
        }
        if (next) {
            return;
        }

        BehaviorSubject<T> s = state;
        Lock lock = s.readLock;

        lock.lock();
        index = s.index;
        o = s.value.get();
        lock.unlock();

        emitting = o != null;
        next = true;
    }

    if (o != null) {
        if (test(o)) {
            return;
        }

        emitLoop();
    }
}
```
```
public boolean test(Object o) {
    return cancelled || NotificationLite.accept(o, actual);
}
```
```
public static <T> boolean accept(Object o, Observer<? super T> s) {
    if (o == COMPLETE) {
        s.onComplete();
        return true;
    } else
    if (o instanceof ErrorNotification) {
        s.onError(((ErrorNotification)o).e);
        return true;
    }
    s.onNext((T)o);
    return false;
}
```

紧接着，如果订阅后，再发射OnNext

```
//解释compareAndSet：
@Test
public void test20(){
    AtomicInteger atomicInteger=new AtomicInteger(3);
    System.out.println(atomicInteger.compareAndSet(3,4));
    System.out.println(atomicInteger.get());
    System.out.println(atomicInteger.compareAndSet(2,5));
    System.out.println(atomicInteger.get());
}
true
4
false
4
//如果第一个参数和之前的值相同则返回true，并将值设置为第二个参数；
//例子中一开始是3，然后第一个参数是3，则参数相同返回true，并设置值为4（第二个参数）；然后又做了一次操作，这次4和2不同则返回false，值不变还是4；
```

**PublishSubject**
```
final AtomicReference<PublishDisposable<T>[]> subscribers;
static final PublishDisposable[] EMPTY = new PublishDisposable[0];
PublishSubject() {
    subscribers = new AtomicReference<PublishDisposable<T>[]>(EMPTY);
}
```
一开始初始了一个subscribers,是个空数组
```
public void subscribeActual(Observer<? super T> t) {
    PublishDisposable<T> ps = new PublishDisposable<T>(t, this);
    t.onSubscribe(ps);
    if (add(ps)) {
        // if cancellation happened while a successful add, the remove() didn't work
        // so we need to do it again
        if (ps.isDisposed()) {
            remove(ps);
        }
    } else {
        Throwable ex = error;
        if (ex != null) {
            t.onError(ex);
        } else {
            t.onComplete();
        }
    }
}
```
```
boolean add(PublishDisposable<T> ps) {
    for (;;) {
        PublishDisposable<T>[] a = subscribers.get();
        if (a == TERMINATED) {
            return false;
        }

        int n = a.length;
        @SuppressWarnings("unchecked")
        PublishDisposable<T>[] b = new PublishDisposable[n + 1];
        System.arraycopy(a, 0, b, 0, n);
        b[n] = ps;

        if (subscribers.compareAndSet(a, b)) {
            return true;
        }
    }
}
```
```
public void onNext(T t) {
    ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");

    if (subscribers.get() == TERMINATED) {
        return;
    }
    for (PublishDisposable<T> s : subscribers.get()) {
        s.onNext(t);
    }
}
```
可以先看OnNext，因为在订阅前可能也发出OnNext方法，里面有一个增强for循环，主要是看subscribers.get()中是否有值，目前从初始化到发出OnNext，是一直没有值的；
然后看订阅方法subscribeActual（），先调用了add方法，做了一个增加值操作,将传入的PublishDisposable放入到b数组中，然后添加到subscribers的EMPTY数组中。这样再次发生OnNext的时候那个增强for循环就会回调OnNext
```
subscribers.compareAndSet(a, b)
```
在对应的onComplete/onError方法中，也有一个增强for循环
```
for (PublishDisposable<T> s : subscribers.getAndSet(TERMINATED)) {
    s.onError(t);/s.onComplete();
}
```
这里就是将subscribers设置为TERMINATED，然后调用对应的方法。

这个getAndSet可能会有点疑惑，我既然设置成了TERMINATED，那就应该是空数组，怎么还执行了for循环里面的东西呢？这里是这样的，这个getAndSet是返回之前的值，设置的是新的值；所以我们的那个之前的是EMPTY，一旦订阅就有值了，所以我们执行的for里面的内容是没有问题的。
我做了一个测试：
```
@Test
public void test19(){
    AtomicInteger atomicInteger=new AtomicInteger(1);
    System.out.println(atomicInteger.getAndSet(2));
    System.out.println(atomicInteger.get());
}
```
```
结果打印：
1
2
```


**AsyncSubject**

```
protected void subscribeActual(Observer<? super T> s) {
    AsyncDisposable<T> as = new AsyncDisposable<T>(s, this);
    s.onSubscribe(as);
    if (add(as)) {
        if (as.isDisposed()) {
            remove(as);
        }
    } else {
        Throwable ex = error;
        if (ex != null) {
            s.onError(ex);
        } else {
            T v = value;
            if (v != null) {
                //注意
                as.complete(v);
            } else {
                as.onComplete();
            }
        }
    }
}
```
```
public final void complete(T value) {
    int state = get();
        if ((state & (FUSED_READY | FUSED_CONSUMED | TERMINATED | DISPOSED)) != 0) {
            return;
        }
        if (state == FUSED_EMPTY) {
            this.value = value;
        lazySet(FUSED_READY);
    } else {
        lazySet(TERMINATED);
    }
    Observer<? super T> a = actual;
    //1
    a.onNext(value);
    if (get() != DISPOSED) {
        //2
        a.onComplete();
    }
}
```
宏观来说就是，可以看到是先执行OnNext，然后执行onComplete

**UnicastSubject**

```
public static <T> UnicastSubject<T> create() {
    //最大128
    return new UnicastSubject<T>(bufferSize(), true);
}
```
```
public static int bufferSize() {
    return Flowable.bufferSize();
}

public static int bufferSize() {
    return BUFFER_SIZE;
}

static final int BUFFER_SIZE;
static {
    BUFFER_SIZE = Math.max(1, Integer.getInteger("rx2.buffer-size", 128));
}
```
```
UnicastSubject(int capacityHint, boolean delayError) {
    this.queue = new SpscLinkedArrayQueue<T>(ObjectHelper.verifyPositive(capacityHint, "capacityHint"));
    this.onTerminate = new AtomicReference<Runnable>();
    this.delayError = delayError;
    this.actual = new AtomicReference<Observer<? super T>>();
    this.once = new AtomicBoolean();
    this.wip = new UnicastQueueDisposable();
}
```
```
protected void subscribeActual(Observer<? super T> observer) {
    if (!once.get() && once.compareAndSet(false, true)) {
        observer.onSubscribe(wip);
        actual.lazySet(observer); // full barrier in drain
        if (disposed) {
            actual.lazySet(null);
            return;
        }
        drain();
    } else {
        EmptyDisposable.error(new IllegalStateException("Only a single observer allowed."), observer);
    }
}
```
关注这一行代码
```
if (!once.get() && once.compareAndSet(false, true)) {
```
这个应该compareAndSet前面说过，这个once一开始默认是false，所以第一个条件判断是否为false，第二个又是判断是否为false；一旦为true则不可进入if，抛出异常
```
EmptyDisposable.error(new IllegalStateException("Only a single observer allowed."), observer);
```
又是层层调用
```
queue.offer(t);drain();->drainFused(Observer)/drainNormal(Observer)->onNext(T)；
```
中间省略一些代码，从宏观上来说就是只能订阅一次，然后将订阅前发射的事件收集起来一块发射，然后订阅后继续发射事件；