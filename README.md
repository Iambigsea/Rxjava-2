RxJava map操作
=============
# Rxjava-2
今天将一下map操作，map操作的作用就是把Onsubsciber里面的参数转换为你需要的参数，比如这样，

          Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("llalala");
                subscriber.onNext("asdfasda");
                subscriber.onCompleted();
            }
        }).map(new Func1<String, Integer>() {
            @Override
            public Integer call(String s) {
                return s.hashCode();
            }
        }).subscribe(new Subscriber<Integer>() {
            @Override
            public void onCompleted() {

            }

            @Override
            public void onError(Throwable e) {

            }

            @Override
            public void onNext(Integer integer) {
                Log.i(TAG, "integer " + integer);
            }
        });

这里是把OnSubscribe里面的字符串"llalala","asdfasda"的hashcode值打印出来，就是把new Observable.OnSubscribe<String>()里面的String，</br>
转为subscribe(new Subscriber<Integer>()里面的integer。那么他是怎么完成的呢？我们先看下类图，这里我只把这里需要的方法画出来了，其他都省略了

对比上篇文章的类图，RxJavaObservableExecutionHook类多了个onLift(lift: Operator<? extends R, ? super T>): Operator<? extends R, ? super T>

Observable.create(new Observable.OnSubscribe<String>()就不看了，上篇文章看过，就是创建个observable然后把onsubscribe对象设置成它的成员变量</br>
onSubscribe,然后后面调用的方法就是map(new Func1<String, Integer>()，同样，点进去看看

      public final <R> Observable<R> map(Func1<? super T, ? extends R> func) {
        return lift(new OperatorMap<T, R>(func));
    }

这里一个new OperatorMap<T, R>(func) 对象，然后把我们的new的func1对象作为构造函数的参数穿进去了，看下OperatorMap是干嘛的

    public final class OperatorMap<T, R> implements Operator<R, T> {

    final Func1<? super T, ? extends R> transformer;

    public OperatorMap(Func1<? super T, ? extends R> transformer) {
        this.transformer = transformer;
    }

    @Override
    public Subscriber<? super T> call(final Subscriber<? super R> o) {
        return new Subscriber<T>(o) {

            @Override
            public void onCompleted() {
                o.onCompleted();
            }

            @Override
            public void onError(Throwable e) {
                o.onError(e);
            }

            @Override
            public void onNext(T t) {
                try {
                    o.onNext(transformer.call(t));
                } catch (Throwable e) {
                    Exceptions.throwOrReport(e, this, t);
                }
            }
        };
    }
  }

这个类就是把

  map(new Func1<String, Integer>() {
            @Override
            public Integer call(String s) {
                return s.hashCode();
            }
        }
        
设置进去，我们主要关注public Subscriber<? super T> call(final Subscriber<? super R> o)，那么这个o是哪个Subscriber呢，这里暂时不知道
