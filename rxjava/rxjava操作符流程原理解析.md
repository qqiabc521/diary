# 实例分析
本文将会对我们使用频率最高的map操作符进行分析，在了解本文内容之前，建议先看一下[rxjava基本流程解析]

```java
Observable.create(new ObservableOnSubscribe<String>() {

            @Override
            public void subscribe(ObservableEmitter<String> e) throws Exception {
                e.onNext("1");
                e.onNext("2");
                e.onComplete();
            }
        }).map(new Function<String, Integer>() {

            @Override
            public Integer apply(String s) throws Exception {
                return Integer.parseInt(s) + 10;
            }
        }).doOnNext(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                System.out.println("doOnNext "+integer.intValue());
            }
        }).subscribe(new Observer<Integer>() {
            @Override
            public void onSubscribe(Disposable d) {
                System.out.println("onSubscribe "+d.toString());
            }

            @Override
            public void onNext(Integer value) {
                System.out.println("onNext "+value);
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
```

## Map()

我们先看一下map()方法的代码结构：

```java
public final <R> Observable<R> map(Function<? super T, ? extends R> mapper) {
        ObjectHelper.requireNonNull(mapper, "mapper is null");
        return RxJavaPlugins.onAssembly(new ObservableMap<T, R>(this, mapper));
    }
```

默认的map内部方法创建了一个ObservableMap对象。

## ObservableMap

```java 
public final class ObservableMap<T, U> extends AbstractObservableWithUpstream<T, U> {
    final Function<? super T, ? extends U> function;

    public ObservableMap(ObservableSource<T> source, Function<? super T, ? extends U> function) {
        super(source);
        this.function = function;
    }

    @Override
    public void subscribeActual(Observer<? super U> t) {
    	//用MapObserver订阅上游Observable。
        source.subscribe(new MapObserver<T, U>(t, function));
    }


    static final class MapObserver<T, U> extends BasicFuseableObserver<T, U> {
        final Function<? super T, ? extends U> mapper;

        MapObserver(Observer<? super U> actual, Function<? super T, ? extends U> mapper) {
        		//super()将actual保存起来
            super(actual);
             //保存Function变量
            this.mapper = mapper;
        }

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
            //这一步执行变换,将上游传过来的T，利用Function转换成下游需要的U。
            //值得一提的是，我们可以看到，在数据进行了转换之后，会将转换后的数据进行一次ObjectHelper.requireNonNull()非空校验，如果转换后的数据为空，则会抛出错误，
            //这一点我们一定要注意，数据流的传递和转换过程中，一定要避免null值的情况。
                v = ObjectHelper.requireNonNull(mapper.apply(t), "The mapper function returned a null value.");
            } catch (Throwable ex) {
                fail(ex);
                return;
            }
             //变换后传递给下游Observer
            actual.onNext(v);
        }

        @Override
        public int requestFusion(int mode) {
            return transitiveBoundaryFusion(mode);
        }

        @Override
        public U poll() throws Exception {
            T t = qs.poll();
            return t != null ? ObjectHelper.<U>requireNonNull(mapper.apply(t), "The mapper function returned a null value.") : null;
        }
    }
}
```
和ObservableCreate不同的是，ObservableMap类继承了AbstractObservableWithUpstream类。当ObservableMap执行subscribeActual()时，我们接收到一个下游的Observer参数，并将其和内部存储的function函数一起作为参数实例化了一个MapObserver对象，并交给上游数据源Observable进行订阅。

## AbstractObservableWithUpstream

```java
abstract class AbstractObservableWithUpstream<T, U> extends Observable<U> implements HasUpstreamObservableSource<T> {

    /** The source consumable Observable. */
    protected final ObservableSource<T> source;

    /**
     * Constructs the ObservableSource with the given consumable.
     * @param source the consumable Observable
     */
    AbstractObservableWithUpstream(ObservableSource<T> source) {
        this.source = source;
    }

    @Override
    public final ObservableSource<T> source() {
        return source;
    }

}
```
通过源码发现，AbstractObservableWithUpstream也是Observable的子类，即Observable的装饰器。其作用就是在初始化的同时，将上游的Observable数据源作为成员变量存储起来。

目前我们看到，可以把Observable和其装饰器分为两种：

1. 最上游数据源 型Observable，类似ObservableCreate等等。
2. 类似ObservableMap，这些类实际上继承了Observable的装饰器AbstractObservableWithUpstream，而不是直接继承Observable。其作用就是在初始化的同时，将上游的Observable数据源作为成员变量存储起来。

> 总结一句话，第一种是起点Observable，第二种则是过程Observable，起到起承转合的作用，处理业务数据，并将下游的Observer订阅上游的Observable。

## Function

```java
public interface Function<T, R> {
    /**
     * Apply some calculation to the input value and return some other value.
     * @param t the input value
     * @return the output value
     * @throws Exception on error
     */
    R apply(T t) throws Exception;
}
```
其作用就是把上游的数据进行变换，因此我们看到，在源码中，这个Function函数也被当做成员进行了存储，在订阅的时候通过ObservableMap.subscribeActual()作为参数传入。

## MapObserver
MapObserver对象在执行onNext的时候，先执行funtion函数进行数据变换，将转换后的数据，传给下游传入的Observer，执行下游Observer的onNext方法。

> 在数据进行了转换之后，会将转换后的数据进行一次ObjectHelper.requireNonNull()非空校验，如果转换后的数据为空，则会抛出错误，这一点我们一定要注意，数据流的传递和转换过程中，一定要避免null值的情况。


# 归纳总结
## 创建
在未执行subscribe订阅之前，执行的流程应该是这样的：



## 订阅

Observable中存储的上游的Observable 会被下游的Observer订阅。
每一步都会生成对应的Observer对上一步生成并存储的Observable进行订阅。
在订阅时，实际上这个顺序是逆向的，从下游往上游进行订阅（Observable的装饰器模式）。

## 上游发射数据源

和订阅不同，数据的传递和变换则是正常，方向从上游往下游进行依次处理，最终执行我们subscribe中传递的Observer.onNext()。

总结

1. 创建：订阅前，每一步都生成对应的Observable对象，中间的每一步都将上游的Observable存储；
2. 订阅： 每一步都会生成对应的Observer对上一步生成并存储的Observable进行订阅。**订阅的执行顺序是由下到上的**。
3. 执行：先执行每一步传入的函数操作，然后将操作后的数据交给下游的Observer继续处理。 **数据的传递和处理顺序是由上到下的**。