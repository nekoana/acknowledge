> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7148675037501849636)

我正在参加「掘金 · 启航计划」

多点触控实现缩放并拖拽
===========

> [参考官方文档 - 跟踪多个指针](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Ftraining%2Fgestures%2Fmulti%3Fhl%3Dzh-cn%23track "https://developer.android.google.cn/training/gestures/multi?hl=zh-cn#track")
> 
> [参考官方文档 - 拖动并缩放](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Ftraining%2Fgestures%2Fscale%3Fhl%3Dzh-cn%23drag "https://developer.android.google.cn/training/gestures/scale?hl=zh-cn#drag")

Touch 事件中用到的 Action
-------------------

结合官方文档解释，可以知道触摸事件的数据是存到一个数组中，每个 action 都会有一个`actionIndex`，它记录这当前指针信息在数组的存放位置和 Id

*   `ACTION_DOWN` 第一根手指按下屏幕时会触发的 Action, 并且他的坐标数据会存在`MotionEvent`中索引`0`的位置
*   `ACTION_POINTER_DOWN` 第二根手指以上的手指按下屏幕时触发，这时我们如果要获取当前按下这根手指的坐标信息，可以通过`event.getActionIndex` 来获取当前手指的信息索引
*   `ACTION_MOVE` 手指按下并拖动时触发
*   `ACTION_POINTER_UP` 官方说非主要指针抬起时触发，意思应该是假如两个手指以上按下屏幕，其中一个手指抬起时会触发这个 Action, 只有一个手指并且抬起的时候不会触发的
*   `ACTION_UP` 最后一根手指抬起时触发
*   `event.actionIndex` get 的时候，拿的是当前 Action 事件的指针数据存放索引

例如从按下两只手指到屏幕到抬起手指过程，上面事件大致流程如下: 第一根手指按下，触发事件 `ACTION_DOWN`，获取指针信息 第二根手指按下，触发 `ACTION_POINTER_DOWN` 拖拽屏幕时，触发`ACTION_MOVE` 抬起一根手指时，触发`ACTION_POINTER_UP` 抬起最后一根手指时，触发`ACTION_UP`

_在多点触控中，不能写死索引，必须要用索引来取数据，不然容易取错数据导致各种问题。_

接下来会使用到的`MotionEvent`的 API 有如下：

*   `getActionMasked` 获取当前 Action 类型
*   `getActionIndex` 获取当前 Action 指针索引
*   `getPointerId`获取指定索引的指针 Id
*   `findPointerIndex` 根据 Id 寻找指针信息的索引
*   `getX、getY` 获取指定索引的 x、y 坐标

拖动并缩放
-----

官方文档提到几个要注意的点，这里就不贴出来了，主要说的是： 当单手指 A 拖动的时候，把第二根手指 B 按下屏幕，然后抬起第一根手指 A，这时，默认的指针会变成 B，但是如果仅跟着单个指针，那么抬起 A 瞬间，你代码如果只跟踪默认指针，将可能会检测到拖动目标本来在 A 的位置，突然飞到了 B 的位置，实际上只是松开了一个手指，并没有发生平移操作，这样就相当于出现异常了。 要解决也不难，我们用一个变量把默认指针 Id 记下来，在 A 按下的时候把 A 的 Id 存下，在`MotionEvent.ACTION_POINTER_UP` 时，获取松开的手指的 Id 是不是当前记录的默认指针的 Id，如果是，那就把当前默认 Id 改为当前按下的 ID，然后每次都使用默认 id 的指针坐标作为初始位置，在`ACTION_MOVE` 的时候，使用当前位置减去这个初始位置即可得到正确的移动距离。

实现代码
----

```
class TouchController {
    private val MIN_THRESHOLD = 2

    var touchListener: TouchMapListener? = null

    val zoomCenterPoint = doubleArrayOf(0.0, 0.0)
    val translateStartPoint = doubleArrayOf(0.0, 0.0)

    var lastDistance = 0.0
    //通过官方文档描述，需要记录下当前的默认手指Id
    private var mActivePointerId = INVALID_POINTER_ID
     fun onTouch(event: MotionEvent): Boolean {
        when (event.actionMasked) {
            MotionEvent.ACTION_DOWN -> {
            //第一个手指按下，把这个Id记录下来，并且把其坐标作为起始点击坐标
                event.actionIndex.also { pointerIndex ->
                    translateStartPoint[0] = event.getX(pointerIndex).toDouble()
                    translateStartPoint[1] = event.getY(pointerIndex).toDouble()
                }

                mActivePointerId = event.getPointerId(0)
            }
           MotionEvent.ACTION_POINTER_DOWN -> {
           //第二个以上手指按下，计算两个手指之间的中心位置，因为我们要实现缩放的同时支持平移，
           //所以，一个手指按下时，平移的基点就是这个手指的坐标，两个手指按下时，我们就以两个手指的中心
           //点坐标作为基点，同时也以这个基点作为缩放的基准点
                calculateCenter(event).also {
                    zoomCenterPoint[0] = it[0]
                    zoomCenterPoint[1] = it[1]
                    translateStartPoint[0] = it[0]
                    translateStartPoint[1] = it[1]
                }
                //计下按下时两个手指的直线距离，为双指缩放准备
                lastDistance = spacing(event)
            }
            MotionEvent.ACTION_POINTER_UP -> {
            //多点触摸的时候，当有一个手指抬起时，会触发这个Action，我们需要判断，抬起的手指是不是
            //当前记录的第一个手指的Id，如果是，说明默认的手指Id已经发生变化，变成另一个了，所以
            //我们的平移基准点坐标也要改变成变成另一个，否则会出现松开第一种手指时，没有进行拖拽却计算到很大的平移距离的问题，这里也是参考官方文档解释后发现的问题
                event.actionIndex.also { pointerIndex ->
                    val pointerId = event.getPointerId(pointerIndex)
                    if (pointerId == mActivePointerId) {
                        val newPointerIndex = if (pointerIndex == 0) 1 else 0
                        translateStartPoint[0] = event.getX(newPointerIndex).toDouble()
                        translateStartPoint[1] = event.getY(newPointerIndex).toDouble()
                        //重新设置默认手指的id 保证取的坐标没问题
                        mActivePointerId = event.getPointerId(newPointerIndex)
                    } else {
                    //松开的不是第一个按下的手指，直接把基准点重置成第一个手指的坐标
                        translateStartPoint[0] =
                            event.getX(event.findPointerIndex(mActivePointerId)).toDouble()
                        translateStartPoint[1] =
                            event.getY(event.findPointerIndex(mActivePointerId)).toDouble()
                    }
                }
            }
            MotionEvent.ACTION_MOVE -> {
                val translateEndPoint = doubleArrayOf(0.0, 0.0)
                //拖放结束点先用默认手指的当前位置
                event.findPointerIndex(mActivePointerId).let { pointerIndex ->
                    translateEndPoint[0] = event.getX(pointerIndex).toDouble()
                    translateEndPoint[1] = event.getY(pointerIndex).toDouble()
                }
                //如果多点触控，我们要计算两个手指的中心点作为终点
                if (event.pointerCount >= 2) {
                    calculateCenter(event).also {
                        zoomCenterPoint[0] = it[0]
                        zoomCenterPoint[1] = it[1]

                        translateEndPoint[0] = it[0]
                        translateEndPoint[1] = it[1]
                    }

                    val currentDistance = spacing(event)
                    val distanceDiff = currentDistance - lastDistance
                    if (abs(distanceDiff) > MIN_THRESHOLD) {
                    //把事件传给地图去控制比例尺缩放
                        if (distanceDiff < 0) {
                            touchListener?.onZoomIn(zoomCenterPoint[0], zoomCenterPoint[1])
                        } else {
                            touchListener?.onZoomOut(zoomCenterPoint[0], zoomCenterPoint[1])
                        }
                        lastDistance = currentDistance
                    }
                }
	                //计算x、y方向的平移量
	              	val xDiff = translateEndPoint[0] - translateStartPoint[0]
	                val yDiff = translateEndPoint[1] - translateStartPoint[1]
	
	                if (abs(xDiff) > MIN_THRESHOLD || abs(yDiff) > MIN_THRESHOLD) {
	                	//把事件传给地图去控制平移
	                    touchListener?.onDrag(xDiff, yDiff)
	                }
	              	translateStartPoint[0] = translateEndPoint[0]
	                translateStartPoint[1] = translateEndPoint[1]
            }
            MotionEvent.ACTION_CANCEL,
            MotionEvent.ACTION_UP -> {
                mActivePointerId = INVALID_POINTER_ID
            }
        }
        return true 
	}

  /**
  * 计算连个手指连线中心点
  */
  private fun calculateCenter(event: MotionEvent): DoubleArray {
    //计算起点中心坐标
    val x0 = event.getX(0)
    val y0 = event.getY(0)
    
            val x1 = event.getX(1)
            val y1 = event.getY(1)
    
            return doubleArrayOf(
                x0 + (x1 - x0) / 2.0,
                y0 + (y1 - y0) / 2.0
            )
      }
      
      /**
       * 计算两个点的距离
       *
       * @param event
       * @return
       */
      private fun spacing(event: MotionEvent): Double {
          return if (event.pointerCount == 2) {
              val x = event.getX(0) - event.getX(1)
              val y = event.getY(0) - event.getY(1)
              sqrt((x * x + y * y).toDouble())
          } else 0.0
      }
  
  
  interface TouchMapListener {
    fun onDrag(xDiff: Double, yDiff: Double)
    fun onZoomIn(centerX: Double, centerY: Double)
    fun onZoomOut(centerX: Double, centerY: Double)
  }
复制代码
```

实现效果
----

> [居于之前的文章写的一个 Demo](https://juejin.cn/post/7111545294642708510 "https://juejin.cn/post/7111545294642708510")

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/2c657f0842084346a93bb787cf1b889d~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)