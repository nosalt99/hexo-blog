---
title: ReactNative-启动流程
date: 2018-12-09 09:30:33
layout: config.default_layout
categories: 'React Native'
tags:
- React Native
---
# ReactNative-启动流程

## 背景

ReactNative在真机iOS和Android上，使用的是**JavaScriptCore**引擎，也就是Safari所使用的JavaScript引擎。但是在iOS上JavaScriptCore并没有使用即时编译技术（JIT），因为在iOS中应用无权拥有可写可执行的内存页（因而无法动态生成代码），在安卓上，理论上是可以使用的。JavaScriptCore引擎也是使用C++编写，在iOS和安卓中，JavaScriptCore都做了一层封装，可以无须关心引擎和系统桥接的那一层。iOS/Android系统通JavaScriptCore引擎将定制好的各种原生组件注入。我们编写少量Java代码大量JavaScript代码，中间作为桥梁的核心是 C/C++ 来处理的。

![img](https://cdn.nlark.com/yuque/0/2018/png/199077/1542525129415-a4873dbe-d967-4554-a866-aa6a275c2594.png)

## RN 启动流程框架浅析

集成 RN 无非就是通过继承 ReactActivity 或者自己通过 ReactRootView 进行处理，但是实质都是触发了ReactRootView 的 startReactApplication 方法

- ReactActivity

先来看看ReactActivity,明显的代理模式，实际操作全部交给了ReactActivityDelegate

```
public abstract class ReactActivity extends Activity
    implements DefaultHardwareBackBtnHandler, PermissionAwareActivity {
  private final ReactActivityDelegate mDelegate;
  protected ReactActivity() {
    mDelegate = createReactActivityDelegate();
  } 
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mDelegate.onCreate(savedInstanceState);
  }
  @Override
  protected void onPause() {
    super.onPause();
    mDelegate.onPause();
  }
  @Override
  protected void onResume() {
    super.onResume();
    mDelegate.onResume();
  }
   ..........
}
```

- ReactActivityDelegate

再来看看ReactActivityDelegate,可以看见，ReactActivityDelegate 只是一个抽出来的封装，上面的实质就是 new 了一个 ReactRootView（实质是 Android 的 FrameLayout），接着调用 ReactRootView 的 startReactApplication 方法，完事就像常规 Android 代码一样直接通过 Activity 的 setContentView 方法把 View 设置进去。所以可以看出来，RN 的神秘之处一定在于 ReactRootView 中，Activity 对于 RN 来说只是为了让 RN 依附符合 Android 的框架而已，所以说，说白了 RN 依旧是标准 Android，因此在我们集成开发中我们可以选择整个界面（包含多级跳转）都用 React Native 实现，或者一个 Android 现有界面中部分采用 React Native 实现，因为这货就是一个 View.

```
public class ReactActivityDelegate {

  protected void onCreate(Bundle savedInstanceState) {
    //启动流程一定会执行的，mMainComponentName为我们设置的，与JS边保持一致
    if (mMainComponentName != null) {
      loadApp(mMainComponentName);
    }
    mDoubleTapReloadRecognizer = new DoubleTapReloadRecognizer();
  }

  protected void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    // createRootView其实就是create了一个FrameLayout，核心方法 
    mReactRootView = createRootView();
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      appKey,
      getLaunchOptions());
    // 把view设置进Activity
    getPlainActivity().setContentView(mReactRootView);
  }
}
```

- ReactRootView

既然明白了 RN 就是个 View，那就接着看看 ReactRootView，如下：

可以看见，ReactRootView的英文注释已经交代很清楚用途和地位了，直接看上面代码的startReactApplication 方法，可以看见他又调用了一个三个参数的同名方法，具体这三个参数来历如下（也是我们自己集成 RN 时手动 builder 模式创建的）：

1. reactInstanceManager： 大内总管接口类，提供一个构造者模式的初始化 Builder, 这类也是我们在集成 RN 时 new ReactRootView 的之前自己创建的。 

1. moduleName： 与 JS 代码约定的 String 类型识别 name，JS 端通过 AppRegistry.registerComponent 方法设置这个 name，Java 端重写基类的 getMainComponentName 方法设置这个 name，这样两边入口就对上了。 

1. launchOptions： 这里默认是 null 的，如果自己不继承 ReactActivity 而自己实现的话可以通过这个参数在 startActivity 时传入一些参数到 JS 代码，用来依据参数初始化 JS 端代码。

可以看见接着调用了 mReactInstanceManager createReactContextInBackground 方法，mReactInstanceManager 就是上面说的第一个参数，实质是通过一个构造者模式创建的

```
/**
 React Native 的 RootView，负责监听标准 Android 的 View 相关的事件分发和子View 渲染等。
 */
public class ReactRootView extends SizeMonitoringFrameLayout implements RootView {
  public void startReactApplication(ReactInstanceManager reactInstanceManager, String moduleName) {
    startReactApplication(reactInstanceManager, moduleName, null);
  }

/**
调度react component的渲染，使用reactInstanceManager讲js模块attch到JS Context中。一些初始化参数可以通过launchOptions传递给react component
*/
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle initialProperties) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "startReactApplication");
    try {
      UiThreadUtil.assertOnUiThread();
      Assertions.assertCondition(
        mReactInstanceManager == null,
        "This root view has already been attached to a catalyst instance manager");
      mReactInstanceManager = reactInstanceManager;
      mJSModuleName = moduleName;
      mAppProperties = initialProperties;
      // 核心创建ReactContext，先判断是否初始化了
      if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
        mReactInstanceManager.createReactContextInBackground();
      }
      attachToReactInstanceManager();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    }
  }
}
```

- ReactInstanceManager

ReactInstanceManager的createReactContextInBackground 方法看看，如下：

可以看到createReactContext经过一系列的包装，最终开启了一个新线程去创建ReactContext,先从initParams中获取JavaScriptExecutor和JsBundleLoader，JsBundleLoader负责JsBundle的加载，实际情况中我们可以修改加载起实现热更新。开始创建后，有几个核心方法createReactContext和setupReactContext，接下来我们来看看createReactContext:

总的来说，createReactContext()把nativeModules注册到CatalysInstanceImpl里,然后把这两张映射表交给 CatalystInstanceImpl，同时包装创建 ReactContext 对象，然后通过 CatalystInstanceImpl 的 runJSBundle() 方法把 JS bundle 文件的 JS 代码加载进来等待 Task 结束以后调用 JS 入口进行渲染 RN。CatalystInstanceImpl 的 build 方法中调用的 CatalystInstanceImpl 构造方法到底干了什么



build()中其实就是构造者模式，实例化CatalystInstanceImpl, CatalystInstanceImpl 就是个封装总管，负责了 Java 层代码到 JNI 封装初始化的任务和 Java 与 JS 调用的 Java 端控制中心。核心方法是initializeBridge,这是个native方法，我们看看他穿了什么参数进去：

1. ReactCallBack是CatalystInstanceImpl 的内部静态实现类 BridgeCallback，负责相关接口回传和回

1. jsExecutor参数： 前面分析的 XReactInstanceManagerImpl 中赋值为 JSCJavaScriptExecutor 实例，JSCJavaScriptExecutor 中也有自己的 native initHybrid 的 C++ 方法被初始化时调用，具体在 OnLoad.cpp 的 JSCJavaScriptExecutorHolder 类中。 

1. jsQueue参数： 来自于 mReactQueueConfiguration.getJSQueueThread()，mReactQueueConfiguration就是 CatalystInstanceImpl 中创建的 ReactQueueConfigurationImpl.create( ReactQueueConfigurationSpec, new NativeExceptionHandler()); 第一个参数来自于 XReactInstanceManagerImpl 中 CatalystInstanceImpl 的建造者，实质为包装相关线程名字、类型等，然后通过 ReactQueueConfigurationImpl 的 create 创建对应线程的 Handler，这里就是名字为 js 的后台线程 Handler，第二个参数为异常捕获回调实现。 

1. moduleQueue参数： 来自于 mReactQueueConfiguration.getNativeModulesQueueThread（）mReactQueueConfiguration同上

1. javaModules: 来自mNativeModuleRegistry.getJavaModules(this),mNativeModuleRegistry就是构建CatalysInstanceImpl时传入的javaModules。

1. cxxModules：来自mNativeModuleRegistry.getCxxModules(),mNativeModuleRegistry就是构建CatalysInstanceImpl时传入的javaModules，获取c++模块

CatalystInstanceImpl 自己在 Java 层直接把持住了 JavaScriptModuleRegistry 映射表，把 NativeModuleRegistry 映射表、BridgeCallback 回调、JSCJavaScriptExecutor、js 队列 MessageQueueThread、native 队列 MessageQueueThread 都通过 JNI 嫁接到了 C++ 中。那我们现在先把目光转移到 CatalystInstanceImpl.cpp 的 initializeBridge 方法上（关于 JNI 的 OnLoad 中初始化注册模块等等就不介绍了），如下：

```
 @ThreadConfined(UI)
public void recreateReactContextInBackground() {
   Assertions.assertCondition(
       mHasStartedCreatingInitialContext,
       "recreateReactContextInBackground should only be called after the initial " +
           "createReactContextInBackground call.");
   recreateReactContextInBackgroundInner();
 }

@ThreadConfined(UI)
 private void recreateReactContextInBackgroundInner() {
   if (mUseDeveloperSupport
       && mJSMainModulePath != null
       && !Systrace.isTracing(TRACE_TAG_REACT_APPS | TRACE_TAG_REACT_JS_VM_CALLS)) {
     final DeveloperSettings devSettings = mDevSupportManager.getDevSettings();

     // If remote JS debugging is enabled, load from dev server.
     if (mDevSupportManager.hasUpToDateJSBundleInCache() &&
         !devSettings.isRemoteJSDebugEnabled()) {
       // If there is a up-to-date bundle downloaded from server,
       // with remote JS debugging disabled, always use that.
       ........
     return;
   }
    //非调试模式，真实环境下去加载Bundle
   recreateReactContextInBackgroundFromBundleLoader();
 }
     
 public void createReactContextInBackground() {
   ......
   recreateReactContextInBackgroundInner();
 }

 private void recreateReactContextInBackgroundInner() {
   UiThreadUtil.assertOnUiThread();
   
   if (mUseDeveloperSupport && mJSMainModuleName != null) {
   //如果是 dev 模式，BuildConfig.DEBUG=true就走这里，在线更新bundle，手机晃动出现调试菜单等等。
   //这个路线属于RN调试流程原理，后面再写文章分析，这里我们抓住主线分析
 ......
 return;
}
    //非调试模式，即BuildConfig.DEBUG=false时执行
    recreateReactContextInBackgroundFromBundleLoader();
}

private void recreateReactContextInBackgroundFromBundleLoader() {
   //在后台创建ReactContext，两个参数是重点。
   //包装了JavaScript执行上下文的JavaScriptExecutorFactory
   //自定义热更新时setJSBundleFile方法参数就是巧妙的利用这里是走JSBundleLoader.createAssetLoader还是JSBundleLoader.createFileLoader！！！！！！
     recreateReactContextInBackground(mJavaScriptExecutorFactory, mBundleLoader);
}

@ThreadConfined(UI)
 private void recreateReactContextInBackground(
   JavaScriptExecutorFactory jsExecutorFactory,
   JSBundleLoader jsBundleLoader) {
     //包装一下JsExecutorFactor
   final ReactContextInitParams initParams = new ReactContextInitParams(
     jsExecutorFactory,
     jsBundleLoader);
     // 开线程差创建ReactContext
   if (mCreateReactContextThread == null) {
     runCreateReactContextOnNewThread(initParams);
   } else {
     mPendingReactContextInitParams = initParams;
   }
 }

 @ThreadConfined(UI)
 private void runCreateReactContextOnNewThread(final ReactContextInitParams initParams) {
   .....
   //开线程创建ReactContext
   mCreateReactContextThread =
       new Thread(
           new Runnable() {
             @Override
             public void run() {
           .....
               try {
                 Process.setThreadPriority(Process.THREAD_PRIORITY_DISPLAY);
                 //创建ReactContext的核心方法，createReactContext
                 final ReactApplicationContext reactApplicationContext =
                     createReactContext(
                         initParams.getJsExecutorFactory().create(),
                         initParams.getJsBundleLoader());

                 mCreateReactContextThread = null;
                 
                 final Runnable maybeRecreateReactContextRunnable =
                     new Runnable() {
                       @Override
                       public void run() {
                         if (mPendingReactContextInitParams != null) {
                           runCreateReactContextOnNewThread(mPendingReactContextInitParams);
                           mPendingReactContextInitParams = null;
                         }
                       }
                 };
                 Runnable setupReactContextRunnable =
                     new Runnable() {
                       @Override
                       public void run() {
                         try {
                           setupReactContext(reactApplicationContext);
                         } catch (Exception e) {
                           mDevSupportManager.handleException(e);
                         }
                       }
                  };
        reactApplicationContext.runOnNativeModulesQueueThread(setupReactContextRunnable);
                 UiThreadUtil.runOnUiThread(maybeRecreateReactContextRunnable);
               } catch (Exception e) {
                 mDevSupportManager.handleException(e);
               }
             }
           });
   mCreateReactContextThread.start();
 }
  /**
   * @return instance of {@link ReactContext} configured a {@link CatalystInstance} set
   */
  private ReactApplicationContext createReactContext(
      JavaScriptExecutor jsExecutor,
      JSBundleLoader jsBundleLoader) {
    final ReactApplicationContext reactContext = new ReactApplicationContext(mApplicationContext);
// 是否处于开发者模式，提供一些异常处理的机制，捕获崩溃显示在红色弹窗
    if (mUseDeveloperSupport) {
      reactContext.setNativeModuleCallExceptionHandler(mDevSupportManager);
    }
// Java层模块注册表，通过它把NativeModules注册到CatalystInstance.包括我们自定义的继承自NativeModule的Java代码
    NativeModuleRegistry nativeModuleRegistry = processPackages(reactContext, mPackages, false);
// 异常处理Handler
      NativeModuleCallExceptionHandler exceptionHandler = mNativeModuleCallExceptionHandler != null
      ? mNativeModuleCallExceptionHandler
      : mDevSupportManager;
// 开始构建CatalystInstance 
    CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
      .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
      .setJSExecutor(jsExecutor)
      .setRegistry(nativeModuleRegistry)
      .setJSBundleLoader(jsBundleLoader)
      .setNativeModuleCallExceptionHandler(exceptionHandler);
    ........
    catalystInstance.runJSBundle();
    reactContext.initializeWithInstance(catalystInstance);

    return reactContext;
  }
private CatalystInstanceImpl(
      final ReactQueueConfigurationSpec reactQueueConfigurationSpec,
      final JavaScriptExecutor jsExecutor,
      final NativeModuleRegistry nativeModuleRegistry,
      final JSBundleLoader jsBundleLoader,
      NativeModuleCallExceptionHandler nativeModuleCallExceptionHandler) {
    Log.d(ReactConstants.TAG, "Initializing React Xplat Bridge.");
    mHybridData = initHybrid();

    mReactQueueConfiguration = ReactQueueConfigurationImpl.create(
        reactQueueConfigurationSpec,
        new NativeExceptionHandler());
    mBridgeIdleListeners = new CopyOnWriteArrayList<>();
    mNativeModuleRegistry = nativeModuleRegistry;
    mJSModuleRegistry = new JavaScriptModuleRegistry();
    mJSBundleLoader = jsBundleLoader;
    mNativeModuleCallExceptionHandler = nativeModuleCallExceptionHandler;
    mNativeModulesQueueThread = mReactQueueConfiguration.getNativeModulesQueueThread();
    mTraceListener = new JSProfilerTraceListener(this);

    Log.d(ReactConstants.TAG, "Initializing React Xplat Bridge before initializeBridge");
    initializeBridge(
      new BridgeCallback(this),
      jsExecutor,
      mReactQueueConfiguration.getJSQueueThread(),
      mNativeModulesQueueThread,
      mNativeModuleRegistry.getJavaModules(this),
      mNativeModuleRegistry.getCxxModules());
    Log.d(ReactConstants.TAG, "Initializing React Xplat Bridge after initializeBridge");

    mJavaScriptContextHolder = new JavaScriptContextHolder(getJavaScriptContext());
  }
  
    private native void initializeBridge(
      ReactCallback callback,
      JavaScriptExecutor jsExecutor,
      MessageQueueThread jsQueue,
      MessageQueueThread moduleQueue,
      Collection<JavaModuleWrapper> javaModules,
      Collection<ModuleHolder> cxxModules);
```

- CatalystInstance

到此 CatalystInstance 的实例 CatalystInstanceImpl 对象也就初始化 OK 了，同时通过 initializeBridge 建立了 Bridge 连接。关于这个 Bridge 在RN 中是通过 libjsc.so 中 JSObjectRef.h 的 JSObjectSetProperty(m_context, globalObject, jsPropertyName, valueToInject, 0, NULL); 来关联的，这样就可以在 Native 设置 JS 执行，反之同理。

这一小节我们只讨论 RN 的加载启动流程，所以 initializeBridge 的具体实现我们下面分析互相通信交互时再仔细分析，故我们先把思路还是回到 XReactInstanceManagerImpl 中 createReactContext 方法的 reactContext.initializeWithInstance(catalystInstance); 一行，可以看见，这行代码意思就是将刚刚初始化的 catalystInstance 传递给全局唯一的 reactContext 对象，同时在 reactContext 中通过 catalystInstance 拿到 JS、Native、UI 几个 Thread 的引用，方便快速访问使用这些对象。接着调用了 catalystInstance.runJSBundle(); 方法，这个方法实现如下：

通过注释我们假设 Loader 是默认的，也即 JSBundleLoader 类的如下方法：

可以看见，它实质又调用了 CatalystInstanceImpl 的 loadScriptFromAssets 方法，我们继续跟踪 CatalystInstanceImpl 的这个方法吧，如下：

说白了就是 CatalystInstanceImpl.java 中 CatalystInstanceImpl 构造方法中调用 C++ 的 initializeBridge 方法时传入的第一个参数 BridgeCallback 么，就是说 JS bundle 文件被加载完成以后 JS 端调用 Java 端时会触发 Callback 的 onBatchComplete 方法，这货最终又会触发 OnBatchCompleteListener 接口的 onBatchComplete 方法，这不就把 JS Bundle 文件加载完成以后回调 Java 通知 OK 了么。最后我们还差一个runCreateContextOnNewThread中的setupReactContext任务还没分析，方法如下：

这里的核心在rootView.invokeJsEntryPoint(),追踪一下源码，发现最后调用了如下方法

我们知道 AppRegistry.class 是 JS 端暴露给 Java 端的接口方法，所以catalystInstance.getJSModule(AppRegistry.class) 实质就桥接到 JS 端代码去了，那就去看看AppRegistry中写了写啥

到这里ReactNative的启动终于到头了，但react-native还有很多细节，这里先介绍React'Native的启动流程，搭建去Bridge，至于通信方式在后面继续介绍。

```
void CatalystInstanceImpl::initializeBridge(
    jni::alias_ref<ReactCallback::javaobject> callback,
    // This executor is actually a factory holder.
    JavaScriptExecutorHolder* jseh,
    jni::alias_ref<JavaMessageQueueThread::javaobject> jsQueue,
    jni::alias_ref<JavaMessageQueueThread::javaobject> moduleQueue,
    ModuleRegistryHolder* mrh) {
    ......
  // Java CatalystInstanceImpl -> C++ CatalystInstanceImpl -> Bridge -> Bridge::Callback
  // --weak--> ReactCallback -> Java CatalystInstanceImpl
    ......
    //instance_为ReactCommon目录下 Instance.h 中类的实例；
    //第一个参数为JInstanceCallback实现类，父类在cxxreact/Instance.h中。
    //第二个参数为JavaScriptExecutorHolder，实质对应java中JavaScriptExecutor，也就是上面分析java的initializeBridge方法第二个参数JSCJavaScriptExecutor。
    //第三第四个参数都是java线程透传到C++，纯C++的JMessageQueueThread。
    //第五个参数为C++的ModuleRegistryHolder的getModuleRegistry()方法。
  instance_->initializeBridge(folly::make_unique<JInstanceCallback>(callback),
                              jseh->getExecutorFactory(),
                              folly::make_unique<JMessageQueueThread>(jsQueue),
                              folly::make_unique<JMessageQueueThread>(moduleQueue),
                              mrh->getModuleRegistry());
}
@Override
  public void runJSBundle() {
    ......
    mJSBundleHasLoaded = true;
    //mJSBundleLoader就是前面分析的依据不同设置决定是JSBundleLoader的createAssetLoader还是createFileLoader等静态方法的匿名实现类。
    // incrementPendingJSCalls();
    mJSBundleLoader.loadScript(CatalystInstanceImpl.this);
    ......
  }
public static JSBundleLoader createAssetLoader(
      final Context context,
      final String assetUrl) {
    return new JSBundleLoader() {
      @Override
      public void loadScript(CatalystInstanceImpl instance) {
        instance.loadScriptFromAssets(context.getAssets(), assetUrl);
      }

​```
  @Override
  public String getSourceUrl() {
    return assetUrl;
  }
};
  }
native void loadScriptFromAssets(AssetManager assetManager, String assetURL);
1
loadScriptFromAssets 既然是一个 native 方法，我们去 CatalystInstanceImpl.cpp 看下这个方法的实现，如下：

void CatalystInstanceImpl::loadScriptFromAssets(jobject assetManager,
                                                const std::string& assetURL) {
  const int kAssetsLength = 9;  // strlen("assets://");
  //获取source路径名，不计前缀，这里默认就是index.android.bundle
  auto sourceURL = assetURL.substr(kAssetsLength);
    //assetManager是Java传递的AssetManager。
    //extractAssetManager是JSLoader.cpp中通过系统动态链接库android/asset_manager_jni.h的AAssetManager_fromJava方法来获取AAssetManager对象的。
  auto manager = react::extractAssetManager(assetManager);
    //通过JSLoader对象的loadScriptFromAssets方法读文件，得到大字符串script（即index.android.bundle文件的JS内容）。
  auto script = react::loadScriptFromAssets(manager, sourceURL);
    //判断是不是Unbundle，这里不是Unbundle，因为打包命令我们用了react.gradle的默认bundle，没用unbundle命令（感兴趣的自己分析这条路线）。
  if (JniJSModulesUnbundle::isUnbundle(manager, sourceURL)) {
    instance_->loadUnbundle(
      folly::make_unique<JniJSModulesUnbundle>(manager, sourceURL),
      std::move(script),
      sourceURL);
    return;
  } else {
    //bundle命令打包的，所以走这里。
    //instance_为ReactCommon目录下 Instance.h 中类的实例，前面分析过了。
    instance_->loadScriptFromString(std::move(script), sourceURL);
  }
}
private void setupReactContext(final ReactApplicationContext reactContext) {

    //Initialize all the native modules
    catalystInstance.initialize();
    //重置devSupportManager中的reactContext
    mDevSupportManager.onNewReactContextCreated(reactContext);
    mMemoryPressureRouter.addMemoryPressureListener(catalystInstance);
    //置位生命周期
    moveReactContextToCurrentLifecycleState();
// 核心方法
    synchronized (mAttachedRootViews) {
      for (ReactRootView rootView : mAttachedRootViews) {
        attachRootViewToInstance(rootView, catalystInstance);
      }
    }
// 通知UI更新
    UiThreadUtil.runOnUiThread(
        new Runnable() {
          @Override
          public void run() {
            for (ReactInstanceEventListener listener : finalListeners) {
              listener.onReactContextInitialized(reactContext);
            }
          }
        });
    // 开启JS消息队列线程
    reactContext.runOnJSQueueThread(
        new Runnable() {
          @Override
          public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          }
        });
    //开启Native消息队列线程
    reactContext.runOnNativeModulesQueueThread(
        new Runnable() {
          @Override
          public void run() {
            Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
          }
        });
  }

  private void attachRootViewToInstance(
      final ReactRootView rootView,
      CatalystInstance catalystInstance) {
       //通过UIManagerModule设置根布局为ReactRootView
    UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class);
        //设置相关tag
    final int rootTag = uiManagerModule.addRootView(rootView);
    rootView.setRootViewTag(rootTag);
      //Calls into JS to start the React application. 
    rootView.invokeJSEntryPoint();
    ......
    UiThreadUtil.runOnUiThread(new Runnable() {
      @Override
      public void run() {
        //
        rootView.onAttachedToReactInstance();
      }
    });
  }
/**
   * Calls the default entry point into JS which is AppRegistry.runApplication()
   */
  private void defaultJSEntryPoint() {
      try {
        if (mReactInstanceManager == null || !mIsAttachedToInstance) {
          return;
        }
        ReactContext reactContext = mReactInstanceManager.getCurrentReactContext();
        if (reactContext == null) {
          return;
        }
        CatalystInstance catalystInstance = reactContext.getCatalystInstance();
          // 包装相关参数，传给js，rootTag用来告诉js端对应的是native 端哪一个view
        WritableNativeMap appParams = new WritableNativeMap();
        appParams.putDouble("rootTag", getRootViewTag());
        @Nullable Bundle appProperties = getAppProperties();
        if (appProperties != null) {
          appParams.putMap("initialProps", Arguments.fromBundle(appProperties));
        }
        ....
        //核心！！！ReactNative真正的启动流程就是在这里被调用起来的
        String jsAppModuleName = getJSModuleName();
        catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
      } finally {
        Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      }
  }

//`AppRegistry` is the JS entry point to running all React Native apps.  App
// root components should register themselves with
var AppRegistry = {
    ......
    //我们JS端自己在index.android.js文件中调用的入口就是：
    //AppRegistry.registerComponent('TestRN', () => TestRN);
  registerComponent: function(appKey: string, getComponentFunc: ComponentProvider): string {
    runnables[appKey] = {
      run: (appParameters) =>
        renderApplication(getComponentFunc(), appParameters.initialProps, appParameters.rootTag)
    };
    return appKey;
  },
    ......
    //上面java端 AppRegistry 调用的 JS 端就是这个方法，索引到我们设置的appkey=TestRN字符串的JS入口
  runApplication: function(appKey: string, appParameters: any): void {
    ......
    runnables[appKey].run(appParameters);
  },
    ......
};

```

## 小结

到这里ReactNative的启动流程就差不多了，我们小梳理总结，如下图：

![img](https://i.loli.net/2018/11/18/5bf0e39e53802.jpg)上面这幅图已经囊括了我们上面说到的启动流程的全部流程分析