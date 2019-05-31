# RxJava2-Disposable

[TOC]

#### Disposable
```
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
Disposable主要是对事件流的处理；有两个接口方法，一个是用来中断事件流，一个是用来判断事件流是否中断。
目前接触到的实现类有ObservableCreate中的静态内部类CreateEmitter，里面实现了这两个方法
#### CreateEmitter---onNext/onError/onComplete
```
        @Override
        public void setDisposable(Disposable d) {
            DisposableHelper.set(this, d);
        }
        
        @Override
        public void dispose() {
            DisposableHelper.dispose(this);
        }

        @Override
        public boolean isDisposed() {
            return DisposableHelper.isDisposed(get());
        }
```
第一个方法是ObservableEmitter中的方法，也被CreateEmitter实现了。
这里涉及到了一个DisposableHelper
#### DisposableHelper
```
public enum DisposableHelper implements Disposable {
    /**
     * The singleton instance representing a terminal, disposed state, don't leak it.
     */
    DISPOSED
    ;

    /**
     * Checks if the given Disposable is the common {@link #DISPOSED} enum value.
     * @param d the disposable to check
     * @return true if d is {@link #DISPOSED}
     */
    public static boolean isDisposed(Disposable d) {
        //判断Disposable类型的变量的引用是否等于DISPOSED
        //即判断该连接器是否被中断
        return d == DISPOSED;
    }

    /**
     * Atomically sets the field and disposes the old contents.
     * @param field the target field
     * @param d the new Disposable to set
     * @return true if successful, false if the field contains the {@link #DISPOSED} instance.
     */
    public static boolean set(AtomicReference<Disposable> field, Disposable d) {
        for (;;) {
            Disposable current = field.get();
            if (current == DISPOSED) {
                if (d != null) {
                    d.dispose();
                }
                return false;
            }
            if (field.compareAndSet(current, d)) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
    }

    /**
     * Atomically sets the field to the given non-null Disposable and returns true
     * or returns false if the field is non-null.
     * If the target field contains the common DISPOSED instance, the supplied disposable
     * is disposed. If the field contains other non-null Disposable, an IllegalStateException
     * is signalled to the RxJavaPlugins.onError hook.
     * 
     * @param field the target field
     * @param d the disposable to set, not null
     * @return true if the operation succeeded, false
     */
    public static boolean setOnce(AtomicReference<Disposable> field, Disposable d) {
        ObjectHelper.requireNonNull(d, "d is null");
        if (!field.compareAndSet(null, d)) {
            d.dispose();
            if (field.get() != DISPOSED) {
                reportDisposableSet();
            }
            return false;
        }
        return true;
    }

    /**
     * Atomically replaces the Disposable in the field with the given new Disposable
     * but does not dispose the old one.
     * @param field the target field to change
     * @param d the new disposable, null allowed
     * @return true if the operation succeeded, false if the target field contained
     * the common DISPOSED instance and the given disposable (if not null) is disposed.
     */
    public static boolean replace(AtomicReference<Disposable> field, Disposable d) {
        for (;;) {
            Disposable current = field.get();
            if (current == DISPOSED) {
                if (d != null) {
                    d.dispose();
                }
                return false;
            }
            if (field.compareAndSet(current, d)) {
                return true;
            }
        }
    }

    /**
     * Atomically disposes the Disposable in the field if not already disposed.
     * @param field the target field
     * @return true if the current thread managed to dispose the Disposable
     */
    public static boolean dispose(AtomicReference<Disposable> field) {
        Disposable current = field.get();
        Disposable d = DISPOSED;
        if (current != d) {
            //这里会把field给设为DISPOSED
            current = field.getAndSet(d);
            if (current != d) {
                if (current != null) {
                    current.dispose();
                }
                return true;
            }
        }
        return false;
    }

    /**
     * Verifies that current is null, next is not null, otherwise signals errors
     * to the RxJavaPlugins and returns false.
     * @param current the current Disposable, expected to be null
     * @param next the next Disposable, expected to be non-null
     * @return true if the validation succeeded
     */
    public static boolean validate(Disposable current, Disposable next) {
        if (next == null) {
            RxJavaPlugins.onError(new NullPointerException("next is null"));
            return false;
        }
        if (current != null) {
            next.dispose();
            reportDisposableSet();
            return false;
        }
        return true;
    }

    /**
     * Reports that the disposable is already set to the RxJavaPlugins error handler.
     */
    public static void reportDisposableSet() {
        RxJavaPlugins.onError(new ProtocolViolationException("Disposable already set!"));
    }

    /**
     * Atomically tries to set the given Disposable on the field if it is null or disposes it if
     * the field contains {@link #DISPOSED}.
     * @param field the target field
     * @param d the disposable to set
     * @return true if successful, false otherwise
     */
    public static boolean trySet(AtomicReference<Disposable> field, Disposable d) {
        if (!field.compareAndSet(null, d)) {
            if (field.get() == DISPOSED) {
                d.dispose();
            }
            return false;
        }
        return true;
    }

    @Override
    public void dispose() {
        // deliberately no-op
    }

    @Override
    public boolean isDisposed() {
        return true;
    }
```
这是一个枚举类，只有一个值DISPOSED，dispose()方法中会把一个原子引用field设为DISPOSED，即标记为中断状态。因此后面通过isDisposed()方法即可以判断连接器是否被中断。
再去CreateEmitter中看看发射事件中的调用
#### CreateEmitter
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
可以看到，再我们常用的三个方法onNext/onError/onComplete方法中都会先使用isDisposed()来判断一下，然后决定是否发射事件；所以我们在Observer的onSubscribe(Disposable)方法中拿到这个发射器对象，然后在我们调用dispose()方法，设置标签，这样在下发下一个事件的时候都会先调用isDisposed()来判断，从而截断事件流。

