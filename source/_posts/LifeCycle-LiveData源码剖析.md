---
title: LifeCycle & LiveData源码剖析
date: 2022-02-14 20:33:46
tags: LifeCycle, LiveData
---

![类图](https://s3.bmp.ovh/imgs/2022/02/c5d4136edbfe0f20.png)

# 生命周期回调方法映射为事件，并派发给LifeCycle

```java
androidx.fragment.app.FragmentActivity#onStart
@Override
protected void onStart() {
    super.onStart();
    // ……
    // NOTE: HC onStart goes here.
    mFragmentLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_START);
    mFragments.dispatchStart();
}

androidx.lifecycle.LifecycleRegistry#handleLifecycleEvent
/**
 * Sets the current state and notifies the observers.
 * <p>
 * Note that if the {@code currentState} is the same state as the last call to this method,
 * calling this method has no effect.
 *
 * @param event The event that was received
 */
public void handleLifecycleEvent(@NonNull Lifecycle.Event event) {
    State next = getStateAfter(event);
    moveToState(next);
}

private void moveToState(State next) {
    if (mState == next) {
        return;
    }
    mState = next;
    if (mHandlingEvent || mAddingObserverCounter != 0) {
        mNewEventOccurred = true;
        // we will figure out what to do on upper level.
        return;
    }
    mHandlingEvent = true;
    sync();
    mHandlingEvent = false;
}

// happens only on the top of stack (never in reentrance),
// so it doesn't have to take in account parents
private void sync() {
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        throw new IllegalStateException("LifecycleOwner of this LifecycleRegistry is already"
                + "garbage collected. It is too late to change lifecycle state.");
    }
    while (!isSynced()) {
        mNewEventOccurred = false;
        // no need to check eldest for nullability, because isSynced does it for us.
// State descends, such as RESUMED to STARTED
        if (mState.compareTo(mObserverMap.eldest().getValue().mState) < 0) {
            backwardPass(lifecycleOwner);
        }
// State ascends, such at STARTED TO RESUMED
        Entry<LifecycleObserver, ObserverWithState> newest = mObserverMap.newest();
        if (!mNewEventOccurred && newest != null
                && mState.compareTo(newest.getValue().mState) > 0) {
            forwardPass(lifecycleOwner);
        }
    }
    mNewEventOccurred = false;
}
```

![状态图](https://s3.bmp.ovh/imgs/2022/02/a70384d33bf36163.png)

如果是状态降级，从尾部向头部遍历监听队列，如果是状态升级，从头部向尾部遍历监听队列。之所以如此区分是因为要保证FastSafeIterableMap的Invariant，但为什么要维持这个不变量呢？原则上，LifeCycleOwner 从ON_CREATED到ON_RESUMED，再到ON_CREATED是一种状态回退，监听回调顺序也要从向前到向后回退。FastSafeIterableMap 是一个map，同时每个Entry是一个双向链表，新加入元素插入尾部。

```java
androidx.lifecycle.LifecycleRegistry#mObserverMap

/**
 * Custom list that keeps observers and can handle removals / additions during traversal.
 *
 * Invariant: at any moment of time for observer1 & observer2:
 * if addition_order(observer1) < addition_order(observer2), then
 * state(observer1) >= state(observer2),
 */
private FastSafeIterableMap<LifecycleObserver, ObserverWithState> mObserverMap =
        new FastSafeIterableMap<>();
```

```java
androidx.lifecycle.LifecycleRegistry#backwardPass

private void backwardPass(LifecycleOwner lifecycleOwner) {
    Iterator<Entry<LifecycleObserver, ObserverWithState>> descendingIterator =
            mObserverMap.descendingIterator();
    while (descendingIterator.hasNext() && !mNewEventOccurred) {
        Entry<LifecycleObserver, ObserverWithState> entry = descendingIterator.next();
        ObserverWithState observer = entry.getValue();
        while ((observer.mState.compareTo(mState) > 0 && !mNewEventOccurred
                && mObserverMap.contains(entry.getKey()))) {
            Event event = downEvent(observer.mState);
            pushParentState(getStateAfter(event));
            observer.dispatchEvent(lifecycleOwner, event);
            popParentState();
        }
    }
}
```

```java
androidx.lifecycle.LifecycleRegistry.ObserverWithState#dispatchEvent
void dispatchEvent(LifecycleOwner owner, Event event) {
    State newState = getStateAfter(event);
    mState = min(mState, newState);
    mLifecycleObserver.onStateChanged(owner, event);
    mState = newState;
}
```

```java
/**
 * Class that can receive any lifecycle change and dispatch it to the receiver.
 * <p>
 * If a class implements both this interface and
 * {@link androidx.lifecycle.DefaultLifecycleObserver}, then
 * methods of {@code DefaultLifecycleObserver} will be called first, and then followed by the call
 * of {@link LifecycleEventObserver#onStateChanged(LifecycleOwner, Lifecycle.Event)}
 * <p>
 * If a class implements this interface and in the same time uses {@link OnLifecycleEvent}, then
 * annotations will be ignored.
 */
public interface LifecycleEventObserver extends LifecycleObserver {
    /**
     * Called when a state transition event happens.
     *
     * @param source The source of the event
     * @param event The event
     */
    void onStateChanged(@NonNull LifecycleOwner source, @NonNull Lifecycle.Event event);
}
```

LiveData包 androidx.lifecycle.LiveData.LifecycleBoundObserver实现了这个接口，接下来看如何实现和使用

# 监听注册

```java
androidx.lifecycle.LiveData#observe
/**
 * Adds the given observer to the observers list within the lifespan of the given
 * owner. The events are dispatched on the main thread. If LiveData already has data
 * set, it will be delivered to the observer.
 * <p>
 * The observer will only receive events if the owner is in {@link Lifecycle.State#STARTED}
 * or {@link Lifecycle.State#RESUMED} state (active).
 * <p>
 * If the owner moves to the {@link Lifecycle.State#DESTROYED} state, the observer will
 * automatically be removed.
 * <p>
 * When data changes while the {@code owner} is not active, it will not receive any updates.
 * If it becomes active again, it will receive the last available data automatically.
 * <p>
 * LiveData keeps a strong reference to the observer and the owner as long as the
 * given LifecycleOwner is not destroyed. When it is destroyed, LiveData removes references to
 * the observer &amp; the owner.
 * <p>
 * If the given owner is already in {@link Lifecycle.State#DESTROYED} state, LiveData
 * ignores the call.
 * <p>
 * If the given owner, observer tuple is already in the list, the call is ignored.
 * If the observer is already in the list with another owner, LiveData throws an
 * {@link IllegalArgumentException}.
 *
 * @param owner    The LifecycleOwner which controls the observer
 * @param observer The observer that will receive the events
 */
@MainThread
public void observe(@NonNull LifecycleOwner owner, @NonNull Observer<? super T> observer) {
    assertMainThread("observe");
    if (owner.getLifecycle().getCurrentState() == DESTROYED) {
        // ignore
        return;
    }
    LifecycleBoundObserver wrapper = new LifecycleBoundObserver(owner, observer);
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

observer 被封装为LifecycleBoundObserver，并注册到Lifecycle的监听队列里，当生命周期方法调用，并派发事件，最终androidx.lifecycle.LiveData.LifecycleBoundObserver#onStateChanged被调用，对于LiveData而言，它只关注两个方面：

1. 如果是Lifecycle 已经destroy，那么自动移除双向监听（对LiveData，和对Lifecycle的监听）；
2. 判断active & inActive 状态变更方向，如果是inactive →  active，说明Lifecycle重新回到前台，则将LiveData的value派发给监听者，提供数据刷新UI。

```java
class LifecycleBoundObserver extends ObserverWrapper implements GenericLifecycleObserver {
    @NonNull
    final LifecycleOwner mOwner;

    LifecycleBoundObserver(@NonNull LifecycleOwner owner, Observer<? super T> observer) {
        super(observer);
        mOwner = owner;
    }

    @Override
    public void onStateChanged(LifecycleOwner source, Lifecycle.Event event) {
        if (mOwner.getLifecycle().getCurrentState() == DESTROYED) {
						// Remove double direction observers.
            removeObserver(mObserver);
            return;
        }
				// Dispatch value if state changes from inactive to active.
        activeStateChanged(shouldBeActive());
    }
}
```

LifecycleBoundObserver 注册观察Lifecycle时，需要处理注册时，错失的生命周期方法派发，将当前LifecycleBoundObserver的状态推送至Lifecycle一致。

```java
androidx.lifecycle.LifecycleRegistry#addObserver
@Override
public void addObserver(@NonNull LifecycleObserver observer) {
    State initialState = mState == DESTROYED ? DESTROYED : INITIALIZED;
    ObserverWithState statefulObserver = new ObserverWithState(observer, initialState);
    ObserverWithState previous = mObserverMap.putIfAbsent(observer, statefulObserver);

    if (previous != null) {
        return;
    }
    LifecycleOwner lifecycleOwner = mLifecycleOwner.get();
    if (lifecycleOwner == null) {
        // it is null we should be destroyed. Fallback quickly
        return;
    }

    boolean isReentrance = mAddingObserverCounter != 0 || mHandlingEvent;
    State targetState = calculateTargetState(observer);
    mAddingObserverCounter++;
    while ((statefulObserver.mState.compareTo(targetState) < 0
            && mObserverMap.contains(observer))) {
        pushParentState(statefulObserver.mState);
        statefulObserver.dispatchEvent(lifecycleOwner, upEvent(statefulObserver.mState));
        popParentState();
        // mState / subling may have been changed recalculate
        targetState = calculateTargetState(observer);
    }

    if (!isReentrance) {
        // we do sync only on the top level.
        sync();
    }
    mAddingObserverCounter--;
}
```

# 数据变更

```java
androidx.lifecycle.LiveData#setValue
/**
 * Sets the value. If there are active observers, the value will be dispatched to them.
 * <p>
 * This method must be called from the main thread. If you need set a value from a background
 * thread, you can use {@link #postValue(Object)}
 *
 * @param value The new value
 */
@MainThread
protected void setValue(T value) {
    assertMainThread("setValue");
    mVersion++;
    mData = value;
    dispatchingValue(null);
}
```

```java
androidx.lifecycle.LiveData#dispatchingValue
@SuppressWarnings("WeakerAccess") /* synthetic access */
void dispatchingValue(@Nullable ObserverWrapper initiator) {
    if (mDispatchingValue) {
        mDispatchInvalidated = true;
        return;
    }
    mDispatchingValue = true;
    do {
        mDispatchInvalidated = false;
        if (initiator != null) {
// Maxim: Lifecycle changes from inactive to active, but the data keeps still,
// notify the corresponding observer in the lifecycle owner only.
            considerNotify(initiator);
            initiator = null;
        } else {
//Maxim:  The data source has changed, notify all observers.
            for (Iterator<Map.Entry<Observer<? super T>, ObserverWrapper>> iterator =
                    mObservers.iteratorWithAdditions(); iterator.hasNext(); ) {
                considerNotify(iterator.next().getValue());
                if (mDispatchInvalidated) {
                    break;
                }
            }
        }
    } while (mDispatchInvalidated);
    mDispatchingValue = false;
}
```

```java
private void considerNotify(ObserverWrapper observer) {
    if (!observer.mActive) {
        return;
    }
    // Check latest state b4 dispatch. Maybe it changed state but we didn't get the event yet.
    //
    // we still first check observer.active to keep it as the entrance for events. So even if
    // the observer moved to an active state, if we've not received that event, we better not
    // notify for a more predictable notification order.
		// Maxim: Don't notify if the observer is not active, for lifecycle, it should
		// at leat be started or above(resumed).
    if (!observer.shouldBeActive()) {
        observer.activeStateChanged(false);
        return;
    }
    if (observer.mLastVersion >= mVersion) {
        return;
    }
    observer.mLastVersion = mVersion;
    //noinspection unchecked
    observer.mObserver.onChanged((T) mData);
}
```



# 更多资料

https://google-developer-training.github.io/android-developer-advanced-course-practicals/unit-6-working-with-architecture-components/lesson-14-room,-livedata,-viewmodel/14-1-a-room-livedata-viewmodel/14-1-a-room-livedata-viewmodel.html
