reference: https://juejin.cn/post/7341617163353374754
1. 每一个LiveData对象中都有一个mObservers集合。mObservers中可以存储该LiveData对象的所有观察者。即一个LiveData对象可以有多个观察者。
数据更新方法：
（1）setValue
```   
@MainThread
protected void setValue(T value) {
    //检查是否在主线程使用，非主线程报错
    assertMainThread("setValue");
    
    //每一次更新数据将版本属性++
    mVersion++;
    
    //更新数据
    mData = value;
    
    //分发数据
    dispatchingValue(null);
}
```
（2）postValue（一般在子线程调用，与上面[setValue]区别在于[没有mVersion++]）
```
protected void postValue(T value) {
boolean postTask;

    //锁保护线程安全，即支持主线程调用
    synchronized (mDataLock) {
        postTask = mPendingData == NOT_SET;
        mPendingData = value;
    }
    if (!postTask) {
        return;
    }
    
    //切换到主线程run一个runnnable
    ArchTaskExecutor.getInstance().postToMainThread(mPostValueRunnable);
}
```
```
private final Runnable mPostValueRunnable = new Runnable() {
    @SuppressWarnings("unchecked")
    @Override
    public void run() {
        Object newValue;
        synchronized (mDataLock) {
            newValue = mPendingData;
            mPendingData = NOT_SET;
        }
        
        //原来postValue 在切换线程以后又调用了一次 setValue，
        //那么mVersion，自然也是通过setValue更新的
        setValue((T) newValue);
    }
};
```
调用链路：
setValue -> dispatchingValue -> considerNotify(对observer的状态进行过滤，以及版本判断)
-> mObserver.onChanged((T) mData)
```
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    //**********记住这里后面会提到***********
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    observer.mObserver.onChanged((T) mData);
}

```

2. LiveData如何感知生命周期
Lifecycle机制：
LifecycleOwner:(Activity exterds ... ComponentActivity实现了[LifecycleOwner])
considerNotify时，会进行[shouldBeActive]检测生命周期的状态，至少为[STARTED]

3. LiveData具有粘性
更新状态时会先调到 LifecycleRegistry.setCurrentState 更新生命周期状态
-> LifecycleRegistry中moveToState方法的Sync()
-> Sync的forwardPass
-> dispatchEvent中进行mLifecycleObserver.onStateChanged回调

4. 是否遇到过LiveData丢失数据的情况
子线程、多线程频繁调用 liveData.postValue，是会出现的。
解决方法：
   (1) 可以切换到主线程，setValue去更新（浪费性能）
   (2) 切换Rxjava,Flow
5. Fragment使用LiveData
   Fragment加载分为[add]和[replace]两种，add的时候，前一个Fragment走不到onDestroy，无法触发，导致liveData不会去进行解注册操作
（1）可以都去使用replace进行Fragment跳转
（2）使用viewLifecycleOwner
6. 想要一直监听LiveData，无论是否活跃怎么办
使用LiveData的[observerForever]方法

