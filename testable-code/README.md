# Testing 101

## Unit-Testable Code

Wait, my code has a structure. I do MVP. Isn't that good enough?

Well maybe. But first let's have a look at this example.

```java
class FooPresenter extends FooContract.Presenter {

    private FooContract.View mView;
    private Activity mActivity;

    public FooPresenter(FooContract.View view, Activity activity) {
        this.mView = view;
        this.mActivity = activity;
    }

    @Override
    void getBar() {
        ApiHelper.getInstance().getBar("barId", new ApiCallback<Bar>() {
            @Override
            void onSuccess(Bar bar) {
                mView.showBar(bar);
                RealmHelper.saveBar(bar);
                SharedPreference sp = mActivity.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE);
                sp.edit().put("baz", bar.baz).apply();
            }
            @Override
            void onFailure(Throwable t) {
                mView.showError(t.getMessage());
            }
        })
    }
}

```
Typical MVP's presenter, right? Is there something wrong with it?

No, there's nothing wrong with it - if your goal is separation of concern. This is absolutely ok. You have your presenter get some data from an api call, save it to a local DB and tell a view to show it. Done! You have mastered MVP.

Next, let's write a test. Good developer writes test, right? We're gonna check if our presenter wire the flow correctly or not.

```java
class FooPresenterTest {

    @Mock FooContract.View mView;
    @Mock Activity mActivity;
    FooPresenter presenter;

    @Before
    void setup() {
        MockitoAnnotations.initMocks(this);
        presenter = new FooPresenter(mView, mActivity);
    }

    @Test
    void testGetBarFail() {
        presenter.getBar();
        verify(mView).showError(any());
    }
}
```

We'll start with `testGetBarFail()` you just call get bar and see if `mView.showError()` was called. Now run your test.

>`mView.showError()` was not called.

Why?

First, you are using a real api call in test, it's gonna run on another thread and the result is never gonna get back.

Ok, how can we fix this?

```java
class FooPresenter extends FooContract.Presenter {

    private FooContract.View mView;
    private Activity mActivity;
    private ApiHelper mApiHelper;

    public FooPresenter(FooContract.View view, Activity activity, ApiHelper apiHelper) {
        this.mView = view;
        this.mActivity = activity;
        this.mApiHelper = apiHelper;
    }

    @Override
    void getBar() {
        mApiHelper.getBar("barId", new ApiCallback<Bar>() {
            @Override
            void onSuccess(Bar bar) {
                mView.showBar(bar);
                RealmHelper.saveBar(bar);
                SharedPreference sp = mActivity.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE);
                sp.edit().put("baz", bar.baz).apply();
            }
            @Override
            void onFailure(Throwable t) {
                mView.showError(t.getMessage());
            }
        })
    }
}
```
```java
class FooPresenterTest {

    @Mock FooContract.View mView;
    @Mock Activity mActivity;
    @Mock ApiHelper mApiHelper;
    @Captor ArgumentCaptor<ApiCallback<Bar>> callbackCaptor;
    FooPresenter presenter;

    @Before
    void setup() {
        MockitoAnnotations.initMocks(this);
        presenter = new FooPresenter(mView, mActivity, mApiHelper);
    }

    @Test
    void testGetBarFail() {
        presenter.getBar();
        verify(mApiHelper).getBar(any(), callbackCaptor.capture());
        callbackCaptor.getValue().onFailure(new Exception());
        verify(mView).showError(any());
    }
}
```
You don't call real api in tests (except it's an integration test), you have to call a fake api. But with `ApiHelper.getInstance()` you can't replace a real api call with a mock one in a test. You need to inject it instead of calling `getInstance()` directly.

But we are not out of the woods yet. With mock api, it's not gonna do anything, the callback is never gonna get called, so we have to use `ArgumentCaptor` to control the callback ourself. Finally we can check `mView.showError()` invocation.

It might seems a little stupid at first, you fake the result and check if you fake it correctly, but that's how it's done. Imageine a method with more logic, more complex, it'll make sense in time.

Now let's test a success case.

```java
class FooPresenterTest {

    @Mock FooContract.View mView;
    @Mock Activity mActivity;
    @Mock ApiHelper mApiHelper;
    @Captor ArgumentCaptor<ApiCallback<Bar>> callbackCaptor;
    FooPresenter presenter;

    @Before
    void setup() {
        MockitoAnnotations.initMocks(this);
        presenter = new FooPresenter(mView, mActivity, mApiHelper);
    }

    @Test
    void testGetBarSuccess() {
        presenter.getBar();
        verify(mApiHelper).getBar(callbackCaptor.capture());
        callbackCaptor.getValue().onSuccess(new Bar())
        verify(mView).showBar(any());
        verify(RealmHelper).saveBar(any());
        SharedPreference sp = mActivity.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE);
        verify(sp).edit().put("baz", any()).apply();
    }
}
```

The code is not even compile, man! We can't verify static method calls (well there's a way but you shouldn't).

We have to rewrite our `RealmHelper.saveBar()` into an instance method. We can do it like `ApiHelper` but there's a better way.

```java
interface AppStorage {
    void saveBar(Bar bar);
}

class RealmStorage implements AppStorage {
    @Override
    void saveBar(Bar bar) {
        // save to realm, you can even call RealmHelper.saveBar(bar) here.
    }
}
```
```java
class FooPresenter extends FooContract.Presenter {

    private FooContract.View mView;
    private Activity mActivity;
    private ApiHelper mApiHelper;
    private AppStorage mAppStorage;

    public FooPresenter(FooContract.View view, Activity activity, ApiHelper apiHelper, AppStorage appStorage) {
        this.mView = view;
        this.mActivity = activity;
        this.mApiHelper = apiHelper;
        this.mAppStorage = appStorage;
    }

    @Override
    void getBar() {
        mApiHelper.getBar("barId", new ApiCallback<Bar>() {
            @Override
            void onSuccess(Bar bar) {
                mView.showBar(bar);
                mAppStorage.saveBar(bar);
                SharedPreference sp = mActivity.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE);
                sp.edit().put("baz", bar.baz).apply();
            }
            @Override
            void onFailure(Throwable t) {
                mView.showError(t.getMessage());
            }
        })
    }
}
```
```java
class FooPresenterTest {

    @Mock FooContract.View mView;
    @Mock Activity mActivity;
    @Mock ApiHelper mApiHelper;
    @Mock AppStorage mAppStorage;
    @Captor ArgumentCaptor<ApiCallback<Bar>> callbackCaptor;
    FooPresenter presenter;
    Bar FAKE_BAR;

    @Before
    void setup() {
        MockitoAnnotations.initMocks(this);
        presenter = new FooPresenter(mView, mActivity, mApiHelper, mAppStorage);
        FAKE_BAR = new Bar();
    }

    @Test
    void testGetBarSuccess() {
        presenter.getBar();
        verify(mApiHelper).getBar(callbackCaptor.capture());
        callbackCaptor.getValue().onSuccess(FAKE_BAR);
        verify(mView).showBar(FAKE_BAR);
        verify(mAppStorage).saveBar(FAKE_BAR);
        SharedPreference sp = mActivity.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE);
        verify(sp).edit().put("baz", any()).apply();
    }
}
```
That's better. By using interface, our presenter is decoupled from `RealmStorage`, you plug any other implementation of `AppStorage` in without having to rewrite a single line of presenter code. This is called Dependency Inversion Principle or whatever, I don't know, I don't have a computer science degree!

Now run your test and find it red. This is because `mActivity` is just a mock object, it's not gonna do anything useful. In the presenter you'll ended up getting a `NullPointerException` caused by a null `SharedPreference` returned by a mock mActivity.

At this point, many people will tell you to move `Activity` out of the presenter because it is Android. Presenter should be independent of the Android framework. I agree, but I don't think we are going to reuse our presenter somewhere else. The real reason is it is hard to unit test a presenter when you have Android dependencies, you have to run an instrumented unit test or use Robolectric. So it is best to keep Android dependencies out of the presenter.

Here's what we are going to do. We abstract out the SharedPreference logic using interface - just like RealmHelper.

```java
interface AppStorage {
    void saveBar(Bar bar);
    void saveBaz(Baz baz);
}

class AppStorageImpl implements AppStorage {
    private Context mContext;
    public AppStorageImpl(Context context) {
        this.mContext = context;
    }
    @Override
    void saveBar(Bar bar) {
        // save to realm, you can even call RealmHelper.saveBar(bar) here.
    }
    @Override
    void saveBaz(Baz baz) {
        SharedPreference sp = mContext.getSharedPreference("PREF_NAME", Context.MODE_PRIVATE;
        sp.edit().put("baz", baz).apply();
    }
}
```
```java
class FooPresenter extends FooContract.Presenter {

    private FooContract.View mView;
    private ApiHelper mApiHelper;
    private AppStorage mAppStorage;

    public FooPresenter(FooContract.View view, ApiHelper apiHelper, AppStorage appStorage) {
        this.mView = view;
        this.mApiHelper = apiHelper;
        this.mAppStorage = appStorage;
    }

    @Override
    void getBar() {
        mApiHelper.getBar("barId", new ApiCallback<Bar>() {
            @Override
            void onSuccess(Bar bar) {
                mView.showBar(bar);
                mAppStorage.saveBar(bar);
                mAppStorage.saveBaz(bar.baz);
            }
            @Override
            void onFailure(Throwable t) {
                mView.showError(t.getMessage());
            }
        })
    }
}
```
```java
class FooPresenterTest {

    @Mock FooContract.View mView;
    @Mock ApiHelper mApiHelper;
    @Mock AppStorage mAppStorage;
    @Captor ArgumentCaptor<ApiCallback<Bar>> callbackCaptor;
    FooPresenter presenter;
    Bar FAKE_BAR;

    @Before
    void setup() {
        MockitoAnnotations.initMocks(this);
        presenter = new FooPresenter(mView, mActivity, mApiHelper, mAppStorage);
        FAKE_BAR = new Bar();
    }

    @Test
    void testGetBarFail() {
        presenter.getBar();
        verify(mApiHelper).getBar(any(), callbackCaptor.capture());
        callbackCaptor.getValue().onFailure(new Exception());
        verify(mView).showError(any());
    }

    @Test
    void testGetBarSuccess() {
        presenter.getBar();
        verify(mApiHelper).getBar(callbackCaptor.capture());
        callbackCaptor.getValue().onSuccess(FAKE_BAR);
        verify(mView).showBar(FAKE_BAR);
        verify(mAppStorage).saveBar(FAKE_BAR);
        verify(mAppStorage).saveBaz(FAKE_BAR.baz);
    }
}
```

Now everything is testable!

I took a liberty of combining RealmStorage and SharedPreference together for convenience. You can design these things however you want. Some people adopt Repository pattern, but it's your choice.

Finally let's see how it looks like in real code.

```java
//impossible to test version
class FooActivity extends AppCompatActivity implements FooContract.View {
    
    FooContract.Presenter presenter;

    @Override
    void onCreate(Bundle bundle) {
        presenter = new FooPresenter(this, this);
    }
}

//testable version
class FooActivity extends AppCompatActivity implements FooContract.View {

    FooContract.Presenter presenter;

    @Override
    void onCreate(Bundle bundle) {
        presenter = new FooPresenter(this, ApiHelper.getInstance(), new AppStorageImpl(this));
    }
}
```
## In summary

### Don't
- No real api calls
- No `new Foo()`
- No static method calls
- No `Singletion.getInstance()`
- No Android in presenters

### Do's
- Inject your dependencies
- Inject your singleton
- No static method calls EVER!
- Break the class using interface


## Takeaway

There are 2 things to keep in mind when you write a testable code.

1. <b>Sensing.</b> How do you know your code works correctly.
2. <b>Separation.</b> Break it down if it is impossible/difficult to test.