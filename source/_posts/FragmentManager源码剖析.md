---
title: FragmentManager源码剖析
tags: 'Fragment, FragmentManager源码'
date: 2021-09-16 21:55:07
---



# 总结

1. 使用Fragment的Activity需要继承`FragmentActivity` 这里有派发生命周期方法给FragmentManager，再派发给旗下Fragment。
2. 加入Fragment时，需要判断`savedInstanceState == null` ，避免重复创建实例。
3. add Fragment，初识时，其生命周期方法会被升级到started 状态，什么时候resume？TODO: 待确定完善
4. detach fragment，其生命周期方法会被降级到onDestroyView。
5. remove fragment，其生命周期方法会被降级到onDestroy。
6. show or hide fragment，生命周期方法不会回调，只是切换fragment 根view的可见性。



# 注意事项

1. ```java
   beginTransaction
   ```

   > Note: A fragment transaction can only be created/committed prior to an activity saving its state. If you try to commit a transaction after FragmentActivity.onSaveInstanceState() (and prior to a following FragmentActivity.onStart or FragmentActivity.onResume(), you will get an error. This is because the framework takes care of saving your current fragments in the state, and if changes are made after the state is saved then they will be lost.
   > 事务需要在activty 保存其状态前创建&提交

2. ```java
   executePendingTransactions
   ```

   > After a FragmentTransaction is committed with FragmentTransaction.commit(), it is scheduled to be executed asynchronously on the process's main thread. If you want to immediately executing any such pending operations, you can call this function (only from the main thread) to do so. 
   >
   > commit 是在主线程异步提交，如果要立即执行这些挂起的操作，可以从主线程调用executePendingTransactions
   >
   > If you are committing a single transaction that does not modify the fragment back stack, strongly consider using FragmentTransaction.commitNow() instead. This can help avoid unwanted side effects when other code in your app has pending committed transactions that expect different timing.
   >
   > 如果是提交不修改fragment back stack的事物，强烈推荐使用commitNow

# Sample codes

```java
public class ExampleActivity extends AppCompatActivity {
    public ExampleActivity() {
        super(R.layout.example_activity);
    }
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        if (savedInstanceState == null) {
            getSupportFragmentManager().beginTransaction()
                .setReorderingAllowed(true)
                .add(R.id.fragment_container_view, ExampleFragment.class)
								.addToBackStack("name") // name can be null
                .commit();
        }
    }
}
```

Note: the fragment transaction is only created when savedInstanceState is null. This is to ensure that the fragment is added only once, when the activity is first created. When a configuration change occurs and the activity is recreated, savedInstanceState is no longer null, and the fragment does not need to be added a second time, as the fragment is automatically restored from the savedInstanceState.

Note: You should always use setReorderingAllowed(true) when performing a FragmentTransaction. For more information on reordered transactions, see Fragment transactions.

```java
@NonNull
@Override
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);
}
```

```java
/**
 * Sets whether or not to allow optimizing operations within and across
 * transactions. This will remove redundant operations, eliminating
 * operations that cancel. For example, if two transactions are executed
 * together, one that adds a fragment A and the next replaces it with fragment B,
 * the operations will cancel and only fragment B will be added. That means that
 * fragment A may not go through the creation/destruction lifecycle.
 * <p>
 * The side effect of removing redundant operations is that fragments may have state changes
 * out of the expected order. For example, one transaction adds fragment A,
 * a second adds fragment B, then a third removes fragment A. Without removing the redundant
 * operations, fragment B could expect that while it is being created, fragment A will also
 * exist because fragment A will be removed after fragment B was added.
 * With removing redundant operations, fragment B cannot expect fragment A to exist when
 * it has been created because fragment A's add/remove will be optimized out.
 * <p>
 * It can also reorder the state changes of Fragments to allow for better Transitions.
 * Added Fragments may have {@link Fragment#onCreate(Bundle)} called before replaced
 * Fragments have {@link Fragment#onDestroy()} called.
 * <p>
 * {@link Fragment#postponeEnterTransition()} requires {@code setReorderingAllowed(true)}.
 * <p>
 * The default is {@code false}.
 *
 * @param reorderingAllowed {@code true} to enable optimizing out redundant operations
 *                          or {@code false} to disable optimizing out redundant
 *                          operations on this transaction.
 */
@NonNull
public FragmentTransaction setReorderingAllowed(boolean reorderingAllowed) {
    mReorderingAllowed = reorderingAllowed;
    return this;
}
```

传入fragment实例，将对fragment的增删换，显示隐藏等操作封装成op，并加入op列表集合，作为一个操作事物。

```java
/**
 * Add this transaction to the back stack.  This means that the transaction
 * will be remembered after it is committed, and will reverse its operation
 * when later popped off the stack.
 * <p>
 * {@link #setReorderingAllowed(boolean)} must be set to <code>true</code>
 * in the same transaction as addToBackStack() to allow the pop of that
 * transaction to be reordered.
 *
 * @param name An optional name for this back stack state, or null.
 */
@NonNull
public FragmentTransaction addToBackStack(@Nullable String name) {
    if (!mAllowAddToBackStack) {
        throw new IllegalStateException(
                "This FragmentTransaction is not allowed to be added to the back stack.");
    }
    mAddToBackStack = true;
    mName = name;
    return this;
}
```

addToBackStack更新mAddToBackStack标记位，并保存backStack名字，作为未来回退到此backStack的标示。

```java
int commitInternal(boolean allowStateLoss) {
    if (mCommitted) throw new IllegalStateException("commit already called");
    mCommitted = true;
    if (mAddToBackStack) {
        mIndex = mManager.allocBackStackIndex(this);
    } else {
        mIndex = -1;
    }
    mManager.enqueueAction(this, allowStateLoss);
    return mIndex;
}
```

如果加入backStack，则分配一个在backStack的索引，然后将此事务加入提交队列，等待主线程调度提交。

```java
/**
 * Adds an action to the queue of pending actions.
 *
 * @param action the action to add
 * @param allowStateLoss whether to allow loss of state information
 * @throws IllegalStateException if the activity has been destroyed
 */
public void enqueueAction(OpGenerator action, boolean allowStateLoss) {
    if (!allowStateLoss) {
        checkStateLoss();
    }
    synchronized (this) {
        if (mDestroyed || mHost == null) {
            if (allowStateLoss) {
                // This FragmentManager isn't attached, so drop the entire transaction.
                return;
            }
            throw new IllegalStateException("Activity has been destroyed");
        }
        if (mPendingActions == null) {
            mPendingActions = new ArrayList<>();
        }
        mPendingActions.add(action);
        scheduleCommit();
    }
}
```

BackStackRecord实现OpGenerator接口，先看下方法实现内容：

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

接下来看scheduleCommit的执行：将mPendingActions的BackStackRecord复制到records（即成员变量mTmpRecords）

```java
/**
 * Only call from main thread!
 */
public boolean execPendingActions() {
    ensureExecReady(true);// Init mTmpRecords & mTmpIsPop

    boolean didSomething = false;
    // Copy BackStackRecord in mPendingActions into mTmpRecords.
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
            removeRedundantOperationsAndExecute(mTmpRecords, mTmpIsPop);
        } finally {
            cleanupExec();
        }
        didSomething = true;
    }

    updateOnBackPressedCallbackEnabled();
    doPendingDeferredStart();
    burpActive();

    return didSomething;
}
```

```java
/**
 * Adds all records in the pending actions to records and whether they are add or pop
 * operations to isPop. After executing, the pending actions will be empty.
 *
 * @param records All pending actions will generate BackStackRecords added to this.
 *                This contains the transactions, in order, to execute.
 * @param isPop All pending actions will generate booleans to add to this. This contains
 *              an entry for each entry in records to indicate whether or not it is a
 *              pop action.
 */
private boolean generateOpsForPendingActions(ArrayList<BackStackRecord> records,
                                             ArrayList<Boolean> isPop) {
    boolean didSomething = false;
    synchronized (this) {
        if (mPendingActions == null || mPendingActions.size() == 0) {
            return false;
        }

        final int numActions = mPendingActions.size();
        for (int i = 0; i < numActions; i++) {
            didSomething |= mPendingActions.get(i).generateOps(records, isPop);
        }
        mPendingActions.clear();
        mHost.getHandler().removeCallbacks(mExecCommit);
    }
    return didSomething;
}
```

```java
/**
 * Remove redundant BackStackRecord operations and executes them. This method merges operations
 * of proximate records that allow reordering. See
 * {@link FragmentTransaction#setReorderingAllowed(boolean)}.
 * <p>
 * For example, a transaction that adds to the back stack and then another that pops that
 * back stack record will be optimized to remove the unnecessary operation.
 * <p>
 * Likewise, two transactions committed that are executed at the same time will be optimized
 * to remove the redundant operations as well as two pop operations executed together.
 *
 * @param records The records pending execution
 * @param isRecordPop The direction that these records are being run.
 */
private void removeRedundantOperationsAndExecute(ArrayList<BackStackRecord> records,
                                                 ArrayList<Boolean> isRecordPop) {
    if (records == null || records.isEmpty()) {
        return;
    }

    if (isRecordPop == null || records.size() != isRecordPop.size()) {
        throw new IllegalStateException("Internal error with the back stack records");
    }

    // Force start of any postponed transactions that interact with scheduled transactions:
    executePostponedTransaction(records, isRecordPop);

    final int numRecords = records.size();
    int startIndex = 0;
    for (int recordNum = 0; recordNum < numRecords; recordNum++) {
        final boolean canReorder = records.get(recordNum).mReorderingAllowed;
        if (!canReorder) {
            // execute all previous transactions
            if (startIndex != recordNum) {
                executeOpsTogether(records, isRecordPop, startIndex, recordNum);
            }
            // execute all pop operations that don't allow reordering together or
            // one add operation
            int reorderingEnd = recordNum + 1;
            if (isRecordPop.get(recordNum)) {
                while (reorderingEnd < numRecords
                        && isRecordPop.get(reorderingEnd)
                        && !records.get(reorderingEnd).mReorderingAllowed) {
                    reorderingEnd++;
                }
            }
            executeOpsTogether(records, isRecordPop, recordNum, reorderingEnd);
            startIndex = reorderingEnd;
            recordNum = reorderingEnd - 1;
        }
    }
		// Reorder is true as example code. Pay attention here.
    if (startIndex != numRecords) {
        executeOpsTogether(records, isRecordPop, startIndex, numRecords);
    }
}
```

默认允许重排序，看最后一行方法调用。

```java
/**
 * Executes a subset of a list of BackStackRecords, all of which either allow reordering or
 * do not allow ordering.
 * @param records A list of BackStackRecords that are to be executed
 * @param isRecordPop The direction that these records are being run.
 * @param startIndex The index of the first record in <code>records</code> to be executed
 * @param endIndex One more than the final record index in <code>records</code> to executed.
 */
private void executeOpsTogether(ArrayList<BackStackRecord> records,
                                ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    final boolean allowReordering = records.get(startIndex).mReorderingAllowed;
    boolean addToBackStack = false;
    if (mTmpAddedFragments == null) {
        mTmpAddedFragments = new ArrayList<>();
    } else {
        mTmpAddedFragments.clear();
    }
		// 复制已添加Fragment集合到临时变量
    mTmpAddedFragments.addAll(mAdded);
    Fragment oldPrimaryNav = getPrimaryNavigationFragment();
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
				// isPop 为false，
        if (!isPop) {
						// 这里将op替换为更原子的操作，如replace拆解为remove & add，并将ops内的fragment
						// 依照op的cmd命令，从mTmpAddedFragments add 或 remove op里的fragment，
            oldPrimaryNav = record.expandOps(mTmpAddedFragments, oldPrimaryNav);
        } else {
            oldPrimaryNav = record.trackAddedFragmentsInPop(mTmpAddedFragments, oldPrimaryNav);
        }
        addToBackStack = addToBackStack || record.mAddToBackStack;
    }
		// 这里clear了，难道add，attach，remove & detachFragment 不操作mAddedfragment集合变量么？
		// 是的，在这些方法里已经按参数增删fragment元素里。
    mTmpAddedFragments.clear();

    if (!allowReordering) {
        FragmentTransition.startTransitions(this, records, isRecordPop, startIndex, endIndex,
                false);
    }
		// 遍历BackStackRecord集合，将Fragment发送给FragmentManager管理，并更新Fragment标记。
    executeOps(records, isRecordPop, startIndex, endIndex);

    int postponeIndex = endIndex;
    if (allowReordering) {
        ArraySet<Fragment> addedFragments = new ArraySet<>();
				// 将mAdded集合内的Fragment推送至STARTED状态
        addAddedFragments(addedFragments);
				// 从后向前计算推迟Fragment的索引
        postponeIndex = postponePostponableTransactions(records, isRecordPop,
                startIndex, endIndex, addedFragments);
        makeRemovedFragmentsInvisible(addedFragments);
    }
		// 将非推迟的Fragment全部推送到当前FragmentManager的状态。
    if (postponeIndex != startIndex && allowReordering) {
        // need to run something now
        FragmentTransition.startTransitions(this, records, isRecordPop, startIndex,
                postponeIndex, true);
        moveToState(mCurState, true);
    }
		// 调用每个事务的提交监听回掉
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (isPop && record.mIndex >= 0) {
            freeBackStackIndex(record.mIndex);
            record.mIndex = -1;
        }
        record.runOnCommitRunnables();
    }
		// 通知backStack变更的监听回掉
    if (addToBackStack) {
        reportBackStackChanged();
    }
}
```

```java
/**
 * Run the operations in the BackStackRecords, either to push or pop.
 *
 * @param records The list of records whose operations should be run.
 * @param isRecordPop The direction that these records are being run.
 * @param startIndex The index of the first entry in records to run.
 * @param endIndex One past the index of the final entry in records to run.
 */
private static void executeOps(ArrayList<BackStackRecord> records,
                               ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
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
```

遍历事务集合，逐个执行事务

```java
/**
 * Executes the operations contained within this transaction. The Fragment states will only
 * be modified if optimizations are not allowed.
 */
void executeOps() {
    final int numOps = mOps.size();
    for (int opNum = 0; opNum < numOps; opNum++) {
        final Op op = mOps.get(opNum);
        final Fragment f = op.mFragment;
        if (f != null) {
            f.setNextTransition(mTransition, mTransitionStyle);
        }
        switch (op.mCmd) {
            case OP_ADD:
                f.setNextAnim(op.mEnterAnim);
                mManager.addFragment(f, false);
                break;
            case OP_REMOVE:
                f.setNextAnim(op.mExitAnim);
                mManager.removeFragment(f);
                break;
            case OP_HIDE:
                f.setNextAnim(op.mExitAnim);
                mManager.hideFragment(f);
                break;
            case OP_SHOW:
                f.setNextAnim(op.mEnterAnim);
                mManager.showFragment(f);
                break;
            case OP_DETACH:
                f.setNextAnim(op.mExitAnim);
                mManager.detachFragment(f);
                break;
            case OP_ATTACH:
                f.setNextAnim(op.mEnterAnim);
                mManager.attachFragment(f);
                break;
            case OP_SET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(f);
                break;
            case OP_UNSET_PRIMARY_NAV:
                mManager.setPrimaryNavigationFragment(null);
                break;
            case OP_SET_MAX_LIFECYCLE:
                mManager.setMaxLifecycle(f, op.mCurrentMaxState);
                break;
            default:
                throw new IllegalArgumentException("Unknown cmd: " + op.mCmd);
        }
        if (!mReorderingAllowed && op.mCmd != OP_ADD && f != null) {
            mManager.moveFragmentToExpectedState(f);
        }
    }
    if (!mReorderingAllowed) {
        // Added fragments are added at the end to comply with prior behavior.
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

addFragment & removeFragment 主要是操作mAdded fragment集合和mAdded & mRemoving标记

```java
public void addFragment(Fragment fragment, boolean moveToStateNow) {
        if (DEBUG) Log.v(TAG, "add: " + fragment);
				// 加入mActive Map
        makeActive(fragment);
        if (!fragment.mDetached) {
            if (mAdded.contains(fragment)) {
                throw new IllegalStateException("Fragment already added: " + fragment);
            }
						// 加入mAdded集合，并更新mAdded等标记位
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

attachFragment 与addFragment的差异： 无makeActive(fragment)，也没有moveToState(fragment)操作

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

showFrament & hideFragment 主要更新mHidden标记

```java
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

```java
// True if the fragment is in the list of added fragments.
boolean mAdded;

// If set this fragment is being removed from its activity.
boolean mRemoving;

// Set to true when the app has requested that this fragment be hidden
// from the user.
boolean mHidden;

// Set to true when the app has requested that this fragment be deactivated.
boolean mDetached;
```

```java
/**
 * Ensure that fragments that are added are moved to at least the CREATED state.
 * Any newly-added Views are inserted into {@code added} so that the Transaction can be
 * postponed with {@link Fragment#postponeEnterTransition()}. They will later be made
 * invisible (by setting their alpha to 0) if they have been removed when postponed.
 */
private void addAddedFragments(ArraySet<Fragment> added) {
    if (mCurState < Fragment.CREATED) {
        return;
    }
    // We want to leave the fragment in the started state
    final int state = Math.min(mCurState, Fragment.STARTED);
    final int numAdded = mAdded.size();
    for (int i = 0; i < numAdded; i++) {
        Fragment fragment = mAdded.get(i);
        if (fragment.mState < state) {
            moveToState(fragment, state, fragment.getNextAnim(), fragment.getNextTransition(),
                    false);
            if (fragment.mView != null && !fragment.mHidden && fragment.mIsNewlyAdded) {
                added.add(fragment);
            }
        }
    }
}
```

```java
void moveToState(@NonNull Fragment f, int newState) {
        FragmentStateManager fragmentStateManager = mFragmentStore.getFragmentStateManager(f.mWho);
        if (fragmentStateManager == null) {
            // Ideally, we only call moveToState() on active Fragments. However,
            // in restoreSaveState() we can call moveToState() on retained Fragments
            // just to clean them up without them ever being added to mActive.
            // For these cases, a brand new FragmentStateManager is enough.
            fragmentStateManager = new FragmentStateManager(mLifecycleCallbacksDispatcher,
                    mFragmentStore, f);
            // Only allow this FragmentStateManager to go up to CREATED at the most
            fragmentStateManager.setFragmentManagerState(Fragment.CREATED);
        }
        // When inflating an Activity view with a resource instead of using setContentView(), and
        // that resource adds a fragment using the <fragment> tag (i.e. from layout and in layout),
        // the fragment will move to the VIEW_CREATED state before the fragment manager
        // moves to CREATED. So when moving the fragment manager moves to CREATED and the
        // inflated fragment is already in VIEW_CREATED we need to move new state up from CREATED
        // to VIEW_CREATED. This avoids accidentally moving the fragment back down to CREATED
        // which would immediately destroy the Fragment's view. We rely on computeExpectedState()
        // to pull the state back down if needed.
        if (f.mFromLayout && f.mInLayout && f.mState == Fragment.VIEW_CREATED) {
            newState = Math.max(newState, Fragment.VIEW_CREATED);
        }
        newState = Math.min(newState, fragmentStateManager.computeExpectedState());
        if (f.mState <= newState) {
            // If we are moving to the same state, we do not need to give up on the animation.
            if (f.mState < newState && !mExitAnimationCancellationSignals.isEmpty()) {
                // The fragment is currently being animated...  but!  Now we
                // want to move our state back up.  Give up on waiting for the
                // animation and proceed from where we are.
                cancelExitAnimation(f);
            }
            switch (f.mState) {
                case Fragment.INITIALIZING:
                    if (newState > Fragment.INITIALIZING) {
                        fragmentStateManager.attach();
                    }
                    // fall through
                case Fragment.ATTACHED:
                    if (newState > Fragment.ATTACHED) {
                        fragmentStateManager.create();
                    }
                    // fall through
                case Fragment.CREATED:
                    // We want to unconditionally run this anytime we do a moveToState that
                    // moves the Fragment above INITIALIZING, including cases such as when
                    // we move from CREATED => CREATED as part of the case fall through above.
                    if (newState > Fragment.INITIALIZING) {
                        fragmentStateManager.ensureInflatedView();
                    }

                    if (newState > Fragment.CREATED) {
                        fragmentStateManager.createView();
                    }
                    // fall through
                case Fragment.VIEW_CREATED:
                    if (newState > Fragment.VIEW_CREATED) {
                        fragmentStateManager.activityCreated();
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState > Fragment.ACTIVITY_CREATED) {
                        fragmentStateManager.start();
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState > Fragment.STARTED) {
                        fragmentStateManager.resume();
                    }
            }
        } else if (f.mState > newState) {
            switch (f.mState) {
                case Fragment.RESUMED:
                    if (newState < Fragment.RESUMED) {
                        fragmentStateManager.pause();
                    }
                    // fall through
                case Fragment.STARTED:
                    if (newState < Fragment.STARTED) {
                        fragmentStateManager.stop();
                    }
                    // fall through
                case Fragment.ACTIVITY_CREATED:
                    if (newState < Fragment.ACTIVITY_CREATED) {
                        if (isLoggingEnabled(Log.DEBUG)) {
                            Log.d(TAG, "movefrom ACTIVITY_CREATED: " + f);
                        }
                        if (f.mView != null) {
                            // Need to save the current view state if not
                            // done already.
                            if (mHost.onShouldSaveFragmentState(f) && f.mSavedViewState == null) {
                                fragmentStateManager.saveViewState();
                            }
                        }
                    }
                    // fall through
                case Fragment.VIEW_CREATED:
                    if (newState < Fragment.VIEW_CREATED) {
                        FragmentAnim.AnimationOrAnimator anim = null;
                        if (f.mView != null && f.mContainer != null) {
                            // Stop any current animations:
                            f.mContainer.endViewTransition(f.mView);
                            f.mView.clearAnimation();
                            // If parent is being removed, no need to handle child animations.
                            if (!f.isRemovingParent()) {
                                if (mCurState > Fragment.INITIALIZING && !mDestroyed
                                        && f.mView.getVisibility() == View.VISIBLE
                                        && f.mPostponedAlpha >= 0) {
                                    anim = FragmentAnim.loadAnimation(mHost.getContext(),
                                            f, false, f.getPopDirection());
                                }
                                f.mPostponedAlpha = 0;
                                // Robolectric tests do not post the animation like a real device
                                // so we should keep up with the container and view in case the
                                // fragment view is destroyed before we can remove it.
                                ViewGroup container = f.mContainer;
                                View view = f.mView;
                                if (anim != null) {
                                    FragmentAnim.animateRemoveFragment(f, anim,
                                            mFragmentTransitionCallback);
                                }
                                container.removeView(view);
                                if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
                                    Log.v(FragmentManager.TAG, "Removing view " + view + " for "
                                            + "fragment " + f + " from container " + container);
                                }
                                // If the local container is different from the fragment
                                // container, that means onAnimationEnd was called, onDestroyView
                                // was dispatched and the fragment was already moved to state, so
                                // we should early return here instead of attempting to move to
                                // state again.
                                if (container != f.mContainer) {
                                    return;
                                }
                            }
                        }
                        // If a fragment has an exit animation (or transition), do not destroy
                        // its view immediately and set the state after animating
                        if (mExitAnimationCancellationSignals.get(f) == null) {
                            fragmentStateManager.destroyFragmentView();
                        }
                    }
                    // fall through
                case Fragment.CREATED:
                    if (newState < Fragment.CREATED) {
                        if (mExitAnimationCancellationSignals.get(f) != null) {
                            // We are waiting for the fragment's view to finish animating away.
                            newState = Fragment.CREATED;
                        } else {
                            fragmentStateManager.destroy();
                        }
                    }
                    // fall through
                case Fragment.ATTACHED:
                    if (newState < Fragment.ATTACHED) {
                        fragmentStateManager.detach();
                    }
            }
        }

        if (f.mState != newState) {
            if (isLoggingEnabled(Log.DEBUG)) {
                Log.d(TAG, "moveToState: Fragment state for " + f + " not updated inline; "
                        + "expected state " + newState + " found " + f.mState);
            }
            f.mState = newState;
        }
    }
```

将当前Fragment的状态沿着下方箭头方向升级（curState < newState），或降级(curState > newState)至newState，（想象状态值是一个等腰三角形，resumed在上方顶点，created & destroyed位于下方两端。从左顶点到上顶点是升级，上顶点到右顶点是降级）状态值如下，RESUMED状态值最大。当前newState参数值为STARTED。

```java
static final int INITIALIZING = 0;     // Not yet created.
static final int CREATED = 1;          // Created.
static final int ACTIVITY_CREATED = 2; // Fully created, not started.
static final int STARTED = 3;          // Created and started, not resumed.
static final int RESUMED = 4;          // Created started and resumed.
```

![https://developer.android.com/images/guide/fragments/fragment-view-lifecycle.png](https://developer.android.com/images/guide/fragments/fragment-view-lifecycle.png)

由上可见，Fragment 添加后，首先推送升级到STARTED状态。

# Difference between removeFragment and detachFragment

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

# Difference between addFragment & attachFragment

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

# ShowFragment and hideFragment 做了什么

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
/**
 * Fragments that have been shown or hidden don't have their visibility changed or
 * animations run during the {@link #showFragment(Fragment)} or {@link #hideFragment(Fragment)}
 * calls. After fragments are brought to their final state in
 * {@link #moveFragmentToExpectedState(Fragment)} the fragments that have been shown or
 * hidden must have their visibility changed and their animations started here.
 *
 * @param fragment The fragment with mHiddenChanged = true that should change its View's
 *                 visibility and start the show or hide animation.
 */
void completeShowHideFragment(final Fragment fragment) {
    if (fragment.mView != null) {
        AnimationOrAnimator anim = loadAnimation(fragment, fragment.getNextTransition(),
                !fragment.mHidden, fragment.getNextTransitionStyle());
        if (anim != null && anim.animator != null) {
            anim.animator.setTarget(fragment.mView);
            if (fragment.mHidden) {
                if (fragment.isHideReplaced()) {
                    fragment.setHideReplaced(false);
                } else {
                    final ViewGroup container = fragment.mContainer;
                    final View animatingView = fragment.mView;
                    container.startViewTransition(animatingView);
                    // Delay the actual hide operation until the animation finishes,
                    // otherwise the fragment will just immediately disappear
                    anim.animator.addListener(new AnimatorListenerAdapter() {
                        @Override
                        public void onAnimationEnd(Animator animation) {
                            container.endViewTransition(animatingView);
                            animation.removeListener(this);
                            if (fragment.mView != null && fragment.mHidden) {
                                fragment.mView.setVisibility(View.GONE);
                            }
                        }
                    });
                }
            } else {
                fragment.mView.setVisibility(View.VISIBLE);
            }
            anim.animator.start();
        } else {
            if (anim != null) {
                fragment.mView.startAnimation(anim.animation);
                anim.animation.start();
            }
            final int visibility = fragment.mHidden && !fragment.isHideReplaced()
                    ? View.GONE
                    : View.VISIBLE;
            fragment.mView.setVisibility(visibility);
            if (fragment.isHideReplaced()) {
                fragment.setHideReplaced(false);
            }
        }
    }
    if (fragment.mAdded && isMenuAvailable(fragment)) {
        mNeedMenuInvalidate = true;
    }
    fragment.mHiddenChanged = false;
    fragment.onHiddenChanged(fragment.mHidden);
}
```

其次，hidden后，Fragment不再贡献菜单项

```java
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
private void dispatchStateChange(int nextState) {
    try {
        mExecutingActions = true;
        moveToState(nextState, false);
    } finally {
        mExecutingActions = false;
    }
    execPendingActions();
}
```

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

    // Must add them in the proper order. mActive fragments may be out of order
		// 处理add & attach fragment的状态同步，两个API调用的fragment都加入mAdded集合
    final int numAdded = mAdded.size();
    for (int i = 0; i < numAdded; i++) {
        Fragment f = mAdded.get(i);
        moveFragmentToExpectedState(f);
    }

    // Now iterate through all active fragments. These will include those that are removed
    // and detached.
		// 处理remove & detach fragment的状态同步
    for (Fragment f : mActive.values()) {
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

注意`mActive.containsKey(f.mWho)`的判断，只有在mActive集合的Fragment才可以`moveToState` ，attach阶段的Fragment 并未进入此集合，add之后才进入，此后Fragment接收生命周期方法派发。

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

把当前backStackRecord 即事务加入mBackStack 集合。此处可以猜想：当接收back事件时，activity先派发back事件给FragmentManager，如果mBackStack存在元素，则将列表末尾的事务revert，并移出集合，指针向左偏移一位。我们看下具体实现是否这样。

```java
/**
 * Called when the activity has detected the user's press of the back
 * key.  The default implementation simply finishes the current activity,
 * but you can override this to do whatever you want.
 */
public void onBackPressed() {
    if (mActionBar != null && mActionBar.collapseActionView()) {
        return;
    }

    FragmentManager fragmentManager = mFragments.getFragmentManager();

    if (!fragmentManager.isStateSaved() && fragmentManager.popBackStackImmediate()) {
        return;
    }
    if (!isTaskRoot()) {
        // If the activity is not the root of the task, allow finish to proceed normally.
        finishAfterTransition();
        return;
    }
    try {
        // Inform activity task manager that the activity received a back press
        // while at the root of the task. This call allows ActivityTaskManager
        // to intercept or defer finishing.
        ActivityTaskManager.getService().onBackPressedOnTaskRoot(mToken,
                new RequestFinishCallback(new WeakReference<>(this)));
    } catch (RemoteException e) {
        finishAfterTransition();
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

→androidx.fragment.app.BackStackRecord#trackAddedFragmentsInPop

```java
/**
 * Removes fragments that are added or removed during a pop operation.
 *
 * @param added Initialized to the fragments that are in the mManager.mAdded, this
 *              will be modified to contain the fragments that will be in mAdded
 *              after the execution ({@link #executeOps()}.
 * @param oldPrimaryNav The tracked primary navigation fragment as of the beginning of
 *                      this set of ops
 * @return the new oldPrimaryNav fragment after this record's ops would be popped
 */
Fragment trackAddedFragmentsInPop(ArrayList<Fragment> added, Fragment oldPrimaryNav) {
    for (int opNum = mOps.size() - 1; opNum >= 0; opNum--) {
        final Op op = mOps.get(opNum);
        switch (op.mCmd) {
            case OP_ADD:
            case OP_ATTACH:
                added.remove(op.mFragment);
                break;
            case OP_REMOVE:
            case OP_DETACH:
                added.add(op.mFragment);
                break;
            case OP_UNSET_PRIMARY_NAV:
                oldPrimaryNav = op.mFragment;
                break;
            case OP_SET_PRIMARY_NAV:
                oldPrimaryNav = null;
                break;
            case OP_SET_MAX_LIFECYCLE:
                op.mCurrentMaxState = op.mOldMaxState;
                break;
        }
    }
    return oldPrimaryNav;
}
```

