# RxJava 101

## Basic Building Blocks/Terms

### Observable

The observe-able term might be a little confusing but this basically means "I can provide you with data. You can subscribe me to wait for them, I'll call you when they're available."

```java
public interface ObservableSource<T> {
    void subscribe(Observer<? super T> observer);
}
```

### Observer

Observer observes Observable. Suppose you want some data from Observable, you implement an Observer, have it subscribe to an Observable. When the data is available, the Observable will tell you by calling `Observer.onNext(T)`

```java
public interface Observer<T> {
    void onSubscribe(Disposable d);
    void onNext(T t);
    void onError(Throwable e);
    void onComplete();
}
```

You might think this is a lot like an interface callback/listener. Well yes, it is. It is a general purpose callback for just about anything. Your typicallback might look like this.

```java
public interface MyCallback<T> {
    void onStart();
    void onComplete(T t);
    void onFailure(Throwable e);
}
```

But in reactive world, the data might not come only once like `MyCallback.onComplete(T)` does, so we need an `Observer.onNext(T)` to let the data keep comming in - until `Observer.onComplete()` or you unsubscribe.

### Subscribe

Subscribe is the act when you call `Observable.subscribe(Observer)`. It means you want your Observer to listen for some data from an Observable.

In a non-reactive world this is when you call a method, passing in a callback and wait for the result.

### Disposable

When your Observer subscribes to an Observable you get a Disposable. In case you don't want the data anymore you can call `Disposable.dispose()` to unsubscribe.

In a non-reactive world, this is a way to cancel the work in progress before the callback is called.

```java
public interface Disposable {
    void dispose();
    boolean isDisposed();
}
```

### Event/Data/Item/Signal/Message

I have to admit I don't know what the correct term is. But this is the data passing from Observable to Observer through `onNext(T)`. This is your T.

### Stream

In reactive world, `onNext(T)` can be called multiple times, even endlessly. This is why people call it stream, it is a stream of data that keeps flowing.

## Example

The `ObservableSource<T>` is just an interface, we don't use it directly. We use a concrete class `Observable<T>` instead.

I'm not gonna show you how to actually create an Observable just yet. Let's assume we have an Observable ready, for example, a Retrofit interface. You just define your interface with Observable return type and the Retrofit takes care of creating the observables for you.

```java
interface MyApi {
    @GET("/helloworld")
    Observable<String> helloWorld();
}
```

The api is super simple, it's gonna return a "Hello World" String.

```java
myApi.helloWorld()
    .subscribe(new Observer<String>() {
        @Override
        void onNext(String string) {
            myTextView.setText(string);
        }
        //...
    });
```

Basically you encapsulate your logic into an observable, call subscribe to start the work, passing in an observer to wait for the result. When the work is done, you get your "Hello World" in `onNext(String)`.

## Operator

Operators provide many functionality for Observable. To name a few, create, combine, transform.

```java
//create
Observable.just("Hello World");
Observable.fromIterable(myList);

//combine
Observable<String> fullNameObservable = Observable.combineLatest(
    firstNameObservable,
    lastNameObservable,
    new BiFunction<String,String,String>() {
        @Override
        String apply(String s1, String s2) {
            return s1 + " " +s2;
        }
});

//transform
Observable<String> fullNameLengthObservable = fullnameObservable.map(new Function<String,Integer>() {
    @Override
    Integer apply(String string) {
        return string.length;
    }
});
```

## Creation

### Observable.fromCallable()

```java
public interface Callable<V> {
    V call();
}

Observable<String> hwObservable = Observable.fromCallable(new Callable<String>() {
    @Override
     String call() {
         return myApi.getHelloWorldFromServerSync();
     }
})

hwObservable.subscribe(new Observer<String>() {
    @Override
    void onNext(String string) {
        println(string);
    }
})
```

Run a callable when subscribe. You'll get your T in `onNext(T)`

### Observable.create()

Turns a callback into an observable.
