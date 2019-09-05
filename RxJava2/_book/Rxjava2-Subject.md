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

--------------------------------------------------------------------2019/09/05补充-----------------------------------------------------------------------
先看一个例子：
```
@Test
public void test14() {
        ReplaySubject<Object> subject = ReplaySubject.create();
        subject.onNext("one");
        subject.onNext("two");
        subject.onNext("three");

        Observer<Object> observer=new Observer<Object>() {
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
        };

        subject.subscribe(observer);
        subject.subscribe(observer);
        subject.subscribe(observer);
        subject.onNext("one1");
        subject.onNext("two1");
        subject.onNext("three1");
        subject.onComplete();
    }
```
```
onSubscribe
onNext:one
onNext:two
onNext:three
onSubscribe
onNext:one
onNext:two
onNext:three
onSubscribe
onNext:one
onNext:two
onNext:three
onNext:one1
onNext:one1
onNext:one1
onNext:two1
onNext:two1
onNext:two1
onNext:three1
onNext:three1
onNext:three1
onComplete
onComplete
onComplete
```
看到上面的结果是先onSubscribe到three重复3遍，然后后面的单个每个重复3遍；这个是为什么呢？看看源码：
这一次我们跟着例子的调用来看源码，先创建了；
```
@CheckReturnValue
public static <T> ReplaySubject<T> create() {
    return new ReplaySubject<T>(new UnboundedReplayBuffer<T>(16));
}
```
```
ReplaySubject(ReplayBuffer<T> buffer) {
    this.buffer = buffer;
    this.observers = new AtomicReference<ReplayDisposable<T>[]>(EMPTY);
}
```
```
UnboundedReplayBuffer(int capacityHint) {
   this.buffer = new ArrayList<Object>(ObjectHelper.verifyPositive(capacityHint, "capacityHint"));
}
```
在create这个静态方法中，创建了一个ReplaySubject对象，但是这个对象需要传递一个ReplayBuffer<T>对象；
UnboundedReplayBuffer是ReplayBuffer的子类，多态；
这里就创建了两个对象，分别注意它们的构造函数：
UnboundedReplayBuffer的构造函数中实例化了一个buffer，是一个List<Object>，它是干啥的呢？我先剧透一下，它是用来承载我们发OnNext时，装传入的值的；为什么这么做呢，往下看；
ReplaySubject的构造函数中也实例化了一个buffer，不过这个buffer是ReplayBuffer类型的，也就是UnboundedReplayBuffer对象，还实例化了一个observers ，这个是AtomicReference<ReplayDisposable<T>[]>类型的，原子引用里面装了一个ReplayDisposable数组，这个是干啥的呢，也剧透一下，还记得我们的例子是一次订阅了多个吗，这个就是处理多个观察者的；

例子里面我们创建完了就开始发射数据，连着三个OnNext事件；
```
subject.onNext("one");
subject.onNext("two");
subject.onNext("three");
```
看看源码走向：
```
@Override
public void onNext(T t) {
    //空检查
    ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");
    //全局搜索done赋值的地方，默认没赋值，是false；只有2个地方，赋值true；就是onComplete/onError；可以看下方的图；这里也就是如果执行了onComplete或者onError，这个OnNext就被return了；
    if (done) {
        return;
    }
    
    //获取我们传入的ReplayBuffer，也就是UnboundedReplayBuffer，做了add操作；ReplayBuffer本质上是一个接口，他的所有具体方法都在实现类；所以看UnboundedReplayBuffer的add方法;add方法中又做了add操作，不过这次的buffer我们开始说过了，是List<Object>，所以只是将数据添加到List中，同时维持了这个List的size，size做了+1操作；
    /*
     * @Override
     * public void add(T value) {
     *     buffer.add(value);
     *     size++;
     * }
     */
    ReplayBuffer<T> b = buffer;
    b.add(t);
    
    //observers我们前面说过，是AtomicReference<ReplayDisposable<T>[]>类型，这里用了原子引用的get方法，所以获取就是ReplayDisposable数组；但是这里的数组还是空数组，我们还没有做添加操作；
    //那啥时候做添加操作呢？看下图；可以看到只有当执行add/remove/terminate方法的时候才可能被操作;
    //那啥时候执行这些方法呢？可以追踪到，add方法是当订阅的时候有可能会执行到；remove方法也是订阅的时候可能会执行到，同时断流的时候，dispose也可能会执行到；terminate方法则是在onComplete/onError中会执行到；
    //这里还没有进行添加操作，所以这个for循环不会执行；
    for (ReplayDisposable<T> rs : observers.get()) {
        b.replay(rs);
    }
}
```
done的用处：
![1111111111.png-40kB][1]

observers的添加操作：
![222222222222.png-64.5kB][2]

从上面的OnNext方法可以看出，此时我们连着发射了3个OnNext事件，但是只是将发射的值放到了ReplyBuffer中的一个List中，没有做其他操作；

跟着例子走，此时我们执行了订阅操作；这个前面讲了好多遍，都是调用上层被观察者的subscribeActual方法，这里由于中间没有各种操作符，所以就是我们的ReplaySubject；
```
protected void subscribeActual(Observer<? super T> observer) {
    //包装，构建了一个ReplayDisposable对象，将我们的观察者和被观察者包装了一个可以断流的ReplayDisposable；
    //它是实现了Disposable接口，实现了dispose和isDisposed方法，这里主要是对一个断流状态的模拟；主要是里面的一个属性cancelled，只要执行了onComplete/onError，或者在onSubscribe中做了断流操作；都会间接或直接的将cancelled置为true，表示断流，然后的onnext中发射事件回对这个标签进行检测，从而决定是否继续发送事件；
    ReplayDisposable<T> rs = new ReplayDisposable<T>(observer, this);
    //调用观察者的onSubscribe，传入了ReplayDisposable，回调里面可做断流操作；
    observer.onSubscribe(rs);
    
    //此时还没有做断流操作；如果此时在onSubscribe回调中做了断流操作，则这个例子就是只调用3次onSubscribe，然后没了；
    if (!rs.cancelled) {
        //做了一个add操作还记得这个add吗，就是我们说observers.get()的时候，此时add了；先看大概，如果返回true；一直判断断流状态；如果刚add成功就断流了，则又把这个add的remove出来，直接return；如果没这么急眼的，那就replay，这里的buffer是ReplyBuffer，也就是UnboundedReplayBuffer
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
上面我们遇到了三个方法add/remove/replay
```
    boolean add(ReplayDisposable<T> rs) {
        for (;;) {
            //取出ReplayDisposable数组
            ReplayDisposable<T>[] a = observers.get();
            //判断此时是否已经调用了onComplete/onError方法，只有在这俩方法中才会有这么干的可能；
            if (a == TERMINATED) {
                return false;
            }
            //获取数组的长度；
            int len = a.length;
            @SuppressWarnings("unchecked")
            //新建一个数组，做了一个+1扩容
            ReplayDisposable<T>[] b = new ReplayDisposable[len + 1];
            //复制转移
            System.arraycopy(a, 0, b, 0, len);
            //赋值
            b[len] = rs;
            //这里就可以理解为该数组里面已经做了+1操作；
            if (observers.compareAndSet(a, b)) {
                return true;
            }
        }
    }
```
```
void remove(ReplayDisposable<T> rs) {
        for (;;) {
            //一开始同理
            ReplayDisposable<T>[] a = observers.get();
            if (a == TERMINATED || a == EMPTY) {
                return;
            }
            int len = a.length;
            //获取要移除的对象在数组中的坐标；
            int j = -1;
            for (int i = 0; i < len; i++) {
                if (a[i] == rs) {
                    j = i;
                    break;
                }
            }
            
            //没找到就算了
            if (j < 0) {
                return;
            }
            //走到这，说明找到了
            ReplayDisposable<T>[] b;
            //如果容器里面只有一个，直接清空
            if (len == 1) {
                b = EMPTY;
            } else {
                //减容新数组，做了-1操作
                b = new ReplayDisposable[len - 1];
                //从0开始，拷贝到指定索引前面的
                System.arraycopy(a, 0, b, 0, j);
                //接着拷贝，拷贝指定索引后面的
                System.arraycopy(a, j + 1, b, j, len - j - 1);
            }
            //同理，就理解为数组做了,其实是比较新旧俩数组。然后根据规则，新数组替换了旧数组
            //规则：CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。
            //简单理解这个规则：我原本是a，判断第一个参数是否和我相同，现在第一个参数是a，那还用说，肯定相等，就是我自己嘛；所以就赋值为b；
            if (observers.compareAndSet(a, b)) {
                return;
            }
        }
    }
```
```
        //有点长。。。
        public void replay(ReplayDisposable<T> rs) {
            //这个和下面末位相呼应，先取值0，同时做+1操作；往下运行到末位，又做了-missed操作；没看出在代码中的作用；所以猜测应该是为线程安全做的处理，不允许同一个ReplayDisposable在操作过程中同时被别的线程操作；
            if (rs.getAndIncrement() != 0) {
                return;
            }

            int missed = 1;
            //取出对应List中OnNext发送的值
            final List<Object> b = buffer;
            //取出观察者
            final Observer<? super T> a = rs.actual;
            //获取当前已经处理到的索引
            Integer indexObject = (Integer)rs.index;
            //初始化索引
            int index;
            if (indexObject != null) {
                index = indexObject;
            } else {
                index = 0;
                rs.index = 0;
            }

            for (;;) {
                //断流判断
                if (rs.cancelled) {
                    rs.index = null;
                    return;
                }
                //获取List的尺寸
                int s = size;
                //判断是否已经全部处理完了List
                while (s != index) {
                    //断流判断
                    if (rs.cancelled) {
                        rs.index = null;
                        return;
                    }
                    //获取该索引对应的值，我们onNext中发送的；这时就能看出index的作用了，订阅后可能发送一波，记住索引，然后进来的新的数据根据索引往前走，不乱套；
                    Object o = b.get(index);
                    //这个done上面的图看到了，只有onComplete/onError的时候才会为true；
                    if (done) {
                        //判断是否是最后一个
                        //这里有个小疑问：为什么要这么判断处理，假设没有执行这个if，那下面的onNext是不是又得搞起来；
                        if (index + 1 == s) {
                            s = size;
                            //如果是最后一个且done=true，那妥妥的onComplete/onError
                            if (index + 1 == s) {
                                if (NotificationLite.isComplete(o)) {
                                    a.onComplete();
                                } else {
                                    a.onError(NotificationLite.getError(o));
                                }
                                rs.index = null;
                                //断流标签搞起来
                                rs.cancelled = true;
                                return;
                            }
                        }
                    }
                    //如果不是onComplete/onError，那就是OnNext;
                    a.onNext((T)o);
                    index++;
                }
                
                //时时刷新判断size，因为有可能此时List增加了，也就是onNext执行了，这种情况应该是异步操作；然后continue，又上去走一轮，执行了OnNext/onComplete/onError，这样当那个异步OnNextNext/onComplete/onError执行到上面的循环的时候，因为有index，所以会直接跳过，因为执行过了；
                if (index != size) {
                    continue;
                }
                //记录索引
                rs.index = index;

                missed = rs.addAndGet(-missed);
                if (missed == 0) {
                    break;
                }
            }
        }
```
上面有我一些不确定的地方，目前只是猜测是对线程安全相关的处理，只是猜测；后续追究一下；

```
subject.subscribe(observer);
subject.subscribe(observer);
subject.subscribe(observer);
```
按照我们代码的流程，就是我们执行了三次订阅(subscribeActual)，所以之前在List中囤积的onNext发射的数据发射了3遍，也就是三次List
```
onSubscribe
onNext:one
onNext:two
onNext:three
onSubscribe
onNext:one
onNext:two
onNext:three
onSubscribe
onNext:one
onNext:two
onNext:three
```
这是三次订阅那后面为什么都是3个单独重复呢？
```
onNext:one1
onNext:one1
onNext:one1
onNext:two1
onNext:two1
onNext:two1
onNext:three1
onNext:three1
onNext:three1
onComplete
onComplete
onComplete
```
接着代码往下走；
```
subject.onNext("one1");
subject.onNext("two1");
subject.onNext("three1");
```
我们发射了三次onNext；之前我们看过onNext的代码，这里再拿出来
```
public void onNext(T t) {
    ObjectHelper.requireNonNull(t, "onNext called with null. Null values are generally not allowed in 2.x operators and sources.");
    if (done) {
        return;
    }

    ReplayBuffer<T> b = buffer;
    b.add(t);

    for (ReplayDisposable<T> rs : observers.get()) {
        b.replay(rs);
    }
}
```
前面的不变，这次主要是后面的那个for循环；还记得我们在执行订阅方法subscribeActual的时候，执行的那个add(rs)方法中的observers.compareAndSet(a, b)吗？在订阅的时候我们做了+1操作；我们订阅了3次，所以此时for循环3次，然后执行replay方法，后续之前分析过了，index往前走，只要done=false，就执行onNext；所以这里才会单个的循环3次；

最后执行
```
subject.onComplete();
```
```
public void onComplete() {
    //对自己很严格，执行了done就不可再执行了；
    if (done) {
        return;
    }
    //这个眼熟吗
    done = true;
    //这个给了个枚举标签COMPLETE
    Object o = NotificationLite.complete();
    //获取对应的ReplayBuffer
    ReplayBuffer<T> b = buffer;
    //下方细说
    b.addFinal(o);
    //这一块之前也类似的分析过了，就是循环对所有的观察者执行了replay，在里面执行了onComplete回调，就是根据上面这个枚举标签COMPLETE判断来的，然后做了断流标签设置；
    for (ReplayDisposable<T> rs : terminate(o)) {
        b.replay(rs);
    }
}
```
```
public void addFinal(Object notificationLite) {
    //传入枚举值，主要在replay中判断当前是onComplete还是onError
    buffer.add(notificationLite);
    size++;
    //双层保险，前面(onComplete/onError)已经设置过了，这里又设置一次；还是说我自作多情了，丫就是多余的
    done = true;
}
```

在onComplete和onError的最后会有一个方法调用terminate
```
ReplayDisposable<T>[] terminate(Object terminalValue) {
    //buffer之前没有做个原子操作，所以它是null，这里比较修改后buffer的值就是COMPLETE，修改成功返回true，进入if语句;
    if (buffer.compareAndSet(null, terminalValue)) {
        //这里是将TERMINATED设置为observers的值，observers是一个空数组；可能会有疑惑，你都空数组，后面还怎么for循环回调onComplete啊，注意一下，这里是getAndSet，就有点i++的意思，先用后设置值；
        return observers.getAndSet(TERMINATED);
    }
    return TERMINATED;
}
```

ReplaySubject的create方法还有两种：

- createWithSize/createUnbounded
- createWithTime/createWithTimeAndSize

它们的内部对于发射的onNext不是用List装载的，而是用节点Node，但是又不是数据结构中的那种链表，有上有下，它是一个节点，继承了原子引用，原子引用的类型又是Node节点，所以它直接在节点Node中设置值value，然后将下一个节点设置为上一个节点的原子引用值，这样，这条“链”只能往下走，不能往上走；完全没有存储关于上个节点的引用；

在添加的时候，按上述所说的操作，最后调用一个trim方法，在该方法中会比较最大值/时间；如果订阅之前发射了一些事件，则按照节点添加操作，同时计数size，如果size大于maxSize，则将头结点往下走，头结点抛弃，size做-1操作；至于订阅之后的，其实它也抛弃了之前，只保留了maxSize这么多个节点，只不过因为前面的事件已经回调过了，而且又抛弃的是头结点，所以对后续没有影响，也就是后续再定约的观察者，只能收到订阅之前的最多maxSize个事件；

带时间限制的差不多，也是当添加的时候，记录时间，等下次添加的时候就判断是否已过期指定的时间（是否还处于有效时间），OK就飘过，不OK就从头部开始抛弃；

就相当于一个缓存条件；

里面还有一些getValue/getValues：第一个是获取最后一个onNext发射值；第二个是获取所有缓存的onNext发射值数组；
源码实现就是：getValue遍历获取最后一个，getValues遍历并装载在一个空数组中返回；
onSubscribe：不知道暂时可做什么用途；
剩下的就是一些见名知意的简单方法；


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



---

[1]: http://static.zybuluo.com/xiey/9nvtii60x9h6lo1cxea05fz7/1111111111.png
[2]: http://static.zybuluo.com/xiey/e4s88454b1wjekmoamokjo64/222222222222.png