---
title: FragmentManager源码剖析一
date: 2021-10-18 21:23:33
tags:
---



# 总结

1. 使用Fragment的Activity需要继承`FragmentActivity` 这里有派发生命周期方法给FragmentManager，再派发给旗下Fragment。
2. 加入Fragment时，需要判断`savedInstanceState == null` ，避免重复创建实例。
3. add Fragment，初始时，其Fragment生命周期方法会被升级到onStart 状态，随后与FragmentManager的状态同步，如果宿主Activity是Resume状态，则同步至onResume状态。
4. detach fragment，其生命周期方法会被降级到onDestroyView。
5. remove fragment，其生命周期方法会被降级到onDestroyView 或者onDestroy。
6. show or hide fragment，生命周期方法不会回调，只是切换fragment 根view的可见性。

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
								// name can be null
								.addToBackStack("name") 
                .commit();
        }
    }
}
```

Note: the fragment transaction is only created when savedInstanceState is null. This is to ensure that the fragment is added only once, when the activity is first created. When a configuration change occurs and the activity is recreated, savedInstanceState is no longer null, and the fragment does not need to be added a second time, as the fragment is automatically restored from the savedInstanceState.

Note: You should always use setReorderingAllowed(true) when performing a FragmentTransaction. For more information on reordered transactions, see Fragment transactions.

## beginTransaction

```java
@NonNull
@Override
public FragmentTransaction beginTransaction() {
    return new BackStackRecord(this);
}
```

## setReorderingAllowed

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

## add

```java
/**
 * Calls {@link #add(int, Fragment, String)} with a 0 containerViewId.
 */
@NonNull
public FragmentTransaction add(@NonNull Fragment fragment, @Nullable String tag)  {
    doAddOp(0, fragment, tag, OP_ADD);
    return this;
}
```

```java
void doAddOp(int containerViewId, Fragment fragment, @Nullable String tag, int opcmd) {
    final Class<?> fragmentClass = fragment.getClass();
    final int modifiers = fragmentClass.getModifiers();
    if (fragmentClass.isAnonymousClass() || !Modifier.isPublic(modifiers)
            || (fragmentClass.isMemberClass() && !Modifier.isStatic(modifiers))) {
        throw new IllegalStateException("Fragment " + fragmentClass.getCanonicalName()
                + " must be a public static class to be  properly recreated from"
                + " instance state.");
    }

    if (tag != null) {
        if (fragment.mTag != null && !tag.equals(fragment.mTag)) {
            throw new IllegalStateException("Can't change tag of fragment "
                    + fragment + ": was " + fragment.mTag
                    + " now " + tag);
        }
        fragment.mTag = tag;
    }

    if (containerViewId != 0) {
        if (containerViewId == View.NO_ID) {
            throw new IllegalArgumentException("Can't add fragment "
                    + fragment + " with tag " + tag + " to container view with no id");
        }
        if (fragment.mFragmentId != 0 && fragment.mFragmentId != containerViewId) {
            throw new IllegalStateException("Can't change container ID of fragment "
                    + fragment + ": was " + fragment.mFragmentId
                    + " now " + containerViewId);
        }
        fragment.mContainerId = fragment.mFragmentId = containerViewId;
    }

    addOp(new Op(opcmd, fragment));
}
```

```java
void addOp(Op op) {
    mOps.add(op);
    op.mEnterAnim = mEnterAnim;
    op.mExitAnim = mExitAnim;
    op.mPopEnterAnim = mPopEnterAnim;
    op.mPopExitAnim = mPopExitAnim;
}
```

传入fragment实例，将对fragment的增删换，显示隐藏等操作封装成op，并加入op列表集合，作为一个操作事物。

## addToBackStack

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

## commit

```java
androidx.fragment.app.BackStackRecord#commitInternal

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
androidx.fragment.app.FragmentManager#enqueueAction

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

## OpGenerator

BackStackRecord实现OpGenerator接口，先看下方法实现内容：将本次事务加入事务集合。并且判断如果要支持回退栈，则加入回退栈集合。

```java
androidx.fragment.app.BackStackRecord#generateOps

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
androidx.fragment.app.FragmentManager#addBackStackState

void addBackStackState(BackStackRecord state) {
    if (mBackStack == null) {
        mBackStack = new ArrayList<BackStackRecord>();
    }
    mBackStack.add(state);
}
```

# scheduleCommit

## ensureExecReady

条件检查，并实例化mTmpRecords & mTmpIsPop。

```java
androidx.fragment.app.FragmentManager#execPendingActions

/**
 * Only call from main thread!
 */
public boolean execPendingActions() {
    ensureExecReady(true);// 1. Init mTmpRecords & mTmpIsPop

    boolean didSomething = false;
    // 2. Copy BackStackRecord  in mPendingActions into mTmpRecords， so does mTmpIsPop 
    while (generateOpsForPendingActions(mTmpRecords, mTmpIsPop)) {
        mExecutingActions = true;
        try {
						// 3. 
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

## generateOpsForPendingActions

将mPendingActions的BackStackRecord，即我们提交的事务集合，复制到records（即成员变量mTmpRecords，mTmpIsPop同理）

```java
androidx.fragment.app.FragmentManager#generateOpsForPendingActions

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

## removeRedundantOperationsAndExecute

```java
androidx.fragment.app.FragmentManager#removeRedundantOperationsAndExecute

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
				// startIndex 为0， numRecords为批处理的事务个数。
        executeOpsTogether(records, isRecordPop, startIndex, numRecords);
    }
}
```

## executeOpsTogether

一般推荐允许重排序，看最后一行方法调用executeOpsTogether。从API描述看，它执行事务集合的子集，子集里事务要么是允许重排序，要么不允许。

```java
androidx.fragment.app.FragmentManager#executeOpsTogether

/**
 * Executes a subset of a list of BackStackRecords, all of which either allow reordering or
 * do not allow ordering.
 * @param records A list of BackStackRecords that are to be executed
 * @param isRecordPop The direction that these records are being run.
 * @param startIndex The index of the first record in <code>records</code> to be executed
 * @param endIndex One more than the final record index in <code>records</code> to executed.
 */
private void executeOpsTogether(@NonNull ArrayList<BackStackRecord> records,
        @NonNull ArrayList<Boolean> isRecordPop, int startIndex, int endIndex) {
    final boolean allowReordering = records.get(startIndex).mReorderingAllowed;
    boolean addToBackStack = false;
    if (mTmpAddedFragments == null) {
        mTmpAddedFragments = new ArrayList<>();
    } else {
        mTmpAddedFragments.clear();
    }
		// 执行transaction.add(fragment)，事务完成后，fragment会加入mAdded集合内，
		// activity生命周期方法派发给FragmentManager，继而派发给mAdded集合内的Fragment
		// 此处复制已添加Fragment集合到临时变量，
    mTmpAddedFragments.addAll(mFragmentStore.getFragments());
    Fragment oldPrimaryNav = getPrimaryNavigationFragment();
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (!isPop) {
						// 这里将op替换为更原子的操作，如replace拆解为remove & add，并将ops内的fragment
						// 依照op的cmd命令，从mTmpAddedFragments add 或 remove op里的fragment，
            oldPrimaryNav = record.expandOps(mTmpAddedFragments, oldPrimaryNav);
        } else {
            oldPrimaryNav = record.trackAddedFragmentsInPop(mTmpAddedFragments, oldPrimaryNav);
        }
        addToBackStack = addToBackStack || record.mAddToBackStack;
    }
    mTmpAddedFragments.clear();

    if (!allowReordering && mCurState >= Fragment.CREATED) {
        // 一般推荐重排序，此处代码省略
    }
    // 1. 遍历BackStackRecord集合，逐个执行事务：将Fragment发送给FragmentManager管理，并更新Fragment标记。
    executeOps(records, isRecordPop, startIndex, endIndex);

    if (USE_STATE_MANAGER) {
        // The last operation determines the overall direction, this ensures that operations
        // such as push, push, pop, push are correctly considered a push
        boolean isPop = isRecordPop.get(endIndex - 1);
        // Ensure that Fragments directly affected by operations
        // are moved to their expected state in operation order
        for (int index = startIndex; index < endIndex; index++) {
            BackStackRecord record = records.get(index);
            if (isPop) {
                // Pop operations get applied in reverse order
                for (int opIndex = record.mOps.size() - 1; opIndex >= 0; opIndex--) {
                    FragmentTransaction.Op op = record.mOps.get(opIndex);
                    Fragment fragment = op.mFragment;
                    if (fragment != null) {
                        FragmentStateManager fragmentStateManager =
                                createOrGetFragmentStateManager(fragment);
                        fragmentStateManager.moveToExpectedState();
                    }
                }
            } else {
                for (FragmentTransaction.Op op : record.mOps) {
                    Fragment fragment = op.mFragment;
                    if (fragment != null) {
                        FragmentStateManager fragmentStateManager =
                                createOrGetFragmentStateManager(fragment);
												// 2. 事务内的Fragment推送至默认状态
                        fragmentStateManager.moveToExpectedState();
                    }
                }
            }

        }
        // And only then do we move all other fragments to the current state
				// 3. 将Fragment全部推送到当前FragmentManager的状态。
        moveToState(mCurState, true);
        Set<SpecialEffectsController> changedControllers = collectChangedControllers(
                records, startIndex, endIndex);
        for (SpecialEffectsController controller : changedControllers) {
            controller.updateOperationDirection(isPop);
            controller.markPostponedState();
            controller.executePendingOperations();
        }
    } else {
        // USE_STATE_MANAGER 默认为true，此处代码省略
    }
		// 调用每个事务的提交监听回掉
    for (int recordNum = startIndex; recordNum < endIndex; recordNum++) {
        final BackStackRecord record = records.get(recordNum);
        final boolean isPop = isRecordPop.get(recordNum);
        if (isPop && record.mIndex >= 0) {
            record.mIndex = -1;
        }
        record.runOnCommitRunnables();
    }
		// // 通知backStack变更的监听回掉
    if (addToBackStack) {
        reportBackStackChanged();
    }
}
```

### executeOps

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

遍历事务Op集合，逐个执行事务里的Op

```java
androidx.fragment.app.BackStackRecord#executeOps

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
            f.setPopDirection(false);
            f.setAnimations(op.mEnterAnim, op.mExitAnim, op.mPopEnterAnim, op.mPopExitAnim);
            f.setNextTransition(mTransition);
            f.setSharedElementNames(mSharedElementSourceNames, mSharedElementTargetNames);
        }
        switch (op.mCmd) {
            case OP_ADD:
                mManager.setExitAnimationOrder(f, false);
                mManager.addFragment(f);
                break;
            case OP_REMOVE:
                mManager.removeFragment(f);
                break;
            case OP_HIDE:
                mManager.hideFragment(f);
                break;
            case OP_SHOW:
                mManager.setExitAnimationOrder(f, false);
                mManager.showFragment(f);
                break;
            case OP_DETACH:
                mManager.detachFragment(f);
                break;
            case OP_ATTACH:
                mManager.setExitAnimationOrder(f, false);
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
				// 允许排序，以下代码不执行
        if (!mReorderingAllowed && op.mCmd != OP_ADD && f != null) {
            if (!FragmentManager.USE_STATE_MANAGER) {
                mManager.moveFragmentToExpectedState(f);
            }
        }
    }
    if (!mReorderingAllowed && !FragmentManager.USE_STATE_MANAGER) {
        // Added fragments are added at the end to comply with prior behavior.
        mManager.moveToState(mManager.mCurState, true);
    }
}
```

addFragment & removeFragment 主要是操作mAdded fragment集合并更新fragment.mRemoving & fragment.mAdded 标记

```java
androidx.fragment.app.FragmentManager#removeFragment
void removeFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) {
        Log.v(TAG, "remove: " + fragment + " nesting=" + fragment.mBackStackNesting);
    }
    final boolean inactive = !fragment.isInBackStack();
    if (!fragment.mDetached || inactive) {
        mFragmentStore.removeFragment(fragment);
        if (isMenuAvailable(fragment)) {
            mNeedMenuInvalidate = true;
        }
        fragment.mRemoving = true;
        setVisibleRemovingFragment(fragment);
    }
}

androidx.fragment.app.FragmentManager#addFragment
FragmentStateManager addFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "add: " + fragment);
    FragmentStateManager fragmentStateManager = createOrGetFragmentStateManager(fragment);
    fragment.mFragmentManager = this;
    mFragmentStore.makeActive(fragmentStateManager);
    if (!fragment.mDetached) {
        mFragmentStore.addFragment(fragment);
        fragment.mRemoving = false;
        if (fragment.mView == null) {
            fragment.mHiddenChanged = false;
        }
        if (isMenuAvailable(fragment)) {
            mNeedMenuInvalidate = true;
        }
    }
    return fragmentStateManager;
}
```

attachFragment 与addFragment的差异： 

1. 无makeActive(fragment)，这个差异导致两者的Fragment默认升级到started状态后，只有add fragment会继续与FragmentManager状态同步，接收FragmentManager生命周期方法的派发，attach fragment则到此戛然而止。也没有moveToState(fragment)操作（addFragment下，此方法不调用，传入moveToStateNow参数值为false）
2. 只有之前detach操作过，attach才有效果。

```java
androidx.fragment.app.FragmentManager#attachFragment
void attachFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "attach: " + fragment);
    if (fragment.mDetached) {
				// 初始化时，mDetached为false，只有detachFragment才置为true。
        fragment.mDetached = false;
        if (!fragment.mAdded) {
            mFragmentStore.addFragment(fragment);
            if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "add from attach: " + fragment);
            if (isMenuAvailable(fragment)) {
                mNeedMenuInvalidate = true;
            }
        }
    }
}

androidx.fragment.app.FragmentStore#addFragment
void addFragment(@NonNull Fragment fragment) {
    if (mAdded.contains(fragment)) {
        throw new IllegalStateException("Fragment already added: " + fragment);
    }
    synchronized (mAdded) {
        mAdded.add(fragment);
    }
    fragment.mAdded = true;
}
```

remove & detachFragment都将fragment从mAdded集合移除fragment，不同的是分别标记mRemoving & mDetached 为true。

```java
void removeFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) {
        Log.v(TAG, "remove: " + fragment + " nesting=" + fragment.mBackStackNesting);
    }
    final boolean inactive = !fragment.isInBackStack();
    if (!fragment.mDetached || inactive) {
        mFragmentStore.removeFragment(fragment);
        if (isMenuAvailable(fragment)) {
            mNeedMenuInvalidate = true;
        }
        fragment.mRemoving = true;
        setVisibleRemovingFragment(fragment);
    }
}

void removeFragment(@NonNull Fragment fragment) {
    synchronized (mAdded) {
        mAdded.remove(fragment);
    }
    fragment.mAdded = false;
}
```

```java
void detachFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "detach: " + fragment);
    if (!fragment.mDetached) {
        fragment.mDetached = true;
        if (fragment.mAdded) {
            // We are not already in back stack, so need to remove the fragment.
            if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "remove from detach: " + fragment);
            mFragmentStore.removeFragment(fragment);
            if (isMenuAvailable(fragment)) {
                mNeedMenuInvalidate = true;
            }
            setVisibleRemovingFragment(fragment);
        }
    }
}
```

showFrament & hideFragment 主要更新mHidden标记

```java
/**
 * Marks a fragment as hidden to be later animated in with
 * {@link #completeShowHideFragment(Fragment)}.
 *
 * @param fragment The fragment to be shown.
 */
void hideFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "hide: " + fragment);
    if (!fragment.mHidden) {
        fragment.mHidden = true;
        // Toggle hidden changed so that if a fragment goes through show/hide/show
        // it doesn't go through the animation.
        fragment.mHiddenChanged = !fragment.mHiddenChanged;
        setVisibleRemovingFragment(fragment);
    }
}

/**
 * Marks a fragment as shown to be later animated in with
 * {@link #completeShowHideFragment(Fragment)}.
 *
 * @param fragment The fragment to be shown.
 */
void showFragment(@NonNull Fragment fragment) {
    if (isLoggingEnabled(Log.VERBOSE)) Log.v(TAG, "show: " + fragment);
    if (fragment.mHidden) {
        fragment.mHidden = false;
        // Toggle hidden changed so that if a fragment goes through show/hide/show
        // it doesn't go through the animation.
        fragment.mHiddenChanged = !fragment.mHiddenChanged;
    }
}
```

Fragment 主要标记

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

### moveToExpectedState

遍历事务集合，将事务内每个op 对应的Fragment推送到合适的状态，具体状态值在computeExpectedState计算。默认是当前FragmentManager的状态。但会根据Fragment标记重算。

```java
// Assume the Fragment can go as high as the FragmentManager's state
int maxState = mFragmentManagerState;
// Fragments that are not currently added will sit in the CREATED state.
// detach & remove 操作的fragment，目标状态是CREATED
if (!mFragment.mAdded) {
    maxState = Math.min(maxState, Fragment.CREATED);
}

if (mFragment.mRemoving) {
    if (mFragment.isInBackStack()) {
        // Fragments on the back stack shouldn't go higher than CREATED
        maxState = Math.min(maxState, Fragment.CREATED);
    } else {
        // While removing a fragment, we always move to INITIALIZING
				// remove 操作的Fragment，如果不在回退栈，目标状态是INITIALIZING
        maxState = Math.min(maxState, Fragment.INITIALIZING);
    }
}
```

```java
void moveToExpectedState() {
        if (mMovingToState) {
            if (FragmentManager.isLoggingEnabled(Log.VERBOSE)) {
                Log.v(FragmentManager.TAG, "Ignoring re-entrant call to "
                        + "moveToExpectedState() for " + getFragment());
            }
            return;
        }
        try {
            mMovingToState = true;

            int newState;
            while ((newState = computeExpectedState()) != mFragment.mState) {
                if (newState > mFragment.mState) {
                    // Moving upward
                    int nextStep = mFragment.mState + 1;
                    switch (nextStep) {
                        case Fragment.ATTACHED:
                            attach();
                            break;
                        case Fragment.CREATED:
                            create();
                            break;
                        case Fragment.VIEW_CREATED:
                            ensureInflatedView();
                            createView();
                            break;
                        case Fragment.AWAITING_EXIT_EFFECTS:
                            activityCreated();
                            break;
                        case Fragment.ACTIVITY_CREATED:
                            if (mFragment.mView != null && mFragment.mContainer != null) {
                                SpecialEffectsController controller = SpecialEffectsController
                                        .getOrCreateController(mFragment.mContainer,
                                                mFragment.getParentFragmentManager());
                                int visibility = mFragment.mView.getVisibility();
                                SpecialEffectsController.Operation.State finalState =
                                        SpecialEffectsController.Operation.State.from(visibility);
                                controller.enqueueAdd(finalState, this);
                            }
                            mFragment.mState = Fragment.ACTIVITY_CREATED;
                            break;
                        case Fragment.STARTED:
                            start();
                            break;
                        case Fragment.AWAITING_ENTER_EFFECTS:
                            mFragment.mState = Fragment.AWAITING_ENTER_EFFECTS;
                            break;
                        case Fragment.RESUMED:
                            resume();
                            break;
                    }
                } else {
                    // Moving downward
                    int nextStep = mFragment.mState - 1;
                    switch (nextStep) {
                        case Fragment.AWAITING_ENTER_EFFECTS:
                            pause();
                            break;
                        case Fragment.STARTED:
                            mFragment.mState = Fragment.STARTED;
                            break;
                        case Fragment.ACTIVITY_CREATED:
                            stop();
                            break;
                        case Fragment.AWAITING_EXIT_EFFECTS:
                            if (FragmentManager.isLoggingEnabled(Log.DEBUG)) {
                                Log.d(TAG, "movefrom ACTIVITY_CREATED: " + mFragment);
                            }
                            if (mFragment.mView != null) {
                                // Need to save the current view state if not done already
                                // by saveInstanceState()
                                if (mFragment.mSavedViewState == null) {
                                    saveViewState();
                                }
                            }
                            if (mFragment.mView != null && mFragment.mContainer != null) {
                                SpecialEffectsController controller = SpecialEffectsController
                                        .getOrCreateController(mFragment.mContainer,
                                                mFragment.getParentFragmentManager());
                                controller.enqueueRemove(this);
                            }
                            mFragment.mState = Fragment.AWAITING_EXIT_EFFECTS;
                            break;
                        case Fragment.VIEW_CREATED:
                            mFragment.mInLayout = false;
                            mFragment.mState = Fragment.VIEW_CREATED;
                            break;
                        case Fragment.CREATED:
                            destroyFragmentView();
                            mFragment.mState = Fragment.CREATED;
                            break;
                        case Fragment.ATTACHED:
                            destroy();
                            break;
                        case Fragment.INITIALIZING:
                            detach();
                            break;
                    }
                }
            }
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
        } finally {
            mMovingToState = false;
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

同步FragmentManager的状态到mActive集合里的Fragments，mAdded集合的不同步。

```java
androidx.fragment.app.FragmentStore#moveToExpectedState
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
