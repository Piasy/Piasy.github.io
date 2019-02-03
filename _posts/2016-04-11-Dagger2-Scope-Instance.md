---
layout: post
title: Dagger2 Scope 注解能保证依赖在 component 生命周期内的单例性吗？
tags:
    - 安卓开发
    - Dagger2
    - PoC
---

[Dagger2](https://github.com/google/dagger){:target="_blank"} 已经在项目中用了一年多了，之前曾看到过[一篇文章](http://frogermcs.github.io/dependency-injection-with-dagger-2-custom-scopes){:target="_blank"}，里面说 Scope 注解可以保证依赖在每个 component 生命周期内的单例性，即局部单例。但上周一天和同事一起看生成的 injector 代码，却并未发现这一点是怎么做到的，于是得出“单例性需要 module 的 provide 方法实现来保证”的结论。但我依然对此不甚放心，决定仔细实验一番。测试代码可以在 [Github 获取](https://github.com/Piasy/AndroidPlayground/blob/master/showcase/Dagger2ScopeInstanceDemo){:target="_blank"}。

## TL; DR
+ component 的 inject 函数不要声明基类参数；
+ Scope 注解必须用在 module 的 provide 方法上，否则并不能达到局部单例的效果；
+ 如果 module 的 provide 方法使用了 scope 注解，那么 component 就必须使用同一个注解，否则编译会失败；
+ 如果 module 的 provide 方法没有使用 scope 注解，那么 component 和 module 是否加注解都无关紧要，可以通过编译，但是没有局部单例效果；
+ 对于直接使用 @Inject 构造函数的依赖，如果把 scope 注解放到它的类上，而不是构造函数上，就能达到局部单例的效果了；

## 废话不说
测试准备过程很简单，不说废话，直接上代码，然后分析 apt 生成的代码，再看 log 结果。

### 代码
三个依赖：`DemoNewDependency` 和 `DemoInjectDependency`，这两个是 interface，没有定义方法，实际注入的是实现类，但依赖注入声明的是接口类型，`DemoDirectInjectDependency` 是一个 concrete class。

`DemoNewDependencyImpl.java`:

``` java
public class DemoNewDependencyImpl implements DemoNewDependency {
    private final Context mContext;

    public DemoNewDependencyImpl(Context context) {
        mContext = context;
        Timber.d("new DemoNewDependencyImpl");
    }
}
```

`DemoInjectDependencyImpl.java`:

``` java
public class DemoInjectDependencyImpl implements DemoInjectDependency {
    private final Context mContext;

    @Inject
    public DemoInjectDependencyImpl(Context context) {
        mContext = context;
        Timber.d("new DemoInjectDependencyImpl");
    }
}
```

`DemoDirectInjectDependency.java`:

``` java
public class DemoDirectInjectDependency {
    private final Context mContext;

    @ActivityScope
    @Inject
    public DemoDirectInjectDependency(Context context) {
        mContext = context;
        Timber.d("new DemoDirectInjectDependency");
    }
}
```

注意，后两者的构造函数使用了 `@Inject` 注解，这个结合下面的 module 代码解释一下。

`DemoModule.java`:

``` java
@Module
public class DemoModule {
    @ActivityScope
    @Provides
    DemoNewDependency provideDemoNewDependency(Context context) {
        Timber.d("provideDemoNewDependency, context: " + context);
        DemoNewDependency demoNewDependency = new DemoNewDependencyImpl(context);
        Timber.d("provideDemoNewDependency, demoNewDependency: " + demoNewDependency);
        return demoNewDependency;
    }
    
    @ActivityScope
    @Provides
    DemoInjectDependency provideDemoInjectDependency(
            DemoInjectDependencyImpl demoInjectDependency) {
        Timber.d("provideDemoInjectDependency, DemoInjectDependencyImpl: "
                + demoInjectDependency);
        return demoInjectDependency;
    }
}
```

`provideDemoNewDependency` 函数调用了 `DemoNewDependencyImpl` 的构造函数创建了一个新对象后返回，这个构造函数没有用 `@Inject` 注解，而 `provideDemoInjectDependency` 函数则是接收了一个 `DemoInjectDependencyImpl` 实例参数，直接返回了，`DemoInjectDependencyImpl` 的构造函数使用了 `@Inject` 注解，而 `DemoDirectInjectDependency` 则压根儿没有 provide 方法。

这就是 dagger2 的 provide 依赖的两种方式：可以直接给构造函数加 `@Inject` 注解，这样整个 component 就有这个依赖的来源了，如：`DemoDirectInjectDependency`；也可以通过 provide 方法来返回这个依赖，同样也就为 component 提供了依赖的来源，如：`DemoNewDependency`。这里 `DemoInjectDependency` 比较特殊，`@Inject` 注解的是实现类的构造函数，而需要提供的依赖是接口类型，所以这里是无法直接提供 `DemoInjectDependency` 对象的，所以有了 `provideDemoInjectDependency` 函数，接收 `DemoInjectDependencyImpl` 实例，直接返回，由于 `DemoInjectDependencyImpl` 的构造函数加了 `@Inject` 注解，可以直接被提供，所以整个 component 的依赖就完整了。

dagger2 对父子类的支持还有一个小坑（称之为注意事项更恰当）：如果 component 的 inject 函数接收的是父类型参数，而实际调用时如果传入的是子类型对象，此时在子类型中声明要 `@Inject` 的依赖是无法注入的，只有父类型中声明的依赖会被注入。而如果 inject 函数声明接收的就是子类型参数，实际调用时传入子类型（当然也只能是子类型，传父类型存在编译错误），则子类型和父类型中声明要 `@Inject` 的依赖都可以成功注入。所以，**component 的 inject 函数不要声明基类参数**。

生成 component 之后，我们用同一个 component 对象对 `MainActivity` 和 `BlankFragment` 注入上述三种依赖。

### apt 生成的代码
我们查看把依赖注入到 MainActivity 的代码：

`MainActivity_MembersInjector.java`:

``` java
  @Override
  public void injectMembers(MainActivity instance) {
    if (instance == null) {
      throw new NullPointerException("Cannot inject members into a null reference");
    }
    instance.mDemoNewDependency = mDemoNewDependencyProvider.get();
    instance.mDemoInjectDependency = mDemoInjectDependencyProvider.get();
    instance.mDemoDirectInjectDependency = mDemoDirectInjectDependencyProvider.get();
  }
```

三个依赖都是由 provider 提供的，这三个 provider 在 `MainActivity_MembersInjector.create` 函数中传入 `MainActivity_MembersInjector` 的构造函数中进而被设置。而 `create` 方法则在 `DaggerDemoComponent` 的 `initialize` 函数中调用：

`DaggerDemoComponent.java`:

``` java
  @SuppressWarnings("unchecked")
  private void initialize(final Builder builder) {

    this.provideContextProvider = ContextModule_ProvideContextFactory
            .create(builder.contextModule);

    this.provideDemoNewDependencyProvider =
        ScopedProvider.create(
            DemoModule_ProvideDemoNewDependencyFactory.create(
                builder.demoModule, provideContextProvider));

    this.demoInjectDependencyImplProvider =
        DemoInjectDependencyImpl_Factory.create(provideContextProvider);

    this.provideDemoInjectDependencyProvider =
        ScopedProvider.create(
            DemoModule_ProvideDemoInjectDependencyFactory.create(
                builder.demoModule, demoInjectDependencyImplProvider));

    this.demoDirectInjectDependencyProvider =
        DemoDirectInjectDependency_Factory.create(provideContextProvider);

    this.mainActivityMembersInjector =
        MainActivity_MembersInjector.create(
            provideDemoNewDependencyProvider,
            provideDemoInjectDependencyProvider,
            demoDirectInjectDependencyProvider);

    this.blankFragmentMembersInjector =
        BlankFragment_MembersInjector.create(
            provideDemoNewDependencyProvider,
            provideDemoInjectDependencyProvider,
            demoDirectInjectDependencyProvider);
  }
```

这里我们就看到了，`provideDemoNewDependencyProvider` 创建的是一个 `ScopedProvider` 实例，而 `ScopedProvider` 则是实现了局部单例性的一个 provider，对同一个 ScopedProvider 调用 get 方法，得到的就是同一个对象。同样，`provideDemoInjectDependencyProvider` 也是一个 `ScopedProvider`，而 `demoDirectInjectDependencyProvider` 则就不是 ScopedProvider 了。所以这里我们可以得出结论了，mDemoNewDependency 和 mDemoInjectDependency 将会注入相同的实例，而 mDemoDirectInjectDependency 则会注入多个实例。

### log 结果

<img src="/img/201604/dagger2_scoped_instance1.png" alt="局部单例效果">

可以看到，确实验证了我们的结论，mDemoNewDependency 和 mDemoInjectDependency 注入了相同的实例，而 mDemoDirectInjectDependency 则注入了多个实例。

这里另外有两点需要注意

+ 尽管 `DemoDirectInjectDependency` 的构造函数使用了 `@ActivityScope` 注解，dagger2 并不会用 ScopedProvider 来提供这个依赖，所以它依然会存在多个实例。
+ 上面 module 类的两个 provide 方法都使用了 `@ActivityScope` 注解，这个注解是对每个 provide 方法加的，而不是加在了 module 类上，那么加在类上面有用吗？我改了之后进行了一次测试，结果如下：

  <img src="/img/201604/dagger2_scoped_instance2.png" alt="非局部单例效果">
  
  所以这里就还有另一个结论了：**Scope 注解必须用在 module 的 provide 方法上，否则并不能达到局部单例的效果**。
  
+ 那么还有一个疑问：component 是不是必须要用同一个 scope 进行注解呢？这里就不贴截图了，直接说结论：
  + **如果 module 的 provide 方法使用了 scope 注解，那么 component 就必须使用同一个注解，否则编译会失败**；
  + **如果 module 的 provide 方法没有使用 scope 注解，那么 component 和 module 是否加注解都无关紧要，可以通过编译，但是没有局部单例效果**；

## 结论
+ component 的 inject 函数不要声明基类参数；
+ Scope 注解必须用在 module 的 provide 方法上，否则并不能达到局部单例的效果；
+ 如果 module 的 provide 方法使用了 scope 注解，那么 component 就必须使用同一个注解，否则编译会失败；
+ 如果 module 的 provide 方法没有使用 scope 注解，那么 component 和 module 是否加注解都无关紧要，可以通过编译，但是没有局部单例效果；
+ 对于直接使用 @Inject 构造函数的依赖，如果把 scope 注解放到它的类上，而不是构造函数上，就能达到局部单例的效果了；

## 4.11 晚更新
在上面的测试过程中，我发现尽管 `DemoDirectInjectDependency` 的构造函数使用了 `@ActivityScope` 注解，dagger2 并不会用 ScopedProvider 来提供这个依赖，它依然会存在多个实例。一开始我觉得这可能是 dagger2 的一个 bug，所以在 github 上提了[一个 issue](https://github.com/google/dagger/issues/358){:target="_blank"}，很快有人在 issue 下进行了回复，虽然他的回复并不是直接给出解决方案，但却让我发现了一个解决方案：

+ **对于直接使用 @Inject 构造函数的依赖，如果把 scope 注解放到它的类上，而不是构造函数上，就能达到局部单例的效果了**；

而这一点也正符合 JSR330 对 Scope 的规定，只能用于类的注解。~~尽管 dagger2 在 module 的 provide 方法上利用了 scope 注解，但这是为了给我们尽可能大的灵活性，否则对于一个 module 需要有多个 scope 的情形，我们只能使用 module 组合的方式了。~~ 既然 component 必须和它所有的 module 的 provide 方法使用同一个 scope，而且一个 module 的所有 provide 方法又必须使用同一个 scope （允许有的被 scope 注解，有的不被 scope 注解，但所有使用注解的都必须使用同一个 scope 注解），那何不直接使用 scope 注解 module 类呢？

其实还是灵活性，上面一段开头的话也并不算错误，只不过这里的灵活性不是同一个 module 可以提供 ScopeA 和 ScopeB 的依赖，而是同一个 module 可以提供某种 scope 的依赖或者没有 scope 的依赖。而这种需求也是很常见的，不同组件需要同一类型的依赖，但它们必须使用不同的实例。
