---
title: FragmentManager源码剖析二
date: 2021-10-18 21:27:09
tags:
---



# remove & detach Fragment差别

```java
public void removeFragment(Fragment fragment) {
    if (DEBUG) Log.v(TAG, "remove: " + fragment + " nesting=" + fragment.mBackStackNesting);
    final boolean inactive = !fragment.isInBackStack();
    if (!fragment.mDetached || inactive) {
        synchronized (mAdded) {
            mAdded.remove(fragment);
        }
        if (isMenuAvailable(fragment)) {
            mNeedMenuInvalidate = true;
        }
        fragment.mAdded = false;
        fragment.mRemoving = true;
    }
}
```

```java
public void detachFragment(Fragment fragment) {
    if (DEBUG) Log.v(TAG, "detach: " + fragment);
    if (!fragment.mDetached) {
        fragment.mDetached = true;
        if (fragment.mAdded) {
            // We are not already in back stack, so need to remove the fragment.
            if (DEBUG) Log.v(TAG, "remove from detach: " + fragment);
            synchronized (mAdded) {
                mAdded.remove(fragment);
            }
            if (isMenuAvailable(fragment)) {
                mNeedMenuInvalidate = true;
            }
            fragment.mAdded = false;
        }
    }
}
```

都是移出mAdded集合，并将mAdded置为false，区别在更新的标记不同mRemoving & mDetached。在FragmentManager同步状态调用时，有对remove & detach的状态同步。

```java
/**
 * Changes the state of the fragment manager to {@code newState}. If the fragment manager
 * changes state or {@code always} is {@code true}, any fragments within it have their
 * states updated as well.
 *
 * @param newState The new state for the fragment manager
 * @param always If {@code true}, all fragments update their state, even
 *               if {@code newState} matches the current fragment manager's state.
 */
void moveToState(int newState, boolean always) {
    if (mHost == null && newState != Fragment.INITIALIZING) {
        throw new IllegalStateException("No activity");
    }

    if (!always && newState == mCurState) {
        return;
    }

    mCurState = newState;

    if (mActive != null) {

        // Must add them in the proper order. mActive fragments may be out of order
				// Fragment add & attach 后的状态同步
        final int numAdded = mAdded.size();
        for (int i = 0; i < numAdded; i++) {
            Fragment f = mAdded.get(i);
            moveFragmentToExpectedState(f);
        }

        // Now iterate through all active fragments. These will include those that are removed
        // and detached.
        final int numActive = mActive.size();
        for (int i = 0; i < numActive; i++) {
            Fragment f = mActive.valueAt(i);
						// Fragment 激活后执行remove 或者 detach，同步状态。
            if (f != null && (f.mRemoving || f.mDetached) && !f.mIsNewlyAdded) {
                moveFragmentToExpectedState(f);
            }
        }

        startPendingDeferredFragments();

        if (mNeedMenuInvalidate && mHost != null && mCurState == Fragment.RESUMED) {
            mHost.onSupportInvalidateOptionsMenu();
            mNeedMenuInvalidate = false;
        }
    }
}
```

```java
/**
 * Moves a fragment to its expected final state or the fragment manager's state, depending
 * on whether the fragment manager's state is raised properly.
 *
 * @param f The fragment to change.
 */
void moveFragmentToExpectedState(Fragment f) {
    if (f == null) {
        return;
    }
    if (!mActive.containsKey(f.mWho)) {
        if (DEBUG) {
            Log.v(TAG, "Ignoring moving " + f + " to state " + mCurState
                    + "since it is not added to " + this);
        }
        return;
    }
    int nextState = mCurState;
    if (f.mRemoving) {
        if (f.isInBackStack()) {
            nextState = Math.min(nextState, Fragment.CREATED);
        } else {
            nextState = Math.min(nextState, Fragment.INITIALIZING);
        }
    }
    moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);
}
```

对于Remove如果nextState 是CREATED，Fragment只会`f.performDestroyView()`，但如果是INITIALIZING，分两种情况：

1. 不在回退栈，执行`f.performDestroy()`;
2. 在回退栈，执行`f.performDetach()`;并`makeInactive(f);`

对于detach，nextState是CREATED，也即只执行`f.performDestroyView()`

```java
@SuppressWarnings("ReferenceEquality")
void moveToState(Fragment f, int newState, int transit, int transitionStyle,
        boolean keepActive) {
    // Fragments that are not currently added will sit in the onCreate() state.
    if ((!f.mAdded || f.mDetached) && newState > Fragment.CREATED) {
        newState = Fragment.CREATED;
    }
}
```

# add & attach Fragment区别

由于addFragment 有activate调用，fragment 已经在mActive映射里，允许状态同步，但attach没有activate 调用，于是状态不同步。

```java
/**
 * Moves a fragment to its expected final state or the fragment manager's state, depending
 * on whether the fragment manager's state is raised properly.
 *
 * @param f The fragment to change.
 */
void moveFragmentToExpectedState(Fragment f) {
    if (f == null) {
        return;
    }
    if (!mActive.containsKey(f.mWho)) {
        if (DEBUG) {
            Log.v(TAG, "Ignoring moving " + f + " to state " + mCurState
                    + "since it is not added to " + this);
        }
        return;
    }
    int nextState = mCurState;
    if (f.mRemoving) {
        if (f.isInBackStack()) {
            nextState = Math.min(nextState, Fragment.CREATED);
        } else {
            nextState = Math.min(nextState, Fragment.INITIALIZING);
        }
    }
    moveToState(f, nextState, f.getNextTransition(), f.getNextTransitionStyle(), false);
}
```

```java
public void addFragment(Fragment fragment, boolean moveToStateNow) {
    if (DEBUG) Log.v(TAG, "add: " + fragment);
    **makeActive(fragment);// 这是主要区别**
    if (!fragment.mDetached) {
        if (mAdded.contains(fragment)) {
            throw new IllegalStateException("Fragment already added: " + fragment);
        }
        synchronized (mAdded) {
            mAdded.add(fragment);
        }
        fragment.mAdded = true;
        fragment.mRemoving = false;
        if (fragment.mView == null) {
            fragment.mHiddenChanged = false;
        }
        if (isMenuAvailable(fragment)) {
            mNeedMenuInvalidate = true;
        }
        if (moveToStateNow) {
            moveToState(fragment);
        }
    }
}
```

```java
public void attachFragment(Fragment fragment) {
    if (DEBUG) Log.v(TAG, "attach: " + fragment);
    if (fragment.mDetached) {
        fragment.mDetached = false;
        if (!fragment.mAdded) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
            if (DEBUG) Log.v(TAG, "add from attach: " + fragment);
            synchronized (mAdded) {
                mAdded.add(fragment);
            }
            fragment.mAdded = true;
            if (isMenuAvailable(fragment)) {
                mNeedMenuInvalidate = true;
            }
        }
    }
}
```

# Show和hide Fragment区别

首先，更新mHidden标记。

```java
/**
 * Marks a fragment as hidden to be later animated in with
 * {@link #completeShowHideFragment(Fragment)}.
 *
 * @param fragment The fragment to be shown.
 */
public void hideFragment(Fragment fragment) {
    if (DEBUG) Log.v(TAG, "hide: " + fragment);
    if (!fragment.mHidden) {
        fragment.mHidden = true;
        // Toggle hidden changed so that if a fragment goes through show/hide/show
        // it doesn't go through the animation.
        fragment.mHiddenChanged = !fragment.mHiddenChanged;
    }
}

/**
 * Marks a fragment as shown to be later animated in with
 * {@link #completeShowHideFragment(Fragment)}.
 *
 * @param fragment The fragment to be shown.
 */
public void showFragment(Fragment fragment) {
    if (DEBUG) Log.v(TAG, "show: " + fragment);
    if (fragment.mHidden) {
        fragment.mHidden = false;
        // Toggle hidden changed so that if a fragment goes through show/hide/show
        // it doesn't go through the animation.
        fragment.mHiddenChanged = !fragment.mHiddenChanged;
    }
}
```

故事并没有就此结束，看下这个标记如何使用。

首先，hidden后，Fragment下的View为Gone，不贡献视图。

```java
androidx.fragment.app.FragmentStateManager#moveToExpectedState
if (FragmentManager.USE_STATE_MANAGER && mFragment.mHiddenChanged) {
    if (mFragment.mView != null && mFragment.mContainer != null) {
        // Get the controller and enqueue the show/hide
        SpecialEffectsController controller = SpecialEffectsController
                .getOrCreateController(mFragment.mContainer,
                        mFragment.getParentFragmentManager());
        if (mFragment.mHidden) {
            controller.enqueueHide(this);
        } else {
            controller.enqueueShow(this);
        }
    }
    if (mFragment.mFragmentManager != null) {
        mFragment.mFragmentManager.invalidateMenuForFragment(mFragment);
    }
    mFragment.mHiddenChanged = false;
    mFragment.onHiddenChanged(mFragment.mHidden);
}
```

其次，hidden后，Fragment不再贡献菜单项

```java
androidx.fragment.app.FragmentManager#dispatchPrepareOptionsMenu
boolean dispatchPrepareOptionsMenu(@NonNull Menu menu) {
    if (mCurState < Fragment.CREATED) {
        return false;
    }
    boolean show = false;
    for (Fragment f : mFragmentStore.getFragments()) {
        if (f != null) {
            if (isParentMenuVisible(f) && f.performPrepareOptionsMenu(menu)) {
                show = true;
            }
        }
    }
    return show;
}

androidx.fragment.app.Fragment#performPrepareOptionsMenu
boolean performPrepareOptionsMenu(@NonNull Menu menu) {
    boolean show = false;
    if (!mHidden) {
        if (mHasMenu && mMenuVisible) {
            show = true;
            onPrepareOptionsMenu(menu);
        }
        show |= mChildFragmentManager.dispatchPrepareOptionsMenu(menu);
    }
    return show;
}
```

显隐切换完成后，androidx.fragment.app.Fragment#onHiddenChanged会得到通知调用。

```java
androidx.fragment.app.FragmentManager#completeShowHideFragment
androidx.fragment.app.FragmentStateManager#moveToExpectedState

```

# FragmentManager 派发生命周期方法

```java
public class FragmentActivity extends ComponentActivity {
    final FragmentController mFragments = FragmentController.createController(
					new HostCallbacks());
}
```

mFragments 负责Activity 生命周期方法，派发给FragmentManager，再遍历逐个派发给旗下的Fragment。

```java
/**
 * Perform initialization of all fragments.
 */
@SuppressWarnings("deprecation")
@Override
protected void onCreate(@Nullable Bundle savedInstanceState) {
    mFragments.attachHost(null /*parent*/);

    if (savedInstanceState != null) {
        Parcelable p = savedInstanceState.getParcelable(FRAGMENTS_TAG);
        mFragments.restoreSaveState(p);
    }

    super.onCreate(savedInstanceState);

    mFragmentLifecycleRegistry.handleLifecycleEvent(Lifecycle.Event.ON_CREATE);
    mFragments.dispatchCreate();
}
```

```java
/**
 * Moves all Fragments managed by the controller's FragmentManager
 * into the create state.
 * <p>Call when Fragments should be created.
 *
 * @see Fragment#onCreate(Bundle)
 */
public void dispatchCreate() {
    mHost.mFragmentManager.dispatchCreate();
}
```

dispatchCreate 终导致FragmentManagerImpl的当前状态`mCurState` 变更。

```java
androidx.fragment.app.FragmentManager#dispatchCreate
void dispatchCreate() {
    mStateSaved = false;
    mStopped = false;
    mNonConfig.setIsStateSaved(false);
    dispatchStateChange(Fragment.CREATED);
}

androidx.fragment.app.FragmentManager#dispatchStateChange
private void dispatchStateChange(int nextState) {
    try {
        mExecutingActions = true;
        mFragmentStore.dispatchStateChange(nextState);
        moveToState(nextState, false);
        if (USE_STATE_MANAGER) {
            Set<SpecialEffectsController> controllers = collectAllSpecialEffectsController();
            for (SpecialEffectsController controller : controllers) {
                controller.forceCompleteAllOperations();
            }
        }
    } finally {
        mExecutingActions = false;
    }
    execPendingActions(true);
}

// 同步生命周期状态给mActive内fragment集合
void dispatchStateChange(int state) {
    for (FragmentStateManager fragmentStateManager : mActive.values()) {
        if (fragmentStateManager != null) {
            fragmentStateManager.setFragmentManagerState(state);
        }
    }
}
```

```java
androidx.fragment.app.FragmentManager#moveToState(int, boolean)
/**
 * Changes the state of the fragment manager to {@code newState}. If the fragment manager
 * changes state or {@code always} is {@code true}, any fragments within it have their
 * states updated as well.
 *
 * @param newState The new state for the fragment manager
 * @param always If {@code true}, all fragments update their state, even
 *               if {@code newState} matches the current fragment manager's state.
 */
void moveToState(int newState, boolean always) {
    if (mHost == null && newState != Fragment.INITIALIZING) {
        throw new IllegalStateException("No activity");
    }

    if (!always && newState == mCurState) {
        return;
    }

    mCurState = newState;

    if (USE_STATE_MANAGER) {
        mFragmentStore.moveToExpectedState();
    } else {
        // Must add them in the proper order. mActive fragments may be out of order
        for (Fragment f : mFragmentStore.getFragments()) {
            moveFragmentToExpectedState(f);
        }

        // Now iterate through all active fragments. These will include those that are removed
        // and detached.
        for (FragmentStateManager fragmentStateManager :
                mFragmentStore.getActiveFragmentStateManagers()) {
            Fragment f = fragmentStateManager.getFragment();
            if (!f.mIsNewlyAdded) {
                moveFragmentToExpectedState(f);
            }
            boolean beingRemoved = f.mRemoving && !f.isInBackStack();
            if (beingRemoved) {
                mFragmentStore.makeInactive(fragmentStateManager);
            }
        }
    }

    startPendingDeferredFragments();

    if (mNeedMenuInvalidate && mHost != null && mCurState == Fragment.RESUMED) {
        mHost.onSupportInvalidateOptionsMenu();
        mNeedMenuInvalidate = false;
    }
}
```

```java
androidx.fragment.app.FragmentManager#moveFragmentToExpectedState
void moveToExpectedState() {
    // Must add them in the proper order. mActive fragments may be out of order
    for (Fragment f : mAdded) {
        FragmentStateManager fragmentStateManager = mActive.get(f.mWho);
        if (fragmentStateManager != null) {
            fragmentStateManager.moveToExpectedState();
        }
    }

    // Now iterate through all active fragments. These will include those that are removed
    // and detached.
    for (FragmentStateManager fragmentStateManager : mActive.values()) {
        if (fragmentStateManager != null) {
            fragmentStateManager.moveToExpectedState();

            Fragment f = fragmentStateManager.getFragment();
            boolean beingRemoved = f.mRemoving && !f.isInBackStack();
            if (beingRemoved) {
                makeInactive(fragmentStateManager);
            }
        }
    }
}
```

注意只有在mActive集合的Fragment才可以`moveToExpectedState` ，attach阶段的Fragment 并未进入此集合，add之后才进入，此后Fragment接收生命周期方法派发。

# BackStack

```java
/**
 * Implementation of {@link FragmentManagerImpl.OpGenerator}.
 * This operation is added to the list of pending actions during {@link #commit()}, and
 * will be executed on the UI thread to run this FragmentTransaction.
 *
 * @param records Modified to add this BackStackRecord
 * @param isRecordPop Modified to add a false (this isn't a pop)
 * @return true always because the records and isRecordPop will always be changed
 */
@Override
public boolean generateOps(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop) {
    if (FragmentManagerImpl.DEBUG) {
        Log.v(TAG, "Run: " + this);
    }

    records.add(this);
    isRecordPop.add(false);
    if (mAddToBackStack) {
        mManager.addBackStackState(this);
    }
    return true;
}
```

```java
void addBackStackState(BackStackRecord state) {
    if (mBackStack == null) {
        mBackStack = new ArrayList<BackStackRecord>();
    }
    mBackStack.add(state);
}
```

把当前backStackRecord 即事务加入mBackStack 集合。此处可以猜想：当接收back事件时，activity先派发back事件给FragmentManager，如果mBackStack存在元素，则将列表末尾的事务revert，并移出集合，指针向左偏移一位，并直接返回。如果FragmentManager不回应back event，那么由Activity继续回应。我们看下具体实现是否这样。

```java
androidx.activity.ComponentActivity#onBackPressed
/**
 * Called when the activity has detected the user's press of the back
 * key. The {@link #getOnBackPressedDispatcher() OnBackPressedDispatcher} will be given a
 * chance to handle the back button before the default behavior of
 * {@link android.app.Activity#onBackPressed()} is invoked.
 *
 * @see #getOnBackPressedDispatcher()
 */
@Override
@MainThread
public void onBackPressed() {
    mOnBackPressedDispatcher.onBackPressed();
}

private final OnBackPressedDispatcher mOnBackPressedDispatcher =
      new OnBackPressedDispatcher(new Runnable() {
          @Override
          public void run() {
              ComponentActivity.super.onBackPressed();
          }
      });

androidx.activity.OnBackPressedDispatcher#onBackPressed
/**
 * Trigger a call to the currently added {@link OnBackPressedCallback callbacks} in reverse
 * order in which they were added. Only if the most recently added callback is not
 * {@link OnBackPressedCallback#isEnabled() enabled}
 * will any previously added callback be called.
 * <p>
 * It is strongly recommended to call {@link #hasEnabledCallbacks()} prior to calling
 * this method to determine if there are any enabled callbacks that will be triggered
 * by this method as calling this method.
 */
@MainThread
public void onBackPressed() {
    Iterator<OnBackPressedCallback> iterator =
            mOnBackPressedCallbacks.descendingIterator();
    while (iterator.hasNext()) {
        OnBackPressedCallback callback = iterator.next();
        if (callback.isEnabled()) {
						// 如果回调回应back事件，则返回
            callback.handleOnBackPressed();
            return;
        }
    }
		// FragmentManager 没回应，则继续走这里
    if (mFallbackOnBackPressed != null) {
				// 为OnBackPressedDispatcher初始化时，传入的回调方法，其实现是
				// 调用Activity的onBackPressed()
        mFallbackOnBackPressed.run();
    }
}

```

FragmentManager如何注册callback

→androidx.fragment.app.FragmentActivity#init

→androidx.fragment.app.FragmentController#attachHost

→androidx.fragment.app.FragmentManager#attachController

```java
@SuppressWarnings("deprecation")
@SuppressLint("SyntheticAccessor")
void attachController(@NonNull FragmentHostCallback<?> host,
        @NonNull FragmentContainer container, @Nullable final Fragment parent) {
    if (mHost != null) throw new IllegalStateException("Already attached");
    mHost = host;
    mContainer = container;
    mParent = parent;

    // Add a FragmentOnAttachListener to the parent fragment / host to support
    // backward compatibility with the deprecated onAttachFragment() APIs
    if (mParent != null) {
        addFragmentOnAttachListener(new FragmentOnAttachListener() {
            @SuppressWarnings("deprecation")
            @Override
            public void onAttachFragment(@NonNull FragmentManager fragmentManager,
                    @NonNull Fragment fragment) {
                parent.onAttachFragment(fragment);
            }
        });
    } else if (host instanceof FragmentOnAttachListener) {
        addFragmentOnAttachListener((FragmentOnAttachListener) host);
    }

    if (mParent != null) {
        // Since the callback depends on us being the primary navigation fragment,
        // update our callback now that we have a parent so that we have the correct
        // state by default
        updateOnBackPressedCallbackEnabled();
    }
    // Set up the OnBackPressedCallback
    if (host instanceof OnBackPressedDispatcherOwner) {
        OnBackPressedDispatcherOwner dispatcherOwner = ((OnBackPressedDispatcherOwner) host);
        mOnBackPressedDispatcher = dispatcherOwner.getOnBackPressedDispatcher();
        LifecycleOwner owner = parent != null ? parent : dispatcherOwner;
        mOnBackPressedDispatcher.addCallback(owner, mOnBackPressedCallback);
    }
		// 代码省略
}
```

```java
private final OnBackPressedCallback mOnBackPressedCallback =
    new OnBackPressedCallback(false) {
        @Override
        public void handleOnBackPressed() {
            FragmentManager.this.handleOnBackPressed();
        }
    };
```

```java
androidx.fragment.app.FragmentManager#handleOnBackPressed
@SuppressWarnings("WeakerAccess") /* synthetic access */
void handleOnBackPressed() {
    // First, execute any pending actions to make sure we're in an
    // up to date view of the world just in case anyone is queuing
    // up transactions that change the back stack then immediately
    // calling onBackPressed()
    execPendingActions(true);
    if (mOnBackPressedCallback.isEnabled()) {
        // We still have a back stack, so we can pop
        popBackStackImmediate();
    } else {
        // Sigh. Due to FragmentManager's asynchronicity, we can
        // get into cases where we *think* we can handle the back
        // button but because of frame perfect dispatch, we fell
        // on our face. Since our callback is disabled, we can
        // re-trigger the onBackPressed() to dispatch to the next
        // enabled callback
        mOnBackPressedDispatcher.onBackPressed();
    }
}
```

看fragmentManager.popBackStackImmediate()

```java
@Override
public boolean popBackStackImmediate() {
    checkStateLoss();
    return popBackStackImmediate(null, -1, 0);
}
```

```java
/**
 * Used by all public popBackStackImmediate methods, this executes pending transactions and
 * returns true if the pop action did anything, regardless of what other pending
 * transactions did.
 *
 * @return true if the pop operation did anything or false otherwise.
 */
private boolean popBackStackImmediate(String name, int id, int flags) {
    execPendingActions();
    ensureExecReady(true);

    if (mPrimaryNav != null // We have a primary nav fragment
            && id < 0 // No valid id (since they're local)
            && name == null) { // no name to pop to (since they're local)
        final FragmentManager childManager = mPrimaryNav.getChildFragmentManager();
        if (childManager.popBackStackImmediate()) {
            // We did something, just not to this specific FragmentManager. Return true.
            return true;
        }
    }

    boolean executePop = popBackStackState(mTmpRecords, mTmpIsPop, name, id, flags);
    if (executePop) {
        mExecutingActions = true;
        try {
            removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
        } finally {
            cleanupExec();
        }
    }

    updateOnBackPressedCallbackEnabled();
    doPendingDeferredStart();
    burpActive();
    return executePop;
}
```

popBackStackState获取回退事务集合，交由removeRedundantOperationsAndExecute执行事务回退。

```java
boolean popBackStackState(ArrayList<BackStackRecord> records, ArrayList<Boolean> isRecordPop,
                              String name, int id, int flags) {
    if (mBackStack == null) {
        return false;
    }
		// 无参数回退，只回退一个事务
    if (name == null && id < 0 && (flags & POP_BACK_STACK_INCLUSIVE) == 0) {
        int last = mBackStack.size() - 1;
        if (last < 0) {
            return false;
        }
        records.add(mBackStack.remove(last));
        isRecordPop.add(true);
    } else {
				// 指定id或者backStack name回退，可以一次回退多个事务。
        int index = -1;
        if (name != null || id >= 0) {
            // If a name or ID is specified, look for that place in
            // the stack.
            index = mBackStack.size()-1;
            while (index >= 0) {
                BackStackRecord bss = mBackStack.get(index);
                if (name != null && name.equals(bss.getName())) {
                    break;
                }
                if (id >= 0 && id == bss.mIndex) {
                    break;
                }
                index--;
            }
            if (index < 0) {
                return false;
            }
            if ((flags&POP_BACK_STACK_INCLUSIVE) != 0) {
                index--;
                // Consume all following entries that match.
                while (index >= 0) {
                    BackStackRecord bss = mBackStack.get(index);
                    if ((name != null && name.equals(bss.getName()))
                            || (id >= 0 && id == bss.mIndex)) {
                        index--;
                        continue;
                    }
                    break;
                }
            }
        }
        if (index == mBackStack.size()-1) {
            return false;
        }
        for (int i = mBackStack.size() - 1; i > index; i--) {
            records.add(mBackStack.remove(i));
            isRecordPop.add(true);
        }
    }
    return true;
}
```

事务从mBackStack回退栈管理集合移出，并加入records用于执行事务，对应的isRecordPop元素设置为true，标记位回退事务，最后执行逆向操作，实现回退。

→androidx.fragment.app.FragmentManagerImpl#removeRedundantOperationsAndExecute

→androidx.fragment.app.FragmentManagerImpl#executeOpsTogether

→androidx.fragment.app.FragmentManager#executeOps

→androidx.fragment.app.BackStackRecord#executePopOps

```java
androidx.fragment.app.FragmentManager#executeOps
/**
 * Run the operations in the BackStackRecords, either to push or pop.
 *
 * @param records The list of records whose operations should be run.
 * @param isRecordPop The direction that these records are being run.
 * @param startIndex The index of the first entry in records to run.
 * @param endIndex One past the index of the final entry in records to run.
 */
private static void executeOps(@NonNull ArrayList<BackStackRecord> records,
        @NonNull ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    for (int i = startIndex; i < endIndex; i++) {
        final BackStackRecord record = records.get(i);
        final boolean isPop = isRecordPop.get(i);
        if (isPop) {
            record.bumpBackStackNesting(-1);
            // Only execute the add operations at the end of
            // all transactions.
            boolean moveToState = i == (endIndex - 1);
            record.executePopOps(moveToState);
        } else {
            record.bumpBackStackNesting(1);
            record.executeOps();
        }
    }
}

androidx.fragment.app.BackStackRecord#executePopOps
/**
 * Reverses the execution of the operations within this transaction. The Fragment states will
 * only be modified if reordering is not allowed.
 *
 * @param moveToState {@code true} if added fragments should be moved to their final state
 *                    in ordered transactions
 */
void executePopOps(boolean moveToState) {
    for (int opNum = mOps.size() - 1; opNum >= 0; opNum--) {
        final Op op = mOps.get(opNum);
        Fragment f = op.mFragment;
        if (f != null) {
            f.setPopDirection(true);
            f.setAnimations(op.mEnterAnim, op.mExitAnim, op.mPopEnterAnim, op.mPopExitAnim);
            f.setNextTransition(FragmentManager.reverseTransit(mTransition));
            // Reverse the target and source names for pop operations
            f.setSharedElementNames(mSharedElementTargetNames, mSharedElementSourceNames);
        }
        switch (op.mCmd) {
            case OP_ADD:
                mManager.setExitAnimationOrder(f, true);
                mManager.removeFragment(f);
                break;
            case OP_REMOVE:
                mManager.addFragment(f);
                break;
            case OP_HIDE:
                mManager.showFragment(f);
                break;
            case OP_SHOW:
                mManager.setExitAnimationOrder(f, true);
                mManager.hideFragment(f);
                break;
            case OP_DETACH:
                mManager.attachFragment(f);
                break;
            case OP_ATTACH:
                mManager.setExitAnimationOrder(f, true);
                mManager.detachFragment(f);
                break;
            case OP_SET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(null);
                break;
            case OP_UNSET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(f);
                break;
            case OP_SET_MAX_LIFECYCLE:
                mManager.setMaxLifecycle(f, op.mOldMaxState);
                break;
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.mCmd);
        }
        if (!mReorderingAllowed && op.mCmd != OP_REMOVE && f != null) {
            if (!FragmentManager.USE_STATE_MANAGER) {
                mManager.moveFragmentToExpectedState(f);
            }
        }
    }
    if (!mReorderingAllowed && moveToState && !FragmentManager.USE_STATE_MANAGER) {
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

# BackPressedCallback注册和反注册

注册和反注册有些值得揣摩，它和以往的模式略有不同。一般而言，注册和反注册都是提供一对API，add & remove 或者 register & unregister。但这里mOnBackPressedDispatcher只提供addCallback，反注册是mOnBackPressedCallback自己调用cancel 方法实现。

## 注册

```java
androidx.fragment.app.FragmentManager#attachController
@SuppressLint("SyntheticAccessor")
void attachController(@NonNull FragmentHostCallback<?> host,
        @NonNull FragmentContainer container, @Nullable final Fragment parent) {
    if (mHost != null) throw new IllegalStateException("Already attached");
    mHost = host;
    mContainer = container;
    mParent = parent;

    // Set up the OnBackPressedCallback
    if (host instanceof OnBackPressedDispatcherOwner) {
        OnBackPressedDispatcherOwner dispatcherOwner = ((OnBackPressedDispatcherOwner) host);
        mOnBackPressedDispatcher = dispatcherOwner.getOnBackPressedDispatcher();
        LifecycleOwner owner = parent != null ? parent : dispatcherOwner;
				// 注册
        mOnBackPressedDispatcher.addCallback(owner, mOnBackPressedCallback);
    }
}
```

## 反注册

```java
void dispatchDestroy() {
    mDestroyed = true;
    execPendingActions(true);
    endAnimatingAwayFragments();
    dispatchStateChange(Fragment.INITIALIZING);
    mHost = null;
    mContainer = null;
    mParent = null;
    if (mOnBackPressedDispatcher != null) {
        // mOnBackPressedDispatcher can hold a reference to the host
        // so we need to null it out to prevent memory leaks
				// 反注册
        mOnBackPressedCallback.remove();
        mOnBackPressedDispatcher = null;
    }
}
```

注册实现

```java
androidx.activity.OnBackPressedDispatcher#addCallback(androidx.lifecycle.LifecycleOwner, androidx.activity.OnBackPressedCallback)
/**
 * Receive callbacks to a new {@link OnBackPressedCallback} when the given
 * {@link LifecycleOwner} is at least {@link Lifecycle.State#STARTED started}.
 * <p>
 * This will automatically call {@link #addCallback(OnBackPressedCallback)} and
 * remove the callback as the lifecycle state changes.
 * As a corollary, if your lifecycle is already at least
 * {@link Lifecycle.State#STARTED started}, calling this method will result in an immediate
 * call to {@link #addCallback(OnBackPressedCallback)}.
 * <p>
 * When the {@link LifecycleOwner} is {@link Lifecycle.State#DESTROYED destroyed}, it will
 * automatically be removed from the list of callbacks. The only time you would need to
 * manually call {@link OnBackPressedCallback#remove()} is if
 * you'd like to remove the callback prior to destruction of the associated lifecycle.
 *
 * <p>
 * If the Lifecycle is already {@link Lifecycle.State#DESTROYED destroyed}
 * when this method is called, the callback will not be added.
 *
 * @param owner The LifecycleOwner which controls when the callback should be invoked
 * @param onBackPressedCallback The callback to add
 *
 * @see #onBackPressed()
 */
@SuppressLint("LambdaLast")
@MainThread
public void addCallback(@NonNull LifecycleOwner owner,
        @NonNull OnBackPressedCallback onBackPressedCallback) {
    Lifecycle lifecycle = owner.getLifecycle();
    if (lifecycle.getCurrentState() == Lifecycle.State.DESTROYED) {
        return;
    }
		// 注册 action2
    onBackPressedCallback.addCancellable(
            new LifecycleOnBackPressedCancellable(lifecycle, onBackPressedCallback));
}
```

onBackPressedCallback的注册委托给LifecycleOnBackPressedCancellable，同时，持有LifecycleOnBackPressedCancellable的实例引用，但反注册时，onBackPressedCallback调用cancel方法，通知LifecycleOnBackPressedCancellable实例反注册。也就是A 向C注册，其实是委托给B实现注册，反注册通过A持有B的引用，也由B负责完成。

```java
private class LifecycleOnBackPressedCancellable implements LifecycleEventObserver,
        Cancellable {
    private final Lifecycle mLifecycle;
    private final OnBackPressedCallback mOnBackPressedCallback;

    @Nullable
    private Cancellable mCurrentCancellable;

    LifecycleOnBackPressedCancellable(@NonNull Lifecycle lifecycle,
            @NonNull OnBackPressedCallback onBackPressedCallback) {
        mLifecycle = lifecycle;
        mOnBackPressedCallback = onBackPressedCallback;
				// 注册 action1
        lifecycle.addObserver(this);
    }

    @Override
    public void onStateChanged(@NonNull LifecycleOwner source,
            @NonNull Lifecycle.Event event) {
        if (event == Lifecycle.Event.ON_START) {
						// 实际注册
            mCurrentCancellable = addCancellableCallback(mOnBackPressedCallback);
        } else if (event == Lifecycle.Event.ON_STOP) {
            // Should always be non-null
            if (mCurrentCancellable != null) {
                mCurrentCancellable.cancel();
            }
        } else if (event == Lifecycle.Event.ON_DESTROY) {
            cancel();
        }
    }

    @Override
    public void cancel() {
				// 反注册 action1
        mLifecycle.removeObserver(this);
				// 反注册 action2
        mOnBackPressedCallback.removeCancellable(this);
        if (mCurrentCancellable != null) {
						// 实际反注册
            mCurrentCancellable.cancel();
            mCurrentCancellable = null;
        }
    }
}

androidx.activity.OnBackPressedDispatcher#addCancellableCallback
/**
 * Internal implementation of {@link #addCallback(OnBackPressedCallback)} that gives
 * access to the {@link Cancellable} that specifically removes this callback from
 * the dispatcher without relying on {@link OnBackPressedCallback#remove()} which
 * is what external developers should be using.
 *
 * @param onBackPressedCallback The callback to add
 * @return a {@link Cancellable} which can be used to {@link Cancellable#cancel() cancel}
 * the callback and remove it from the set of OnBackPressedCallbacks.
 */
@SuppressWarnings("WeakerAccess") /* synthetic access */
@MainThread
@NonNull
Cancellable addCancellableCallback(@NonNull OnBackPressedCallback onBackPressedCallback) {
    // start后，实际注册进callback列表
		mOnBackPressedCallbacks.add(onBackPressedCallback);
		// 建立OnBackPressedCancellable实例 与onBackPressedCallback的双向引用并返回
    OnBackPressedCancellable cancellable = new OnBackPressedCancellable(onBackPressedCallback);
    onBackPressedCallback.addCancellable(cancellable);
    return cancellable;
}

private class OnBackPressedCancellable implements Cancellable {
    private final OnBackPressedCallback mOnBackPressedCallback;
    OnBackPressedCancellable(OnBackPressedCallback onBackPressedCallback) {
        mOnBackPressedCallback = onBackPressedCallback;
    }

    @Override
    public void cancel() {
				// stop后，实际反注册
        mOnBackPressedCallbacks.remove(mOnBackPressedCallback);
				// 断开引用
        mOnBackPressedCallback.removeCancellable(this);
    }
}
```

