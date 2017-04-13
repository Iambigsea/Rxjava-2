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
![类图](https://github.com/Iambigsea/Rxjava-2/blob/master/map.png?raw=true)
对比上篇文章的类图，RxJavaObservableExecutionHook类多了个onLift(lift: Operator<? extends R, ? super T>): Operator<? extends R, ? super T> Observable.create(new Observable.OnSubscribe<String>()就不看了，上篇文章看过，就是创建个observable然后把onsubscribe对象设置成它的成员变量
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
        
设置进去，我们主要关注public Subscriber<? super T> call(final Subscriber<? super R> o)，那么这个o是哪个Subscriber呢，这里暂时不知道,就先不纠结了，看看return new Subscriber<T>(o）的对象，它在onNext方法里面调用参数o的onNext方法，参数接收的transformer.call(t)。这个transformer就是new Func1<String, Integer>，它的call方法就是把接收到的String转为.subscribe(new Subscriber<Integer>()接收到参数interge。这里已经很明白了，这个OperatorMap就是个转换器。</br>
回到方法lift(new OperatorMap<T, R>(func))中，继续点进lift方法里

        public final <R> Observable<R> lift(final Operator<? extends R, ? super T> operator) {
		return new Observable<R>(new OnSubscribe<R>() {
		    @Override
		    public void call(Subscriber<? super R> o) {
			try {
			    Subscriber<? super T> st = hook.onLift(operator).call(o);
			    try {
				st.onStart();
				onSubscribe.call(st);
			    } catch (Throwable e) {
				Exceptions.throwIfFatal(e);
				st.onError(e);
			    }
			} catch (Throwable e) {
			    Exceptions.throwIfFatal(e);
			    o.onError(e);
			}
		    }
		});
    	}
    
    
这里重新创建new Observable<R>(new OnSubscribe<R>()，在方法call(Subscriber<? super R> o)里调用hook.onLift(operator).call(o)重新生成</br> Subscriber<? super T> st,点进去看看这个st是什么东东，hook在上一章说过这个hook就是类的RxJavaObservableExecutionHook对象，完整方法onLift为

	   public abstract class RxJavaObservableExecutionHook {
		......
	    public <T, R> Operator<? extends R, ? super T> onLift(final Operator<? extends R, ? super T> lift) {
		return lift;
	    }
	  }

好吧，还是和之前一样，什么都没干，直接把传进来的lift返回了，就是说这个Subscriber<? super T> st是operator.call(o)返回的，也就是我们刚刚说的，然后st.onStart()；这个就是调用st的模版方法，没什么好说的
来看看这个onSubscribe.call(st);这个onSubscribe是什么？其实就是我们一开始在

          Observable.create(new Observable.OnSubscribe<String>() {
            @Override
            public void call(Subscriber<? super String> subscriber) {
                subscriber.onNext("llalala");
                subscriber.onNext("asdfasda");
                subscriber.onCompleted();
            }
        }

创建的的OnSubscribe对象，它的call方法，就是把String字符串传传到参数Subscriber<? super String> subscriber上，那么这个public void call(Subscriber<? super R> o)的参数o到底是什么呢？这里对应昨天我们的最后简化写法，这里也可以简化成这样
	
	     new Observable<R>(new OnSubscribe<R>() {
		     @Override
		     public void call(Subscriber<? super R> o) {
			 try {
			     Subscriber<? super T> st = hook.onLift(operator).call(o);
			     try {
				 st.onStart();
				 onSubscribe.call(st);
			     } catch (Throwable e) {
				 Exceptions.throwIfFatal(e);
				 st.onError(e);
			     }
			 } catch (Throwable e) {
			     Exceptions.throwIfFatal(e);
			     o.onError(e);
			 }
		     }
		 }.subscribe(new Subscriber<Integer>() {
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
	
	
这样就很明白了这个o就是我们在.subscribe(new Subscriber<Integer>()创建的Subscriber对象，然后倒回去看

	public void call(Subscriber<? super R> o) {
			try {
			    Subscriber<? super T> st = hook.onLift(operator).call(o);
			    try {
				st.onStart();
				onSubscribe.call(st);
			    } catch (Throwable e) {
				Exceptions.throwIfFatal(e);
				st.onError(e);
			    }
			} catch (Throwable e) {
			    Exceptions.throwIfFatal(e);
			    o.onError(e);
			}
		    }
		});
		
就是先通过OperatorMap创建一个新的Subscriber，这个Subscriber负责把原始数据String转换为我们需要的Integer,接着调用st.onStart()，再接着调用一开始在create里面创建的参数对象OnSubscribe的call方法，设置call(st)的st为我们负责转换数据的Subscriber，Subscriber接受到原始参数后，再转换成目标参数，再传递到最后的subscribe(new Subscriber<Integer>()的方法onNext(Integer integer)里面，
下面看下这个流程图就应该更清楚了


o了，flatmap见

