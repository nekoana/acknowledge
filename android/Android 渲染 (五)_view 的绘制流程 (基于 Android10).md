> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7133878536414167076)

一、 概述
=====

继续上篇，在 ViewRootImpl 的`performTraversals()`中，执行了 relayoutWindow() 后，mSurface 就已经和 native 层的 Surface 对象建立起了联系。接下来就是 `测量、布局、绘制`三大流程。

在测量、布局后，`那么绘制怎么执行的呢？`

二、 引出两种渲染方式
===========

2.1 ViewRootImpl.performTraversals()
------------------------------------

> ViewRootImpl.java

```
private void performTraversals() {
    // 调用 relayoutWindow 
    relayoutResult = relayoutWindow(params, viewVisibility, insetsPending);
    //...
    
    // Ask host how big it wants to be
    // 三大流程
    performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
    //...
    performLayout(lp, mWidth, mHeight);
    //``
    if (mAttachInfo.mThreadedRenderer != null) {
        // 如果硬件绘制开启，初始化
        hwInitialized = mAttachInfo.mThreadedRenderer.initialize(
                mSurface);
        if (hwInitialized && (host.mPrivateFlags
                & View.PFLAG_REQUEST_TRANSPARENT_REGIONS) == 0) {
            //  想SF请求GraphicBuffer
            mAttachInfo.mThreadedRenderer.allocateBuffers();
        }
    }
    //继续调用performDraw()
    performDraw();
    //...
}

// 
private void performDraw() {
    //...
     try {
       // 继续调用draw()
      boolean canUseAsync = draw(fullRedrawNeeded);
      
    } finally {
        mIsDrawing = false;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
    }
}

复制代码
```

2.2 ViewRootImpl.draw()
-----------------------

> ViewRootImpl.java

```
private boolean draw(boolean fullRedrawNeeded) {
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            //...
            if (invalidateRoot) {
                mAttachInfo.mThreadedRenderer.invalidateRoot();
            }
            dirty.setEmpty();
            //...
            // 硬件渲染
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            // ...
            // 软件渲染
            if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                    scalingRequired, dirty, surfaceInsets)) {
                return false;
            }
        }

    }
}
复制代码
```

有两种绘制方式：`硬件和软件绘制`。 先看看软件绘制：

三、软件绘制
======

3.1 drawSoftware()
------------------

```
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    try{ 
        // Draw with software renderer.
         final Canvas canvas;
        // 获取canvas 
        canvas = mSurface.lockCanvas(dirty);
    
        // TODO: Do this in native
        canvas.setDensity(mDensity);
        // ...
         dirty.setEmpty();
        // 开始view的绘制流程
         mView.draw(canvas);
    }finally {
        // 最终绘制完成，确保提交    
        surface.unlockCanvasAndPost(canvas);
           
    }
        return true;
}
复制代码
```

1.  通过 Surface 获取 Canvas 对象。 `Canvas看做是一个工具集`，它负责把图形数据写入到 Bitmap 中。包含了绘图用的`各种命令和工具`，同时还包含了`画纸Bitmap`。
2.  拿到 canvas 后，从根 View 开始绘制流程。
3.  绘制完成后，把结果提交

3.2 Surface.lockCanvas()
------------------------

> Surface.java

```
public Canvas lockCanvas(Rect inOutDirty)
        throws Surface.OutOfResourcesException, IllegalArgumentException {
    synchronized (mLock) {
        checkNotReleasedLocked();
        if (mLockedObject != 0) {
            throw new IllegalArgumentException("Surface was already locked");
        }
        // 调用的是native方法 ,传入了native引用、mCanvas成员变量
        mLockedObject = nativeLockCanvas(mNativeObject, mCanvas, inOutDirty);
        return mCanvas;
    }
}
    
>frameworks/base/core/jni/android_view_Surface.cpp
static jlong nativeLockCanvas(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj, jobject dirtyRectObj) {
        
    // 得到c层的 surface 对象    
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));

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

    //声明 buffer
    ANativeWindow_Buffer buffer;
    // 获取GraphicBuffer 
    status_t err = surface->lock(&buffer, dirtyRectPtr);

    // 创建 canvas对象,并且与java层的联系起来
    graphics::Canvas canvas(env, canvasObj);
    // 设置buffer
    canvas.setBuffer(&buffer, static_cast<int32_t>(surface->getBuffersDataSpace()));

    if (dirtyRectPtr) {
        canvas.clipRect({dirtyRect.left, dirtyRect.top, dirtyRect.right, dirtyRect.bottom});
    }

  

    // Create another reference to the surface and return it.  This reference
    // should be passed to nativeUnlockCanvasAndPost in place of mNativeObject,
    // because the latter could be replaced while the surface is locked.
    // 创建新的 surface对象 返回给java层。
    sp<Surface> lockedSurface(surface);
    
    lockedSurface->incStrong(&sRefBaseOwner);
    // 返回 
    return (jlong) lockedSurface.get();
}

复制代码
```

1.  获得之前创建的 native 层 Surface 对象
2.  调用 Surface 的 lock() 方法，传入 ANativeWindow_Buffer
3.  创建 canvas 对象，设置 buffer

### 3.2.0 ANativeWindow_Buffer 结构体

> frameworks/native/libs/nativewindow/include/android/native_window.h

`ANativeWindow_Buffer` 用来接收 surface.lock() 返回回来的内存地址、宽高等信息。后面的 SKBitmap 的`setPixels()`方法就是和 该结构体的 `bits指针` 绑定。

```
/**
 * Struct that represents a windows buffer.
 * A pointer can be obtained using {@link ANativeWindow_lock()}.
 */
typedef struct ANativeWindow_Buffer {
    /// The number of pixels that are shown horizontally.
    int32_t width;

    /// The number of pixels that are shown vertically.
    int32_t height;

    /// The number of *pixels* that a line in the buffer takes in
    /// memory. This may be >= width.
    int32_t stride;

    /// The format of the buffer. One of AHardwareBuffer_Format.
    int32_t format;

    /// The actual bits.， 这个就是实际的内存地址，也就是bitmap存放数据的地方。
    void* bits;

    /// Do not touch.
    uint32_t reserved[6];
} ANativeWindow_Buffer;

复制代码
```

我们注意到 Surface 中有一个 mCanvas 成员对象。先看看 Canvas 的构造流程。

### 3.2.1 Canvas 创建过程

```
> Surface.java

private final Canvas mCanvas = new CompatibleCanvas();
// CompatibleCanvas是 Surface的内部类 
private final class CompatibleCanvas extends Canvas {...


> Canvas.java  空参构造 
public Canvas() {
    if (!isHardwareAccelerated()) { // 软件绘制才开启？ 
        // 0 means no native bitmap
        // 0 表示native层没有bitmap，也就是空画布，不占内存。
        //nInitRaster是一个native方法 
        mNativeCanvasWrapper = nInitRaster(0);
        
        mFinalizer = NoImagePreloadHolder.sRegistry.registerNativeAllocation(
                this, mNativeCanvasWrapper);
    } else {
        mFinalizer = null;
    }
}

>frameworks/base/libs/hwui/jni/android_graphics_Canvas.cpp

 {"nInitRaster", "(J)J", (void*) CanvasJNI::initRaster},
 
 // Native wrapper constructor used by Canvas(Bitmap)
static jlong initRaster(JNIEnv* env, jobject, jlong bitmapHandle) {
    SkBitmap bitmap;
    // 我们在上面传入了0，因此不进入
    if (bitmapHandle != 0) {
        bitmap::toBitmap(bitmapHandle).getSkBitmap(&bitmap);
    }
    // 调用 create_canvas()，返回的是一个 SkiaCanvas 对象，
    return reinterpret_cast<jlong>(Canvas::create_canvas(bitmap));
}


>frameworks/base/libs/hwui/SkiaCanvas.cpp
Canvas* Canvas::create_canvas(const SkBitmap& bitmap) {
    return new SkiaCanvas(bitmap);
}

复制代码
```

总结：

Java 层的 Canvas 对应 native 层的 SkiaCanvas 对象。内部通过 SKBitmap 来承载绘制内容。

3.3 Surface.lock()
------------------

> frameworks/native/libs/gui/Surface.cpp

```
status_t Surface::lock(ANativeWindow_Buffer* outBuffer, ARect* inOutDirtyBounds)
{
    if (mLockedBuffer != nullptr) {
        ALOGE("Surface::lock failed, already locked");
        return INVALID_OPERATION;
    }

    
    // 声明 ANativeWindowBuffer 对象
    ANativeWindowBuffer* out;
    int fenceFd = -1;
    //1  调用dequeueBuffer获取 ANativeWindowBuffer 对象
    status_t err = dequeueBuffer(&out, &fenceFd);
    // 2 创建GraphicBuffer 对象
    sp<GraphicBuffer> backBuffer(GraphicBuffer::getSelf(out));
    
    // ...
    void* vaddr;
    //3  锁定 GraphicBuffer
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
    
    return err;
}

复制代码
```

1.  调用 dequeueBuffer 获取 ANativeWindowBuffer 对象
2.  创建 GraphicBuffer 对象
3.  锁定 GraphicBuffer

### 3.3.1 dequeueBuffer()

> frameworks/native/libs/gui/Surface.cpp

```
int Surface::dequeueBuffer(android_native_buffer_t** buffer, int* fenceFd) {
    ATRACE_CALL();
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
            // 如果开启共享buffer模式    
             // 从共享的位置mSharedBufferSlot，拿出GraphicBuffer，
            sp<GraphicBuffer>& gbuf(mSlots[mSharedBufferSlot].buffer);
            if (gbuf != nullptr) {
                 // 不为空，则返回
                *buffer = gbuf.get();
                *fenceFd = -1;
                return OK;
            }
        }
    } // Drop the lock so that we can still touch the Surface while blocking in IGBP::dequeueBuffer

    int buf = -1; //传递过去到SF，返回mSlots的下标值
    sp<Fence> fence;
    nsecs_t startTime = systemTime();

    FrameEventHistoryDelta frameTimestamps;
    // 跨进程 从SF的 BufferQueue获取 GraphicBuffer
    status_t result = mGraphicBufferProducer->dequeueBuffer(&buf, &fence, dqInput.width,
                                                            dqInput.height, dqInput.format,
                                                            dqInput.usage, &mBufferAge,
                                                            dqInput.getTimestamps ?
                                                                    &frameTimestamps : nullptr);
    mLastDequeueDuration = systemTime() - startTime;

    //NUM_BUFFER_SLOTS=64 , 不能为负数，也不能大于等于64
    if (buf < 0 || buf >= NUM_BUFFER_SLOTS) {
        ALOGE("dequeueBuffer: IGraphicBufferProducer returned invalid slot number %d", buf);
        android_errorWriteLog(0x534e4554, "36991414"); // SafetyNet logging
        return FAILED_TRANSACTION;
    }

    Mutex::Autolock lock(mMutex);

    // Write this while holding the mutex
    mLastDequeueStartTime = startTime;
    // 得到GraphicBuffer
    sp<GraphicBuffer>& gbuf(mSlots[buf].buffer);

    // this should never happen
    ALOGE_IF(fence == nullptr, "Surface::dequeueBuffer: received null Fence! buf=%d", buf);

    if (CC_UNLIKELY(atrace_is_tag_enabled(ATRACE_TAG_GRAPHICS))) {
        static FenceMonitor hwcReleaseThread("HWC release");
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
        //请求 buffer
        result = mGraphicBufferProducer->requestBuffer(buf, &gbuf);
        if (result != NO_ERROR) {
            ALOGE("dequeueBuffer: IGraphicBufferProducer::requestBuffer failed: %d", result);
            mGraphicBufferProducer->cancelBuffer(buf, fence);
            return result;
        }
    }
    // 赋值 buffer
    *buffer = gbuf.get();
    // 如果这个位置的buffer可以共享
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
复制代码
```

重点：

1.  mGraphicBufferProducer->dequeueBuffer((&buf...)，从 SF 进程中返回一个`下标值`，对应`mSlots`中的位置。此时`还没有分配内存`。
2.  mGraphicBufferProducer->requestBuffer(buf, &gbuf); 向 SF 进程请求分配对应 mSlots 位置的`GraphicBuffer内存空间`。

`mSlots 又是什么？`

### 3.3.2 mSlots 是什么？

```
>frameworks/native/libs/gui/include/gui/Surface.h
 BufferSlot mSlots[NUM_BUFFER_SLOTS];
 
 // BufferSlot是一个结构体 
 struct BufferSlot {
        sp<GraphicBuffer> buffer;
        Region dirtyRegion;
    };
 
 >frameworks/native/libs/ui/include/ui/BufferQueueDefs.h
   static constexpr int NUM_BUFFER_SLOTS = 64;
   
   
复制代码
```

在 Surface 中有成员变量 mSlots 来存储获取到的 GraphicBuffer 对象。GraphicBuffer 就是表示内存缓冲区。 而与之对应的在 SF 进程中的 BufferQueueProducer.cpp 中也有对应 mSlots 数组。内部就存储着 GraphicBuffer 内存缓冲区。 `真正 分配内存是在SF进程中完成的`。App 进程只是映射到了对应的内存。

由于 APP 进程与 SF 进程通过 `匿名共享内存来实现共享GraphicBuffer缓冲区`。 因此，app 进程只要通过 mGraphicBufferProducer 就能把数据写入到 SF 进程提供的 GraphicBuffer 内存空间中。

在获取到 GraphicBuffer 缓冲区后，`对canvas工具箱设置"画布"`。

3.4 Canvas.setBuffer()
----------------------

> frameworks/base/libs/hwui/apex/android_canvas.cpp

```
bool ACanvas_setBuffer(ACanvas* canvas, const ANativeWindow_Buffer* buffer,
                       int32_t /*android_dataspace_t*/ dataspace) {
    //skBitmap表示是 skia绘制引擎的类
    SkBitmap bitmap; 
    // convert 将buffer对应的缓冲区与 bitmap 绑定起来
    bool isValidBuffer = (buffer == nullptr) ? false : convert(buffer, dataspace, &bitmap);
    // 设置canvas的btimap，调用的是 SkiaCanvas 的setBitmap()
    TypeCast::toCanvas(canvas)->setBitmap(bitmap);
    return isValidBuffer;
}

> frameworks/base/libs/hwui/SkiaCanvas.cpp
void SkiaCanvas::setBitmap(const SkBitmap& bitmap) {
    // deletes the previously owned canvas (if any)
    // 具体实现就是 SkCanvas对象。
    mCanvasOwned.reset(new SkCanvas(bitmap));
    mCanvas = mCanvasOwned.get();

    // clean up the old save stack
    mSaveStack.reset(nullptr);
}

// 将 buffer 和 bitmap 绑定
>frameworks/base/libs/hwui/apex/android_canvas.cpp
/*
 * Converts a buffer and dataspace into an SkBitmap 
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
            
            // 关键： 将bits指针设置给SKBitmap
            outBitmap->setPixels(buffer->bits);
        }
        return true;
    }
    return false;
}

复制代码
```

*   经过 Surface 的 lock()，使得 SkiaCanvas 的 mCanvas 有了 SKBitmap 对象。
*   SKBitmap 与 ANativeWindow_Buffer 中的 `bits指针`完成绑定，也就是说`SKBitmap`指向了分配好的内存空间。
*   `SkiaCanvas是与Java层的 Canvas 对象`绑定的，至此，Java 层的 Canvas 终于不是空对象了，有了 bitmap 可以使用了。

我们以 drawLines() 接口来看看内部的流程。

### 3.4.1 Canvas.drawLines() 流程

```
>Canvas.java
public void drawLines(@Size(multiple = 4) @NonNull float[] pts, int offset, int count,
        @NonNull Paint paint) {
    super.drawLines(pts, offset, count, paint);
}
>BaseCanvas.java
public void drawLines(@Size(multiple = 4) @NonNull float[] pts, int offset, int count,
        @NonNull Paint paint) {
    throwIfHasHwBitmapInSwMode(paint);
    // native 方法，最后一个参数 paint的native层引用
    nDrawLines(mNativeCanvasWrapper, pts, offset, count, paint.getNativeInstance());
}
复制代码
```

### 3.4.2 android_graphics_Canvas.drawLines()

```
>frameworks/base/libs/hwui/jni/android_graphics_Canvas.cpp
  {"nDrawLines", "(J[FIIJ)V", (void*) CanvasJNI::drawLines},
  
static void drawLines(JNIEnv* env, jobject, jlong canvasHandle, jfloatArray jptsArray,
                      jint offset, jint count, jlong paintHandle) {
    //得到native层的 paint对象
    const Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    // 获取 SKiaCanvas 对象，调用drawLines()
    get_canvas(canvasHandle)->drawLines(floats + offset, count, *paint);
}

>frameworks/base/libs/hwui/SkiaCanvas.cpp
void SkiaCanvas::drawLines(const float* points, int count, const Paint& paint) {
    if (CC_UNLIKELY(count < 4 || paint.nothingToDraw())) return;
    this->drawPoints(points, count, paint, SkCanvas::kLines_PointMode);
}
复制代码
```

### 3.4.3 SkiaCanvas.drawPoints()

```
void SkiaCanvas::drawPoints(const float* points, int count, const Paint& paint,
                            SkCanvas::PointMode mode) {
    std::unique_ptr<SkPoint[]> pts(new SkPoint[count]);
    for (int i = 0; i < count; i++) {
        pts[i].set(points[0], points[1]);
        points += 2;
    }

    applyLooper(&paint, [&](const SkPaint& p) {
    // 调用 SkCanvas 的drawPoints()
     mCanvas->drawPoints(mode, count, pts.get(), p); 
     });
}
复制代码
```

### 3.4.4 SKCanvas.drawPoints()

```
> external/skia/src/core/SkCanvas.cpp
void SkCanvas::drawPoints(PointMode mode, size_t count, const SkPoint pts[], const SkPaint& paint) {
    TRACE_EVENT0("skia", TRACE_FUNC);
    this->onDrawPoints(mode, count, pts, paint);
}


void SkCanvas::onDrawPoints(PointMode mode, size_t count, const SkPoint pts[],
                            const SkPaint& paint) {
    if ((long)count <= 0 || paint.nothingToDraw()) {
        return;
    }
    SkASSERT(pts != nullptr);

    SkRect bounds;
    // Compute bounds from points (common for drawing a single line)
    if (count == 2) {
        bounds.set(pts[0], pts[1]);
    } else {
        bounds.setBounds(pts, SkToInt(count));
    }

    // Enforce paint style matches implicit behavior of drawPoints
    SkPaint strokePaint = paint;
    strokePaint.setStyle(SkPaint::kStroke_Style);
    if (this->internalQuickReject(bounds, strokePaint)) {
        return;
    }

    AutoLayerForImageFilter layer(this, strokePaint, &bounds);
    // 获取device实现类 开始绘制
    this->topDevice()->drawPoints(mode, count, pts, layer.paint());
}
复制代码
```

#### 3.4.5 topDevice.drawPoints()

```
// device的实现类有两个 ：
 
>external/skia/src/core/SkBitmapDevice.cpp
void SkBitmapDevice::drawPaint(const SkPaint& paint) {
    BDDraw(this).drawPaint(paint);
}

>external/skia/src/gpu/SkGpuDevice.cpp
void SkGpuDevice::drawPoints(SkCanvas::PointMode mode,
                             size_t count, const SkPoint pts[], const SkPaint& paint) {
    ASSERT_SINGLE_OWNER
    GR_CREATE_TRACE_MARKER_CONTEXT("SkGpuDevice", "drawPoints", fContext.get());
    SkScalar width = paint.getStrokeWidth();
    if (width < 0) {
        return;
    }

    GrAA aa = fSurfaceDrawContext->chooseAA(paint);

    if (paint.getPathEffect() && 2 == count && SkCanvas::kLines_PointMode == mode) {
        GrStyle style(paint, SkPaint::kStroke_Style);
        GrPaint grPaint;
        if (!SkPaintToGrPaint(this->recordingContext(), fSurfaceDrawContext->colorInfo(), paint,
                              this->asMatrixProvider(), &grPaint)) {
            return;
        }
        SkPath path;
        path.setIsVolatile(true);
        path.moveTo(pts[0]);
        path.lineTo(pts[1]);
        fSurfaceDrawContext->drawPath(this->clip(), std::move(grPaint), aa, this->localToDevice(),
                                      path, style);
        return;
    }

    SkScalar scales[2];
    bool isHairline = (0 == width) ||
                       (1 == width && this->localToDevice().getMinMaxScales(scales) &&
                        SkScalarNearlyEqual(scales[0], 1.f) && SkScalarNearlyEqual(scales[1], 1.f));
    // we only handle non-antialiased hairlines and paints without path effects or mask filters,
    // else we let the SkDraw call our drawPath()
    if (!isHairline || paint.getPathEffect() || paint.getMaskFilter() || aa == GrAA::kYes) {
        SkRasterClip rc(this->devClipBounds());
        SkDraw draw;
        draw.fDst = SkPixmap(SkImageInfo::MakeUnknown(this->width(), this->height()), nullptr, 0);
        draw.fMatrixProvider = this;
        draw.fRC = &rc;
        draw.drawPoints(mode, count, pts, paint, this);
        return;
    }

    GrPrimitiveType primitiveType = point_mode_to_primitive_type(mode);

    const SkMatrixProvider* matrixProvider = this;
#ifdef SK_BUILD_FOR_ANDROID_FRAMEWORK
    SkTLazy<SkPostTranslateMatrixProvider> postTranslateMatrixProvider;
    // This offsetting in device space matches the expectations of the Android framework for non-AA
    // points and lines.
    if (GrIsPrimTypeLines(primitiveType) || GrPrimitiveType::kPoints == primitiveType) {
        static const SkScalar kOffset = 0.063f; // Just greater than 1/16.
        matrixProvider = postTranslateMatrixProvider.init(*matrixProvider, kOffset, kOffset);
    }
#endif

    GrPaint grPaint;
    if (!SkPaintToGrPaint(this->recordingContext(), fSurfaceDrawContext->colorInfo(), paint,
                          *matrixProvider, &grPaint)) {
        return;
    }

    static constexpr SkVertices::VertexMode kIgnoredMode = SkVertices::kTriangles_VertexMode;
    sk_sp<SkVertices> vertices = SkVertices::MakeCopy(kIgnoredMode, SkToS32(count), pts, nullptr,
                                                      nullptr);

    fSurfaceDrawContext->drawVertices(this->clip(), std::move(grPaint), *matrixProvider,
                                      std::move(vertices), &primitiveType);
}

复制代码
```

最后，通过 skia/OpenGL 的 api 来绘制完成的。当绘制完成后，还需要调用 `Surface的unlockCanvasAndPost()`来提交才能显示。

3.5 软件绘制提交
----------

### 3.5.1 Surface.unlockCanvasAndPost()

```
> Surface.java
/**
 * Posts the new contents of the {@link Canvas} to the surface and
 * releases the {@link Canvas}.
 * @param canvas The canvas previously obtained from {@link #lockCanvas}.
 */
public void unlockCanvasAndPost(Canvas canvas) {
    synchronized (mLock) {
        checkNotReleasedLocked();

        if (mHwuiContext != null) {
            mHwuiContext.unlockAndPost(canvas);
        } else {
            // 软件提交
            unlockSwCanvasAndPost(canvas);
        }
    }
}

复制代码
```

### 3.5.2 unlockSwCanvasAndPost()

> Surface.java

```
private long mLockedObject;
 
private void unlockSwCanvasAndPost(Canvas canvas) {
    if (canvas != mCanvas) {
        throw new IllegalArgumentException("canvas object must be the same instance that "
                + "was previously returned by lockCanvas");
    }
    if (mNativeObject != mLockedObject) {
        Log.w(TAG, "WARNING: Surface's mNativeObject (0x" +
                Long.toHexString(mNativeObject) + ") != mLockedObject (0x" +
                Long.toHexString(mLockedObject) +")");
    }
    if (mLockedObject == 0) {
        throw new IllegalStateException("Surface was not locked");
    }
    try {
        // 调用native方法，传入 上锁的surface引用，和canvas对象
        nativeUnlockCanvasAndPost(mLockedObject, canvas);
    } finally {
        nativeRelease(mLockedObject);
        mLockedObject = 0;
    }
}

>frameworks/base/core/jni/android_view_Surface.cpp
static void nativeUnlockCanvasAndPost(JNIEnv* env, jclass clazz,
        jlong nativeObject, jobject canvasObj) {
    // 得到native层绘制之前锁定的 Surface对象    
    sp<Surface> surface(reinterpret_cast<Surface *>(nativeObject));
    if (!isSurfaceValid(surface)) {
        return;
    }

    // detach the canvas from the surface
    //获取之前的canvas工具箱 
    Canvas* nativeCanvas = GraphicsJNI::getNativeCanvas(env, canvasObj);
    // 设置一个空的bitmap，释放canvas的画布SKBitmap。也就是说此时SKBitmap不在引用到buffer了。
    nativeCanvas->setBitmap(SkBitmap());
    
    //此时已经绘制完了吗？是的！

    // unlock surface
    // 调用surface的 unlockAndPost() 
    status_t err = surface->unlockAndPost();
    if (err < 0) {
        doThrowIAE(env);
    }
}

>frameworks/native/libs/gui/Surface.cpp
status_t Surface::unlockAndPost()
{
    if (mLockedBuffer == nullptr) {
        ALOGE("Surface::unlockAndPost failed, no locked buffer");
        return INVALID_OPERATION;
    }

    int fd = -1;
    status_t err = mLockedBuffer->unlockAsync(&fd); //解除buffer的绑定
    ALOGE_IF(err, "failed unlocking buffer (%p)", mLockedBuffer->handle);
    // 加入队列 
    err = queueBuffer(mLockedBuffer.get(), fd);
    ALOGE_IF(err, "queueBuffer (handle=%p) failed (%s)",
            mLockedBuffer->handle, strerror(-err));

    mPostedBuffer = mLockedBuffer;
    mLockedBuffer = nullptr;
    return err;
}

复制代码
```

*   清空 Canvas 的 SKBitmap 画布，`解除与buffer`的引用。
*   `解除surface的buffer与 SF进程的GraphicBuffer的绑定`。
*   将 SF 的 GraphicBuffer`入队BufferQueue`, 请求下一个 vsync 信号，通知 SF 来进行合成消费。

至此，App 的软件绘制工作已经完成。 我们在来看看硬件绘制。

3.6 小结
------

1.  调用 Surface 的 lock() 方法，native 层内部调用 mGraphicBufferProducer->dequeueBuffer() 获得缓冲区 GraphicBuffer。 此时，Java 层 Surface 中的 Canvas 对象是空的，没有与任何 Bitmap 绑定的。
2.  生成一个 SKBimap 对象，与 GraphicBuffer 绑定，本质上共享一块内存 Buffer。设置给 native 层的 Canvas 对象。至此，java 层的 Canvas 才有了画布。
3.  调用 SKCanvas 提供的 API 开始绘制，最终会调用 Skia 来完成绘制
4.  调用 Surface 的 unlockAndPost() 来解除与 GraphicBuffer 的绑定。内部清空 Cnavas 的 Bitmap，解除 SKBitmap 与 GraphicBuffer 的绑定。
5.  通过 mGraphicBufferProducer->queueBuffer() 入队 SF 的 BufferQueue，请求下一次 vsync 信号，通知 SF 来进行消费合成。

四、硬件绘制
======

包括 `构建` 和`绘制`阶段两个分开的独立阶段。

4.1 ViewRootImpl.draw()
-----------------------

> ViewRootImpl.java

```
private boolean draw(boolean fullRedrawNeeded) {
    if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
        if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
            // 硬件渲染
            mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this);
        } else {
            // 软件渲染
        }

    }
}
复制代码
```

在开始之前我们先熟悉一些概念。`方便后续理解`。

4.1.1 ThreadedRenderer
----------------------

继承关系：

> ThreadedRenderer.java

```
public final class ThreadedRenderer extends HardwareRenderer {...
public class HardwareRenderer {
复制代码
```

**什么时候赋值的呢？**

ViewRootImpl 的`setView()`会调用 `enableHardwareAcceleration()`：

```
>ViewRootImpl.java 
private void enableHardwareAcceleration(WindowManager.LayoutParams attrs) {
     // ...
    
      else if (!ThreadedRenderer.sRendererDisabled
                    || (ThreadedRenderer.sSystemRendererDisabled && forceHwAccelerated)) {
     // 进程默认 开启硬件绘制         
     mAttachInfo.mThreadedRenderer = ThreadedRenderer.create(mContext, translucent,
                        attrs.getTitle().toString());
     // ...
     }                   
}


>ThreadedRenderer.java 
  public static ThreadedRenderer create(Context context, boolean translucent, String name) {
        ThreadedRenderer renderer = null;
        if (isAvailable()) {
            renderer = new ThreadedRenderer(context, translucent, name);
        }
        return renderer;
    }
 // 构造方法 
 ThreadedRenderer(Context context, boolean translucent, String name) {
        // 调用父类的构造
        super();
        setName(name);
        setOpaque(!translucent);

        // ...
    }
    
    
>HardwareRenderer.java 
//     父类的构造
 public HardwareRenderer() {
 
        // 创建一个Java层的 根RootNode 节点
        mRootNode = RenderNode.adopt(nCreateRootRenderNode());
        mRootNode.setClipToBounds(false);
        
        // 创建一个 native层的 RenderProxy 对象,  用来和RenderThread线程通信
        mNativeProxy = nCreateProxy(!mOpaque, mRootNode.mNativeRenderNode);
        if (mNativeProxy == 0) {
            throw new OutOfMemoryError("Unable to create hardware renderer");
        }
        Cleaner.create(this, new DestroyContextRunnable(mNativeProxy));
        
        // 往AMS设置 renderThread的线程tid
        ProcessInitializer.sInstance.init(mNativeProxy);
    }

  public static RenderNode adopt(long nativePtr) {
        //创建Java层的RenderNode, 持有native层的node 引用 
        return new RenderNode(nativePtr);
    }
    
    
复制代码
```

1.  在 Java 层和 native 层同时创建 `RootRenderNode` 节点，Java 层持有 native 的引用
2.  创建 native 层的 `RenderProxy` 对象，用来和`RenderThread线程`通信
3.  往 AMS 设置 renderThread 的线程 tid

#### 4.1.1.1 nCreateRootRenderNode()

> frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp

```
static jlong android_view_ThreadedRenderer_createRootRenderNode(JNIEnv* env, jobject clazz) {
    // 创建 native层的 node对象
    RootRenderNode* node = new RootRenderNode(std::make_unique<JvmErrorReporter>(env));
    node->incStrong(0);
    // 名字：RootRenderNode
    node->setName("RootRenderNode");
    return reinterpret_cast<jlong>(node);
}

复制代码
```

#### 4.1.1.2. nCreateProxy()

> frameworks/base/libs/hwui/jni/android_graphics_HardwareRenderer.cpp

```
static jlong android_view_ThreadedRenderer_createProxy(JNIEnv* env, jobject clazz,
        jboolean translucent, jlong rootRenderNodePtr) {
     // 获取之前创建的  rootRenderNode
    RootRenderNode* rootRenderNode = reinterpret_cast<RootRenderNode*>(rootRenderNodePtr);
    // 创建 ContextFactoryImpl 对象
    ContextFactoryImpl factory(rootRenderNode);
    // new 一个 RenderProxy对象 。 translucent：是否半透明
    RenderProxy* proxy = new RenderProxy(translucent, rootRenderNode, &factory);
    return (jlong) proxy;
}

// RenderProxy 的构造函数
>frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
RenderProxy::RenderProxy(bool translucent, RenderNode* rootRenderNode,
                         IContextFactory* contextFactory)
        : mRenderThread(RenderThread::getInstance()), mContext(nullptr) {
        // mRenderThread 赋值。 RenderThread是一个单例线程。因此一个应用只会拥有一个RenderThread线程
        
    mContext = mRenderThread.queue().runSync([&]() -> CanvasContext* {
        // 在RenderThread线程中，创建CanvasContext对象。这个对象很重要！！是链接OpenGL/Vulkan和graphicBuffer缓冲区的关键
        return CanvasContext::create(mRenderThread, translucent, rootRenderNode, contextFactory);
    });
    // mDrawFrameTask 设置context 
    mDrawFrameTask.setContext(&mRenderThread, mContext, rootRenderNode,
                              pthread_gettid_np(pthread_self()), getRenderThreadTid());
}

>frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
CanvasContext* CanvasContext::create(RenderThread& thread, bool translucent,
                                     RenderNode* rootRenderNode, IContextFactory* contextFactory) {
    // 根据渲染管道的配置
    auto renderType = Properties::getRenderPipelineType();

    switch (renderType) {
        case RenderPipelineType::SkiaGL:
            // skiaOpenGl渲染管道
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaOpenGLPipeline>(thread));
        case RenderPipelineType::SkiaVulkan:
            // skiaVulkan渲染管道
            return new CanvasContext(thread, translucent, rootRenderNode, contextFactory,
                                     std::make_unique<skiapipeline::SkiaVulkanPipeline>(thread));
        default:
            LOG_ALWAYS_FATAL("canvas context type %d not supported", (int32_t)renderType);
            break;
    }
    return nullptr;
}

复制代码
```

### 4.1.1.3 小结

ThreadedRenderer 在 ViewRootImpl 的 setView() 中被初始化。构造方法做了如下初始化：

1.  在构造方法中会创建 Java 和 native 层两个 根 RenderNode 节点。
2.  创建 native 层的 RenderProxy 对象，持有 RenderThread 单例线程的引用。
3.  给 RenderThread 线程，设置渲染的上下文。根据配置该 CanvasContext 采用的是管道：SkiaOpenGLPipeline 或者 SkiaVulkanPipeline。

### 4.1.2 RenderNode

在每个 view 的构造方法中都会生成一个 RenderNode 对象。

```
>View.java
public View(Context context) {
        mContext = context;
        mResources = context != null ? context.getResources() : null;
        // ...
        
        // 生成一个 RenderNode节点
        mRenderNode = RenderNode.create(getClass().getName(), new ViewAnimationHostBridge(this));
        
}

> RenderNode.java
/** @hide */
public static RenderNode create(String name, @Nullable AnimationHost animationHost) {
    return new RenderNode(name, animationHost);
}

//构造方法 
 private RenderNode(String name, AnimationHost animationHost) {
        // 调用native方法 生成
        mNativeRenderNode = nCreate(name);
        NoImagePreloadHolder.sRegistry.registerNativeAllocation(this, mNativeRenderNode);
        mAnimationHost = animationHost;
    }


>frameworks/base/libs/hwui/jni/android_graphics_RenderNode.cpp
static jlong android_view_RenderNode_create(JNIEnv* env, jobject, jstring name) {
    // 创建一个 renderNode节点 
    RenderNode* renderNode = new RenderNode();
    renderNode->incStrong(0);
    if (name != NULL) {
        const char* textArray = env->GetStringUTFChars(name, NULL);
        renderNode->setName(textArray);
        env->ReleaseStringUTFChars(name, textArray);
    }
    return reinterpret_cast<jlong>(renderNode);
}
复制代码
```

> RenderNode 是根据 view 树来建立的。每一个 View 对应一个 RenderNode 节点。一个 RenderNode 节点包含了当前 view 的绘制命令 drawOp(如 drawLines->drawLinesOp), 同时还包括绘制子 RenderNode 节点的命令：DrawRenderNodeOp。因此，可看成一颗 RenderNode 树。 每一个 drawXXXOp 命令都有对应的 OpenGL/Vulkan 命令与之对应。

### 4.1.3 RecordingCanvas

RecordingCanvas 用来记录 View 树 中的硬件加速绘制动作 drawOp。对应 native 层的 SkiaRecordingCanvas。 主要功能都是由 native 来完成。 通过 obtain() 方法来获得一个 RecordingCanvas

### 4.1.4 obtain()

```
static RecordingCanvas obtain(@NonNull RenderNode node, int width, int height) {
   // 必须要跟一个renderNode绑定
  if (node == null) throw new IllegalArgumentException("node cannot be null");
  RecordingCanvas canvas = sPool.acquire();
  if (canvas == null) {
      // 构造新的对象，传入了node、宽、高
      canvas = new RecordingCanvas(node, width, height);
  } else {
      nResetDisplayListCanvas(canvas.mNativeCanvasWrapper, node.mNativeRenderNode,
              width, height);
  }
  // 记录当前node、宽、高信息
  canvas.mNode = node;
  canvas.mWidth = width;
  canvas.mHeight = height;
  return canvas;
}
 
 
 // 构造方法 
protected RecordingCanvas(@NonNull RenderNode node, int width, int height) {
     // 调用父类 DisplayListCanvas 的构造方法 ，一直调用父类，知道canvas类的 赋值到成员变量： long mNativeCanvasWrapper;
     super(nCreateDisplayListCanvas(node.mNativeRenderNode, width, height));
     mDensity = 0; // disable bitmap density scaling
 } 
 
// native 构造
>frameworks/base/libs/hwui/jni/android_graphics_DisplayListCanvas.cpp (新版本的源码路径)
>frameworks/base/core/jni/android_view_DisplayListCanvas.cpp( Android10 路径)
static jlong android_view_DisplayListCanvas_createDisplayListCanvas(CRITICAL_JNI_PARAMS_COMMA jlong renderNodePtr,
        jint width, jint height) {
        // 拿到 node对象
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    //
    return reinterpret_cast<jlong>(Canvas::create_recording_canvas(width, height, renderNode));
}

// canvas.cpp 
>frameworks/base/libs/hwui/hwui/Canvas.cpp
Canvas* Canvas::create_recording_canvas(int width, int height, uirenderer::RenderNode* renderNode) {
   //返回一个 SkiaRecordingCanvas 对象，跟java层的RecordingCanvas 对应
    return new uirenderer::skiapipeline::SkiaRecordingCanvas(renderNode, width, height);
} 
复制代码
```

**回过头来，继续分析** `ThreadedRenderer的 draw()`方法：

4.2 ThreadedRenderer.draw()
---------------------------

```
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    
    // 拿到ViewRootImpl中的 Choreographer
    final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
    choreographer.mFrameInfo.markDrawStart();
    // 构建View的 drawXXXOp树 
    updateRootDisplayList(view, callbacks);

    // ...
    
    // 通知 RenderThread  线程 开始绘制
    int syncResult = syncAndDrawFrame(choreographer.mFrameInfo);
    
     // ...
}

复制代码
```

1.  构建 root view tree 的 drawOp 树，也就是 displayList 绘制命令
2.  通知 RenderThread 线程 开始绘制

4.3 [构建阶段] ThreadedRenderer.updateRootDisplayList()
---------------------------------------------------

```
> ThreadedRenderer.java 
private void updateRootDisplayList(View view, DrawCallbacks callbacks) {
    Trace.traceBegin(Trace.TRACE_TAG_VIEW, "Record View#draw()");
    // 更新传入view对应的 RenderNode中的displayList
    updateViewTreeDisplayList(view);

    // ...
    // 根节点需要更新，并且拥有drawOp命令
    if (mRootNodeNeedsUpdate || !mRootNode.hasDisplayList()) {
        // 从RecordingCanvas缓存池中，获取一个 RecordingCanvas 对象，包含了当前RenderNode、width、height信息
        RecordingCanvas canvas = mRootNode.beginRecording(mSurfaceWidth, mSurfaceHeight);
        try {
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onPreDraw(canvas);

            canvas.enableZ();
            // 返回view对应的node，开始绘制RenderNode
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            canvas.disableZ();

            callbacks.onPostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
            // 结束绘制，加入到Nodes集合 
            mRootNode.endRecording();
        }
    }
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
}

>RenderNode.java
public @NonNull RecordingCanvas beginRecording(int width, int height) {
    // 一个 RenderNode节点 只能有一个 RecordingCanvas对象 
    if (mCurrentRecordingCanvas != null) {
        throw new IllegalStateException(
                "Recording currently in progress - missing #endRecording() call?");
    }
    mCurrentRecordingCanvas = RecordingCanvas.obtain(this, width, height);
    return mCurrentRecordingCanvas;
}

复制代码
```

### 4.3.1 updateViewTreeDisplayList()

> ThreadedRenderer.java

```
private void updateViewTreeDisplayList(View view) {
    view.mPrivateFlags |= View.PFLAG_DRAWN;
    view.mRecreateDisplayList = (view.mPrivateFlags & View.PFLAG_INVALIDATED)
            == View.PFLAG_INVALIDATED;
    view.mPrivateFlags &= ~View.PFLAG_INVALIDATED;
    // 调用了view的 updateDisplayListIfDirty
    view.updateDisplayListIfDirty();
    view.mRecreateDisplayList = false;
}

复制代码
```

### 4.3.2 view.updateDisplayListIfDirty()

获取当前 view 的 RenderNode，同时更新 displayList

```
public RenderNode updateDisplayListIfDirty() {
    // 拿到对应的 renderNode
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
        // 如果当前view的缓存可用，则重用drawOp展示列表。告知子view去重建displayList
        if (renderNode.hasDisplayList()
                && !mRecreateDisplayList) {
            mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
            mPrivateFlags &= ~PFLAG_DIRTY_MASK;
            // 分发子view去 重建displayList
            dispatchGetDisplayList();

            return renderNode; // no work needed
        }

        // If we got here, we're recreating it. Mark it as such to ensure that
        // we copy in child display lists into ours in drawChild()
        mRecreateDisplayList = true;

        int width = mRight - mLeft;
        int height = mBottom - mTop;
        int layerType = getLayerType();
        // 到这里表示需要重建，开始记录drawOp
        final RecordingCanvas canvas = renderNode.beginRecording(width, height);

        try {
            if (layerType == LAYER_TYPE_SOFTWARE) {
               // 如果当前view拥有LAYER_TYPE_SOFTWARE类型，及时开了硬件加速，也只会使用
               //Android软件渲染管道来绘制。
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
                    // 如果自身不需要绘制，则分发到子view，让子view去完成绘制
                    dispatchDraw(canvas);
                    
                    drawAutofilledHighlight(canvas);
                    if (mOverlay != null && !mOverlay.isEmpty()) {
                        mOverlay.getOverlayView().draw(canvas);
                    }
                    if (debugDraw()) {
                        debugDrawFocus(canvas);
                    }
                } else {
                    // 如果自身也需要，开始按照绘制顺序执行 
         //  *      1. Draw the background
         //*      2. If necessary, save the canvas' layers to prepare for fading
         //*      3. Draw view's content
         //*      4. Draw children
         //*      5. If necessary, draw the fading edges and restore layers
         //*      6. Draw decorations (scrollbars for instance)
         
                    draw(canvas);
                }
            }
        } finally {
            // 结束记录drawOp
            renderNode.endRecording();
            setDisplayListProperties(renderNode);
        }
    } else {
        mPrivateFlags |= PFLAG_DRAWN | PFLAG_DRAWING_CACHE_VALID;
        mPrivateFlags &= ~PFLAG_DIRTY_MASK;
    }
    return renderNode;
}

// draw 方法 
public void draw(Canvas canvas) {
     final int privateFlags = mPrivateFlags;
     mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;

     /*
      * Draw traversal performs several drawing steps which must be executed
      * in the appropriate order:
      *
      *      1. Draw the background
      *      2. If necessary, save the canvas' layers to prepare for fading
      *      3. Draw view's content
      *      4. Draw children
      *      5. If necessary, draw the fading edges and restore layers
      *      6. Draw decorations (scrollbars for instance)
      */

     // Step 1, draw the background, if needed
     int saveCount;

     drawBackground(canvas);

     // skip step 2 & 5 if possible (common case)
     final int viewFlags = mViewFlags;
     boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;
     boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;
     if (!verticalEdges && !horizontalEdges) {
         // Step 3, draw the content
         onDraw(canvas);

         // Step 4, draw the children
         dispatchDraw(canvas);

         drawAutofilledHighlight(canvas);

         // Overlay is part of the content and draws beneath Foreground
         if (mOverlay != null && !mOverlay.isEmpty()) {
             mOverlay.getOverlayView().dispatchDraw(canvas);
         }

         // Step 6, draw decorations (foreground, scrollbars)
         onDrawForeground(canvas);

         // Step 7, draw the default focus highlight
         drawDefaultFocusHighlight(canvas);

         if (debugDraw()) {
             debugDrawFocus(canvas);
         }

         // we're done...
         return;
     }
// ... 

}       




复制代码
```

1.  获取成员变量对应的 `mRootNode`
2.  调用`node.beginRecording()`开始记录，得到 java 层的 RecordingCanvas 对象, 同时得到 native 层的 SkiaRecordingCanvas 对象
3.  当前 view 开始执行 draw(canvas)，以及自己的子 view 的 draw(canvas) 方法，最终都是调用 canvas 的 api，记录到`RecordingCanvas`中
4.  调用`node.endRecording()`结束绘制，内部其实调用的是`RecordingCanvas`的 finishRecording()，结束记录。

### 4.3.3 RenderNode.endRecording()

```
> RenderNode.java 
public void endRecording() {
    if (mCurrentRecordingCanvas == null) {
        throw new IllegalStateException(
                "No recording in progress, forgot to call #beginRecording()?");
    }
    RecordingCanvas canvas = mCurrentRecordingCanvas;
    mCurrentRecordingCanvas = null;
    // 1 结束绘制动作记录，调用到native，返回native层的SkiaDisplayList对象的引用
    long displayList = canvas.finishRecording();
    // 2 设置？？
    nSetDisplayList(mNativeRenderNode, displayList);
    canvas.recycle();
}
复制代码
```

### 4.3.3.1 RecordingCanvas.finishRecording()

> RecordingCanvas.java

```
> RecordingCanvas.java 
/** @hide */
long finishRecording() {
    // jni方法 
    return nFinishRecording(mNativeCanvasWrapper);
}

>frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static jlong android_view_DisplayListCanvas_finishRecording(jlong canvasPtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
     // 最终调用canvas的 finishRecording()
    return reinterpret_cast<jlong>(canvas->finishRecording());
}

// 此处的canvas对应的是 SkiaRecordingCanvas：

>frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp
std::unique_ptr<SkiaDisplayList> SkiaRecordingCanvas::finishRecording() {
    // close any existing chunks if necessary
    enableZ(false);
    mRecorder.restoreToCount(1);
    // 返回 SkiaDisplayList 的引用
    return std::move(mDisplayList);
}

>
>frameworks/base/libs/hwui/pipeline/skia/SkiaDisplayList.h
 /**
     * We use std::deque here because (1) we need to iterate through these
     * elements and (2) mDisplayList holds pointers to the elements, so they
     * cannot relocate.
     */
    std::deque<RenderNodeDrawable> mChildNodes; 
    std::deque<FunctorDrawable*> mChildFunctors;
    std::vector<SkImage*> mMutableImages;

复制代码
```

`mDisplayList` 是 SkiaDisplayList 对象，内部保存了对应的绘制绘制动作。返回引用 long 到 java 层。

至此，node 调用 endRecording()，内部最终调用的是 native 层的 SkiaRecordingCanvas 的 finishRecording() 方法，返回 `displayList对象的引用给java层`。

我们继续看看 RenderNode 中的 nSetDisplayList(mNativeRenderNode, displayList)。displayList 就是上一步放回的引用。

### 4.3.3.2 RenderNode.nSetDisplayList()

```
> frameworks/base/core/jni/android_view_RenderNode.cpp
static void android_view_RenderNode_setDisplayList(JNIEnv* env,
        jobject clazz, jlong renderNodePtr, jlong displayListPtr) {
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    DisplayList* newData = reinterpret_cast<DisplayList*>(displayListPtr);
    renderNode->setStagingDisplayList(newData);
}

> frameworks/base/libs/hwui/RenderNode.cpp
void RenderNode::setStagingDisplayList(DisplayList* displayList) {
      //
    mValid = (displayList != nullptr);
    mNeedsDisplayListSync = true;
    //释放之前的 displayList
    delete mStagingDisplayList;
    // 赋值新的 displayList
    mStagingDisplayList = displayList;
}

复制代码
```

对应 native 层的 RnderNode 中的 displayList 进行了更新！

4.4 构建小结:
---------

1.  通过 ThreadedRenderer.updateRootDisplayList()，根据 view 树来构建对应的 RenderNode 树 (一个 view 对应一个 node)。每个 node 中包含了当前 view 的绘制指令，如 drawXXXop，如果有子 view 还会包含 drawRenderNodeOp 命令。 这些 Op 又称为统称为 displayList。
    
2.  调用`node.beginRecording()`开始记录，得到 java 层的 RecordingCanvas 对象, 同时得到 native 层的 SkiaRecordingCanvas 对象
    
    *   当前 view 开始执行 draw(canvas)，以及自己的子 view 的 draw(canvas) 方法，最终都是调用 canvas 的 api，记录到`RecordingCanvas`中
3.  调用`node.endRecording()`结束绘制， RenderNode.endRecording() 做了两件事：
    
    *   调用 native 层 SkiaRecordingCanvas 的 finishRecording()，结束记录，`同时返回displayList的引用到java层`
    *   根据返回的 displayList 引用，把 native 层的 displayList 设置到 `node中的成员 mStagingDisplayList中`。

至此，所有的绘制指令都存储到了对应的 node 中。

**疑问：**

**但是，这些绘制了是怎么生成的呢？**

再回过头来，具体绘制如何构建的呢？ 看看 `canvas.drawRenderNode()`方法

4.5 drawRenderNode()
--------------------

```
/**
 * Draws the specified display list onto this canvas.
 *
 * @param renderNode The RenderNode to draw.
 */
@Override
public void drawRenderNode(@NonNull RenderNode renderNode) {
     // 其实就是调用native的 canvas->drawRenderNode(...
    nDrawRenderNode(mNativeCanvasWrapper, renderNode.mNativeRenderNode);
}

>frameworks/base/core/jni/android_view_DisplayListCanvas.cpp
static void android_view_DisplayListCanvas_drawRenderNode(jlong canvasPtr, jlong renderNodePtr) {
    Canvas* canvas = reinterpret_cast<Canvas*>(canvasPtr);
    RenderNode* renderNode = reinterpret_cast<RenderNode*>(renderNodePtr);
    canvas->drawRenderNode(renderNode);
}

>frameworks/base/libs/hwui/pipeline/skia/SkiaRecordingCanvas.cpp
void SkiaRecordingCanvas::drawRenderNode(uirenderer::RenderNode* renderNode) {

    // Record the child node. Drawable dtor will be invoked when mChildNodes deque is cleared.
    // mChildNodes 是 std::deque<RenderNodeDrawable>类型。 
    //把renderNode转换成drawable对象，创建一个 RenderNodeDrawable对象，存入队列中 
    mDisplayList->mChildNodes.emplace_back(renderNode, asSkCanvas(), true, mCurrentBarrier);
    // 取出 RenderNodeDrawable 对象
    auto& renderNodeDrawable = mDisplayList->mChildNodes.back();
    
    if (Properties::getRenderPipelineType() == RenderPipelineType::SkiaVulkan) {
        // Put Vulkan WebViews with non-rectangular clips in a HW layer
        renderNode->mutateStagingProperties().setClipMayBeComplex(mRecorder.isClipMayBeComplex());
    }
    // 开始绘制 
    drawDrawable(&renderNodeDrawable);

    // use staging property, since recording on UI thread
    if (renderNode->stagingProperties().isProjectionReceiver()) {
        mDisplayList->mProjectionReceiver = &renderNodeDrawable;
    }
}

## RenderNodeDrawable 封装了一个node对象，让node可以被记录为 一系列的 skia 绘制命令。

Android中的各种drawable都是啥？  draw是绘制意思，那么drawable就是可以绘制的东西~


// 开始绘制
>frameworks/base/libs/hwui/SkiaCanvas.h
void drawDrawable(SkDrawable* drawable) { 
   //mCanvas 是 SKCanvas对象
   mCanvas->drawDrawable(drawable); 
}

// 此时已经到了skia引擎库
>external/skia/src/core/SkCanvas.cpp
void SkCanvas::drawDrawable(SkDrawable* dr, const SkMatrix* matrix) {
#ifndef SK_BUILD_FOR_ANDROID_FRAMEWORK
    TRACE_EVENT0("skia", TRACE_FUNC);
#endif
    RETURN_ON_NULL(dr);
    if (matrix && matrix->isIdentity()) {
        matrix = nullptr;
    }
    // 继续调用
    this->onDrawDrawable(dr, matrix);
}

void SkCanvas::onDrawDrawable(SkDrawable* dr, const SkMatrix* matrix) {
    // drawable bounds are no longer reliable (e.g. android displaylist)
    // so don't use them for quick-reject
    if (this->predrawNotify()) {
        // 
        this->topDevice()->drawDrawable(this, dr, matrix);
    }
}

SkBaseDevice* SkCanvas::topDevice() const {
    SkASSERT(fMCRec->fDevice);
    return fMCRec->fDevice;
}

topDevice() 返回的是 SkDevice对象。

>external/skia/src/core/SkDevice.cpp  
void SkBaseDevice::drawDrawable(SkCanvas* canvas, SkDrawable* drawable, const SkMatrix* matrix) {
    drawable是 SKDrawable类型，上面传入的具体实现类是 RenderNodeDrawable 
    drawable->draw(canvas, matrix);
}

> external/skia/src/core/SkDrawable.cpp
void SkDrawable::draw(SkCanvas* canvas, const SkMatrix* matrix) {
    SkAutoCanvasRestore acr(canvas, true);
    if (matrix) {
      //结合skia的 matrix 
        canvas->concat(*matrix);
    }
    // onDraw是一个抽象方法，具体实现的是xxxDrawable，而这里传入的是 RenderNodeDrawable
    this->onDraw(canvas);

    if ((false)) {
        draw_bbox(canvas, this->getBounds());
    }
}


RenderNodeDrawable
>frameworks/base/libs/hwui/pipeline/skia/RenderNodeDrawable.cpp
void RenderNodeDrawable::onDraw(SkCanvas* canvas) {
    // negative and positive Z order are drawn out of order, if this render node drawable is in
    // a reordering section
    if ((!mInReorderingSection) || MathUtils::isZero(mRenderNode->properties().getZ())) {
        this->forceDraw(canvas);
    }
}

void RenderNodeDrawable::forceDraw(SkCanvas* canvas) const {
    RenderNode* renderNode = mRenderNode.get();
    MarkDraw _marker{*canvas, *renderNode};

    // We only respect the nothingToDraw check when we are composing a layer. This
    // ensures that we paint the layer even if it is not currently visible in the
    // event that the properties change and it becomes visible.
    if ((mProjectedDisplayList == nullptr && !renderNode->isRenderable()) ||
        (renderNode->nothingToDraw() && mComposeLayer)) {
        return;
    }

    SkiaDisplayList* displayList = renderNode->getDisplayList().asSkiaDl();

    SkAutoCanvasRestore acr(canvas, true);
    const RenderProperties& properties = this->getNodeProperties();
    // pass this outline to the children that may clip backward projected nodes
    displayList->mProjectedOutline =
            displayList->containsProjectionReceiver() ? &properties.getOutline() : nullptr;
    if (!properties.getProjectBackwards()) {
        drawContent(canvas);
        if (mProjectedDisplayList) {
            acr.restore();  // draw projected children using parent matrix
            LOG_ALWAYS_FATAL_IF(!mProjectedDisplayList->mProjectedOutline);
            const bool shouldClip = mProjectedDisplayList->mProjectedOutline->getPath();
            SkAutoCanvasRestore acr2(canvas, shouldClip);
            canvas->setMatrix(mProjectedDisplayList->mParentMatrix);
            if (shouldClip) {
                canvas->clipPath(*mProjectedDisplayList->mProjectedOutline->getPath());
            }
            drawBackwardsProjectedNodes(canvas, *mProjectedDisplayList);
        }
    }
    displayList->mProjectedOutline = nullptr;
}
复制代码
```

也就就是说，各种绘制都会转换成 skia 绘制命令。

### 4.5.1 RenderNodeDrawable

```
This drawable wraps a RenderNode and enables it to be recorded into a list of Skia drawing commands.
class RenderNodeDrawable : public SkDrawable {
...

复制代码
```

4.6 看看 drawLine() 接口
--------------------

```
public void drawLine(float startX, float startY, float stopX, float stopY,
      @NonNull Paint paint) {
  super.drawLine(startX, startY, stopX, stopY, paint);
}

public void drawLine(float startX, float startY, float stopX, float stopY,
      @NonNull Paint paint) {
  throwIfHasHwBitmapInSwMode(paint);
  nDrawLine(mNativeCanvasWrapper, startX, startY, stopX, stopY, paint.getNativeInstance());
}

private static native void nDrawLine(long nativeCanvas, float startX, float startY, float stopX,
         float stopY, long nativePaint);
     
     
>frameworks/base/core/jni/android_graphics_Canvas.cpp     
static void drawLine(JNIEnv* env, jobject, jlong canvasHandle, jfloat startX, jfloat startY,
                     jfloat stopX, jfloat stopY, jlong paintHandle) {
    // 画笔
    Paint* paint = reinterpret_cast<Paint*>(paintHandle);
    // 硬件绘制java层的canvas 对应的就是SkiaRecordingCanvas
    get_canvas(canvasHandle)->drawLine(startX, startY, stopX, stopY, *paint);
}  

>frameworks/base/libs/hwui/SkiaCanvas.cpp
void SkiaCanvas::drawLine(float startX, float startY, float stopX, float stopY,
                          const Paint& paint) {
    applyLooper(&paint,
                [&](const SkPaint& p) { mCanvas->drawLine(startX, startY, stopX, stopY, p); });
}


>frameworks/base/libs/hwui/SkiaCanvas.cpp
void SkiaCanvas::drawLines(const float* points, int count, const Paint& paint) {
    if (CC_UNLIKELY(count < 4 || paint.nothingToDraw())) return;
    this->drawPoints(points, count, paint, SkCanvas::kLines_PointMode);
}

>external/skia/src/core/SkCanvas.cpp
void SkCanvas::drawLine(SkScalar x0, SkScalar y0, SkScalar x1, SkScalar y1, const SkPaint& paint) {
    SkPoint pts[2];
    pts[0].set(x0, y0);
    pts[1].set(x1, y1);
    this->drawPoints(kLines_PointMode, 2, pts, paint);
}

void SkCanvas::drawPoints(PointMode mode, size_t count, const SkPoint pts[], const SkPaint& paint) {
    TRACE_EVENT0("skia", TRACE_FUNC);
    this->onDrawPoints(mode, count, pts, paint);
}

void SkCanvas::onDrawPoints(PointMode mode, size_t count, const SkPoint pts[],
                            const SkPaint& paint) {
    if ((long)count <= 0 || paint.nothingToDraw()) {
        return;
    }
    SkASSERT(pts != nullptr);

    SkRect bounds;
    // Compute bounds from points (common for drawing a single line)
    if (count == 2) {
        bounds.set(pts[0], pts[1]);
    } else {
        bounds.setBounds(pts, SkToInt(count));
    }

    // Enforce paint style matches implicit behavior of drawPoints
    SkPaint strokePaint = paint;
    strokePaint.setStyle(SkPaint::kStroke_Style);
    if (this->internalQuickReject(bounds, strokePaint)) {
        return;
    }

    auto layer = this->aboutToDraw(this, strokePaint, &bounds);
    if (layer) {
         // 内部直接转换成skia绘制命令
        this->topDevice()->drawPoints(mode, count, pts, layer->paint());
    }
}      
复制代码
```

native canvas 继承关系

```
class SkiaRecordingCanvas : public SkiaCanvas {...
class SkiaCanvas : public Canvas {...

复制代码
```

至此，所有的绘制指令都存储到了 node 中。 可以通知`RenderThread线程`来绘制了。

4.7 [绘制阶段] syncAndDrawFrame()
-----------------------------

在父类中 HardwareRenderer.java

> HardwareRenderer.java

```
/**
* Syncs the RenderNode tree to the render thread and requests a frame to be drawn.
*
* @hide
*/
@SyncAndDrawResult
public int syncAndDrawFrame(@NonNull FrameInfo frameInfo) {
   // 把rendernode树同步到 渲染线程
  return nSyncAndDrawFrame(mNativeProxy, frameInfo.frameInfo, frameInfo.frameInfo.length);
}


private static native int nSyncAndDrawFrame(long nativeProxy, long[] frameInfo, int size);

>frameworks/base/core/jni/android_view_ThreadedRenderer.cpp
static int android_view_ThreadedRenderer_syncAndDrawFrame(JNIEnv* env, jobject clazz,
        jlong proxyPtr, jlongArray frameInfo, jint frameInfoSize) {
    LOG_ALWAYS_FATAL_IF(frameInfoSize != UI_THREAD_FRAME_INFO_SIZE,
            "Mismatched size expectations, given %d expected %d",
            frameInfoSize, UI_THREAD_FRAME_INFO_SIZE);
            
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    env->GetLongArrayRegion(frameInfo, 0, frameInfoSize, proxy->frameInfo());
    // 调用 RenderProxy的 syncAndDrawFrame()
    return proxy->syncAndDrawFrame();
}

>frameworks/base/libs/hwui/renderthread/RenderProxy.cpp
int RenderProxy::syncAndDrawFrame() {
    //调用 mDrawFrameTask
    return mDrawFrameTask.drawFrame();
}


复制代码
```

4.8 DrawFrameTask::drawFrame()
------------------------------

> frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp

```
int DrawFrameTask::drawFrame() {
    LOG_ALWAYS_FATAL_IF(!mContext, "Cannot drawFrame with no CanvasContext!");

    mSyncResult = SyncResult::OK;
    mSyncQueued = systemTime(SYSTEM_TIME_MONOTONIC);
    // post到 RenderTHread 线程
    postAndWait();

    return mSyncResult;
}

void DrawFrameTask::postAndWait() {
    ATRACE_CALL();
    AutoMutex _lock(mLock);
    //往渲染线程的队列中 post任务进去 ，回调run方法。
    mRenderThread->queue().post([this]() { run(); });
   
    mSignal.wait(mLock);
}
复制代码
```

在继续看`run()`方法前，我们先了解下 `CanvasContext`。

4.9 CanvasContext.cpp
---------------------

> frameworks/base/libs/hwui/renderthread/CanvasContext.cpp

**CanvasContext 作用**：每个 RenderThread 对象有一个 CanvasContext。 管理全局 EGL context 和当前绘制区域 surface。

### 4.9.1 CanvasContext 构造函数

```
CanvasContext::CanvasContext(RenderThread& thread, bool translucent, RenderNode* rootRenderNode,
                             IContextFactory* contextFactory,
                             std::unique_ptr<IRenderPipeline> renderPipeline)
        : mRenderThread(thread) // 渲染线程
        , mGenerationID(0)
        , mOpaque(!translucent)
        , mAnimationContext(contextFactory->createAnimationContext(mRenderThread.timeLord()))
        , mJankTracker(&thread.globalProfileData()) // jank
        , mProfiler(mJankTracker.frames(), thread.timeLord().frameIntervalNanos())
        , mContentDrawBounds(0, 0, 0, 0)
        , mRenderPipeline(std::move(renderPipeline)) { // 渲染管道
    rootRenderNode->makeRoot();
    mRenderNodes.emplace_back(rootRenderNode);
    mProfiler.setDensity(DeviceInfo::getDensity());
}
复制代码
```

因此，CanvasContext 要管理当前绘制区域，又要给完成 egl 的配置。我们先来看看是如何`跟surface挂钩`的。

还记的在 ViewRootImpl 的 setView() 中，如果是硬件绘制，则会先请求 buffer。

### 4.9.2 mAttachInfo.mThreadedRenderer.allocateBuffers()

具体实现在父类 HardwareRenderer.java 中：

```
public void allocateBuffers() {
    // JNI方法
     nAllocateBuffers(mNativeProxy);
}

private static native void nAllocateBuffers(long nativeProxy);

>frameworks/base/core/jni/android_view_ThreadedRenderer.cpp 
static void android_view_ThreadedRenderer_allocateBuffers(JNIEnv* env, jobject clazz,jlong proxyPtr) {
    RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
    // 
    proxy->allocateBuffers();
}
 
>frameworks/base/libs/hwui/renderthread/RenderProxy.cpp 
void RenderProxy::allocateBuffers() {
    // post到renderThread线程中，CanvasContext方法
    mRenderThread.queue().post([=]() { mContext->allocateBuffers(); });
} 

>frameworks/base/libs/hwui/renderthread/CanvasContext.cpp
void CanvasContext::allocateBuffers() {
    if (mNativeSurface && Properties::isDrawingEnabled()) {
        // ANativeWindow的hook方法
        ANativeWindow_tryAllocateBuffers(mNativeSurface->getNativeWindow());
    }
}
void ANativeWindow_tryAllocateBuffers(ANativeWindow* window) {
    if (!window || !query(window, NATIVE_WINDOW_IS_VALID)) {
        return;
    }
    // 最后调用到 ANativeWindow 本地化窗口的hook_perform钩子函数，在surface中
    window->perform(window, NATIVE_WINDOW_ALLOCATE_BUFFERS);
}

复制代码
```

最后调用到 ANativeWindow 本地化窗口的 `hook_perform()` 钩子函数，在 `Surface` 中：

#### 4.9.2.1 Surface.hook_perform()

```
> frameworks/native/libs/gui/Surface.cpp
 // Initialize the ANativeWindow function pointers.
    ANativeWindow::setSwapInterval  = hook_setSwapInterval;
    ANativeWindow::dequeueBuffer    = hook_dequeueBuffer;
    ANativeWindow::cancelBuffer     = hook_cancelBuffer;
    ANativeWindow::queueBuffer      = hook_queueBuffer;
    ANativeWindow::query            = hook_query;
    ANativeWindow::perform          = hook_perform; //钩子

int Surface::hook_perform(ANativeWindow* window, int operation, ...) {
    va_list args;
    va_start(args, operation);
    Surface* c = getSelf(window);
    // ...
    
    result = c->perform(operation, args);
    va_end(args);
    return result;
}

int Surface::perform(int operation, va_list args)
{
    int res = NO_ERROR;
    switch (operation) {
     //...
     //这里
    case NATIVE_WINDOW_ALLOCATE_BUFFERS: 
        allocateBuffers();
        res = NO_ERROR;
        break;
    case NATIVE_WINDOW_GET_LAST_QUEUED_BUFFER:
        res = dispatchGetLastQueuedBuffer(args);
        break;
   // ...
    default:
        res = NAME_NOT_FOUND;
        break;
    }
    return res;
}

void Surface::allocateBuffers() {
    uint32_t reqWidth = mReqWidth ? mReqWidth : mUserWidth;
    uint32_t reqHeight = mReqHeight ? mReqHeight : mUserHeight;
    // 熟悉的身影
    mGraphicBufferProducer->allocateBuffers(reqWidth, reqHeight,
            mReqFormat, mReqUsage);
}

复制代码
```

兜兜转转，还是通过 `gbp` 来请求`缓存buffer`。 请求到 buffer 后，最后通过 ThreadedRenderer.java 的 initialize() 方法，内部会调用 setSurface()，最终调用到 mContext->setSurface()，绑定起来。

### 4.9.3 ThreadedRenderer.initialize()

```
boolean initialize(Surface surface) throws OutOfResourcesException {
    boolean status = !mInitialized;
    mInitialized = true;
    updateEnabledState(surface);
    
    setSurface(surface);
    return status;
}

public void setSurface(@Nullable Surface surface) {
    if (surface != null && !surface.isValid()) {
        throw new IllegalArgumentException("Surface is invalid. surface.isValid() == false.");
    }
    nSetSurface(mNativeProxy, surface);
}

static void android_view_ThreadedRenderer_setSurface(JNIEnv* env, jobject clazz,
      jlong proxyPtr, jobject jsurface) {
  RenderProxy* proxy = reinterpret_cast<RenderProxy*>(proxyPtr);
  sp<Surface> surface;
  if (jsurface) {
      surface = android_view_Surface_getSurface(env, jsurface);
  }
  proxy->setSurface(surface);
}

void RenderProxy::setSurface(ANativeWindow* window, bool enableTimeout) {
    if (window) { ANativeWindow_acquire(window); }
    mRenderThread.queue().post([this, win = window, enableTimeout]() mutable {
        mContext->setSurface(win, enableTimeout);
        if (win) { ANativeWindow_release(win); }
    });
}
复制代码
```

至于如何给 EGL 配置的，接着看 run() 方法：

4.10 DrawFrameTask::run()
-------------------------

> frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp

```
// run方法：
void DrawFrameTask::run() {
    
    bool canUnblockUiThread;
    bool canDrawThisFrame;
    {
        // 构造一个 treeInfo 对象 
        TreeInfo info(TreeInfo::MODE_FULL, *mContext);
        
        // ...
        // 遍历renderNode结合，转变为TreeInfo 信息
        canUnblockUiThread = syncFrameState(info);
       
        ...
    }

    // Grab a copy of everything we need
    // 复制 CanvasContext 对象
    CanvasContext* context = mContext;
    // ... 
    nsecs_t dequeueBufferDuration = 0;
    if (CC_LIKELY(canDrawThisFrame)) {
         // 开始绘制 
        dequeueBufferDuration = context->draw();
    } else {
         // ...
        // wait on fences so tasks don't overlap next frame
        context->waitOnFences();
    }
   
    // ... 
    
}

复制代码
```

1.  把遍历 renderNode 结合，转变为 TreeInfo 信息；配置 eglcontext
2.  调用 CanvasContext 的 draw() 方法，开始绘制

### 4.11 DrawFrameTask::syncFrameState()

> frameworks/base/libs/hwui/renderthread/DrawFrameTask.cpp

```
bool DrawFrameTask::syncFrameState(TreeInfo& info) {
    ATRACE_CALL();
    int64_t vsync = mFrameInfo[static_cast<int>(FrameInfoIndex::Vsync)];
    mRenderThread->timeLord().vsyncReceived(vsync);
    // 准备eglcontext 上下文，用来链接OpenGL和 GraphicBuffer之间的桥梁
    bool canDraw = mContext->makeCurrent();
    mContext->unpinImages();

    for (size_t i = 0; i < mLayers.size(); i++) {
        mLayers[i]->apply();
    }
    mLayers.clear();
    mContext->setContentDrawBounds(mContentDrawBounds);
    // 准备
    mContext->prepareTree(info, mFrameInfo, mSyncQueued, mTargetNode);

    //...
    // If prepareTextures is false, we ran out of texture cache space
    return info.prepareTextures;
}

// makeCurrent()方法的具体实现为 SkiaOpenGLPipeline或者 SkiaVulkanPipeline。这里看SkiaOpenGLPipeline：
>frameworks/base/libs/hwui/pipeline/skia/SkiaOpenGLPipeline.cpp
MakeCurrentResult SkiaOpenGLPipeline::makeCurrent() {
    // TODO: Figure out why this workaround is needed, see b/13913604
    // In the meantime this matches the behavior of GLRenderer, so it is not a regression
    EGLint error = 0;
    // 调用 mEglManager 来初始化
    if (!mEglManager.makeCurrent(mEglSurface, &error)) {
        return MakeCurrentResult::AlreadyCurrent;
    }
    return error ? MakeCurrentResult::Failed : MakeCurrentResult::Succeeded;
}


>frameworks/base/libs/hwui/renderthread/EglManager.cpp 

bool EglManager::makeCurrent(EGLSurface surface, EGLint* errOut, bool force) {
    if (!force && isCurrent(surface)) return false;

    if (surface == EGL_NO_SURFACE) {
        // Ensure we always have a valid surface & context
        // 确保surface有效
        surface = mPBufferSurface;
    }
    // 最终 让egl模块给OpenGL提供了窗口、上下文
    if (!eglMakeCurrent(mEglDisplay, surface, surface, mEglContext)) {
        ...
    }
    mCurrentSurface = surface;
    if (Properties::disableVsync) {
        eglSwapInterval(mEglDisplay, 0);
    }
    return true;
}

复制代码
```

到这里，我们通过 EglManager 的 `makeCurrent()`终于让`OpenGL`和本`地窗口surface` 联系了起来。因此，通过`CanvasContext.draw()` 就可以实现 OpenGL/Vulkan 来绘制了。

### 4.12 CanvasContext::draw()

```
nsecs_t CanvasContext::draw() {
    //...
    SkRect dirty;
    mDamageAccumulator.finish(&dirty);

   // ...
    ATRACE_FORMAT("Drawing " RECT_STRING, SK_RECT_ARGS(dirty));
    // 开始绘制
    const auto drawResult = mRenderPipeline->draw(frame, windowDirty, dirty, mLightGeometry,
                                                  &mLayerUpdateQueue, mContentDrawBounds, mOpaque,
                                                  mLightInfo, mRenderNodes, &(profiler()));

    // ...
    bool requireSwap = false;
    int error = OK;
    // 绘制完成，交换缓冲区
    bool didSwap = mRenderPipeline->swapBuffers(frame, drawResult.success, windowDirty,
                                                mCurrentFrameInfo, &requireSwap);

    mCurrentFrameInfo->set(FrameInfoIndex::CommandSubmissionCompleted) = std::max(
            drawResult.commandSubmissionTime, mCurrentFrameInfo->get(FrameInfoIndex::SwapBuffers));

    mIsDirty = false;

   // ...
    return mCurrentFrameInfo->get(FrameInfoIndex::DequeueBufferDuration);
}
复制代码
```

### 4.2.1 SkiaOpenGLPipeline::draw()

```
bool SkiaOpenGLPipeline::draw(const Frame& frame, const SkRect& screenDirty, const SkRect& dirty,
                              const LightGeometry& lightGeometry,
                              LayerUpdateQueue* layerUpdateQueue, const Rect& contentDrawBounds,
                              bool opaque, const LightInfo& lightInfo,
                              const std::vector<sp<RenderNode>>& renderNodes,
                              FrameInfoVisualizer* profiler) {
    mEglManager.damageFrame(frame, dirty);

    // ...
    // Draw visual debugging features
    if (CC_UNLIKELY(Properties::showDirtyRegions ||
                    ProfileType::None != Properties::getProfileType())) {
        SkCanvas* profileCanvas = surface->getCanvas();
        
        SkiaProfileRenderer profileRenderer(profileCanvas);
        // 调用SKcanvas 通过skia api来完成绘制
        profiler->draw(profileRenderer);
        profileCanvas->flush();
    }
    

    return true;
}
复制代码
```

五、总结
====

总体分为软件和硬件绘制：

*   `软件绘制`
    
    1.  调用`Surface的lock()`方法，native 层内部调用 mGraphicBufferProducer->dequeueBuffer() 获得缓冲区 GraphicBuffer。 此时，Java 层 Surface 中的 Canvas 对象是空的，没有与任何 Bitmap 绑定的。
    2.  生成一个`SKBimap`对象，与 GraphicBuffer 绑定，本质上`共享一块内存Buffer`。设置给 native 层的 Canvas 对象。至此，java 层的 Canvas 才有了画布。
    3.  调用 SKCanvas 提供的 API 开始绘制，最终会调用 Skia 来完成绘制
    4.  调用 Surface 的`unlockAndPost()`来解除与 GraphicBuffer 的绑定。内部清空 Cnavas 的 Bitmap，`解除SKBitmap与GraphicBuffer` 的绑定。
    5.  通过 mGraphicBufferProducer->queueBuffer() 入队 SF 的 BufferQueue，请求下一次 vsync 信号，通知 SF 来进行消费合成。
*   `硬件绘制`
    

分为构建和绘制两个阶段：

*   构建阶段
    
    1.  通过 ThreadedRenderer.updateRootDisplayList()，把 view 树转换成对应的 drawXXXOp 指令的`RenderNode树`。
    2.  一个 view 对应一个 RenderNode。每个 node 中包含了当前 view 的绘制指令，如 drawXXXop，如果有子 view 还会包含 drawRenderNodeOp 命令。 这些 Op 又称为统称为`displayList`。
*   绘制阶段
    
    1.  调用 HardwareRenderer 的 syncAndDrawFrame()，把生成的 displayList 同步到 RenderThread 线程去绘制
    2.  CanvasContext 负责 提供 OpenGL 的本地化窗口 Surface，至此在 RenderThread 线程中，调用 CanvasContext.draw()。
    3.  最终通过`Skia/Vulkan 引擎(SkiaOpenGLPipeline/SkiaVulkanPipeline)`把数据渲染到缓冲区 GraphicBuffer 中，入队 BufferQueue, 请求下一次 vsync 信号，通知 SF 来消费合成上屏。

六、参考
====

[ljd1996.github.io/2020/11/09/…](https://link.juejin.cn?target=https%3A%2F%2Fljd1996.github.io%2F2020%2F11%2F09%2FAndroid-Surface%25E5%258E%259F%25E7%2590%2586%25E8%25A7%25A3%25E6%259E%2590%2F "https://ljd1996.github.io/2020/11/09/Android-Surface%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/")

[blog.csdn.net/tyyj90/arti…](https://link.juejin.cn?target=https%3A%2F%2Fblog.csdn.net%2Ftyyj90%2Farticle%2Fdetails%2F107447549 "https://blog.csdn.net/tyyj90/article/details/107447549")