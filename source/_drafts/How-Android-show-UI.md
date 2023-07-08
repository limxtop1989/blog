---
title: How Android show UI
tags:
---

# TODO
EventControlThread
DispSyncThread
SF EventThread
App EventThread

ViewRootImpl
Choreographer
DisplayList（绘制指令）
surfaceview
TextureView
SurfaceTexture

frameworks/base/core/java/android/view/InputEventReceiver.java

# Traversals
```Java
    @Override
    public void requestLayout() {
        if (!mHandlingLayoutInLayoutRequest) {
            checkThread();
            mLayoutRequested = true;
            scheduleTraversals();
        }
    }
```  

```Java
    @UnsupportedAppUsage
    void invalidate() {
        mDirty.set(0, 0, mWidth, mHeight);
        if (!mWillDrawSoon) {
            scheduleTraversals();
        }
    }
```      
`requestLayout` 额外比 `invalidate` 多了一个标记 `mLayoutRequested = true` 

```Java
    void scheduleTraversals() {
        if (!mTraversalScheduled) {
            mTraversalScheduled = true;
            mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
            mChoreographer.postCallback(
                    Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
            notifyRendererOfFramePending();
            pokeDrawLockIfNeeded();
        }
    }
```  
`mTraversalRunnable` 是`TraversalRunnable`
```
    final class TraversalRunnable implements Runnable {
        @Override
        public void run() {
            doTraversal();
        }
    }
```

```Java
    void doTraversal() {
        if (mTraversalScheduled) {
            mTraversalScheduled = false;
            mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);

            if (mProfile) {
                Debug.startMethodTracing("ViewAncestor");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
        }
    }
```        
```Java
    private void performTraversals() {
    // measureHierarchy
    // mAttachInfo.mThreadedRenderer.initialize(mSurface)
    // threadedRenderer.setup(mWidth, mHeight, mAttachInfo,
                            mWindowAttributes.surfaceInsets);
   // performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
   // performLayout(lp, mWidth, mHeight);
   // performDraw()                            
```
`mTraversalRunnable` 最终放在`mCallbackQueues`, `callbackType = Choreographer.CALLBACK_TRAVERSAL`，然后`scheduleFrameLocked`，发送一个通知：
安排在下一显示帧开始时提供一个单一的垂直同步脉冲。

# 接收垂直同步信号
```
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;
        private VsyncEventData mLastVsyncEventData = new VsyncEventData();

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource, 0);
        }

        // TODO(b/116025192): physicalDisplayId is ignored because SF only emits VSYNC events for
        // the internal display and DisplayEventReceiver#scheduleVsync only allows requesting VSYNC
        // for the internal display implicitly.
        @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
                VsyncEventData vsyncEventData) {
            try {
                if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                    Trace.traceBegin(Trace.TRACE_TAG_VIEW,
                            "Choreographer#onVsync "
                                    + vsyncEventData.preferredFrameTimeline().vsyncId);
                }
                // Post the vsync event to the Handler.
                // The idea is to prevent incoming vsync events from completely starving
                // the message queue.  If there are no messages in the queue with timestamps
                // earlier than the frame time, then the vsync event will be processed immediately.
                // Otherwise, messages that predate the vsync event will be handled first.
                long now = System.nanoTime();
                if (timestampNanos > now) {
                    Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                            + " ms in the future!  Check that graphics HAL is generating vsync "
                            + "timestamps using the correct timebase.");
                    timestampNanos = now;
                }

                if (mHavePendingVsync) {
                    Log.w(TAG, "Already have a pending vsync event.  There should only be "
                            + "one at a time.");
                } else {
                    mHavePendingVsync = true;
                }

                mTimestampNanos = timestampNanos;
                mFrame = frame;
                mLastVsyncEventData = vsyncEventData;
                Message msg = Message.obtain(mHandler, this);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
        }
    }
```    

> Schedules a single vertical sync pulse to be delivered when the next display frame begins.



```Java
    private void postCallbackDelayedInternal(int callbackType,
            Object action, Object token, long delayMillis) {
        if (DEBUG_FRAMES) {
            Log.d(TAG, "PostCallback: type=" + callbackType
                    + ", action=" + action + ", token=" + token
                    + ", delayMillis=" + delayMillis);
        }

        synchronized (mLock) {
            final long now = SystemClock.uptimeMillis();
            final long dueTime = now + delayMillis;
            mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

            if (dueTime <= now) {
                scheduleFrameLocked(now);
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
                msg.arg1 = callbackType;
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, dueTime);
            }
        }
    }
```  

```Java
    private void scheduleFrameLocked(long now) {
        if (!mFrameScheduled) {
            mFrameScheduled = true;
            if (USE_VSYNC) {
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame on vsync.");
                }

                // If running on the Looper thread, then schedule the vsync immediately,
                // otherwise post a message to schedule the vsync from the UI thread
                // as soon as possible.
                if (isRunningOnLooperThreadLocked()) {
                    scheduleVsyncLocked();// mDisplayEventReceiver.scheduleVsync();
                } else {
                    Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                    msg.setAsynchronous(true);
                    mHandler.sendMessageAtFrontOfQueue(msg);
                }
            } else {
                final long nextFrameTime = Math.max(
                        mLastFrameTimeNanos / TimeUtils.NANOS_PER_MS + sFrameDelay, now);
                if (DEBUG_FRAMES) {
                    Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
                }
                Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, nextFrameTime);
            }
        }
    }
```

```Java
android.view.DisplayEventReceiver#scheduleVsync
    /**
     * Schedules a single vertical sync pulse to be delivered when the next
     * display frame begins.
     */
    @UnsupportedAppUsage
    public void scheduleVsync() {
        if (mReceiverPtr == 0) {
            Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                    + "receiver has already been disposed.");
        } else {
            nativeScheduleVsync(mReceiverPtr);
        }
    }

android.view.DisplayEventReceiver#dispatchVsync
    // Called from native code.
    @SuppressWarnings("unused")
    private void dispatchVsync(long timestampNanos, long physicalDisplayId, int frame,
            VsyncEventData vsyncEventData) {
        onVsync(timestampNanos, physicalDisplayId, frame, vsyncEventData);
    }
```    
``` C
base/core/jni/android_view_DisplayEventReceiver.cpp:245
static void nativeScheduleVsync(JNIEnv* env, jclass clazz, jlong receiverPtr) {
    sp<NativeDisplayEventReceiver> receiver =
            reinterpret_cast<NativeDisplayEventReceiver*>(receiverPtr);
    status_t status = receiver->scheduleVsync();
    if (status) {
        String8 message;
        message.appendFormat("Failed to schedule next vertical sync pulse.  status=%d", status);
        jniThrowRuntimeException(env, message.string());
    }
}
```

``` C
frameworks/native/libs/gui/DisplayEventDispatcher.cpp
status_t DisplayEventDispatcher::scheduleVsync() {
    if (!mWaitingForVsync) {
        ALOGV("dispatcher %p ~ Scheduling vsync.", this);

        // Drain all pending events.
        nsecs_t vsyncTimestamp;
        PhysicalDisplayId vsyncDisplayId;
        uint32_t vsyncCount;
        VsyncEventData vsyncEventData;
        if (processPendingEvents(&vsyncTimestamp, &vsyncDisplayId, &vsyncCount, &vsyncEventData)) {
            ALOGE("dispatcher %p ~ last event processed while scheduling was for %" PRId64 "", this,
                  ns2ms(static_cast<nsecs_t>(vsyncTimestamp)));
        }

        status_t status = mReceiver.requestNextVsync();
        if (status) {
            ALOGW("Failed to request next vsync, status=%d", status);
            return status;
        }

        mWaitingForVsync = true;
        mLastScheduleVsyncTime = systemTime(SYSTEM_TIME_MONOTONIC);
    }
    return OK;
}
``` 
``` C
bool DisplayEventDispatcher::processPendingEvents(nsecs_t* outTimestamp,
                                                  PhysicalDisplayId* outDisplayId,
                                                  uint32_t* outCount,
                                                  VsyncEventData* outVsyncEventData) {
    bool gotVsync = false;
    DisplayEventReceiver::Event buf[EVENT_BUFFER_SIZE];
    ssize_t n;
    while ((n = mReceiver.getEvents(buf, EVENT_BUFFER_SIZE)) > 0) {
        ALOGV("dispatcher %p ~ Read %d events.", this, int(n));
        mFrameRateOverrides.reserve(n);
        for (ssize_t i = 0; i < n; i++) {
            const DisplayEventReceiver::Event& ev = buf[i];
            switch (ev.header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    // Later vsync events will just overwrite the info from earlier
                    // ones. That's fine, we only care about the most recent.
                    gotVsync = true;
                    *outTimestamp = ev.header.timestamp;
                    *outDisplayId = ev.header.displayId;
                    *outCount = ev.vsync.count;
                    *outVsyncEventData = ev.vsync.vsyncData;
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                    dispatchHotplug(ev.header.timestamp, ev.header.displayId, ev.hotplug.connected);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_MODE_CHANGE:
                    dispatchModeChanged(ev.header.timestamp, ev.header.displayId,
                                        ev.modeChange.modeId, ev.modeChange.vsyncPeriod);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_NULL:
                    dispatchNullEvent(ev.header.timestamp, ev.header.displayId);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE:
                    mFrameRateOverrides.emplace_back(ev.frameRateOverride);
                    break;
                case DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE_FLUSH:
                    dispatchFrameRateOverrides(ev.header.timestamp, ev.header.displayId,
                                               std::move(mFrameRateOverrides));
                    break;
                default:
                    ALOGW("dispatcher %p ~ ignoring unknown event type %#x", this, ev.header.type);
                    break;
            }
        }
    }
    if (n < 0) {
        ALOGW("Failed to get events from display event dispatcher, status=%d", status_t(n));
    }
    return gotVsync;
}
```
``` C
frameworks/native/libs/gui/DisplayEventReceiver.cpp
status_t DisplayEventReceiver::requestNextVsync() {
    if (mEventConnection != nullptr) {
        mEventConnection->requestNextVsync();
        return NO_ERROR;
    }
    return mInitError.has_value() ? mInitError.value() : NO_INIT;
}
```

``` C
binder::Status EventThreadConnection::requestNextVsync() {
    ATRACE_CALL();
    mEventThread->requestNextVsync(this);
    return binder::Status::ok();
}
```

``` C
void EventThread::requestNextVsync(const sp<EventThreadConnection>& connection) {
    if (connection->resyncCallback) {
        connection->resyncCallback();
    }

    std::lock_guard<std::mutex> lock(mMutex);

    if (connection->vsyncRequest == VSyncRequest::None) {
        connection->vsyncRequest = VSyncRequest::Single;
        mCondition.notify_all();
    } else if (connection->vsyncRequest == VSyncRequest::SingleSuppressCallback) {
        connection->vsyncRequest = VSyncRequest::Single;
    }
}
```
` connection->vsyncRequest = VSyncRequest::Single; ` 赋值后，唤醒等待线程。shouldConsumeEvent返回true，将connection 压consumers栈，然后便利consumers，派发事件。

```
frameworks/native/services/surfaceflinger/Scheduler/EventThread.cpp
	
void EventThread::threadMain(std::unique_lock<std::mutex>& lock) {
    DisplayEventConsumers consumers;

    while (mState != State::Quit) {
        std::optional<DisplayEventReceiver::Event> event;

        // Determine next event to dispatch.
        if (!mPendingEvents.empty()) {
            event = mPendingEvents.front();
            mPendingEvents.pop_front();

            switch (event->header.type) {
                case DisplayEventReceiver::DISPLAY_EVENT_HOTPLUG:
                    if (event->hotplug.connected && !mVSyncState) {
                        mVSyncState.emplace(event->header.displayId);
                    } else if (!event->hotplug.connected && mVSyncState &&
                               mVSyncState->displayId == event->header.displayId) {
                        mVSyncState.reset();
                    }
                    break;

                case DisplayEventReceiver::DISPLAY_EVENT_VSYNC:
                    if (mInterceptVSyncsCallback) {
                        mInterceptVSyncsCallback(event->header.timestamp);
                    }
                    break;
            }
        }

        bool vsyncRequested = false;

        // Find connections that should consume this event.
        auto it = mDisplayEventConnections.begin();
        while (it != mDisplayEventConnections.end()) {
            if (const auto connection = it->promote()) {
                vsyncRequested |= connection->vsyncRequest != VSyncRequest::None;

                if (event && shouldConsumeEvent(*event, connection)) {
                    consumers.push_back(connection);
                }

                ++it;
            } else {
                it = mDisplayEventConnections.erase(it);
            }
        }

        if (!consumers.empty()) {
            dispatchEvent(*event, consumers);
            consumers.clear();
        }

        State nextState;
        if (mVSyncState && vsyncRequested) {
            nextState = mVSyncState->synthetic ? State::SyntheticVSync : State::VSync;
        } else {
            ALOGW_IF(!mVSyncState, "Ignoring VSYNC request while display is disconnected");
            nextState = State::Idle;
        }

        if (mState != nextState) {
            if (mState == State::VSync) {
                mVSyncSource->setVSyncEnabled(false);
            } else if (nextState == State::VSync) {
                mVSyncSource->setVSyncEnabled(true);
            }

            mState = nextState;
        }

        if (event) {
            continue;
        }

        // Wait for event or client registration/request.
        if (mState == State::Idle) {
            mCondition.wait(lock);
        } else {
            // Generate a fake VSYNC after a long timeout in case the driver stalls. When the
            // display is off, keep feeding clients at 60 Hz.
            const std::chrono::nanoseconds timeout =
                    mState == State::SyntheticVSync ? 16ms : 1000ms;
            if (mCondition.wait_for(lock, timeout) == std::cv_status::timeout) {
                if (mState == State::VSync) {
                    ALOGW("Faking VSYNC due to driver stall for thread %s", mThreadName);
                    std::string debugInfo = "VsyncSource debug info:\n";
                    mVSyncSource->dump(debugInfo);
                    // Log the debug info line-by-line to avoid logcat overflow
                    auto pos = debugInfo.find('\n');
                    while (pos != std::string::npos) {
                        ALOGW("%s", debugInfo.substr(0, pos).c_str());
                        debugInfo = debugInfo.substr(pos + 1);
                        pos = debugInfo.find('\n');
                    }
                }

                LOG_FATAL_IF(!mVSyncState);
                const auto now = systemTime(SYSTEM_TIME_MONOTONIC);
                const auto deadlineTimestamp = now + timeout.count();
                const auto expectedVSyncTime = deadlineTimestamp + timeout.count();
                mPendingEvents.push_back(makeVSync(mVSyncState->displayId, now,
                                                   ++mVSyncState->count, expectedVSyncTime,
                                                   deadlineTimestamp));
            }
        }
    }
}
```

```
void EventThread::dispatchEvent(const DisplayEventReceiver::Event& event,
                                const DisplayEventConsumers& consumers) {
    for (const auto& consumer : consumers) {
        DisplayEventReceiver::Event copy = event;
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_VSYNC) {
            const int64_t frameInterval = mGetVsyncPeriodFunction(consumer->mOwnerUid);
            copy.vsync.vsyncData.frameInterval = frameInterval;
            generateFrameTimeline(copy.vsync.vsyncData, frameInterval, copy.header.timestamp,
                                  event.vsync.vsyncData.preferredExpectedPresentationTime(),
                                  event.vsync.vsyncData.preferredDeadlineTimestamp());
        }
        switch (consumer->postEvent(copy)) {
            case NO_ERROR:
                break;

            case -EAGAIN:
                // TODO: Try again if pipe is full.
                ALOGW("Failed dispatching %s for %s", toString(event).c_str(),
                      toString(*consumer).c_str());
                break;

            default:
                // Treat EPIPE and other errors as fatal.
                removeDisplayEventConnectionLocked(consumer);
        }
    }
}
```
`consumer` 其实就是`EventThreadConnection`
```
status_t EventThreadConnection::postEvent(const DisplayEventReceiver::Event& event) {
    constexpr auto toStatus = [](ssize_t size) {
        return size < 0 ? status_t(size) : status_t(NO_ERROR);
    };

    if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE ||
        event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE_FLUSH) {
        mPendingEvents.emplace_back(event);
        if (event.header.type == DisplayEventReceiver::DISPLAY_EVENT_FRAME_RATE_OVERRIDE) {
            return status_t(NO_ERROR);
        }

        auto size = DisplayEventReceiver::sendEvents(&mChannel, mPendingEvents.data(),
                                                     mPendingEvents.size());
        mPendingEvents.clear();
        return toStatus(size);
    }

    auto size = DisplayEventReceiver::sendEvents(&mChannel, &event, 1);
    return toStatus(size);
}
```
TODO: 到这里，我就不知道代码怎么调用了。
```
ssize_t DisplayEventReceiver::sendEvents(Event const* events, size_t count) {
    return DisplayEventReceiver::sendEvents(mDataChannel.get(), events, count);
}

ssize_t DisplayEventReceiver::sendEvents(gui::BitTube* dataChannel,
        Event const* events, size_t count)
{
    return gui::BitTube::sendObjects(dataChannel, events, count);
}
```

```
    private final class FrameDisplayEventReceiver extends DisplayEventReceiver
            implements Runnable {
        private boolean mHavePendingVsync;
        private long mTimestampNanos;
        private int mFrame;
        private VsyncEventData mLastVsyncEventData = new VsyncEventData();

        public FrameDisplayEventReceiver(Looper looper, int vsyncSource) {
            super(looper, vsyncSource, 0);
        }

        // TODO(b/116025192): physicalDisplayId is ignored because SF only emits VSYNC events for
        // the internal display and DisplayEventReceiver#scheduleVsync only allows requesting VSYNC
        // for the internal display implicitly.
        @Override
        public void onVsync(long timestampNanos, long physicalDisplayId, int frame,
                VsyncEventData vsyncEventData) {
            try {
                if (Trace.isTagEnabled(Trace.TRACE_TAG_VIEW)) {
                    Trace.traceBegin(Trace.TRACE_TAG_VIEW,
                            "Choreographer#onVsync "
                                    + vsyncEventData.preferredFrameTimeline().vsyncId);
                }
                // Post the vsync event to the Handler.
                // The idea is to prevent incoming vsync events from completely starving
                // the message queue.  If there are no messages in the queue with timestamps
                // earlier than the frame time, then the vsync event will be processed immediately.
                // Otherwise, messages that predate the vsync event will be handled first.
                long now = System.nanoTime();
                if (timestampNanos > now) {
                    Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                            + " ms in the future!  Check that graphics HAL is generating vsync "
                            + "timestamps using the correct timebase.");
                    timestampNanos = now;
                }

                if (mHavePendingVsync) {
                    Log.w(TAG, "Already have a pending vsync event.  There should only be "
                            + "one at a time.");
                } else {
                    mHavePendingVsync = true;
                }

                mTimestampNanos = timestampNanos;
                mFrame = frame;
                mLastVsyncEventData = vsyncEventData;
                Message msg = Message.obtain(mHandler, this);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtTime(msg, timestampNanos / TimeUtils.NANOS_PER_MS);
            } finally {
                Trace.traceEnd(Trace.TRACE_TAG_VIEW);
            }
        }

        @Override
        public void run() {
            mHavePendingVsync = false;
            doFrame(mTimestampNanos, mFrame, mLastVsyncEventData);
        }
    }
```    

``` Java
    void doFrame(long frameTimeNanos, int frame,
            DisplayEventReceiver.VsyncEventData vsyncEventData) {
        final long startNanos;
        final long frameIntervalNanos = vsyncEventData.frameInterval;
        try {
            FrameData frameData = new FrameData(frameTimeNanos, vsyncEventData);
            synchronized (mLock) {
            
            AnimationUtils.lockAnimationClock(frameTimeNanos / TimeUtils.NANOS_PER_MS);

            mFrameInfo.markInputHandlingStart();
            doCallbacks(Choreographer.CALLBACK_INPUT, frameData, frameIntervalNanos);

            mFrameInfo.markAnimationsStart();
            doCallbacks(Choreographer.CALLBACK_ANIMATION, frameData, frameIntervalNanos);
            doCallbacks(Choreographer.CALLBACK_INSETS_ANIMATION, frameData,
                    frameIntervalNanos);

            mFrameInfo.markPerformTraversalsStart();
            doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameData, frameIntervalNanos);

            doCallbacks(Choreographer.CALLBACK_COMMIT, frameData, frameIntervalNanos);
        } finally {
            AnimationUtils.unlockAnimationClock();
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```  

``` Java
    void doCallbacks(int callbackType, FrameData frameData, long frameIntervalNanos) {
        CallbackRecord callbacks;
        long frameTimeNanos = frameData.mFrameTimeNanos;
        synchronized (mLock) {
            
        try {
            Trace.traceBegin(Trace.TRACE_TAG_VIEW, CALLBACK_TRACE_TITLES[callbackType]);
            for (CallbackRecord c = callbacks; c != null; c = c.next) {
                c.run(frameData);
            }
        } finally {
            synchronized (mLock) {
                mCallbacksRunning = false;
                do {
                    final CallbackRecord next = callbacks.next;
                    recycleCallbackLocked(callbacks);
                    callbacks = next;
                } while (callbacks != null);
            }
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```    
```Java 
    private static final class CallbackRecord {
        public CallbackRecord next;
        public long dueTime;
        /** Runnable or FrameCallback or VsyncCallback object. */
        public Object action;
        /** Denotes the action type. */
        public Object token;

        @UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
        public void run(long frameTimeNanos) {
            if (token == FRAME_CALLBACK_TOKEN) {
                ((FrameCallback)action).doFrame(frameTimeNanos);
            } else {
                ((Runnable)action).run();
            }
        }

        void run(FrameData frameData) {
            if (token == VSYNC_CALLBACK_TOKEN) {
                ((VsyncCallback) action).onVsync(frameData);
            } else {
                run(frameData.getFrameTimeNanos());
            }
        }
    }
```  

```Java
    private final class FrameHandler extends Handler {
        public FrameHandler(Looper looper) {
            super(looper);
        }

        @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
                case MSG_DO_FRAME:
                    doFrame(System.nanoTime(), 0, new DisplayEventReceiver.VsyncEventData());
                    break;
                case MSG_DO_SCHEDULE_VSYNC:
                    doScheduleVsync();
                    break;
                case MSG_DO_SCHEDULE_CALLBACK:
                    doScheduleCallback(msg.arg1);
                    break;
            }
        }
    }
```  


```Java
    /**
     * Implement this interface to receive a callback when a new display frame is
     * being rendered.  The callback is invoked on the {@link Looper} thread to
     * which the {@link Choreographer} is attached.
     */
    public interface FrameCallback {
        /**
         * Called when a new display frame is being rendered.
         * <p>
         * This method provides the time in nanoseconds when the frame started being rendered.
         * The frame time provides a stable time base for synchronizing animations
         * and drawing.  It should be used instead of {@link SystemClock#uptimeMillis()}
         * or {@link System#nanoTime()} for animations and drawing in the UI.  Using the frame
         * time helps to reduce inter-frame jitter because the frame time is fixed at the time
         * the frame was scheduled to start, regardless of when the animations or drawing
         * callback actually runs.  All callbacks that run as part of rendering a frame will
         * observe the same frame time so using the frame time also helps to synchronize effects
         * that are performed by different callbacks.
         * </p><p>
         * Please note that the framework already takes care to process animations and
         * drawing using the frame time as a stable time base.  Most applications should
         * not need to use the frame time information directly.
         * </p>
         *
         * @param frameTimeNanos The time in nanoseconds when the frame started being rendered,
         * in the {@link System#nanoTime()} timebase.  Divide this value by {@code 1000000}
         * to convert it to the {@link SystemClock#uptimeMillis()} time base.
         */
        public void doFrame(long frameTimeNanos);
    }
```

```Java
    /**
     * Implement this interface to receive a callback to start the next frame. The callback is
     * invoked on the {@link Looper} thread to which the {@link Choreographer} is attached. The
     * callback payload contains information about multiple possible frames, allowing choice of
     * the appropriate frame based on latency requirements.
     *
     * @see FrameCallback
     */
    public interface VsyncCallback {
        /**
         * Called when a new display frame is being rendered.
         *
         * @param data The payload which includes frame information. Divide nanosecond values by
         *             {@code 1000000} to convert it to the {@link SystemClock#uptimeMillis()}
         *             time base.
         * @see FrameCallback#doFrame
         **/
        void onVsync(@NonNull FrameData data);
    }
```  
## 参考
- [1] [desc] (https://esligh.github.io/[object%20Object]/aosp-choreographer/)
- [2] [desc] (https://www.jianshu.com/p/996bca12eb1d)
- [3] [desc] (https://juejin.cn/post/6844904084139425806)