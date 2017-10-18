# 引子
这篇文章会告诉你

- **什么是路由**，是为了解决什么问题才产生的
- **业界现状是怎么样的**，我们可以做什么来优化当前的问题
- **路由设计思路**是怎么样的，该怎么设计比较好

具体实现见《51信用卡移动端APP路由设计最佳实践》。

# 前言
> 当前Android的路由库实在太多了，51信用卡管家作为一个移动金融APP，稳定、高效还是很重要的，所以分析了一下当前各个Android路由库的优缺点

# 背景

### 什么是路由

根据`路由表`将`页面请求`分发到指定页面，`路由表`指的是URL与原生页面的对应关系

### 使用场景
1. 点击通知打开App的某个页面
2. 点击链接打开App的某个页面
3. 如果app没打开，会打开app主页面再打开页面
4. 打开页面需要先验证完条件再去打开页面，譬如登录校验
5. 原生页面出bug，动态把原生的页面替换成H5页面
6. 屏蔽掉不合法打开App页面的请求

### 原生和路由库的比较
在Android中原生已经支持`AndroidManifest`去管理App跳转，为什么要有路由库，这可能是大部分人接触到Android各种路由库不太明白的地方，这里讲一下我的理解
- 显式Intent：项目庞大以后，类依赖 **耦合太大**，不适合组件化拆分
- 隐式Intent：协作困难，调用时候 **不知道调什么参数**
- 每个注册了Scheme的Activity都可以直接打开，有 **安全风险**
- AndroidMainfest集中式管理比较 **臃肿**
-  **无法动态修改路由**，如果页面出错，无法动态降级
-  **无法动态拦截跳转**，譬如未登录的情况下，打开登录页面，登录成功后接着打开刚才想打开的页面
- H5、Android、iOS地址不一样， **不利于统一跳转**

在iOS中可以通过[运行时实现](https://casatwy.com/iOS-Modulization.html)，但是我们需要考虑到两端甚至三端的兼容，所以iOS和Android一样都是通过URL跳转。


### 怎么样的路由才算好路由
路由说到底还是为了解决开发者遇到的各种奇葩需求，先找出实际要解决的问题还是比较重要的

- 生成 **路由表**
- 获取 **路由参数**
- 拦截 **路由**
-  **异步调用**
-  **路由回调**
-  **安全拦截**

# 详细比较

Android中路由库都用`Apt`(在javacompile任务之前生成java文件)生成路由表，然后用路由表转发到指定页面，`OkDeepLink`是51信用卡管家综合各个路由库的特点产生。

方案对比 |51信用卡 OkDeepLink | Airbnb [DeepLinkDispatch](https://github.com/airbnb/DeepLinkDispatch) | 阿里 [ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go) | 天猫 [统跳协议](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go) |[ActivityRouter](https://github.com/mzule/ActivityRouter)
---|---|---|---|---|---
路由注册 | **注解式接口注册**|每个module都要手动注册|每个module的路由表都要类查找|AndroidManiFest配置|每个module都要手动注册
路由查找 | 路由表|路由表|路由表|系统Intent|路由表
路由分发 | Activity转发|Activity转发|Activity转发|Activity转发|Activity转发
动态替换 | **Rxjava实现异步拦截器**|不支持|线程等待|不支持|不支持
动态拦截 | **Rxjava实现异步拦截器**|不支持|线程等待|不支持|主线程
安全拦截 | **Rxjava实现异步拦截器**|不支持|线程等待|不支持|主线程
方法调用 |**接口**| 手动拼装|手动拼装|手动拼装|手动拼装
参数获取 |**Apt依赖注入，支持所有类型，不需要在Activity的`onCreate`中手动调用get方法**|参数定义在path，不利于多人协作|Apt依赖注入，但是要手动调用get方法|手动调用|手动调用
路由回调 |**Rxjava回调**|onActivityResult|onActivityResult|onActivityResult|onActivityResult
Module接入不同App|**支持**|不支持|支持|不支持|支持

iOS中也有较多的路由方案，但都有各种优缺点:

1. [JLRoutes](https://github.com/joeldev/JLRoutes)
2. [routable-ios](https://github.com/clayallsopp/routable-ios)
3. [HHRouter](https://github.com/lightory/HHRouter)
4. [MGJRouter](https://github.com/mogujie/MGJRouter)

方案1、方案2都是基于遍历来进行路由查找，效率不够高<br/>
方案3、方案4为了提高查找效率都使用匹配的方式，方案4是在方案3的基础上做了一些改进，但在参数的查找上都做了比较复杂的处理逻辑
[这篇文章](https://halfrost.com/ios_router/)里面做了详细的比较


其实说到底，路由的本质就是注册再转发，围绕着转发可以进行各种操作，拦截，替换，参数获取等等，其他Apt、Rxjava说到底都只是为了方便使用出现的，这里你会发现各种路由库为了修复各种工具的弊端反而带来一些问题，譬如[DeepLinkDispatch](https://github.com/airbnb/DeepLinkDispatch)为了解决Apt没法汇总所有Module路由，每个module都要手动注册，[ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go)为了解决Apt没法汇总所有Module路由，通过类查找的方式实现，才出现分组的概念。


### 定义路由

![路由定义](http://upload-images.jianshu.io/upload_images/53953-054d5e9096445d84.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

对应路由的定义，业界有两种做法
1. 参数放在path里面
2. 参数放在query里面

参数定义在path里面的做法，有不需要额外传参数的好处，但是没有那么灵活，调试起来也没有那么方便。


### 注册路由

`AndroidManifest`里面的`acitivity`声明scheme码是不安全的，所有App都可以打开这个页面，这里就产生了三种方式去注册
1. 注解产生路由表，通过`DispatchActivity`转发到指定页面
2. `AndroidManifest`注册，将其`export=fasle`，但是再通过DispatchActivity转发Intent，天猫就是这么做的，比上面的方法的好处是路由查找都是系统调用，省掉了维护路由表的过程，但是AndroidManifest配置还是比较不方便的
3. 注解自动修改AndroidManifest，这种方式可以避免路由表汇总的问题，方案是这样的，用自定义`Lint`扫描出注解相关的Activity，然后在`processManifestTask后面`修改Manifest，51信用卡里面原来想用这个方案，但是现在接入风险比较大，没采用这种做法

#### 生成路由表
思路都是用Apt生成URL和activity的对应关系

**Airbnb**
``` java
@DeepLink("foo://example.com/deepLink/{id}")
public class MainActivity extends Activity {
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
  }
}
```
生成
``` java
public final class SampleModuleLoader implements Parser {
  public static final List<DeepLinkEntry> REGISTRY = Collections.unmodifiableList(Arrays.asList(
    new DeepLinkEntry("foo://example.com/deepLink/{id}", DeepLinkEntry.Type.METHOD, MainActivity.class, null)
    ));

  @Override
  public DeepLinkEntry parseUri(String uri) {
    for (DeepLinkEntry entry : REGISTRY) {
      if (entry.matches(uri)) {
        return entry;
      }
    }
    return null;
  }
}
```

**阿里Arouter**
``` java
@Route(path = "/deepLink")
public class MainActivity extends Activity {
 @Autowired
    String id;
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
  }
}
```
生成
```java

public class ARouter$$Group$$m2 implements IRouteGroup {
  @Override
  public void loadInto(Map<String, RouteMeta> atlas) {
    atlas.put("/deepLink", RouteMeta.build(RouteType.ACTIVITY, MainActivity.class, "/deepLink", null, null, -1, -2147483648));
  }
}
```


**Activity Router**

```java
@Router("deeplink")
public class ModuleActivity extends Activity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
    }
}
```
生成
```java
public final class RouterMapping_sdk {
  public static final void map() {
    java.util.Map<String,String> transfer = null;
    com.github.mzule.activityrouter.router.ExtraTypes extraTypes;

    transfer = null;
    extraTypes = new com.github.mzule.activityrouter.router.ExtraTypes();
    extraTypes.setTransfer(transfer);
    com.github.mzule.activityrouter.router.Routers.map("deeplink", ModuleActivity.class, null, extraTypes);

  }
}
```

#### 汇总路由表

这里就要提一下使用Apt会造成每个module都要手动注册，因为APT是在javacompile任务前插入了一个task，所以只对自己的moudle处理注解

[DeepLinkDispatch](https://github.com/airbnb/DeepLinkDispatch)是这么做的
```java
@DeepLinkModule
public class SampleModule {
}
```
```
@DeepLinkHandler({ SampleModule.class, LibraryDeepLinkModule.class })
public class DeepLinkActivity extends Activity {
  @Override protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    DeepLinkDelegate deepLinkDelegate = new DeepLinkDelegate(
        new SampleModuleLoader(), new LibraryDeepLinkModuleLoader());
    deepLinkDelegate.dispatchFrom(this);
    finish();
  }
}
```
[ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go)是通过类查找,就比较耗时了，所以他又加入了分组的概念，按需加载
```java
/**
     * 通过指定包名，扫描包下面包含的所有的ClassName
     *
     * @param context     U know
     * @param packageName 包名
     * @return 所有class的集合
     */
    public static List<String> getFileNameByPackageName(Context context, String packageName) throws PackageManager.NameNotFoundException, IOException {
        List<String> classNames = new ArrayList<>();
        for (String path : getSourcePaths(context)) {
            DexFile dexfile = null;

            try {
                if (path.endsWith(EXTRACTED_SUFFIX)) {
                    //NOT use new DexFile(path), because it will throw "permission error in /data/dalvik-cache"
                    dexfile = DexFile.loadDex(path, path + ".tmp", 0);
                } else {
                    dexfile = new DexFile(path);
                }
                Enumeration<String> dexEntries = dexfile.entries();
                while (dexEntries.hasMoreElements()) {
                    String className = dexEntries.nextElement();
                    if (className.contains(packageName)) {
                        classNames.add(className);
                    }
                }
            } catch (Throwable ignore) {
                Log.e("ARouter", "Scan map file in dex files made error.", ignore);
            } finally {
                if (null != dexfile) {
                    try {
                        dexfile.close();
                    } catch (Throwable ignore) {
                    }
                }
            }
        }

        Log.d("ARouter", "Filter " + classNames.size() + " classes by packageName <" + packageName + ">");
        return classNames;
    }

```

[ActivityRouter](https://github.com/mzule/ActivityRouter)就比较巧妙了，通过Stub项目，其他地方都是provide的，只有主工程里面用Apt生成RouterInit类,虽然还是要写`module`的注解
```java
        // RouterInit
        if (hasModules) {
            debug("generate modules RouterInit");
            generateModulesRouterInit(moduleNames);
        } else if (!hasModule) {
            debug("generate default RouterInit");
            generateDefaultRouterInit();
        }
```

[美柚路由](https://github.com/gybin02/RouterKit)是通过生成每个module的路由表，然后复制到app的assets目录，运行的时候遍历asset目录，反射对应的activity
```groovy
//拷贝生成的 assets/目录到打包目录
android.applicationVariants.all { variant ->
    def variantName = variant.name
    def variantNameCapitalized = variantName.capitalize()
    def copyMetaInf = tasks.create "copyMetaInf$variantNameCapitalized", Copy
    copyMetaInf.from project.fileTree(javaCompile.destinationDir)
    copyMetaInf.include "assets/**"
    copyMetaInf.into "build/intermediates/sourceFolderJavaResources/$variantName"
    tasks.findByName("transformResourcesWithMergeJavaResFor$variantNameCapitalized").dependsOn copyMetaInf
}
```
[Metis](https://github.com/yangxlei/metis)是一个android中解决服务发现的库，他是这么解决的，在app主工程中transfomer的时候去扫描所有modlue和jar带注解的文件去生成路由表，然后把这个java文件编译，但是这种方式需要扫描整个app会慢一点，而且手动去编译java感觉不太稳定

```groovy
 def destDir
        List<String> classpaths = new ArrayList<>()
        transformInvocation.inputs.each { input ->

            input.jarInputs.each { jarInput ->

                def jarName = jarInput.name
                if (jarName.endsWith(".jar")) {
                    jarName = jarName.substring(0, jarName.length() - 4)
                }

                def dest = transformInvocation.outputProvider.getContentLocation(jarName, jarInput.contentTypes, jarInput.scopes, Format.JAR)

                classpaths.add(dest)
                mAction.loadJar(new JarFile(jarInput.file), jarInput.status)
                FileUtils.copyFile(jarInput.file, dest)

                mProject.logger.info("scan file:\t ${jarInput.file} status:${jarInput.status}")
            }

            input.directoryInputs.each { dirInput ->

                // 测试发现: 如果目录下的文件没有任何改变，不会进入到这个 transform
                Map<File, Status> changedFiles = dirInput.changedFiles
                if (changedFiles == null || changedFiles.isEmpty()) {
                    // clean 后进入， changed 为空
                    mAction.loadDirectory(dirInput.file)
                    mProject.logger.info("scan dir:\t ${dirInput.file}")
                } else {
                    mAction.loadChangedFiles(changedFiles)
                }

                destDir = transformInvocation.outputProvider.getContentLocation(dirInput.name, dirInput.contentTypes, dirInput.scopes, Format.DIRECTORY)
                classpaths.add(destDir)
                FileUtils.copyDirectory(dirInput.file, destDir)
            }
        }
```

天猫 [统跳协议](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go) 是最简单的，转发一下Intent就可以，但是这样就没法享受注解的好处了。

而51信用卡的OkDeepLink用`aspectj`解决了这个问题，会自动汇总所有module的路由省略了这些多余的代码。

```java
@After("execution(* okdeeplink.DeepLinkClient.init(..))")
  public void init() {
    DeepLinkClient.addAddress(new Address("/main", MainActivity.class));
  }
```
okdeeplink.DeepLinkClient.init会调用所有路由表的init方法。
### 参数获取
大部分路由库都是手动获取参数的，这样还要传入参数key比较麻烦，有三种做法
1. Hook掉`Instrumentation`的`newActivity`方法，注入参数
2. 注册`ActivityLifecycleCallbacks`方法，注入参数
3. `Apt`生成注入代码，`onCreate`的时候bind一下


Hook掉`Instrumentation`的`newActivity`方法是这么实现的

```java
@Deprecated
public class InstrumentationHook extends Instrumentation {
    /**
     * Hook the instrumentation's newActivity, inject
     * <p>
     * Perform instantiation of the process's {@link Activity} object.  The
     * default implementation provides the normal system behavior.
     *
     * @param cl        The ClassLoader with which to instantiate the object.
     * @param className The name of the class implementing the Activity
     *                  object.
     * @param intent    The Intent object that specified the activity class being
     *                  instantiated.
     * @return The newly instantiated Activity object.
     */
    public Activity newActivity(ClassLoader cl, String className,
                                Intent intent)
            throws InstantiationException, IllegalAccessException,
            ClassNotFoundException {

//        return (Activity)cl.loadClass(className).newInstance();

        Class<?> targetActivity = cl.loadClass(className);
        Object instanceOfTarget = targetActivity.newInstance();

        if (ARouter.canAutoInject()) {
            String[] autoInjectParams = intent.getStringArrayExtra(ARouter.AUTO_INJECT);
            if (null != autoInjectParams && autoInjectParams.length > 0) {
                for (String paramsName : autoInjectParams) {
                    Object value = intent.getExtras().get(TextUtils.getLeft(paramsName));
                    if (null != value) {
                        try {
                            Field injectField = targetActivity.getDeclaredField(TextUtils.getLeft(paramsName));
                            injectField.setAccessible(true);
                            injectField.set(instanceOfTarget, value);
                        } catch (Exception e) {
                            ARouter.logger.error(Consts.TAG, "Inject values for activity error! [" + e.getMessage() + "]");
                        }
                    }
                }
            }
        }

        return (Activity) instanceOfTarget;
    }
}
```

业界的统一做法都是用apt，其他方式不稳定，[ARouter](https://github.com/androidannotations/androidannotations)、[androidannotations](https://github.com/androidannotations/androidannotations)、[Jet](https://github.com/gybin02/Jet), 思路都是一样的，这里拿ARouter的代码说明一下是怎么实现的

用`Autowired`生成Test1Activity$$ARouter$$Autowired类，用inject方法找到`AutowiredServiceImpl`方法，`AutowiredServiceImpl`调用到`Test1Activity$$ARouter$$Autowired`

```java
@Route(path = "/test/activity1")
public class Test1Activity extends AppCompatActivity {

    @Autowired
    String name;
     @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_test1);

        ARouter.getInstance().inject(this);
        }
    
    }

```

```java
@Route(path = "/arouter/service/autowired")
public class AutowiredServiceImpl implements AutowiredService {
    private LruCache<String, ISyringe> classCache;
    private List<String> blackList;

    @Override
    public void init(Context context) {
        classCache = new LruCache<>(66);
        blackList = new ArrayList<>();
    }

    @Override
    public void autowire(Object instance) {
        String className = instance.getClass().getName();
        try {
            if (!blackList.contains(className)) {
                ISyringe autowiredHelper = classCache.get(className);
                if (null == autowiredHelper) {  // No cache.
                    autowiredHelper = (ISyringe) Class.forName(instance.getClass().getName() + SUFFIX_AUTOWIRED).getConstructor().newInstance();
                }
                autowiredHelper.inject(instance);
                classCache.put(className, autowiredHelper);
            }
        } catch (Exception ex) {
            blackList.add(className);    // This instance need not autowired.
        }
    }
}
```

```java
public class Test1Activity$$ARouter$$Autowired implements ISyringe {

  @Override
  public void inject(Object target) {
    Test1Activity substitute = (Test1Activity)target;
    substitute.name = substitute.getIntent().getStringExtra("name");
  }
}
```

### 路由分发

现在所有路由方案分发都是用`Activity`做分发的，这样做会有这几个缺点
1. 每次都要启动一个Activity，而Activity就算不写任何代码启动都要0.1秒
2. 如果是异步等待的话，Activiy要在合适时间`finish`，不然会有一层透明的页面阻挡操作

对于第一个问题，有两个方法
1. QQ音乐是把`DispatchActivity`设为`SingleInstacne`,但是这样的话，动画会奇怪，堆栈也会乱掉，后退会有一层透明的页面阻挡操作
2. `DispatchActivity`只在外部打开的时候调用

我选择了第二种

对于第二个问题，有两个方法
1. `DispatchActivity`再把Intent转发到`Service`,再finish，这种方法唯一的缺陷是拦截器里面的context是Servcie的activity，就没发再拦截器里面弹出对话框了。
2. `DispatchActivity`在打开和错误的时候`finish`,如果`activity`已经finish了，就用application的context去转发路由

我选择了第二种

```java
  public void dispatchFrom(Intent intent) {
        new DeepLinkClient(this)
                .buildRequest(intent)
                .dispatch()
                .subscribe(new Subscriber<Request>() {
                    @Override
                    public void onCompleted() {
                        finish();
                    }

                    @Override
                    public void onError(Throwable e) {
                        finish();
                    }

                    @Override
                    public void onNext(Request request) {
                        Intent dispatchIntent = request.getIntent();
                        startActivity(dispatchIntent);
                    }
                });
    }
```
其实处理透明Activity阻挡操作可以采用取消所有事件变成无感页面的方法
我找到一种方式解决这个问题[解决透明Activity点击不影响用户操作](http://www.jianshu.com/p/09d18717c433)



### 动态拦截

拦截器是重中之重，有了拦截器可以做好多事情，可以说之所以要做页面路由，就是为了要实现拦截器。[ARouter](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go)是异步的,只有一个拦截器做完了，下一个拦截器才会调用到

```java
@Interceptor(priority = 7)
public class Test1Interceptor implements IInterceptor {
    Context mContext;

    /**
     * The operation of this interceptor.
     *
     * @param postcard meta
     * @param callback cb
     */
    @Override
    public void process(final Postcard postcard, final InterceptorCallback callback) {
        if ("/test/activity4".equals(postcard.getPath())) {
            final AlertDialog.Builder ab = new AlertDialog.Builder(MainActivity.getThis());
            ab.setCancelable(false);
            ab.setTitle("温馨提醒");
            ab.setMessage("想要跳转到Test4Activity么？(触发了\"/inter/test1\"拦截器，拦截了本次跳转)");
            ab.setNegativeButton("继续", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onContinue(postcard);
                }
            });
            ab.setNeutralButton("算了", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    callback.onInterrupt(null);
                }
            });
            ab.setPositiveButton("加点料", new DialogInterface.OnClickListener() {
                @Override
                public void onClick(DialogInterface dialog, int which) {
                    postcard.withString("extra", "我是在拦截器中附加的参数");
                    callback.onContinue(postcard);
                }
            });

            MainLooper.runOnUiThread(new Runnable() {
                @Override
                public void run() {
                    ab.create().show();
                }
            });
        } else {
            callback.onContinue(postcard);
        }
    }

    /**
     * Do your init work in this method, it well be call when processor has been load.
     *
     * @param context ctx
     */
    @Override
    public void init(Context context) {
        mContext = context;
        Log.e("testService", Test1Interceptor.class.getName() + " has init.");
    }
}
```
```java
LogisticsCenter.executor.execute(new Runnable() {
                @Override
                public void run() {
                    CancelableCountDownLatch interceptorCounter = new CancelableCountDownLatch(Warehouse.interceptors.size());
                    try {
                        _excute(0, interceptorCounter, postcard);
                        interceptorCounter.await(postcard.getTimeout(), TimeUnit.SECONDS);
                        if (interceptorCounter.getCount() > 0) {    // Cancel the navigation this time, if it hasn't return anythings.
                            callback.onInterrupt(new HandlerException("The interceptor processing timed out."));
                        } else if (null != postcard.getTag()) {    // Maybe some exception in the tag.
                            callback.onInterrupt(new HandlerException(postcard.getTag().toString()));
                        } else {
                            callback.onContinue(postcard);
                        }
                    } catch (Exception e) {
                        callback.onInterrupt(e);
                    }
                }
            });
```      
也有现成的库支持实现异步拦截器，譬如[Rxjava](https://github.com/ReactiveX/RxJava)、[android-promise](https://github.com/hnakagawa/android-promise)

51信用卡用Rxjava实现了仿造Okhttp的异步拦截器，，可以见《51信用卡移动端APP路由设计最佳实践》
### 方法调用
大部分路由库都是手动拼参数调用路由的，其中有两个方案比较特殊
1. [LiteRouter](http://www.jianshu.com/p/79e9a54e85b2)模仿了`Retrofit`接口式调用
2. [joyrun的ActivityRouter](https://github.com/joyrun/ActivityRouter)通过生成ActivityHelper类去调用


```java
public interface IntentService { 
  @ClassName("com.hiphonezhu.test.demo.ActivityDemo2")     
  @RequestCode(100) 
  IntentWrapper intent2ActivityDemo2Raw(@Key("platform") String platform, @Key("year") int year);
}
```
```java
public <T> T create(final Class<T> service, final Context context)
    {
        return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
                new InvocationHandler() {
                    @Override public Object invoke(Object proxy, Method method, Object... args)
                            throws Throwable {
                        IntentWrapper intentWrapper = loadIntentWrapper(context, method, args);

                        Class returnTYpe = method.getReturnType();
                        if (returnTYpe == void.class)
                        {
                            if (interceptor == null || !interceptor.intercept(intentWrapper))
                            {
                                intentWrapper.start();
                            }
                            return null;
                        }
                        else if (returnTYpe == IntentWrapper.class)
                        {
                            return intentWrapper;
                        }
                        throw new RuntimeException("method return type only support 'void' or 'IntentWrapper'");
                    }
                });
    }
```    
51信用卡用`apt`实现了`Retrofit`接口式调用，具体实现可以见《51信用卡移动端APP路由设计最佳实践》

# 参考文献
### 业界做法
- [airbnb开源的页面路由](https://github.com/airbnb/DeepLinkDispatch) 
- [阿里开源的页面路由](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go)
- [天猫的统跳协议](https://yq.aliyun.com/articles/71687?spm=5176.100240.searchblog.7.8os9Go)
- [蘑菇街的页面路由](http://pingguohe.net/2015/11/24/Navigator-and-Rewrite.html)
- [Google App Link](http://limboy.me/tech/2016/03/10/mgj-components.html)
- [移动DeepLink的前世今生](https://sanwen8.cn/p/1a3vKVl.html)
### 设计方案
- [UrlRouter路由框架的设计](http://zhengxiaoyong.me/2016/04/24/UrlRouter%E8%B7%AF%E7%94%B1%E6%A1%86%E6%9E%B6%E7%9A%84%E8%AE%BE%E8%AE%A1/)
- [移动端路由层设计](http://www.jianshu.com/p/be7da3ed4100)
- [客户端路由动态配置](http://www.sixwolf.net/blog/2016/12/02/%E7%83%AD%E6%9B%B4%E6%96%B0%E6%96%B9%E6%A1%88%E4%B9%8B%E8%B7%AF%E7%94%B1%E5%8A%A8%E6%80%81%E9%85%8D%E7%BD%AE/)
- [移动端基于动态路由的架构设计](http://www.jianshu.com/p/cc55ce2b3ff4)
- [Android组件化通信(多进程)](http://blog.spinytech.com/2016/12/28/android_modularization/)
- [iOS 组件化 —— 路由设计思路分析](https://halfrost.com/ios_router/?hmsr=toutiao.io&utm_medium=toutiao.io&utm_source=toutiao.io)
- [QQ音乐首页Activity的单例实现](http://chuansong.me/n/425965251535)
### 个人开发
- [LiteRouter](http://www.jianshu.com/p/79e9a54e85b2)            模仿retrofit，各个业务分根据需求约定好接口，就像一份接口文档一样
- [ActivityRouter](https://joyrun.github.io/2016/08/01/ActivityRouter/)   
- [ActivityRouter2](https://github.com/mzule/ActivityRouter)     
- [AndRouter](https://github.com/campusappcn/AndRouter)
- [Router](https://github.com/yjfnypeu/Router)
- [Router2](https://github.com/chenenyu/Router) 
- [router-android](https://github.com/eju-front/router-android)
### 安全讨论
- [如何在Activity中获取调用者](http://blog.csdn.net/u013553529/article/details/53856800)             讨论了android里面原生支持找到路由来源的可能性，分析了referrer是如何产生的
- [LauncherFrom](https://github.com/YoungShoo/LaunchedFrom)
提供了一种hook activitythread找到launchedFromPackage的方法，不过也只支持5.0以上
- [高效过滤Intents](http://www.jianshu.com/p/7069b330c328)
只有包含特定Package URL的 intent 才会唤起页面



