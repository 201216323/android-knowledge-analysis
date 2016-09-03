# Activity的启动过程分析

Activity作为Android的四大组件之一，也是最基本的组件，负责与用户交互的所有功能。Activity的启动过程也并非一件神秘的事情，接下来就简单的从源码的角度分析一下Activity的启动过程。

根Activity一般就是指我们项目中的MainActivity，代表了一个Android应用程序，一般也是在一个新的进程中启动起来。在Android系统中，所有的Activity组件都保存在堆栈中，我们启动一个新的Activity组件就位于上一个Activity的上面。那么我们从桌面（Launcher）打开一个App是一个怎样的过程呢，如下所示：

1. Launcher向ActivityManagerService发送一个启动MainActivity的请求；
2. ActivityManagerService首先将MainActivity的相关信息保存下来，然后向Launcher发送一个使之进入中止状态的请求；
3. Launcher收到中止状态之后，就会想ActivityManagerService发送一个已进入中止状态的请求，便于ActivityManagerService继续执行启动MainActivity的操作；
4. ActivityManagerService检查用于运行MainActivity的进程，如果不存在，则启动一个新的进程；
5. 新的应用程序进程启动完成之后，就会向ActivityManagerService发送一个启动完成的请求，便于ActivityManagerService继续执行启动MainActivity的操作；
6. ActivityManagerService将第2步保存下来的MainActivity相关信息发送给新创建的进程，便于该进程启动MainActivity组件。

## Launcher.startActivitySafely
```java
boolean startActivitySafely(Intent intent, Object tag) {    
    intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);    
    try {    
        startActivity(intent);    
        return true;    
    } catch (ActivityNotFoundException e) {
    }    
}
```
当我们在Launcher上点击应用程序图标时，startActivitySafely方法会被调用。需要启动的Activity信息保存在intent中，包括action、category等等。那么Launcher是如何获得intent里面的这些信息呢？首先，系统在启动时会启动一个叫做PackageManagerService的管理服务，并且通过他来安装系统中的应用程序，在这个过程中，PackageManagerService会对应用程序的配置文件AndroidManifest.xml进行解析，从而得到程序里的组件信息（包括Activity、Service、Broadcast等），然后PackageManagerService去查询所有action为“android.intent.action.MAIN”并且category为“android.intent.category.LAUNCHER”的Activity，然后为每个应用程序创建一个快捷方式图标，并把程序信息与之关联。上述代码中，Activity的启动标志位设置为“Intent.FLAG_ACTIVITY_NEW_TASK”，便于他可以在一个新的任务中启动。

## Activity.startActivity
```java
@Override  
public void startActivity(Intent intent, @Nullable Bundle options) {  
    if (options != null) {  
        startActivityForResult(intent, -1, options);  
    } else {  
        startActivityForResult(intent, -1);  
    }  
} 
```
调用startActivityForResult，第二个参数(requestCode)为-1则表示在Activity关闭时不需要将结果传回来。

## Activity.startActivityForResult
```java
public void startActivityForResult(Intent intent, int requestCode, @Nullable Bundle options) {  
    if (mParent == null) { //一般的Activity其mParent都为null  
        Instrumentation.ActivityResult ar = mInstrumentation.execStartActivity(this,   
                   mMainThread.getApplicationThread(), mToken, this,intent, requestCode, options);  
        if (ar != null) { //发送结果，即onActivityResult会被调用  
            mMainThread.sendActivityResult(mToken, mEmbeddedID, requestCode, ar.getResultCode(),  
                    ar.getResultData());  
        }  
        if (requestCode >= 0) {  
            mStartedActivity = true;  
        }  

        final View decor = mWindow != null ? mWindow.peekDecorView() : null;  
        if (decor != null) {  
            decor.cancelPendingInputEvents();  
        }  
    } else { //在ActivityGroup内部的Activity，内部处理逻辑和上面是类似的  
        if (options != null) {  
            mParent.startActivityFromChild(this, intent, requestCode, options);  
        } else {  
            mParent.startActivityFromChild(this, intent, requestCode);  
        }  
    }  
    if (options != null && !isTopOfTask()) {  
        mActivityTransitionState.startExitOutTransition(this, options);  
    }  
} 
```
不难发现，最后实际上是调用mInstrumentation.execStartActivity来启动Activity，mInstrumentation类型为Instrumentation，用于监控程序和系统之间的交互操作。mInstrumentation代为执行Activity的启动操作，便于他可以监控这一个交互过程。mMainThread的类型为ActivityThread，用于描述一个应用程序进程，系统每启动一个程序都会在它里面加载一个ActivityThread的实例，并且将该实例保存在Activity的成员变量mMainThread中，而mMainThread.getApplicationThread()则用于获取其内部一个类型为ApplicationThread的本地Binder对象。mToken的类型为IBinder，他是一个Binder的代理对象，只想了ActivityManagerService中一个类型为ActivityRecord的本地Binder对象。每一个已经启动的Activity在ActivityManagerService中都有一个对应的ActivityRecord对象，用于维护Activity的运行状态及信息。

## Instrumentation.execStartActivity
```java
public ActivityResult execStartActivity(Context who, IBinder contextThread, IBinder token, Activity target,  
                    Intent intent, int requestCode, Bundle options) {  
    IApplicationThread whoThread = (IApplicationThread) contextThread;  
    if (mActivityMonitors != null) {  
        synchronized (mSync) {  
            final int N = mActivityMonitors.size();  
            for (int i=0; i<N; i++) { //先查找一遍看是否存在这个activity  
                final ActivityMonitor am = mActivityMonitors.get(i);  
                if (am.match(who, null, intent)) {  
                    am.mHits++;  
                    if (am.isBlocking()) {  
                        return requestCode >= 0 ? am.getResult() : null;  
                    }  
                    break;  
                }  
            }  
        }  
    }  
    try {  
        intent.migrateExtraStreamToClipData();  
        intent.prepareToLeaveProcess();  
        int result = ActivityManagerNative.getDefault().startActivity(whoThread, who.getBasePackageName(), intent,  
                    intent.resolveTypeIfNeeded(who.getContentResolver()),token, target != null ? target.mEmbeddedID : null,  
                    requestCode, 0, null, options); //这里才是真正打开activity的地方，其核心功能在whoThread中完成。  
        checkStartActivityResult(result, intent); // 处理各种异常，如ActivityNotFound  
    } catch (RemoteException e) {  
    }  
    return null;  
}  
```
上述代码可知，通过ActivityManagerNative.getDefault()获取一个ActivityManagerService的代理对象，然后调用他的startActivity方法来通知ActivityManagerService去启动Activity。
中间还有一系列过程，跟着源码走下去，不难发现，最后，是调用ApplicationThread的scheduleLaunchActivity来进行Activity的启动。

## Application.scheduleLaunchActivity
```java
public final void scheduleLaunchActivity(Intent intent, IBinder token, int ident,  
                ActivityInfo info, Configuration curConfig, CompatibilityInfo compatInfo,  
                String referrer, IVoiceInteractor voiceInteractor, int procState, Bundle state,  
                PersistableBundle persistentState, List<ResultInfo> pendingResults,  
                List<ReferrerIntent> pendingNewIntents, boolean notResumed, boolean isForward,  
                ProfilerInfo profilerInfo) {  
  
    updateProcessState(procState, false);  

    ActivityClientRecord r = new ActivityClientRecord();  

    r.token = token;  
    r.ident = ident;  
    r.intent = intent;  
    r.referrer = referrer;  
    r.voiceInteractor = voiceInteractor;  
    r.activityInfo = info;  
    r.compatInfo = compatInfo;  
    r.state = state;  
    r.persistentState = persistentState;  

    r.pendingResults = pendingResults;  
    r.pendingIntents = pendingNewIntents;  

    r.startsNotResumed = notResumed;  
    r.isForward = isForward;  

    r.profilerInfo = profilerInfo;  

    updatePendingConfiguration(curConfig);  

    sendMessage(H.LAUNCH_ACTIVITY, r);  
}  
```
上述代码主要做的事就是构造一个ActivityClientRecord，然后调用sendMessage发送一个消息。在应用程序对应的进程中，每一个Activity组件都使用一个ActivityClientRecord对象来描述，他们保存在ActivityThread类的成员变量mActivities中。那么Handler是如何处理这个消息的呢？

## H.handleMessage
```java
switch (msg.what) { // 消息类型  
    case LAUNCH_ACTIVITY: {  
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");  
        final ActivityClientRecord r = (ActivityClientRecord) msg.obj;  

        r.packageInfo = getPackageInfoNoCheck(  
                r.activityInfo.applicationInfo, r.compatInfo);  
        handleLaunchActivity(r, null); // 处理消息  
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);  
    } break;  
    case RELAUNCH_ACTIVITY: {  
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityRestart");  
        ActivityClientRecord r = (ActivityClientRecord)msg.obj;  
        handleRelaunchActivity(r);  
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);  
    } break;  
    case PAUSE_ACTIVITY:  
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityPause");  
        handlePauseActivity((IBinder)msg.obj, false, (msg.arg1&1) != 0, msg.arg2,  
                (msg.arg1&2) != 0);  
        maybeSnapshot();  
        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);  
        break;  
    ... ...  
}  
```
首先将msg里面的obj转成一个ActivityClientRecord对象，然后调用来获取一个LoaderApk对象并保存在ActivityClientRecord对象的成员变量packageInfo中。Loader对象用于描述一个已经加载的APK文件。最后调用handleLaunchActivity来启动Activity组件。

## ActivityThread.handleLaunchActivity
```java
private void handleLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
    unscheduleGcIdler();  
    mSomeActivitiesChanged = true;  

    if (r.profilerInfo != null) {  
        mProfiler.setProfiler(r.profilerInfo);  
        mProfiler.startProfiling();  
    }  

    handleConfigurationChanged(null, null);  

    if (localLOGV) Slog.v(  
        TAG, "Handling launch of " + r);  

    WindowManagerGlobal.initialize();  

    Activity a = performLaunchActivity(r, customIntent); //performLaunchActivity真正完成了activity的调起,Activity被实例化，onCreate被调用  

    if (a != null) {  
        r.createdConfig = new Configuration(mConfiguration);  
        Bundle oldState = r.state;  
        handleResumeActivity(r.token, false, r.isForward, // 再调用Activity实例的Resume(用户界面可见)  
                !r.activity.mFinished && !r.startsNotResumed);  

        if (!r.activity.mFinished && r.startsNotResumed) {  
            try {  
                r.activity.mCalled = false;  
                mInstrumentation.callActivityOnPause(r.activity); // finish的时候先调onPause  
                if (r.isPreHoneycomb()) {  
                    r.state = oldState;  
                }  
                if (!r.activity.mCalled) {  
                    throw new SuperNotCalledException(  
                        "Activity " + r.intent.getComponent().toShortString() +  
                        " did not call through to super.onPause()");  
                }  

            } catch (SuperNotCalledException e) {  
                throw e;  

            } catch (Exception e) {  
                if (!mInstrumentation.onException(r.activity, e)) {  
                    throw new RuntimeException(  
                            "Unable to pause activity "  
                            + r.intent.getComponent().toShortString()  
                            + ": " + e.toString(), e);  
                }  
            }  
            r.paused = true;  
        }  
    } else {  
        try {  
            ActivityManagerNative.getDefault() // finishActivity    一样的原理  
                .finishActivity(r.token, Activity.RESULT_CANCELED, null, false);  
        } catch (RemoteException ex) {  
        }  
    }  
}  
```
到了这一步，那就很清晰了。憋了一口气到这里，是不是突然放松了一下~~  再来看看performLaunchActivity做的事儿~~performLaunchActivity函数加载用户自定义的Activity的派生类，并执行其onCreate函数，它将返回此Activity对象。

## ActivityThread.performLaunchActivity
```java
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {  
    ActivityInfo aInfo = r.activityInfo;  
    if (r.packageInfo == null) {  
        r.packageInfo = getPackageInfo(aInfo.applicationInfo, r.compatInfo,  
                Context.CONTEXT_INCLUDE_CODE);  
    }  
    //从intent中取出目标activity的启动参数（包名、类名等）  
    ComponentName component = r.intent.getComponent();  
    if (component == null) {  
        component = r.intent.resolveActivity(  
            mInitialApplication.getPackageManager());  
        r.intent.setComponent(component);  
    }  

    if (r.activityInfo.targetActivity != null) {  
        component = new ComponentName(r.activityInfo.packageName,  
                r.activityInfo.targetActivity);  
    }  

    Activity activity = null;  
    try {  
        java.lang.ClassLoader cl = r.packageInfo.getClassLoader(); // 将Activity类文件加载到内存中  
        activity = mInstrumentation.newActivity( // 创建Activity实例  
                cl, component.getClassName(), r.intent);  
        StrictMode.incrementExpectedActivityCount(activity.getClass());  
        r.intent.setExtrasClassLoader(cl);  
        r.intent.prepareToEnterProcess();  
        if (r.state != null) {  
            r.state.setClassLoader(cl);  
        }  
    } catch (Exception e) {  
        if (!mInstrumentation.onException(activity, e)) {  
            throw new RuntimeException(  
                "Unable to instantiate activity " + component  
                + ": " + e.toString(), e);  
        }  
    }  

    try {  
        Application app = r.packageInfo.makeApplication(false, mInstrumentation);  

        if (localLOGV) Slog.v(TAG, "Performing launch of " + r);  
        if (localLOGV) Slog.v(  
                TAG, r + ": app=" + app  
                + ", appName=" + app.getPackageName()  
                + ", pkg=" + r.packageInfo.getPackageName()  
                + ", comp=" + r.intent.getComponent().toShortString()  
                + ", dir=" + r.packageInfo.getAppDir());  

        if (activity != null) {  
            Context appContext = createBaseContextForActivity(r, activity); // 初始化Context对象，作为Activity的上下文  
            CharSequence title = r.activityInfo.loadLabel(appContext.getPackageManager());  
            Configuration config = new Configuration(mCompatConfiguration);  
            if (DEBUG_CONFIGURATION) Slog.v(TAG, "Launching activity "  
                    + r.activityInfo.name + " with config " + config);  
            activity.attach(appContext, this, getInstrumentation(), r.token,  
                    r.ident, app, r.intent, r.activityInfo, title, r.parent,  
                    r.embeddedID, r.lastNonConfigurationInstances, config,  
                    r.referrer, r.voiceInteractor);  

            if (customIntent != null) {  
                activity.mIntent = customIntent;  
            }  
            r.lastNonConfigurationInstances = null;  
            activity.mStartedActivity = false;  
            int theme = r.activityInfo.getThemeResource();  
            if (theme != 0) {  
                activity.setTheme(theme);  
            }  

            activity.mCalled = false;  
            if (r.isPersistable()) { //下面就是调用到acitivity的onCreate方法了  
                mInstrumentation.callActivityOnCreate(activity, r.state, r.persistentState);  
            } else {  
                mInstrumentation.callActivityOnCreate(activity, r.state);  
            } // 至此，Activity启动过程就结束了，其生命周期由ApplicationThread来管理  
            if (!activity.mCalled) {  
                throw new SuperNotCalledException(  
                    "Activity " + r.intent.getComponent().toShortString() +  
                    " did not call through to super.onCreate()");  
            }  
            r.activity = activity;  
            r.stopped = true;  
            if (!r.activity.mFinished) {  
                activity.performStart();  
                r.stopped = false;  
            }  
            if (!r.activity.mFinished) {  
                if (r.isPersistable()) {  
                    if (r.state != null || r.persistentState != null) {  
                        mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state,  
                                r.persistentState);  
                    }  
                } else if (r.state != null) {  
                    mInstrumentation.callActivityOnRestoreInstanceState(activity, r.state);  
                }  
            }  
            if (!r.activity.mFinished) {  
                activity.mCalled = false;  
                if (r.isPersistable()) {  
                    mInstrumentation.callActivityOnPostCreate(activity, r.state,  
                            r.persistentState);  
                } else {  
                    mInstrumentation.callActivityOnPostCreate(activity, r.state);  
                }  
                if (!activity.mCalled) {  
                    throw new SuperNotCalledException(  
                        "Activity " + r.intent.getComponent().toShortString() +  
                        " did not call through to super.onPostCreate()");  
                }  
            }  
        }  
        r.paused = true;  
        mActivities.put(r.token, r); // 将ActivityRecord对象保存在ActivityThread的mActivities中  
    } catch (SuperNotCalledException e) {  
        throw e;  
    } catch (Exception e) {  
        if (!mInstrumentation.onException(activity, e)) {  
            throw new RuntimeException(  
                "Unable to start activity " + component  
                + ": " + e.toString(), e);  
        }  
    }  
    return activity;  
} 
```
ActivityRecord里面的token，是一个Binder的代理对象，和ActivityClientRecord对象一样，都是用来描述所启动的Activity组件，只不过前者是在ActivityManagerService中使用，后者是在应用程序进程中使用。

至此，Activity的启动过程就分析完了。MainActivity的启动过程，其实也可以认为是应用程序的启动过程。

子Activity的启动过程和根Activity的启动过程也是类似的，过程如下：

1. MainActivity向ActivityManagerService发送一个自动ChildActivity的请求；
2. ActivityManagerService首先将ChildActivity的信息保存下来，再向MainActivity发送一个中止的请求；
3. MainActivity收到请求进入中止状态，告诉ActivityManagerService，便于ActivityManagerService继续执行启动ChildActivity的操作
4. ActivityManagerService检查ChildActivity所运行的进程是否存在，存在就发送ChildActivity信息给他，以进行启动。

源代码方面，原理类似，相比起来会比MainActivity的稍微简单一些，这里就不再详细叙述了，各位可以自行根据前面步骤，阅读源代码。

另附：Android 5.0 系统源码下载地址

[https://pan.baidu.com/s/1dElkPlZ](https://pan.baidu.com/s/1dElkPlZ) 密码：4pue 