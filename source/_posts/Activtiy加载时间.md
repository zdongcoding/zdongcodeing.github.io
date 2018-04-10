---
title: Activtiy加载时间调研
copyright: true
categories: android
date: 2018-04-10 17:22:12
tags:
---

顾名思义
>个人理解，进入Activity开始一直到首屏页面被渲染出来（用户可见状态）这个时间差不多就是Activity的加载时间！

### 方案一、onCreate(),onResume()的时间差
首先脑海里弄出来的方案就是分别在onCreate(),onResume() 统计时间计算时间差，
查看源码发现 Activity#onResume()
```java
/**
     * Called after {@link #onRestoreInstanceState}, {@link #onRestart}, or
     * {@link #onPause}, for your activity to start interacting with the user.
     * This is a good place to begin animations, open exclusive-access devices
     * (such as the camera), etc.
     *
     * <p>Keep in mind that onResume is not the best indicator that your activity
     * is visible to the user; a system window such as the keyguard may be in
     * front.  Use {@link #onWindowFocusChanged} to know for certain that your
     * activity is visible to the user (for example, to resume a game).
     *
     * <p><em>Derived classes must call through to the super class's
     * implementation of this method.  If they do not, an exception will be
     * thrown.</em></p>
     *
     * @see #onRestoreInstanceState
     * @see #onRestart
     * @see #onPostResume
     * @see #onPause
     */
    @CallSuper
    protected void onResume() {
        if (DEBUG_LIFECYCLE) Slog.v(TAG, "onResume " + this);
        getApplication().dispatchActivityResumed(this);
        mActivityTransitionState.onResume();
        mCalled = true;
    }
```
onResume() 用户可交互的状态，显然作为用户可见的状态 activity加载时间有误差较大


```java
//android.app.ActivityThread#handleResumeActivity
final void handleResumeActivity(IBinder token,
            boolean clearHide, boolean isForward, boolean reallyResume) {
       
       ...

        // TODO Push resumeArgs into the activity for consideration
        //内部调用了   r.activity.performResume();
        ActivityClientRecord r = performResumeActivity(token, clearHide);

       ...
       
            if (r.window == null && !a.mFinished && willBeVisible) {
                r.window = r.activity.getWindow();
                View decor = r.window.getDecorView();
                decor.setVisibility(View.INVISIBLE);
                ViewManager wm = a.getWindowManager();
                WindowManager.LayoutParams l = r.window.getAttributes();
                a.mDecor = decor;
                l.type = WindowManager.LayoutParams.TYPE_BASE_APPLICATION;
                l.softInputMode |= forwardBit;
                if (a.mVisibleFromClient) {
                    a.mWindowAdded = true;
                    wm.addView(decor, l);
                }
        /** wm对象其实是 WindowManager（WindowManagerImpl） 
        在android.view.Window#setWindowManager(android.view.WindowManager, android.os.IBinder, java.lang.String, boolean) 创建
        **/
         ...
         
            // Get rid of anything left hanging around.
            cleanUpPendingRemoveWindows(r);

            // The window is now visible if it has been added, we are not
            // simply finishing, and we are not starting another activity.
            if (!r.activity.mFinished && willBeVisible
                    && r.activity.mDecor != null && !r.hideForNow) {
                if (r.newConfig != null) {
                    r.tmpConfig.setTo(r.newConfig);
                    if (r.overrideConfig != null) {
                        r.tmpConfig.updateFrom(r.overrideConfig);
                    }
                    if (DEBUG_CONFIGURATION) Slog.v(TAG, "Resuming activity "
                            + r.activityInfo.name + " with newConfig " + r.tmpConfig);
                    performConfigurationChanged(r.activity, r.tmpConfig);
                    freeTextLayoutCachesIfNeeded(r.activity.mCurrentConfig.diff(r.tmpConfig));
                    r.newConfig = null;
                }
                if (localLOGV) Slog.v(TAG, "Resuming " + r + " with isForward="
                        + isForward);
                WindowManager.LayoutParams l = r.window.getAttributes();
                if ((l.softInputMode
                        & WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION)
                        != forwardBit) {
                    l.softInputMode = (l.softInputMode
                            & (~WindowManager.LayoutParams.SOFT_INPUT_IS_FORWARD_NAVIGATION))
                            | forwardBit;
                    if (r.activity.mVisibleFromClient) {
                        ViewManager wm = a.getWindowManager();
                        View decor = r.window.getDecorView();
                        wm.updateViewLayout(decor, l);
                    }
                }
                r.activity.mVisibleFromServer = true;
                mNumVisibleActivities++;
                if (r.activity.mVisibleFromClient) {
                    r.activity.makeVisible();
                }
            }
        ...
    }
```

performResumeActivity 之后才开始正真的绘制界面   wm.addView(decor, l);
```java 
handleResumeActivity()
  --- performResumeActivity()
      --- r.activity.performResume();
          --- onResume()
  ---add DecorView   (wm.addView(decor, l);)
  ---wm.updateViewLayout(decor, l);
     --- scheduleTraversals
          ---  performDraw（）， performMeasure（） ， performLayout（）
```
onResume() 之后才开始 绘制布局
上面代码更加证明了onResume()不能计算结束加载时间标志 

### 方案二、onWindowFocusChanged()
```java
/**
     * <p>Keep in mind that onResume is not the best indicator that your activity
     * is visible to the user; a system window such as the keyguard may be in
     * front.  Use {@link #onWindowFocusChanged} to know for certain that your
     * activity is visible to the user (for example, to resume a game).
**/

-> Activity 构造函数
-> Activity.setTheme()
-> Activity.onCreate()
-> Activity.onStart
-> Activity.onResume
-> Activity.onAttachedToWindow
-> Activity.onWindowFocusChanged

```
onResume()的注释发现用户可见状态  onWindowFocusChanged

android.app.Activity#onWindowFocusChanged 在什么时候调用的呢？？？
>   android.view.Window.Callback#onWindowFocusChanged

```java
android.app.ActivityThread#handleResumeActivity()
--- wm.updateViewLayout(decor, l);  //DecorView  , wm (ViewManager) a.getWindowManager();
    --- wm (WindowManagerImpl)  实际上 android.view.WindowManagerGlobal#updateViewLayout
        --- android.view.ViewRootImpl#setLayoutParams
                --- scheduleTraversals
                    --- doTraversal（） -> performTraversals()
                            ---  performDraw（）， performMeasure（） ， performLayout（）
          ---
performTraversals()中有这两句话  host就是mDecorView
host.dispatchAttachedToWindow(mAttachInfo, 0);  //将mAttachInfo  和mView 绑定
mAttachInfo.mTreeObserver.dispatchOnWindowAttachedChange(true);

android.view.ViewRootImpl.W   
@Override
public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
    final ViewRootImpl viewAncestor = mViewAncestor.get();
    if (viewAncestor != null) {
       viewAncestor.windowFocusChanged(hasFocus, inTouchMode);
    }
}
android.view.ViewRootImpl#windowFocusChanged
public void windowFocusChanged(boolean hasFocus, boolean inTouchMode) {
        Message msg = Message.obtain();
        msg.what = MSG_WINDOW_FOCUS_CHANGED;
        msg.arg1 = hasFocus ? 1 : 0;
        msg.arg2 = inTouchMode ? 1 : 0;
        mHandler.sendMessage(msg);
    }
    
android.view.ViewRootImpl.ViewRootHandler
//android.view.ViewRootImpl#windowFocusChanged() // mHandler.sendMessage(msg); MSG_WINDOW_FOCUS_CHANGED
 if (mView != null) { //mView就是mDecorView
   mAttachInfo.mKeyDispatchState.reset();
    mView.dispatchWindowFocusChanged(hasWindowFocus);
    mAttachInfo.mTreeObserver.dispatchOnWindowFocusChange(hasWindowFocus);
}
```


### 方案三、IdleHandler

![Handler_looper](http://ow9n8vqns.bkt.clouddn.com/QQ20180409-102332.png)
> 空闲线程MessageQueue 回调
向消息队列中添加一个新的MessageQueue.IdleHandler。当调用IdleHandler.queueIdle()返回false时，此MessageQueue.IdleHandler会自动的从消息队列中移除。或者调用removeIdleHandler(MessageQueue.IdleHandler)也可以从消息队列中移除MessageQueue.IdleHandler

activity 所有的生命周期方法都是通过 Handler 处理的，

结果测试  
```java
-> Activity 构造函数
-> Activity.setTheme()
-> Activity.onCreate()
-> Activity.onStart
-> Activity.onResume
-> Activity.onAttachedToWindow
-> Activity.onWindowFocusChanged
-> IdleHandler 回调
```
问题 ：IdleHandler回调的时机会受到 Handler.sendxxxMessage 受影响
>如果再onCreate方法中 Handler.sendxxxMessage 会影响回调时间
