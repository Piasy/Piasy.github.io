---
layout: post
title: APP启动加速，以及使用FSA来处理状态转换避免Activity state loss
tags:
    - 安卓开发
    - 性能优化
---

随着APP的不断开发，启动时（Application类的onCreate函数中）需要做的事情越来越多，这将导致APP冷启动（杀死进程后的第一次启动）变慢，有分析表明，APP冷启动时间大于2s时，大部分用户将失去耐心。

# 提升APP启动速度
其实并不是所有的事情都需要在Application类的onCreate函数中执行，例如一些第三方库的初始化，可以专门增加一个SplashActivity来初始化这些第三方库，但是同样的道理，如果这些初始化工作放到SplashActivity的onCreate函数中执行，APP的冷启动依然很慢，进一步的尝试是把这些初始化工作异步化。

另外如果使用[Dagger](https://github.com/google/dagger)来实现依赖注入，还应该避免在Application类中注入依赖，毕竟创建依赖对象也是需要时间的，更多关于这个细节[请阅读](http://frogermcs.github.io/dagger-graph-creation-performance)。

# 实践
在我的[AndroidTDDBootStrap repo](https://github.com/Piasy/AndroidTDDBootStrap)中，我就尝试创建了一个SplashActivity，在其onCreate函数中首先显示一个SplashFragment，该Fragment用于显示启动页面，与此同时在后台线程进行初始化工作（使用[rx](https://github.com/ReactiveX/RxAndroid)），初始化完成后，再切换到新的GithubSearchFragment。

此时的代码是这样子的：

``` java
@Override
protected void onCreate(final Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mFragmentManager = getSupportFragmentManager();
    mFragmentManager.beginTransaction()
            .add(android.R.id.content, new SplashFragment(), SPLASH_FRAGMENT)
            .commit();

    Observable.create(subscriber -> {
            Timber.plant(new Timber.DebugTree());
            Fresco.initialize(mApp);
            // simulate heavy library initialization
            try {
                Thread.sleep(10 * 1000);
            } catch (InterruptedException e) {
                Timber.e(Constants.ERROR_LOG_FORMAT, TAG, e);
            }
            subscriber.onNext(true);
            subscriber.onCompleted();
        }).subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread())
        .subscribe(success -> {
            mFragmentManager.beginTransaction()
                .remove(mFragmentManager.findFragmentByTag(SPLASH_FRAGMENT))
                .add(android.R.id.content, new GithubSearchFragment(), GITHUB_SEARCH_FRAGMENT)
                .commit();
        });
}
```

## Activity State Loss
第一次测试运行一切OK，但是当我在初始化过程中按下home键，过了10s后app crash了，报错：

``` java
java.lang.IllegalStateException: Can not perform this action after onSaveInstanceState
```

原来是发生了Activity state loss。更多关于Activity state loss的细节[请阅读](http://www.androiddesignpatterns.com/2013/08/fragment-transaction-commit-state-loss.html)。

那么如何避免这一问题呢？总的来说就是一旦Activity onPause了，初始化工作仍然后台执行，完成后先别急着切换Fragment，应该等到onResumeFragments时再切换Fragment。

这一句话说得简单，但背后涉及的逻辑/实现却比较复杂，如何等到onResumeFragments时再切换Fragment，用一个flag变量？初始化过程中又已经onResumeFragments了怎么办，如何记录当前所处状态？这其实是一个状态转换的过程，可以抽象为一个有穷状态自动机，手绘状态机如下：

<img src="/img/1/app-launch-accelerate-state-machine.jpg" alt="StateMachine" style="height:660px">

## EasyFlow
[EasyFlow](https://github.com/Beh01der/EasyFlow)是用Java实现的轻量级有穷状态自动机，API也比较简洁，虽然已较长时间没有更新，但是作者依然能回应issue。

我采用了EasyFlow来实现上述自动机，这里有一点需要指出，由于Activity启动的时候，onResumeFragments也会被调用一次，所以仍需要用一个flag变量特殊处理一下，这里确实不太优雅，但我也没想到更好的办法，欢迎提建议！

``` java
public class SplashActivity extends BaseActivity implements HasComponent<SplashComponent> {

    private static final String SPLASH_FRAGMENT = "SplashFragment";
    private static final String GITHUB_SEARCH_FRAGMENT = "GithubSearchFragment";
    private static final String RELEASE = "release";
    private static final int TIME = 10000;
    private static final String TAG = "SplashActivity";

    @Inject
    TemplateApp mApp;
    private SplashComponent mSplashComponent;
    private FragmentManager mFragmentManager;
    private EasyFlow<StatefulContext> mFlow;
    private final StatefulContext mStatefulContext = new StatefulContext();
    private boolean mIsPaused;

    @Override
    protected void initializeInjector() {
        mSplashComponent = TemplateApp.get(this)
                .visitorComponent()
                .plus(getActivityModule(), new SplashModule());
        mSplashComponent.inject(this);
    }

    @Override
    protected void onCreate(final Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        mFragmentManager = getSupportFragmentManager();
        mFragmentManager.beginTransaction()
                .add(android.R.id.content, new SplashFragment(), SPLASH_FRAGMENT)
                .commit();

        initFlow();
        mFlow/*.trace()*/.start(mStatefulContext);
    }

    private void initFlow() {
        mFlow = FlowBuilder.from(State.Start)
                .transit(on(Event.Initialize).to(State.Initializing)
                        .transit(on(Event.Finish).finish(State.Transaction),
                                on(Event.Pause).to(State.Wait4InitializedAndResume)
                                        .transit(on(Event.Resume).to(State.Initializing),
                                                on(Event.Finish).to(State.Wait4Resume)
                                                        .transit(on(Event.Resume).finish(
                                                                State.Transaction)))))
                .executor(new UiThreadExecutor());

        mFlow.whenEnter(State.Start, context -> {
            context.setState(State.Start);
            Observable.create(subscriber -> {
                Timber.plant(new Timber.DebugTree());
                try {
                    mFlow.trigger(Event.Initialize, mStatefulContext);
                } catch (LogicViolationError logicViolationError) {
                    Timber.e(Constants.ERROR_LOG_FORMAT, TAG, logicViolationError);
                }
                Fresco.initialize(mApp);
                // simulate heavy library initialization
                try {
                    Thread.sleep(10 * 1000);
                } catch (InterruptedException e) {
                    Timber.e(Constants.ERROR_LOG_FORMAT, TAG, e);
                }
                subscriber.onNext(true);
                subscriber.onCompleted();
            }).subscribeOn(Schedulers.io()).subscribe(success -> {
                try {
                    mFlow.trigger(Event.Finish, mStatefulContext);
                } catch (LogicViolationError logicViolationError) {
                    Timber.e(Constants.ERROR_LOG_FORMAT, TAG, logicViolationError);
                }
            });
        }).whenEnter(State.Transaction, context -> {
            context.setState(State.Transaction);
            mFragmentManager.beginTransaction()
                    .remove(mFragmentManager.findFragmentByTag(SPLASH_FRAGMENT))
                    .add(android.R.id.content, new GithubSearchFragment(), GITHUB_SEARCH_FRAGMENT)
                    .commit();
        });
    }

    @Override
    protected void onResumeFragments() {
        super.onResumeFragments();
        if (mIsPaused) {
            try {
                mFlow.trigger(Event.Resume, mStatefulContext);
            } catch (LogicViolationError logicViolationError) {
                Timber.e(Constants.ERROR_LOG_FORMAT, TAG, logicViolationError);
            }
        }
    }

    @Override
    protected void onPause() {
        super.onPause();
        mIsPaused = true;
        try {
            mFlow.trigger(Event.Pause, mStatefulContext);
        } catch (LogicViolationError logicViolationError) {
            Timber.e(Constants.ERROR_LOG_FORMAT, TAG, logicViolationError);
        }
    }

    @Override
    public SplashComponent getComponent() {
        return mSplashComponent;
    }

    /**
     * Init state enum.
     * TODO modify EasyFlow to avoid enum.
     */
    enum State implements StateEnum {
        Start, Initializing, Wait4InitializedAndResume, Wait4Resume, Transaction
    }

    /**
     * Init event enum.
     * TODO modify EasyFlow to avoid enum.
     */
    enum Event implements EventEnum {
        Initialize, Pause, Resume, Finish
    }
}
```

EasyFlow通过enum定义状态和触发状态切换的事件，对于安卓平台来说，使用enum不是一个好的方式，会影响性能，这一点需要改进。

## 实测
经过测试，在初始化过程中按下home键，然后又重新返回APP，如此反复多次，也不会发生Activity state loss了，并且能及时切换Fragment，APP启动速度也很可观，使用[NimbleDroid](https://www.nimbledroid.com)进行测试，APP冷启动耗时1.39s，击败了绝大多数应用市场上的APP :) 。

<img src="/img/1/app-launch-accelerate-nimbledroid-summary.png" alt="NimbleDroidSummary" style="height:300px">

当然这只是个玩笑，但是在模拟中加了一个10s的sleep，足以应对日后可能的初始化需求了。

完整的代码可以在[AndroidTDDBootStrap repo](https://github.com/Piasy/AndroidTDDBootStrap/blob/master/presentation%2Fsrc%2Fmain%2Fjava%2Fcom%2Fgithub%2Fpiasy%2Ftemplate%2Ffeatures%2Fsplash%2FSplashActivity.java)中获取。
