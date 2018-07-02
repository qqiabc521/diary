我们先从一个简单demo入手分析：

```java
Observable.create(new ObservableOnSubscribe<String>() {
            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("1");
                e.onNext("2");
                e.onComplete();
            }
        }).subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("onSubscribe");
            }

            @Override
            public void onNext(String value) {
                System.out.println("onNext "+value);
            }

            @Override
            public void onError(Throwable e) {
                System.out.println("onError "+e.getMessage());
            }

            @Override
            public void onComplete() {
                System.out.println("onComplete");
            }
        });
```

运行结果：

> onSubscribe

> onNext 1

> onNext 2

> onComplete

这个demo有两步操作：

> * Observable.create();
> * Observable.subscribe();

## Observable.create()

```java
 public static <T> Observable<T> create(ObservableOnSubscribe <T> source) {
        ObjectHelper.requireNonNull(source, "source is null");
        //RxJavaPlugins提供了一系列的Hook function，
        //通过钩子函数这种方法对RxJava的标准操作进行加工，
        //当我们没有进行配置时，默认是直接返回原来的对象，也就是返回ObservableCreate对象
        return RxJavaPlugins.onAssembly(new ObservableCreate<T>(source));
    }
```
RxJavaPlugins.onAssembly源码如下：

```java
public static <T> Observable<T> onAssembly(Observable<T> source) {
        Function<Observable, Observable> f = onObservableAssembly;
        if (f != null) {
            return apply(f, source);
        }
        return source;
    }
```



其中RxJavaPlugins.onAssembly我们先不管，就当直接返回source。

在create中将ObservableOnSubscribe作为构造方法参数创建了一个ObservableCreate对象。ObservableCreate是什么东东呢，我们先看一下源码：

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

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }

    static final class CreateEmitter<T>
    extends AtomicReference<Disposable>
    implements ObservableEmitter<T>, Disposable {
    // 1.注意，CreateEmitter实现了Disposable接口
    // 这也就说明了在执行上面subscribeActual()步骤3 observer.onSubscribe()的回调时，可以将ObservableEmitter作为参数传入了
    // ObservableEmitter也是Disposable

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
             //在ObservableEmitter.onNext()发射数据时，实际上调用了observer的onNext()回调打印数据
                observer.onNext(t);
            }
        }

        @Override
        public void onError(Throwable t) {
            if (t == null) {
                t = new NullPointerException("onError called with null. Null values are generally not allowed in 2.x operators and sources.");
            }
            if (!isDisposed()) {
                try {
                    observer.onError(t);
                } finally {
                //当执行onError()时，会执行dispose()方法取消订阅
                    dispose();
                }
            } else {
                RxJavaPlugins.onError(t);
            }
        }

        @Override
        public void onComplete() {
            if (!isDisposed()) {
                try {
                    observer.onComplete();
                } finally {
                //当执行onComplete()时，会执行dispose()方法取消订阅
                //故onComplete()和onError()方法只能触发其中的一个
                    dispose();
                }
            }
        }

        ...
    }
    ...
    ...
}
```
ObservableCreate是Observable的子类，内部有一个静态内部类CreateEmitter，这个内部类实现了ObservableEmitter接口。

**Observable.create()总结：**

Observable.create()返回一个ObservableCreate对象。我们暂且先知道这一点就可以了。

## Observable.subscribe()

```java
public final void subscribe(Observer<? super T> observer) {
        ObjectHelper.requireNonNull(observer, "observer is null");
        try {
            observer = RxJavaPlugins.onSubscribe(this, observer);

            ObjectHelper.requireNonNull(observer, "Plugin returned null Observer");
            
			//重点在这里
            subscribeActual(observer);
        } catch (NullPointerException e) { // NOPMD
            throw e;
        } catch (Throwable e) {
           ...
        }
    }
```
还是先看源码，传递一个observer参数，核心是observer被subscribeActual接收。那么subscribeActual做了什么呢？第一步我们知道，从Observable.create()获得ObservableCreate对象，这个subscribeActual也是ObservableCreate的方法，我们跟进去看看：

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

        try {
            source.subscribe(parent);
        } catch (Throwable ex) {
            Exceptions.throwIfFatal(ex);
            parent.onError(ex);
        }
    }
```
 subscribeActual方法内部，首先实例化一个CreateEmitter发射器对象，observer作为构造参数存储在CreateEmitter中。
 
然后执行Observer的onSubscribe回调。

最后执行ObservableOnSubscribe对象的subscribe方法，将CreateEmitter发射器发送数据，即：
 
 ```java
  public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("1");
                e.onNext("2");
                e.onComplete();
            }
 ```

### 总结
我们先分析各个类的职责：

* Observable:业务主线，所有的操作都是基于其子类实现。可以理解为装饰器模式下的基类。
* ObservableOnSubscribe:接口，定义了数据源的发射行为。
* ObservableCreate:装饰器模式的具体体现，内部存储了数据源的发射事件，关联了订阅关系。
* ObservableEmitter:数据源发射器，内部存储了Observer。
* Observer:接收数据源并回调，可任务的处理者。

总结如下：

1. Observable.create(),实例化ObservableCreate和ObservableOnSubscribe，存储数据源，等待被订阅。
2. Observable.subscribe(),实例化ObservableEmitter，准备一个发射器，随时等待确认订阅关系发送数据。
3. 执行Observable.onSubscribe()回调，将ObservableEmitter作为Disposable参数传入，用于取消数据流。
4. 执行ObservableOnSubscribe.subscribe()，通过ObservableEmitter发射数据，将数据流流入Observer中。
5. Observable与Observer产生订阅关系后，只有没有被dispose()掉，才会回调Observer。
6. Observer的onComplete()和onError() 互斥只能执行一次，因为CreateEmitter在回调他们两中任意一个后，都会自动dispose()。
7. onSubscribe()是在我们执行subscribe()这句代码的那个线程回调的，并不受线程调度影响