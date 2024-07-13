---
title: view render
date: 2024-01-06 16:27:41
tags:
---

TODO: native surface 如何创建， mGraphicBufferProducer 是哪个类？
android.view.ViewRootImpl#draw
```java
if (isHardwareEnabled()) {
    mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
} else {
    if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
            scalingRequired, dirty, surfaceInsets)) {
        return false;
    }
}
```

# drawSoftware
```java
/**
    * @return true if drawing was successful, false if an error occurred
    */
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
        boolean scalingRequired, Rect dirty, Rect surfaceInsets) {

    canvas = mSurface.lockCanvas(dirty);

    mView.draw(canvas);

    surface.unlockCanvasAndPost(canvas);
}

```
## lockCanvas
```cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

    if (!isSurfaceValid(surface)) {
        jniThrowException(env, IllegalArgumentException, NULL);
        return 0;
    }

    if (!ACanvas_isSupportedPixelFormat(ANativeWindow_getFormat(surface.get()))) {
        native_window_set_buffers_format(surface.get(), PIXEL_FORMAT_RGBA_8888);
    }

    Rect dirtyRect(Rect::EMPTY_RECT);
    Rect* dirtyRectPtr = NULL;

    if (dirtyRectObj) {
        dirtyRect.left   = env->GetIntField(dirtyRectObj, gRectClassInfo.left);
        dirtyRect.top    = env->GetIntField(dirtyRectObj, gRectClassInfo.top);
        dirtyRect.right  = env->GetIntField(dirtyRectObj, gRectClassInfo.right);
        dirtyRect.bottom = env->GetIntField(dirtyRectObj, gRectClassInfo.bottom);
        dirtyRectPtr = &dirtyRect;
    }

    ANativeWindow_Buffer buffer;
    status_t err = surface->lock(&buffer, dirtyRectPtr);
    if (err < 0) {
        const char* const exception = (err == NO_MEMORY) ?
                OutOfResourcesException : IllegalArgumentException;
        jniThrowException(env, exception, NULL);
        return 0;
    }

    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(&buffer, static_cast<int32_t>(surface->getBuffersDataSpace()));

    if (dirtyRectPtr) {
        canvas.clipRect({dirtyRect.left, dirtyRect.top, dirtyRect.right, dirtyRect.bottom});
    }

    if (dirtyRectObj) {
        env->SetIntField(dirtyRectObj, gRectClassInfo.left,   dirtyRect.left);
        env->SetIntField(dirtyRectObj, gRectClassInfo.top,    dirtyRect.top);
        env->SetIntField(dirtyRectObj, gRectClassInfo.right,  dirtyRect.right);
        env->SetIntField(dirtyRectObj, gRectClassInfo.bottom, dirtyRect.bottom);
    }

    // Create another reference to the surface and return it.  This reference
    // should be passed to nativeUnlockCanvasAndPost in place of mNativeObject,
    // because the latter could be replaced while the surface is locked.
    sp<Surface> lockedSurface(surface);
    lockedSurface->incStrong(&sRefBaseOwner);
    return (jlong) lockedSurface.get();
}
```

### surface->lock
```cpp

status_t Surface::lock(
        ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    if (mLockedBuffer != nullptr) {
        ALOGE("Surface::lock failed, already locked");
        return INVALID_OPERATION;
    }

    if (!mConnectedToCpu) {
        int err = Surface::connect(NATIVE_WINDOW_API_CPU);
        if (err) {
            return err;
        }
        // we're intending to do software rendering from this point
        setUsage(GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN);
    }

    ANativeWindowBuffer* out;
    int fenceFd = -1;
    status_t err = dequeueBuffer(&out, &fenceFd);
    ALOGE_IF(err, "dequeueBuffer failed (%s)", strerror(-err));
    if (err == NO_ERROR) {
        sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
        const Rect bounds(backBuffer->width, backBuffer->height);

        Region newDirtyRegion;
        if (inOutDirtyBounds) {
            newDirtyRegion.set(static_cast<Rect const&>(*inOutDirtyBounds));
            newDirtyRegion.andSelf(bounds);
        } else {
            newDirtyRegion.set(bounds);
        }

        // figure out if we can copy the frontbuffer back
        const sp<GraphicBuffer>& frontBuffer(mPostedBuffer);
        const bool canCopyBack = (frontBuffer != nullptr &&
                backBuffer->width  == frontBuffer->width &&
                backBuffer->height == frontBuffer->height &&
                backBuffer->format == frontBuffer->format);

        if (canCopyBack) {
            // copy the area that is invalid and not repainted this round
            const Region copyback(mDirtyRegion.subtract(newDirtyRegion));
            if (!copyback.isEmpty()) {
                copyBlt(backBuffer, frontBuffer, copyback, &fenceFd);
            }
        } else {
            // if we can't copy-back anything, modify the user's dirty
            // region to make sure they redraw the whole buffer
            newDirtyRegion.set(bounds);
            mDirtyRegion.clear();
            Mutex::Autolock lock(mMutex);
            for (size_t i=0 ; i<NUM_BUFFER_SLOTS ; i++) {
                mSlots[i].dirtyRegion.clear();
            }
        }


        { // scope for the lock
            Mutex::Autolock lock(mMutex);
            int backBufferSlot(getSlotFromBufferLocked(backBuffer.get()));
            if (backBufferSlot >= 0) {
                Region& dirtyRegion(mSlots[backBufferSlot].dirtyRegion);
                mDirtyRegion.subtract(dirtyRegion);
                dirtyRegion = newDirtyRegion;
            }
        }

        mDirtyRegion.orSelf(newDirtyRegion);
        if (inOutDirtyBounds) {
            *inOutDirtyBounds = newDirtyRegion.getBounds();
        }

        void* vaddr;
        status_t res = backBuffer->lockAsync(
                GRALLOC_USAGE_SW_READ_OFTEN | GRALLOC_USAGE_SW_WRITE_OFTEN,
                newDirtyRegion.bounds(), &vaddr, fenceFd);

        ALOGW_IF(res, "failed locking buffer (handle = %p)",
                backBuffer->handle);

        if (res != 0) {
            err = INVALID_OPERATION;
        } else {
            mLockedBuffer = backBuffer;
            outBuffer->width  = backBuffer->width;
            outBuffer->height = backBuffer->height;
            outBuffer->stride = backBuffer->stride;
            outBuffer->format = backBuffer->format;
            outBuffer->bits   = vaddr;
        }
    }
    return err;
}
```
#### dequeueBuffer
```cpp

int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
    ATRACE_FORMAT("dequeueBuffer - %s", getDebugName());
    ALOGV("Surface::dequeueBuffer");

    IGraphicBufferProducer::DequeueBufferInput dqInput;
    {
        Mutex::Autolock lock(mMutex);
        if (mReportRemovedBuffers) {
            mRemovedBuffers.clear();
        }

        getDequeueBufferInputLocked(&dqInput);

        if (mSharedBufferMode && mAutoRefresh && mSharedBufferSlot !=
                BufferItem::INVALID_BUFFER_SLOT) {
            sp<GraphicBuffer>& gbuf(mSlots[mSharedBufferSlot].buffer);
            if (gbuf != nullptr) {
                *buffer = gbuf.get();
                *fenceFd = -1;
                return OK;
            }
        }
    } // Drop the lock so that we can still touch the Surface while blocking in IGBP::dequeueBuffer

    int buf = -1;
    sp<Fence> fence;
    nsecs_t startTime = systemTime();

    FrameEventHistoryDelta frameTimestamps;
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &mBufferAge,
                                                            dqInput.getTimestamps ?
                                                                    &frameTimestamps : nullptr);
    mLastDequeueDuration = systemTime() - startTime;

    if (result < 0) {
        ALOGV("dequeueBuffer: IGraphicBufferProducer::dequeueBuffer"
                "(%d, %d, %d, %#" PRIx64 ") failed: %d",
                dqInput.width, dqInput.height, dqInput.format, dqInput.usage, result);
        return result;
    }

    if (buf < 0 || buf >= NUM_BUFFER_SLOTS) {
        ALOGE("dequeueBuffer: IGraphicBufferProducer returned invalid slot number %d", buf);
        android_errorWriteLog(0x534e4554, "36991414"); // SafetyNet logging
        return FAILED_TRANSACTION;
    }

    Mutex::Autolock lock(mMutex);

    // Write this while holding the mutex
    mLastDequeueStartTime = startTime;

    sp<GraphicBuffer>& gbuf(mSlots[buf].buffer);

    // this should never happen
    ALOGE_IF(fence == nullptr, "Surface::dequeueBuffer: received null Fence! buf=%d", buf);

    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG_GRAPHICS))) {
        static gui::FenceMonitor hwcReleaseThread("HWC release");
        hwcReleaseThread.queueFence(fence);
    }

    if (result & IGraphicBufferProducer::RELEASE_ALL_BUFFERS) {
        freeAllBuffers();
    }

    if (dqInput.getTimestamps) {
         mFrameEventHistory->applyDelta(frameTimestamps);
    }

    if ((result & IGraphicBufferProducer::BUFFER_NEEDS_REALLOCATION) || gbuf == nullptr) {
        if (mReportRemovedBuffers && (gbuf != nullptr)) {
            mRemovedBuffers.push_back(gbuf);
        }
        result = mGraphicBufferProducer->requestBuffer(buf, &gbuf);
        if (result != NO_ERROR) {
            ALOGE("dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: %d", result);
            mGraphicBufferProducer->cancelBuffer(buf, fence);
            return result;
        }
    }

    if (fence->isValid()) {
        *fenceFd = fence->dup();
        if (*fenceFd == -1) {
            ALOGE("dequeueBuffer: error duping fence: %d", errno);
            // dup() should never fail; something is badly wrong. Soldier on
            // and hope for the best; the worst that should happen is some
            // visible corruption that lasts until the next frame.
        }
    } else {
        *fenceFd = -1;
    }

    *buffer = gbuf.get();

    if (mSharedBufferMode && mAutoRefresh) {
        mSharedBufferSlot = buf;
        mSharedBufferHasBeenQueued = false;
    } else if (mSharedBufferSlot == buf) {
        mSharedBufferSlot = BufferItem::INVALID_BUFFER_SLOT;
        mSharedBufferHasBeenQueued = false;
    }

    mDequeuedSlots.insert(buf);

    return OK;
}
```

###### mGraphicBufferProducer->dequeueBuffer
frameworks/native/libs/gui/BufferQueueProducer.cpp
```cpp 
```
###### mGraphicBufferProducer->requestBuffer

external/minigbm/cros_gralloc/mapper_stablec/Mapper.cpp
```cpp

AIMapper_Error CrosGrallocMapperV5::lock(buffer_handle_t _Nonnull bufferHandle, uint64_t cpuUsage,
                                         ARect region, int acquireFenceRawFd,
                                         void* _Nullable* _Nonnull outData) {
    // We take ownership of the FD in all cases, even for errors
    unique_fd acquireFence(acquireFenceRawFd);
    VALIDATE_DRIVER_AND_BUFFER_HANDLE(bufferHandle)
    if (cpuUsage == 0) {
        ALOGE("Failed to lock. Bad cpu usage: %" PRIu64 ".", cpuUsage);
        return AIMAPPER_ERROR_BAD_VALUE;
    }

    uint32_t mapUsage = cros_gralloc_convert_map_usage(cpuUsage);

    cros_gralloc_handle_t crosHandle = cros_gralloc_convert_handle(bufferHandle);
    if (crosHandle == nullptr) {
        ALOGE("Failed to lock. Invalid handle.");
        return AIMAPPER_ERROR_BAD_VALUE;
    }

    struct rectangle rect;

    // An access region of all zeros means the entire buffer.
    if (region.left == 0 && region.top == 0 && region.right == 0 && region.bottom == 0) {
        rect = {0, 0, crosHandle->width, crosHandle->height};
    } else {
        if (region.left < 0 || region.top < 0 || region.right <= region.left ||
            region.bottom <= region.top) {
            ALOGE("Failed to lock. Invalid accessRegion: [%d, %d, %d, %d]", region.left, region.top,
                  region.right, region.bottom);
            return AIMAPPER_ERROR_BAD_VALUE;
        }

        if (region.right > crosHandle->width) {
            ALOGE("Failed to lock. Invalid region: width greater than buffer width (%d vs %d).",
                  region.right, crosHandle->width);
            return AIMAPPER_ERROR_BAD_VALUE;
        }

        if (region.bottom > crosHandle->height) {
            ALOGE("Failed to lock. Invalid region: height greater than buffer height (%d vs "
                  "%d).",
                  region.bottom, crosHandle->height);
            return AIMAPPER_ERROR_BAD_VALUE;
        }

        rect = {static_cast<uint32_t>(region.left), static_cast<uint32_t>(region.top),
                static_cast<uint32_t>(region.right - region.left),
                static_cast<uint32_t>(region.bottom - region.top)};
    }

    uint8_t* addr[DRV_MAX_PLANES];
    int32_t status = mDriver->lock(bufferHandle, acquireFence.get(),
                                   /*close_acquire_fence=*/false, &rect, mapUsage, addr);
    if (status) {
        return AIMAPPER_ERROR_BAD_VALUE;
    }

    *outData = addr[0];
    return AIMAPPER_ERROR_NONE;
}
```

external/minigbm/cros_gralloc/cros_gralloc_driver.cc
```cpp
int32_t cros_gralloc_driver::lock(buffer_handle_t handle, int32_t acquire_fence,
				  bool close_acquire_fence, const struct rectangle *rect,
				  uint32_t map_flags, uint8_t *addr[DRV_MAX_PLANES])
{
	int32_t ret = cros_gralloc_sync_wait(acquire_fence, close_acquire_fence);
	if (ret)
		return ret;

	std::lock_guard<std::mutex> lock(mutex_);

	auto hnd = cros_gralloc_convert_handle(handle);
	if (!hnd) {
		ALOGE("Invalid handle.");
		return -EINVAL;
	}

	auto buffer = get_buffer(hnd);
	if (!buffer) {
		ALOGE("Invalid reference (lock() called on unregistered handle).");
		return -EINVAL;
	}

	return buffer->lock(rect, map_flags, addr);
}

cros_gralloc_buffer *cros_gralloc_driver::get_buffer(cros_gralloc_handle_t hnd)
{
	/* Assumes driver mutex is held. */
	if (handles_.count(hnd))
		return handles_[hnd].buffer;

	return nullptr;
}
```

handles_ 
```cpp
// external/minigbm/cros_gralloc/cros_gralloc_driver.h
class cros_gralloc_driver
{
      public:
	~cros_gralloc_driver();

	static std::shared_ptr<cros_gralloc_driver> get_instance();
	bool is_supported(const struct cros_gralloc_buffer_descriptor *descriptor);
	int32_t allocate(const struct cros_gralloc_buffer_descriptor *descriptor,
			 native_handle_t **out_handle);

	int32_t retain(buffer_handle_t handle);
	int32_t release(buffer_handle_t handle);

	int32_t lock(buffer_handle_t handle, int32_t acquire_fence, bool close_acquire_fence,
		     const struct rectangle *rect, uint32_t map_flags,
		     uint8_t *addr[DRV_MAX_PLANES]);
	int32_t unlock(buffer_handle_t handle, int32_t *release_fence);

	struct cros_gralloc_imported_handle_info {
		/*
		 * The underlying buffer for referred to by this handle (as multiple handles can
		 * refer to the same buffer).
		 */
		cros_gralloc_buffer *buffer = nullptr;

		/* The handle's refcount as a handle can be imported multiple times.*/
		int32_t refcount = 1;
	};

	std::unordered_map<cros_gralloc_handle_t, cros_gralloc_imported_handle_info> handles_;
};
```

buffer->lock
```cpp
external/minigbm/cros_gralloc/cros_gralloc_buffer.cc
int32_t cros_gralloc_buffer::lock(const struct rectangle *rect, uint32_t map_flags,
				  uint8_t *addr[DRV_MAX_PLANES])
{
	void *vaddr = nullptr;

	memset(addr, 0, DRV_MAX_PLANES * sizeof(*addr));

	if (map_flags) {
		if (lock_data_[0]) {
			drv_bo_invalidate(bo_, lock_data_[0]);
			vaddr = lock_data_[0]->vma->addr;
		} else {
			struct rectangle r = *rect;

			if (!r.width && !r.height && !r.x && !r.y) {
				/*
				 * Android IMapper.hal: An accessRegion of all-zeros means the
				 * entire buffer.
				 */
				r.width = drv_bo_get_width(bo_);
				r.height = drv_bo_get_height(bo_);
			}

			vaddr = drv_bo_map(bo_, &r, map_flags, &lock_data_[0], 0);
		}

		if (vaddr == MAP_FAILED) {
			ALOGE("Mapping failed.");
			return -EFAULT;
		}
	}

	for (uint32_t plane = 0; plane < hnd_->num_planes; plane++)
		addr[plane] = static_cast<uint8_t *>(vaddr) + drv_bo_get_plane_offset(bo_, plane);

	lockcount_++;
	return 0;
}
```
#### backBuffer->lockAsync

### canvas.setBuffer

```cpp
canvas->bitmap->buffer->bits
bool ACanvas_setBuffer(ACanvas* canvas, const ANativeWindow_Buffer* buffer,
                       int32_t /*android_dataspace_t*/ dataspace) {
    SkBitmap bitmap;
    bool isValidBuffer = (buffer == nullptr) ? false : convert(buffer, dataspace, &bitmap);
    TypeCast::toCanvas(canvas)->setBitmap(bitmap);
    return isValidBuffer;
}

/*
 * Converts a buffer and dataspace into an SkBitmap only if the resulting bitmap can be treated as a
 * rendering destination for a Canvas.  If the buffer is null or the format is one that we cannot
 * render into with a Canvas then false is returned and the outBitmap param is unmodified.
 */
static bool convert(const ANativeWindow_Buffer* buffer,
                    int32_t /*android_dataspace_t*/ dataspace,
                    SkBitmap* outBitmap) {
    if (buffer == nullptr) {
        return false;
    }

    sk_sp<SkColorSpace> cs(uirenderer::DataSpaceToColorSpace((android_dataspace)dataspace));
    SkImageInfo imageInfo = uirenderer::ANativeWindowToImageInfo(*buffer, cs);
    size_t rowBytes = buffer->stride * imageInfo.bytesPerPixel();

    // If SkSurface::MakeRasterDirect fails then we should as well as we will not be able to
    // draw into the canvas.
    sk_sp<SkSurface> surface = SkSurface::MakeRasterDirect(imageInfo, buffer->bits, rowBytes);
    if (surface.get() != nullptr) {
        if (outBitmap) {
            outBitmap->setInfo(imageInfo, rowBytes);
            outBitmap->setPixels(buffer->bits);
        }
        return true;
    }
    return false;
}
```

(canvas)->setBitmap
```cpp
SkCanvas::SkCanvas(const SkBitmap& bitmap,
                   std::unique_ptr<SkRasterHandleAllocator> alloc,
                   SkRasterHandleAllocator::Handle hndl,
                   const SkSurfaceProps* props)
        : fMCStack(sizeof(MCRec), fMCRecStorage, sizeof(fMCRecStorage))
        , fProps(SkSurfacePropsCopyOrDefault(props))
        , fAllocator(std::move(alloc)) {
    inc_canvas();

    sk_sp<SkBaseDevice> device(new SkBitmapDevice(bitmap, fProps, hndl));
    this->init(device);
}
```




## draw


external/skia/src/gpu/ganesh/SurfaceDrawContext.cpp
```cpp
// If we are using dmsaa or reduced shader mode then attempt to draw with FillRRectOp.
if (this->caps()->drawInstancedSupport() &&
    (this->alwaysAntialias() ||
        (fContext->priv().caps()->reducedShaderMode() && aa == GrAA::kYes))) {
    SkMatrix localMatrix = SkMatrix::MakeAll(p1.fX - p0.fX, ortho.fX, p0.fX,
                                                p1.fY - p0.fY, ortho.fY, p0.fY,
                                                0, 0, 1);
    if (auto op = FillRRectOp::Make(fContext,
                                    this->arenaAlloc(),
                                    std::move(paint),
                                    SkMatrix::Concat(viewMatrix, localMatrix),
                                    SkRRect::MakeRect({0,-1,1,1}),
                                    localMatrix,
                                    GrAA::kYes)) {
        this->addDrawOp(clip, std::move(op));
        return;
    }
}
```


## unlockCanvasAndPost
```java
private void unlockSwCanvasAndPost(Canvas canvas) {
    try {
        nativeUnlockCanvasAndPost(mLockedObject, canvas);
    } finally {
        nativeRelease(mLockedObject);
        mLockedObject = 0;
    }
}
```
frameworks/base/core/jni/android_view_Surface.cpp
```cpp

static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    if (!isSurfaceValid(surface)) {
        return;
    }

    // detach the canvas from the surface
    graphics::Canvas canvas(env, canvasObj);
    canvas.setBuffer(nullptr, ADATASPACE_UNKNOWN);

    // unlock surface
    status_t err = surface->unlockAndPost();
    if (err < 0) {
        jniThrowException(env, IllegalArgumentException, NULL);
    }
}
```

frameworks/native/libs/gui/Surface.cpp
```cpp
status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE("Surface::unlockAndPost failed, no locked buffer");
        return INVALID_OPERATION;
    }

    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd);
    ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);

    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
            mLockedBuffer->handle, strerror(-err));

    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}
```
### unlockAsync 解除内存映射

hardware/interfaces/graphics/mapper/stable-c/implutils/include/android/hardware/graphics/mapper/utils/IMapperProvider.h
```cpp
    .lock = [](buffer_handle_t _Nonnull buffer, uint64_t cpuUsage, ARect accessRegion,
                int acquireFence, void* _Nullable* _Nonnull outData) -> AIMapper_Error {
        return impl().lock(buffer, cpuUsage, accessRegion, acquireFence, outData);
    },

    .unlock = [](buffer_handle_t _Nonnull buffer, int* _Nonnull releaseFence)
            -> AIMapper_Error { return impl().unlock(buffer, releaseFence); },
```


```cpp
// external/minigbm/cros_gralloc/mapper_stablec/Mapper.cpp
AIMapper_Error CrosGrallocMapperV5::unlock(buffer_handle_t _Nonnull buffer,
                                           int* _Nonnull releaseFence) {
    VALIDATE_DRIVER_AND_BUFFER_HANDLE(buffer)
    int ret = mDriver->unlock(buffer, releaseFence);
    if (ret) {
        ALOGE("Failed to unlock.");
        return AIMAPPER_ERROR_BAD_BUFFER;
    }
    return AIMAPPER_ERROR_NONE;
}
```

```cpp
// external/minigbm/cros_gralloc/cros_gralloc_driver.cc
int32_t cros_gralloc_driver::unlock(buffer_handle_t handle, int32_t *release_fence)
{
	std::lock_guard<std::mutex> lock(mutex_);

	auto hnd = cros_gralloc_convert_handle(handle);
	if (!hnd) {
		ALOGE("Invalid handle.");
		return -EINVAL;
	}

	auto buffer = get_buffer(hnd);
	if (!buffer) {
		ALOGE("Invalid reference (unlock() called on unregistered handle).");
		return -EINVAL;
	}

	/*
	 * From the ANativeWindow::dequeueBuffer documentation:
	 *
	 * "A value of -1 indicates that the caller may access the buffer immediately without
	 * waiting on a fence."
	 */
	*release_fence = -1;
	return buffer->unlock();
}
```

```cpp
// external/minigbm/cros_gralloc/cros_gralloc_buffer.cc
int32_t cros_gralloc_buffer::unlock()
{
	if (lockcount_ <= 0) {
		ALOGE("Buffer was not locked.");
		return -EINVAL;
	}

	if (!--lockcount_) {
		if (lock_data_[0]) {
			drv_bo_flush_or_unmap(bo_, lock_data_[0]);
			lock_data_[0] = nullptr;
		}
	}

	return 0;
}
```

```cpp
// external/minigbm/drv.c
int drv_bo_flush_or_unmap(struct bo *bo, struct mapping *mapping)
{
	int ret = 0;

	assert(mapping);
	assert(mapping->vma);
	assert(mapping->refcount > 0);
	assert(mapping->vma->refcount > 0);
	assert(!(bo->meta.use_flags & BO_USE_PROTECTED));

	if (bo->drv->backend->bo_flush)
		ret = bo->drv->backend->bo_flush(bo, mapping);
	else
		ret = drv_bo_unmap(bo, mapping);

	return ret;
}
```
### queueBuffer
```cpp
// frameworks/native/libs/gui/Surface.cpp

int Surface::queueBuffer(android_native_buffer_t* buffer, int fenceFd) {
    ATRACE_CALL();
    ALOGV("Surface::queueBuffer");
    Mutex::Autolock lock(mMutex);

    int i = getSlotFromBufferLocked(buffer);
    if (i < 0) {
        if (fenceFd >= 0) {
            close(fenceFd);
        }
        return i;
    }
    if (mSharedBufferSlot == i && mSharedBufferHasBeenQueued) {
        if (fenceFd >= 0) {
            close(fenceFd);
        }
        return OK;
    }

    IGraphicBufferProducer::QueueBufferOutput output;
    IGraphicBufferProducer::QueueBufferInput input;
    getQueueBufferInputLocked(buffer, fenceFd, mTimestamp, &input);
    applyGrallocMetadataLocked(buffer, input);
    sp<Fence> fence = input.fence;

    nsecs_t now = systemTime();

    status_t err = mGraphicBufferProducer->queueBuffer(i, input, &output);
    mLastQueueDuration = systemTime() - now;
    if (err != OK)  {
        ALOGE("queueBuffer: error queuing buffer, %d", err);
    }

    onBufferQueuedLocked(i, fence, output);
    return err;
}
```

TODO: queueBuffer
```cpp

```


# drawHardware

android.view.ViewRootImpl#setView -> android.view.ViewRootImpl#enableHardwareAcceleration
```java
     mAttachInfo.mThreadedRenderer = ThreadedRenderer.create(mContext, translucent,
                        attrs.getTitle().toString());
```

```java
// android.graphics.HardwareRenderer#HardwareRenderer
/**
 * Creates a new instance of a HardwareRenderer. The HardwareRenderer will default
 * to opaque with no light source configured.
 */
public HardwareRenderer() {
    ProcessInitializer.sInstance.initUsingContext();
    mRootNode = RenderNode.adopt(nCreateRootRenderNode());
    mRootNode.setClipToBounds(false);
    mNativeProxy = nCreateProxy(!mOpaque, mRootNode.mNativeRenderNode);
    if (mNativeProxy == 0) {
        throw new OutOfMemoryError("Unable to create hardware renderer");
    }
    Cleaner.create(this, new DestroyContextRunnable(mNativeProxy));
    ProcessInitializer.sInstance.init(mNativeProxy);
}

/**
 * Adopts an existing native render node.
 *
 * Note: This will *NOT* incRef() on the native object, however it will
 * decRef() when it is destroyed. The caller should have already incRef'd it
 *
 * @hide
 */
public static RenderNode adopt(long nativePtr) {
    return new RenderNode(nativePtr);
}
```

```cpp
// frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp
static jlong android_view_ThreadedRenderer_createRootRenderNode(JNIEnv* env, jobject clazz) {
    RootRenderNode* node = new RootRenderNode(std::make_unique<JvmErrorReporter>(env));
    node->incStrong(0);
    node->setName("RootRenderNode");
    return reinterpret_cast<jlong>(node);
}

static jlong android_view_ThreadedRenderer_createProxy(JNIEnv* env, jobject clazz,
        jboolean translucent, jlong rootRenderNodePtr) {
    RootRenderNode* rootRenderNode = reinterpret_cast<RootRenderNode*>(rootRenderNodePtr);
    ContextFactoryImpl factory(rootRenderNode);
    RenderProxy* proxy = new RenderProxy(translucent, rootRenderNode, &factory);
    return (jlong) proxy;
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance()), mContext(nullptr) {
            // 单例初始化 mRenderThread 
    pid_t uiThreadId = pthread_gettid_np(pthread_self());
    pid_t renderThreadId = getRenderThreadTid();
    mContext = mRenderThread.queue().runSync([=, this]() -> CanvasContext* {
        // CanvasContext 是链接OpenGL/Vulkan和graphicBuffer缓冲区的关键
        CanvasContext* context = CanvasContext::create(mRenderThread, translucent, rootRenderNode,
                                                       contextFactory, uiThreadId, renderThreadId);
        if (context != nullptr) {
            mRenderThread.queue().post([=] { context->startHintSession(); });
        }
        return context;
    });
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode);
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
CanvasContext* CanvasContext::create(RenderThread& thread, bool translucent,
                                     RenderNode* rootRenderNode, IContextFactory* contextFactory,
                                     int32_t uiThreadId, int32_t renderThreadId) {
    auto renderType = Properties::getRenderPipelineType();

    switch (renderType) {
        case RenderPipelineType::SkiaGL:
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaOpenGLPipeline>(thread),
                                     uiThreadId, renderThreadId);
        case RenderPipelineType::SkiaVulkan:
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaVulkanPipeline>(thread),
                                     uiThreadId, renderThreadId);
        default:
            LOG_ALWAYS_FATAL("canvas context type %d not supported", (int32_t)renderType);
            break;
    }
    return nullptr;
}

CanvasContext::CanvasContext(RenderThread& thread, bool translucent, RenderNode* rootRenderNode,
                             IContextFactory* contextFactory,
                             std::unique_ptr<IRenderPipeline> renderPipeline, pid_t uiThreadId,
                             pid_t renderThreadId)
        : mRenderThread(thread)
        , mGenerationID(0)
        , mOpaque(!translucent)
        , mAnimationContext(contextFactory->createAnimationContext(mRenderThread.timeLord()))
        , mJankTracker(&thread.globalProfileData())
        , mProfiler(mJankTracker.frames(), thread.timeLord().frameIntervalNanos())
        , mContentDrawBounds(0, 0, 0, 0)
        , mRenderPipeline(std::move(renderPipeline))
        , mHintSessionWrapper(uiThreadId, renderThreadId) {
    mRenderThread.cacheManager().registerCanvasContext(this);
    rootRenderNode->makeRoot();
    mRenderNodes.emplace_back(rootRenderNode);
    mProfiler.setDensity(DeviceInfo::getDensity());
}
```

```cpp
// frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp
void DrawFrameTask::setContext(RenderThread* thread, CanvasContext* context,
                               RenderNode* targetNode) {
    mRenderThread = thread;
    mContext = context;
    mTargetNode = targetNode;
}
```

```java
// android.view.View#View(android.content.Context)
/**
 * Simple constructor to use when creating a view from code.
 *
 * @param context The Context the view is running in, through which it can
 *        access the current theme, resources, etc.
 */
public View(Context context) {
    mRenderNode = RenderNode.create(getClass().getName(), new ViewAnimationHostBridge(this));
}

```

```java
// android.view.ThreadedRenderer#draw
/**
 * Draws the specified view.
 *
 * @param view The view to draw.
 * @param attachInfo AttachInfo tied to the specified view.
 */
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    attachInfo.mViewRootImpl.mViewFrameInfo.markDrawStart();

    updateRootDisplayList(view, callbacks);

    final FrameInfo frameInfo = attachInfo.mViewRootImpl.getUpdatedFrameInfo();

    int syncResult = syncAndDrawFrame(frameInfo);
}

private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}

private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    updateViewTreeDisplayList(view);

    // Consume and set the frame callback after we dispatch draw to the view above, but before
    // onPostDraw below which may reset the callback for the next frame.  This ensures that
    // updates to the frame callback during scroll handling will also apply in this frame.
    if (mNextRtFrameCallbacks != null) {
        final ArrayList<FrameDrawingCallback> frameCallbacks = mNextRtFrameCallbacks;
        mNextRtFrameCallbacks = null;
        setFrameCallback(new FrameDrawingCallback() {
            @Override
            public void onFrameDraw(long frame) {
            }

            @Override
            public FrameCommitCallback onFrameDraw(int syncResult, long frame) {
                ArrayList<FrameCommitCallback> frameCommitCallbacks = new ArrayList<>();
                for (int i = 0; i < frameCallbacks.size(); ++i) {
                    FrameCommitCallback frameCommitCallback = frameCallbacks.get(i)
                            .onFrameDraw(syncResult, frame);
                    if (frameCommitCallback != null) {
                        frameCommitCallbacks.add(frameCommitCallback);
                    }
                }

                if (frameCommitCallbacks.isEmpty()) {
                    return null;
                }

                return didProduceBuffer -> {
                    for (int i = 0; i < frameCommitCallbacks.size(); ++i) {
                        frameCommitCallbacks.get(i).onFrameCommit(didProduceBuffer);
                    }
                };
            }
        });
    }

    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onPreDraw(canvas);

            canvas.enableZ();
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            canvas.disableZ();

            callbacks.onPostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
            mRootNode.endRecording();
        }
    }
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}
```

```java
// frameworks/base/core/java/android/view/View.java
/**
 * Gets the RenderNode for the view, and updates its DisplayList (if needed and supported)
 * @hide
 */
@NonNull
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
public RenderNode updateDisplayListIfDirty() {
    final RenderNode renderNode = mRenderNode;
    if (!canHaveDisplayList()) {
        // can't populate RenderNode, don't try
        return renderNode;
    }

    if ((mPrivateFlags & PFLAG_DRAWING_CACHE_VALID) == 0
            || !renderNode.hasDisplayList()
            || (mRecreateDisplayList)) {
        // Don't need to recreate the display list, just need to tell our
        // children to restore/recreate theirs
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            dispatchGetDisplayList();

            return renderNode; // no work needed
        }

        // If we got here, we're recreating it. Mark it as such to ensure that
        // we copy in child display lists into ours in drawChild()
        mRecreateDisplayList = true;

        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();

        // Hacky hack: Reset any stretch effects as those are applied during the draw pass
        // instead of being "stateful" like other RenderNode properties
        renderNode.clearStretch();

        final RecordingCanvas canvas = renderNode.beginRecording(width, height);

        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
                buildDrawingCache(true);
                Bitmap cache = getDrawingCache(true);
                if (cache != null) {
                    canvas.drawBitmap(cache, 0, 0, mLayerPaint);
                }
            } else {
                computeScroll();

                canvas.translate(-mScrollX, -mScrollY);
                mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
                mPrivateFlags &= ~PFLAG_DIRTY_MASK;

                // Fast path for layouts with no backgrounds
                if ((mPrivateFlags & PFLAG_SKIP_DRAW) == PFLAG_SKIP_DRAW) {
                    dispatchDraw(canvas);
                    drawAutofilledHighlight(canvas);
                    if (mOverlay != null && !mOverlay.isEmpty()) {
                        mOverlay.getOverlayView().draw(canvas);
                    }
                    if (isShowingLayoutBounds()) {
                        debugDrawFocus(canvas);
                    }
                } else {
                    draw(canvas);
                }
            }
        } finally {
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}
```

```java
// frameworks/base/graphics/java/android/graphics/RenderNode.java
/**
 * Starts recording a display list for the render node. All
 * operations performed on the returned canvas are recorded and
 * stored in this display list.
 *
 * {@link #endRecording()} must be called when the recording is finished in order to apply
 * the updated display list. Failing to call {@link #endRecording()} will result in an
 * {@link IllegalStateException} if {@link #beginRecording(int, int)} is called again.
 *
 * @param width  The width of the recording viewport. This will not alter the width of the
 *               RenderNode itself, that must be set with {@link #setPosition(Rect)}.
 * @param height The height of the recording viewport. This will not alter the height of the
 *               RenderNode itself, that must be set with {@link #setPosition(Rect)}.
 * @return A canvas to record drawing operations.
 * @throws IllegalStateException If a recording is already in progress. That is, the previous
 * call to {@link #beginRecording(int, int)} did not call {@link #endRecording()}.
 * @see #endRecording()
 * @see #hasDisplayList()
 */
public @NonNull RecordingCanvas beginRecording(int width, int height) {
    if (mCurrentRecordingCanvas != null) {
        throw new IllegalStateException(
                "Recording currently in progress - missing #endRecording() call?");
    }
    mCurrentRecordingCanvas = RecordingCanvas.obtain(this, width, height);
    return mCurrentRecordingCanvas;
}
```

```java
// frameworks/base/graphics/java/android/graphics/RecordingCanvas.java
/*package*/
static RecordingCanvas obtain(@NonNull RenderNode node, int width, int height) {
    if (node == null) throw new IllegalArgumentException("node cannot be null");
    RecordingCanvas canvas = sPool.acquire();
    if (canvas == null) {
        canvas = new RecordingCanvas(node, width, height);
    } else {
        nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, node.mNativeRenderNode,
                width, height);
    }
    canvas.mNode = node;
    canvas.mWidth = width;
    canvas.mHeight = height;
    return canvas;
}

private RecordingCanvas(@NonNull RenderNode node, int width, int height) {
    super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
    mDensity = 0; // disable bitmap density scaling
}
```

```cpp
// frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(CRITICAL_JNI_PARAMS_COMMA jlong renderNodePtr,
        jint width, jint height) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}
```

```cpp
// frameworks/base/libs/hwui/hwui/Canvas.cpp
Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
    return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
}
```

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.h
/**
 * A SkiaCanvas implementation that records drawing operations for deferred rendering backed by a
 * SkLiteRecorder and a SkiaDisplayList.
 */
class SkiaRecordingCanvas : public SkiaCanvas {
public:
    explicit SkiaRecordingCanvas(uirenderer::RenderNode* renderNode, int width, int height) {
        initDisplayList(renderNode, width, height);
    }
}
```

```cpp
// frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp
void SkiaRecordingCanvas::initDisplayList(uirenderer::RenderNode* renderNode, int width,
                                          int height) {
    mCurrentBarrier = nullptr;
    LOG_FATAL_IF(mDisplayList.get() != nullptr);

    if (renderNode) {
        mDisplayList = renderNode->detachAvailableList();
    }
    if (!mDisplayList) {
        mDisplayList.reset(new SkiaDisplayList());
    }

    mDisplayList->attachRecorder(&mRecorder, SkIRect::MakeWH(width, height));
    SkiaCanvas::reset(&mRecorder);
    mDisplayList->setHasHolePunches(false);
}
```