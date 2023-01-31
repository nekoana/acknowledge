> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7139132774615056414)

再懒也要逼自己每个星期输出至少一篇文章，哪怕没人看。这篇是 Android 爬坑日记第三篇，也是小鹅爬坑日记的第二篇，小鹅事务所是我开源的记录事务 APP 可以看看我之前的一篇文章。

什么是输入 Dialog，输入 Dialog 一般出现在评论区、聊天框，我下面放一张动图，分别是微博轻享版，掘金和微信（当然微信这个不是 Dialog）

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/dd836d456cef4265aed014f721b4d11d~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

软键盘入场动画
-------

不知道大家有没有发现，微博和掘金的效果是一样的，也是市面上大部分产品的软键盘入场效果。虽然不影响使用，但是视觉上的效果会差一点。微信的软键盘入场动画是贴着输入框的，看起来非常丝滑流畅。

> 动画交互非常影响用户对 APP 的第一印象。 —— 米奇律师

以下我将以【小鹅事务所】的输入框为案例来实现流畅的软键盘 DialogFragment 输入框动画。

效果
--

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c259149d9ac04dae858cdba882394eae~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

好像区别也不是很大。

> 细节决定着一个 APP 的品质。 —— 米奇律师

先上代码：[github.com/MReP1/Littl…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMReP1%2FLittleGooseOffice%2Fblob%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Flittle%2Fgoose%2Faccount%2Fcommon%2Fdialog%2FInputTextDialogFragment.kt "https://github.com/MReP1/LittleGooseOffice/blob/master/app/src/main/java/little/goose/account/common/dialog/InputTextDialogFragment.kt")

爬坑
--

使用 Dialog 来实现会比较好管理，由于这个输入框在底部，因此使用了`BottomSheetDialogFragment`来实现。并且弹出 Dialog 的时候弹出软键盘，软键盘收起的同时关闭 Dialog。

### BottomSheetDialogFragment

`BottomSheetDialogFragment`是 Material Design 里的一个组件。自带了缓入缓出的入场出场动画。如果需要自己到 XML 文件里写动画，比较难实现缓入缓出的效果，因此考虑到我比较懒这一点，直接使用`BottomSheetDialogFragment`是明智之选。

### 弹出软键盘

#### 坑点

弹出 Dialog 的时候同时弹出软键盘也是有坑的，如果说到网上去找工具类，大概率软键盘弹不出来。举个例子：

```
object KeyBoard {
    fun show(focusView: View){
        val inputManager = appContext.getSystemService(Context.INPUT_METHOD_SERVICE) as InputMethodManager
        inputManager.showSoftInput(focusView, InputMethodManager.SHOW_IMPLICIT)
    }
}
复制代码
```

如果在`DialogFragment`中使用这个工具方法，你会发现软键盘弹不出来。

```
class InputTextDialogFragment : BottomSheetDialogFragment() {    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.etInput.requestFocus()
        KeyBoard.show(binding.etInput)
    }
}
复制代码
```

我猜测是因为 Fragment 的视图没有绑定到 Dialog 的 Window 上，或者 Dialog 还没有弹出来，导致没有办法弹出软键盘。

这个时候就可能会有人告诉大家：这还不简单，我等它几百毫秒绑定好了再弹不就好了。

```
class InputTextDialogFragment : BottomSheetDialogFragment() {    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.etInput.postDelay(300) {
            binding.etInput.requestFocus()
            KeyBoard.show(binding.etInput)
        }
    }
}
复制代码
```

这种方法还真可以，但是在我看来不优雅。

于是我看了米奇律师的[沉浸式状态栏](https://juejin.cn/post/7133619873376108558 "https://juejin.cn/post/7133619873376108558")灵光一动写出了以下代码。

```
binding.etInput.doOnAttach { 
    binding.etInput.requestFocus()
    KeyBoard.show(binding.etInput)
}
复制代码
```

很遗憾这种方法也不能实现，View 绑定到 Root View 了，但是可能 Dialog 还没有弹出来，因此软键盘也弹不出来，具体原因我没有深究，有不同理解的欢迎到评论区交流。

#### 解决方法

其实想要启动`DialogFragment`的时候弹出软键盘没那么复杂。只需要给 Dialog 所在的 Window 设置以下软键盘输入标志位就好了，只要 Dialog 一出现，软键盘马上弹出来。

```
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    binding.etInput.requestFocus()
    dialog?.window?.setSoftInputMode(WindowManager.LayoutParams.SOFT_INPUT_STATE_VISIBLE)
}
复制代码
```

### 监听软键盘弹起

在 Android API 达到 30 及以上，Android 引入了`WindowInsetsAnimation`API，此时就可以监听 WindowInsets 的占用屏幕大小的变化了。因此就可以实现类似于图 1 动图中微信那样流畅的动画了。那么具体怎么实现呢？我先放一下代码，在注释中讲解一下。

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    ViewCompat.setWindowInsetsAnimationCallback(
        binding.root,
        object : WindowInsetsAnimationCompat.Callback(DISPATCH_MODE_STOP) {
            // 存放根视图的的初始高度
            private var startHeight = 0
            private var lastDiffH = 0
            
            override fun onPrepare(animation: WindowInsetsAnimationCompat) {
                if (startHeight == 0) {
                    // 此处赋值根视图的初始高度，由于动画开始前，根视图已经绑定到Window上了
                    // 因此能够获得初始高度
                    startHeight = binding.root.height
                }
            }
            override fun onProgress(
                insets: WindowInsetsCompat,
                runningAnimations: MutableList<WindowInsetsAnimationCompat>
            ): WindowInsetsCompat {
                // 获取软键盘的Inset
                val typesInset = insets.getInsets(WindowInsetsCompat.Type.ime())
                // 获取系统状态栏、导航栏的Inset
                val otherInset = insets.getInsets(WindowInsetsCompat.Type.systemBars())
                // 获取它们的差值
                val diff = Insets.subtract(typesInset, otherInset).let {
                    Insets.max(it, Insets.NONE)
                }
                // 根据差值来调整整个根布局的位移
                binding.root.translationX = (diff.left - diff.right).toFloat()
                
                // 我的主要布局Gravity是贴着底部的，所以需要往上提
                // 实际按需来加，如果布局贴着顶部则不需要调整translationY
                binding.root.translationY = (diff.top - diff.bottom).toFloat()
                
                // 获取Y轴位移的绝对值
                val diffH = abs(diff.top - diff.bottom)
                
                // 更新整个父View的高度，不然视图跑上去了，父View高度没跟上
                binding.root.updateLayoutParams { height = startHeight + diffH }
                
                // 当察觉到软键盘高度越来越小（说明在收起了），就可以dismiss该DialogFragment了
                if (diffH < lastDiffH) {
                    dismiss()
                    ViewCompat.setWindowInsetsAnimationCallback(binding.root, null)
                }
                
                // 存放每一次软键盘的高度
                lastDiffH = diffH
                return insets
            }
        }
    )
}
复制代码
```

我的布局是贴着父 layout 底部的，所以需要调整 translationY，如果布局不贴着底部，则不需要调整 translationY。

```
app:layout_constraintBottom_toBottomOf="parent"
复制代码
```

记得给该 Window 设置软键盘的插入模式为`WindowManager.LayoutParams.SOFT_INPUT_ADJUST_NOTHING`，这意味着该`DialogFragment`不会被软键盘顶上去，也就是说软键盘背后的内容其实也是 View 的一部分，但是是空白的，因为在动画中一直将父 View 的 translationY 往上提。

```
dialog?.window?.setSoftInputMode(
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
        WindowManager.LayoutParams.SOFT_INPUT_STATE_VISIBLE or WindowManager.LayoutParams.SOFT_INPUT_ADJUST_NOTHING
    } else {
        WindowManager.LayoutParams.SOFT_INPUT_STATE_VISIBLE
    }
)
复制代码
```

这里还需要分情况，在 Android11 以下的手机无法使用`WindowInsetsAnimation` API，因此就没有办法用这个 API 享受到如此丝滑的动画啦。

其实在这个动图中，左图为 Android10 的效果，右图为 Android11 的效果，你会发现左图的界面没有贴着软键盘走，而右图的界面稳稳地贴着软键盘。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c259149d9ac04dae858cdba882394eae~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

那么 Android11 以下如何监听软键盘收起 dismiss 掉 DialogFragment 呢？

```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.R) {
    // Android11 以上动画
} else {
    binding.root.viewTreeObserver.addOnGlobalLayoutListener(object :
        ViewTreeObserver.OnGlobalLayoutListener {
        var lastBottom = 0
        override fun onGlobalLayout() {
            ViewCompat.getRootWindowInsets(binding.root)?.let { insets ->
                val bottom = insets.getInsets(WindowInsetsCompat.Type.ime()).bottom
                if (lastBottom != 0 && bottom == 0) {
                    // 收起键盘了，可以 dismiss 了
                    dismiss()
                    binding.root.viewTreeObserver.removeOnGlobalLayoutListener(this)
                }
                lastBottom = bottom
            }
        }
    })
}
复制代码
```

`lastBottom`存放软键盘的高度，当它被赋值不为 0 时，说明软键盘弹出来了，此时如果将它赋值为 0，也就也为这软键盘收起了，就可以 dissmiss 掉 DialogFragment 了。

总结
--

文章结束啦，再放一遍[代码](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FMReP1%2FLittleGooseOffice%2Fblob%2Fmaster%2Fapp%2Fsrc%2Fmain%2Fjava%2Flittle%2Fgoose%2Faccount%2Fcommon%2Fdialog%2FInputTextDialogFragment.kt "https://github.com/MReP1/LittleGooseOffice/blob/master/app/src/main/java/little/goose/account/common/dialog/InputTextDialogFragment.kt")吧！

这个实现或许看起来也不算很优雅，可能不是最佳实践，是我界面交互优化的一个尝试，有的时候 UI 给的设计稿是静态的，作为开发赶工期肯定是怎么方便怎么来，功能实现就行、UI 设计师不提 BUG 就行。但是谷歌团队提供了这么多好用的 API 来给大家优化界面，在如今大家手机都性能过剩的背景下，把主线程腾出点空间用于更舒服的用户交互，也未尝不是一种应用优化。

其实我个人比较喜欢上层界面层的开发，基于这篇文章我再开一个坑吧，关于交互、动画，有空慢慢填坑。

参考
--

[zhuanlan.zhihu.com/p/343022200](https://link.juejin.cn?target=https%3A%2F%2Fzhuanlan.zhihu.com%2Fp%2F343022200 "https://zhuanlan.zhihu.com/p/343022200")