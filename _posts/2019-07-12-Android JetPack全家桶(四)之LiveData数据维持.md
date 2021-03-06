---
layout:     post
title:      Android Jetpack全家桶(四)之LiveData数据维持
subtitle:    ""
date:       2019-07-12
author:     Toeii
header-img: img/post-bg-2015.jpg
catalog: true
tags:
    - Android
---



## 前言

Android Jetpack是Google在18年IO大会上推荐的一整套组件库，它的出现填补了之前Android中自带的一些缺陷，例如Handler的内存泄露、Camera的不易用性、后台调度难以管理等等。所以我打算把整个架构组件系统性的学习一下，在这里和大家分享，希望能帮助到其他学习者。本系列文章包含十篇：

- [Android Jetpack全家桶(一)之Jetpack介绍](https://toeii.github.io/2019/07/09/Android-Jetpack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%B8%80)%E4%B9%8BJetpack%E4%BB%8B%E7%BB%8D/)<br />
- [Android Jetpack全家桶(二)之Lifecycle生命周期感知](https://toeii.github.io/2019/07/09/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%BA%8C)%E4%B9%8BLifecycle%E7%94%9F%E5%91%BD%E5%91%A8%E6%9C%9F%E6%84%9F%E7%9F%A5/)<br />
- [Android Jetpack全家桶(三)之ViewModel控制器](https://toeii.github.io/2019/07/10/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%B8%89)%E4%B9%8BViewModel%E6%8E%A7%E5%88%B6%E5%99%A8/)<br />
- [Android Jetpack全家桶(四)之LiveData数据维持](https://toeii.github.io/2019/07/12/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E5%9B%9B)%E4%B9%8BLiveData%E6%95%B0%E6%8D%AE%E7%BB%B4%E6%8C%81/)<br />
- [Android Jetpack全家桶(五)之Room ORM库](https://toeii.github.io/2019/07/17/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%BA%94)%E4%B9%8BRoom-ORM%E5%BA%93/)<br />
- [Android Jetpack全家桶(六)之Paging分页库](https://toeii.github.io/2019/07/19/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E5%85%AD)%E4%B9%8BPaging%E5%88%86%E9%A1%B5%E5%BA%93/)<br />
- [Android Jetpack全家桶(七)之WorkManager工作管理](https://toeii.github.io/2019/08/01/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%B8%83)%E4%B9%8BWorkManager%E5%B7%A5%E4%BD%9C%E7%AE%A1%E7%90%86/)<br />
- [Android Jetpack全家桶(八)之Navigation导航](https://toeii.github.io/2019/08/06/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E5%85%AB)%E4%B9%8BNavigation%E5%AF%BC%E8%88%AA/)<br />
- [Android Jetpack全家桶(九)之DataBinding数据绑定](https://toeii.github.io/2019/08/07/Android-JetPack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E4%B9%9D)%E4%B9%8BDataBinding%E6%95%B0%E6%8D%AE%E7%BB%91%E5%AE%9A/)<br />
- [Android Jetpack全家桶(十)之从0到1写一个Jetpack项目](https://toeii.github.io/2019/11/20/Android-Jetpack%E5%85%A8%E5%AE%B6%E6%A1%B6(%E5%8D%81)%E4%B9%8B%E4%BB%8E0%E5%88%B01%E5%86%99%E4%B8%80%E4%B8%AAJetPack%E9%A1%B9%E7%9B%AE/)<br />



## 介绍

LiveData与常规Observable不同，LiveData是具有生命周期感知的。它在生命周期中扮演观察者的角色，监听生命周期。

官方说，生命周期处于onStart()或onResume状态，则LiveData会将其视为处于活跃状态。
但是根据源码来看，LiveData的活跃状态其实是在onPause()和onStart()前后的，在这一期间LiveData可以对活动观察者进行数据更新。


```java

class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {

    ...

    @Override
    boolean shouldBeActive() {
        return mOwner.getLifecycle().getCurrentState().isAtLeast(STARTED); //onStart开始为活跃状态
    }

}

private void considerNotify(ObserverWrapper observer) {
    
    if (!observer.shouldBeActive()) {//判断是否为活跃状态
        observer.activeStateChanged(false);
        return;
    }

    observer.mObserver.onChanged((T) mData);
}

```

通常我们实现LifecycleOwner接口的对象配对的观察者，之后生命周期在onDestroy()时通过LiveData.observer()调用LifecycleBoundObserver中的removeObserver()回收资源，避免内存泄漏。

一句话概括，它主要是以观察者模式实现数据驱动，并在生命周期结束时回收资源避免内存泄漏。

## 如何使用？

#### 添加依赖

```java

// lifecycle
def lifecycle_version = "2.2.0-alpha02"
implementation "androidx.lifecycle:lifecycle-extensions:$lifecycle_version"
annotationProcessor "android.arch.lifecycle:compiler:$lifecycle_version"
kapt "android.arch.lifecycle:compiler:$lifecycle_version"

```

#### 监听LiveData数据

```java

class MainActivity : AppCompatActivity() {

    private lateinit var mLiveData: MutableLiveData<String>

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        mLiveData = MutableLiveData()

        mLiveData.observe(this, Observer<String> {
            updateText ->
            System.out.println("update new text ---> " + updateText)
        })

    }

    override fun onStart() {
        super.onStart()
        mLiveData.value = "start text from MainThread"
    }

    override fun onResume() {
        super.onResume()
        Thread(Runnable {
            mLiveData.postValue("resume text from BranchThread")
        }).run()
    }

}

```

## 分析LiveData

#### 数据更新

##### LiveData.setValue

LiveData的setValue()方法只能在主线程中调用。

```java

@MainThread //约定主线程运行
protected void setValue(T value) {
    assertMainThread("setValue"); //判断是否在主线程，否则return异常
    mVersion++;
    mData = value;
    dispatchingValue(null);
}

```

##### LiveData.postValue

LiveData的postValue()方法则只能在子线程中调用，这种做法是为了区分调用场景。

```java

protected void postValue(T value) {
    boolean postTask;
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) { //判断是否在子线程，否则return，更新数据失败
        return;
    }
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable); //ArchTaskExecutor内则是调用了Handler去更新数据
}

```

#### 资源回收
LiveData在注册observer的时候，会绑定当前的Lifecycle，注册LifecycleOwner接口并实例LifecycleBoundObserver对象。

```java

@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer); //实例LifecycleBoundObserver
    ObserverWrapper existing = mObservers.putIfAbsent(observer, wrapper);
    if (existing != null && !existing.isAttachedTo(owner)) {
        throw new IllegalArgumentException("Cannot add the same observer"
                + " with different lifecycles");
    }
    if (existing != null) {
        return;
    }
    owner.getLifecycle().addObserver(wrapper);
}

```

而当Lifecycle调用onDestroy时，会被LifecycleBoundObserver监听到，并通过Lifecycle.removeObserver对资源进行回收，从而避免内存泄露。

```java

class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    
    ...

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
            removeObserver(mObserver);//回收操作
            return;
        }
        activeStateChanged(shouldBeActive());
    }

    @Override
    void detachObserver() {
        mOwner.getLifecycle().removeObserver(this); 
    }
}

```

## 结语

最后，希望大家可以结合[demo](https://github.com/toeii/LiveDataSimpleExample)深入源码进行学习。


