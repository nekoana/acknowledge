> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7144621475503276045)

所谓`Activity`共享元素动画，就是从`ActivityA`跳转到`ActivityB` 通过控制某些元素 (`View`) 从`ActivityA`开始帧的位置跳转到`ActivityB` 结束帧的位置，应用过度动画

`Activity`的共享元素动画，其动画核心是使用的`Transition`记录共享元素的开始帧、结束帧，然后使用`TransitionManager`过度动画管理类调用`beginDelayedTransition`方法 应用过度动画

```
注意：Android5.0才开始支持共享元素动画
复制代码
```

所以咱们先介绍一下`TransitionManager`的一些基础知识

`TransitionManager`介绍
---------------------

`TransitionManager`是 `Android5.0`开始提供的一个过渡动画管理类，功能非常强大；其可应用在两个`Activity`之间、`Fragment`之间、`View`之间应用过渡动画

`TransitionManager`有两个比较重要的类`Scene(场景)`和`Transition(过渡)` , 咱们先来介绍一下这两个类

### `Scene(场景)`

顾名思义`Scene`就是场景的意思，在执行动画之前，我们需要创建两个场景 (场景 A 和场景 B), 其动画执行流程如下：

*   根据起始布局和结束布局创建两个 `Scene` 对象 (场景 A 和场景 B); 然而 起始布局的场景通常是根据当前布局自动确定的
*   创建一个 `Transition` 对象以定义所需的动画类型
*   调用 `TransitionManager.go(Scene, Transition)`，使用过渡动画运行到指定的场景

##### 生成场景

生成场景有两种方式; 一种是调用静态方法通过布局生成 `Scene.getSceneForLayout(sceneRoot, R.layout.scene_a, this)`，一种是直接通过构造方法`new Scene(sceneRoot, viewHierarchy)`指定 view 对象生成

这两种方式其实差不多，第一种通过布局生成的方式在使用的时候会自动`inflate`加载布局生成`view`对象

用法比较简单；下面我们来看一下官方的 demo

1.  定义两个布局场景 A 和场景 B
    
    ```
    <!-- res/layout/scene_a.xml -->
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/scene_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
    
        <TextView
            android:id="@+id/text_view2"
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 2(a)" />
    
        <TextView
            android:id="@+id/text_view1"
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 1(a)" />
    
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 3(a)" />
    </LinearLayout>
    复制代码
    ```
    
    ```
    <!-- res/layout/scene_b.xml -->
    <LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
        android:id="@+id/scene_container"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical">
        <TextView
            android:id="@+id/text_view1"
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 1(b)" />
    
        <TextView
            android:id="@+id/text_view2"
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 2(b)" />
    
        <TextView
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 3(b)" />
    </LinearLayout>
    复制代码
    ```
    
2.  使用场景并执行动画
    
    ```
    // 创建a场景
    val aScene: Scene = Scene.getSceneForLayout(binding.sceneRoot, R.layout.scene_a, this)
    // 创建b场景
    val bScene: Scene = Scene.getSceneForLayout(binding.sceneRoot, R.layout.scene_b, this)
    var aSceneFlag = true
    // 添加点击事件，切换不同的场景
    binding.btClick1.setOnClickListener {
        if (aSceneFlag) {
            TransitionManager.go(bScene)
            aSceneFlag = false
        } else {
            TransitionManager.go(aScene)
            aSceneFlag = true
        }
    }
    复制代码
    ```
    
3.  执行效果如下:
    

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3cc44403897943aa8902bc05236c56fd~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

通过上面的效果可以看出，切换的一瞬间会立马变成指定场景的所有 view(文案全都变了)，只是应用了开始帧的位置而已，然后慢慢过渡到结束帧的位置；

```
// Scene的enter()方法源码
public void enter() {

    // Apply layout change, if any
    if (mLayoutId > 0 || mLayout != null) {
        // remove掉场景根视图下的所有view（即上一个场景）
        getSceneRoot().removeAllViews();
		// 添加当前场景的所有view
        if (mLayoutId > 0) {
            LayoutInflater.from(mContext).inflate(mLayoutId, mSceneRoot);
        } else {
            mSceneRoot.addView(mLayout);
        }
    }

    // Notify next scene that it is entering. Subclasses may override to configure scene.
    if (mEnterAction != null) {
        mEnterAction.run();
    }

    setCurrentScene(mSceneRoot, this);
}
复制代码
```

可见切换到指定场景，比如切换到场景 b, 会 remove 掉场景 a 的所有 view，然后添加场景 b 的所有 view

其次；这种方式的两种场景之间的切换动画；是通过 id 确定两个 view 之间的对应关系，从而确定 view 的开始帧和结束帧 来执行过渡动画；如果没有 id 对应关系的 view(即没有开始帧或结束帧), 会执行删除动画 (默认是渐隐动画) 或添加动画（默认是渐显动画）（看源码也可以通过 transtionName 属性来指定对应关系）

其视图匹配对应关系的源码如下：

```
// startValues 是开始帧所有对象的属性
// endValues 是结束帧所有对象的属性
private void matchStartAndEnd(TransitionValuesMaps startValues,
        TransitionValuesMaps endValues) {
    ArrayMap<View, TransitionValues> unmatchedStart =
            new ArrayMap<View, TransitionValues>(startValues.viewValues);
    ArrayMap<View, TransitionValues> unmatchedEnd =
            new ArrayMap<View, TransitionValues>(endValues.viewValues);

    for (int i = 0; i < mMatchOrder.length; i++) {
        switch (mMatchOrder[i]) {
            case MATCH_INSTANCE:
            	// 匹配是否相同的对象（可以通过改变view的属性，使用过渡动画）
                matchInstances(unmatchedStart, unmatchedEnd);
                break;
            case MATCH_NAME:
            	// 匹配transitionName属性是否相同（activity之间就是通过transtionName来匹配的）
                matchNames(unmatchedStart, unmatchedEnd,
                        startValues.nameValues, endValues.nameValues);
                break;
            case MATCH_ID:
            	// 匹配view的id是否相同
                matchIds(unmatchedStart, unmatchedEnd,
                        startValues.idValues, endValues.idValues);
                break;
            case MATCH_ITEM_ID:
            	// 特殊处理listview的item
                matchItemIds(unmatchedStart, unmatchedEnd,
                        startValues.itemIdValues, endValues.itemIdValues);
                break;
        }
    }
    // 添加没有匹配到的对象
    addUnmatched(unmatchedStart, unmatchedEnd);
}
复制代码
```

可见试图的匹配关系有很多种；可以根据 视图对象本身、视图的 id、视图的 transitionName 属性等匹配对应关系

定义场景比较简单，其实相对比较复杂的是`Transition`过度动画

缺点：个人觉得通过创建不同`Scene`对象实现动画效果比较麻烦，需要创建多套布局，后期难以维护；所以一般这种使用`TransitionManager.go(bScene)`方法指定`Scene`对象的方式基本不常用，一般都是使用`TransitionManager.beginDelayedTransition()`方法来实现过渡动画

下面我们来介绍`Transition`，并配合使用`TransitionManager.beginDelayedTransition()`方法实现动画效果

### `Transition(过渡)`

顾名思义 `Transition` 是过渡的意思，里面定义了怎么 记录开始帧的属性、记录结束帧的属性、创建动画或执行动画的逻辑

我们先看看`Transition`源码里比较重要的几个方法

```
// android.transition.Transition的源码
public abstract class Transition implements Cloneable {

	...
	
	// 通过实现这个方法记录view的开始帧的属性
	public abstract void captureStartValues(TransitionValues transitionValues);
	
	// 通过实现这个方法记录view的结束帧的属性
	public abstract void captureEndValues(TransitionValues transitionValues);
	
	// 通过记录的开始帧和结束帧的属性，创建动画
	// 默认返回null,即没有动画；需要你自己创建动画对象
	public Animator createAnimator(ViewGroup sceneRoot, TransitionValues startValues,
            TransitionValues endValues) {
        return null;
    }
    
    // 执行动画
    // mAnimators 里包含的就是上面createAnimator()方法创建的动画对象
    protected void runAnimators() {
        if (DBG) {
            Log.d(LOG_TAG, "runAnimators() on " + this);
        }
        start();
        ArrayMap<Animator, AnimationInfo> runningAnimators = getRunningAnimators();
        // Now start every Animator that was previously created for this transition
        for (Animator anim : mAnimators) {
            if (DBG) {
                Log.d(LOG_TAG, "  anim: " + anim);
            }
            if (runningAnimators.containsKey(anim)) {
                start();
                runAnimator(anim, runningAnimators);
            }
        }
        mAnimators.clear();
        end();
    }
    
    ...

}

复制代码
```

如果我们要自定义`Transition` 过渡动画的话，一般只需要重写前三个方法即可

当前系统也提供了一套完成的`Transition`过渡动画的子类

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/bec5c130ddff4bae903b513e2aba25e0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

上面这些都是系统提供的`Transition`子类 的 实现效果 和 其在`captureStartValues`、`captureEndValues`中记录的属性，然后在`createAnimator`方法中创建的属性动画 不断改变的属性

当然除了上面的一些类以外，系统还提供了`TransitionSet`类，可以指定一组动画；它也是的`Transition`子类

其`TransitionManager`中的默认动画就是 `AutoTransition` , 它是`TransitionSet`的子类

```
public class AutoTransition extends TransitionSet {
    public AutoTransition() {
        init();
    }

    public AutoTransition(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    private void init() {
        setOrdering(ORDERING_SEQUENTIAL);
        addTransition(new Fade(Fade.OUT)).
                addTransition(new ChangeBounds()).
                addTransition(new Fade(Fade.IN));
    }
}
复制代码
```

可见`AutoTransition`包含了 淡出、位移、改变大小、淡入 等一组效果

下面我们来自定义一些`Transition`，达到一些效果

```
// 记录和改变translationX、translationY属性
class XYTranslation : Transition() {
    override fun captureStartValues(transitionValues: TransitionValues?) {
        transitionValues ?: return
        transitionValues.values["translationX"] = transitionValues.view.translationX
        transitionValues.values["translationY"] = transitionValues.view.translationY
    }

    override fun captureEndValues(transitionValues: TransitionValues?) {
        transitionValues ?: return
        transitionValues.values["translationX"] = transitionValues.view.translationX
        transitionValues.values["translationY"] = transitionValues.view.translationY
    }

    override fun createAnimator(
        sceneRoot: ViewGroup?,
        startValues: TransitionValues?,
        endValues: TransitionValues?
    ): Animator? {
        if (startValues == null || endValues == null) return null
        val startX = startValues.values["translationX"] as Float
        val startY = startValues.values["translationY"] as Float
        val endX = endValues.values["translationX"] as Float
        val endY = endValues.values["translationY"] as Float

        var translationXAnim: Animator? = null
        if (startX != endX) {
            translationXAnim = ObjectAnimator.ofFloat(endValues.view, "translationX", startX, endX)
        }

        var translationYAnim: Animator? = null
        if (startY != endY) {
            translationYAnim = ObjectAnimator.ofFloat(endValues.view, "translationY", startY, endY)
        }

        return mergeAnimators(translationXAnim, translationYAnim)
    }

    fun mergeAnimators(animator1: Animator?, animator2: Animator?): Animator? {
        return if (animator1 == null) {
            animator2
        } else if (animator2 == null) {
            animator1
        } else {
            val animatorSet = AnimatorSet()
            animatorSet.playTogether(animator1, animator2)
            animatorSet
        }
    }
}
复制代码
```

```
// 记录和改变backgroundColor属性
class BackgroundColorTransition : Transition() {

    override fun captureStartValues(transitionValues: TransitionValues?) {
        transitionValues ?: return
        val drawable = transitionValues.view.background as? ColorDrawable ?: return
        transitionValues.values["backgroundColor"] = drawable.color
    }

    override fun captureEndValues(transitionValues: TransitionValues?) {
        transitionValues ?: return
        val drawable = transitionValues.view.background as? ColorDrawable ?: return
        transitionValues.values["backgroundColor"] = drawable.color
    }

    override fun createAnimator(
        sceneRoot: ViewGroup?,
        startValues: TransitionValues?,
        endValues: TransitionValues?
    ): Animator? {
        if (startValues == null || endValues == null) return null
        val startColor = (startValues.values["backgroundColor"] as? Int) ?: return null
        val endColor = (endValues.values["backgroundColor"] as? Int) ?: return null
        if (startColor != endColor) {
            return ObjectAnimator.ofArgb(endValues.view, "backgroundColor", startColor, endColor)
        }
        return super.createAnimator(sceneRoot, startValues, endValues)
    }
}
复制代码
```

非常简单，上面就自定义了两个`XYTranslation` 和`BackgroundColorTransition` 类，实现位移和改变背景颜色的效果

下面我们配合使用`TransitionManager.beginDelayedTransition()`方法，应用`XYTranslation` 和`BackgroundColorTransition` 两个动画过渡类，实现效果

```
<!-- res/layout/activity_main.xml -->
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/btClick"
        android:layout_width="100dp"
        android:layout_height="wrap_content"
        android:text="执行动画"/>

    <LinearLayout
        android:id="@+id/beginDelayRoot"
        android:layout_width="match_parent"
        android:layout_height="180dp"
        android:background="#ffff00"
        android:orientation="vertical">

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 1" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 2" />

        <TextView
            android:layout_width="wrap_content"
            android:layout_height="40dp"
            android:gravity="center"
            android:text="Text Line 3" />
    </LinearLayout>
</LinearLayout>
复制代码
```

```
class TransitionDemoActivity : AppCompatActivity() {

    private lateinit var binding: ActivityMainBinding
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityMainBinding.inflate(LayoutInflater.from(this))
        setContentView(binding.root)
        val backgroundColor1 = Color.parseColor("#ffff00")
        val backgroundColor2 = Color.parseColor("#00ff00")
        var index = 0
        binding.btClick.setOnClickListener {
            val view1 = binding.beginDelayRoot.getChildAt(0)
            val view2 = binding.beginDelayRoot.getChildAt(1)
            
            // 设置开始位置的x偏移量为100px（定义开始帧的属性）
            view1.translationX = 100f
            view2.translationX = 100f
            
            // 调用beginDelayedTransition 会立马调用 Transition的captureStartValues方法记录开始帧
            // 同时会添加一个OnPreDrawListener, 在屏幕刷新的下一帧触发onPreDraw() 方法，然后调用captureEndValues方法记录结束帧，然后开始执行动画
            TransitionManager.beginDelayedTransition(binding.beginDelayRoot, TransitionSet().apply {
            	// 实现上下移动（因为没有改变view的left属性所以, 所以它没有左右移动效果）
                addTransition(ChangeBounds())
                // 通过translationX属性实现左右移动
                addTransition(XYTranslation()) 
                // 通过backgroundColor属性改变背景颜色
                addTransition(BackgroundColorTransition())
            })
            
            // 下面开始改变视图的属性（定义结束帧的属性）
            // 将结束位置x偏移量为0px
            view1.translationX = 0f
            view2.translationX = 0f
            binding.beginDelayRoot.removeView(view1)
            binding.beginDelayRoot.removeView(view2)
            binding.beginDelayRoot.addView(view1)
            binding.beginDelayRoot.addView(view2)
            binding.beginDelayRoot.setBackgroundColor(if (index % 2 == 0) backgroundColor2 else backgroundColor1)
            index++
        }
    }
}
复制代码
```

其效果图如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/43194c36d7df4ed5ac91e948f266dc9a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

你可能会有些疑问，为什么上面将 translationX 设置成 100 之后，立马又改成了 0；这样有什么意义吗？？

可见`Transition`的使用和自定义也比较简单，同时也能达到一些比较炫酷的效果

请注意，改变 view 的属性并不会立马重新绘制视图，而是在屏幕的下一帧（60fps 的话，就是 16 毫秒一帧）去绘制；而在绘制下一帧之前调用了`TransitionManager.beginDelayedTransition()`方法，里面会触发`XYTransition`的`captureStartValues`方法记录开始帧（记录的`translationX`为 100）, 同时`TransitionManager`会添加`OnPreDrawListener`, 在屏幕下一帧到来触发 view 去绘制的时候，会先调用`OnPreDrawListener`的`onPreDraw()` 方法，里面又会触发`XYTransition`的`captureEndValues`方法记录结束帧的属性 (记录的`translationX`为 0), 然后应用动画 改变 view 的属性，最后交给 view 去绘制

上面讲了这么多，下面我们简单分析一下`TransitionManager.beginDelayedTransition`方法的源码

首先是`TransitionManager`的`beginDelayedTransition`方法

```
// android.transition.TransitionManager源码
public static void beginDelayedTransition(final ViewGroup sceneRoot, Transition transition) {
    if (!sPendingTransitions.contains(sceneRoot) && sceneRoot.isLaidOut()) {
        if (Transition.DBG) {
            Log.d(LOG_TAG, "beginDelayedTransition: root, transition = " +
                    sceneRoot + ", " + transition);
        }
        sPendingTransitions.add(sceneRoot);
        if (transition == null) {
            transition = sDefaultTransition;
        }
        final Transition transitionClone = transition.clone();
        sceneChangeSetup(sceneRoot, transitionClone);
        Scene.setCurrentScene(sceneRoot, null);
        sceneChangeRunTransition(sceneRoot, transitionClone);
    }
}
复制代码
```

里面代码比较少；我们主要看`sceneChangeSetup`和`sceneChangeRunTransition`方法的实现

```
// android.transition.TransitionManager源码
private static void sceneChangeSetup(ViewGroup sceneRoot, Transition transition) {

    // Capture current values
    ArrayList<Transition> runningTransitions = getRunningTransitions().get(sceneRoot);

    if (runningTransitions != null && runningTransitions.size() > 0) {
        for (Transition runningTransition : runningTransitions) {
        	// 暂停正在运行的动画
            runningTransition.pause(sceneRoot);
        }
    }

    if (transition != null) {
    	// 调用transition.captureValues；
    	// 其实第二个参数 true或false，表示是开始还是结束，对应会调用captureStartValues和captureEndValues 方法
        transition.captureValues(sceneRoot, true);
    }
	...
}
复制代码
```

我们来简单看看`transition.captureValues`的源码

```
// android.transition.Transition源码
void captureValues(ViewGroup sceneRoot, boolean start) {
    clearValues(start);
    // 如果你的 Transition 指定了目标view，就会执行这个if
    if ((mTargetIds.size() > 0 || mTargets.size() > 0)
            && (mTargetNames == null || mTargetNames.isEmpty())
            && (mTargetTypes == null || mTargetTypes.isEmpty())) {
        for (int i = 0; i < mTargetIds.size(); ++i) {
            int id = mTargetIds.get(i);
            View view = sceneRoot.findViewById(id);
            if (view != null) {
                TransitionValues values = new TransitionValues(view);
                if (start) {
                	// 记录开始帧的属性
                    captureStartValues(values);
                } else {
                	// 记录结束帧的属性
                    captureEndValues(values);
                }
                values.targetedTransitions.add(this);
                capturePropagationValues(values);
                if (start) {
                	// 缓存开始帧的属性到mStartValues中
                    addViewValues(mStartValues, view, values);
                } else {
                	// 缓存结束帧的属性到mEndValues中
                    addViewValues(mEndValues, view, values);
                }
            }
        }
        for (int i = 0; i < mTargets.size(); ++i) {
            View view = mTargets.get(i);
            TransitionValues values = new TransitionValues(view);
            if (start) {
            	// 记录开始帧的属性
                captureStartValues(values);
            } else {
            	// 记录结束帧的属性
                captureEndValues(values);
            }
            values.targetedTransitions.add(this);
            capturePropagationValues(values);
            if (start) {
            	// 缓存开始帧的属性到mStartValues中
                addViewValues(mStartValues, view, values);
            } else {
            	// 缓存结束帧的属性到mEndValues中
                addViewValues(mEndValues, view, values);
            }
        }
    } else {
    	// 没有指定目标view的情况
        captureHierarchy(sceneRoot, start);
    }
    ...
}

private void captureHierarchy(View view, boolean start) {
	...
    if (view.getParent() instanceof ViewGroup) {
        TransitionValues values = new TransitionValues(view);
        if (start) {
        	// 记录开始帧的属性
            captureStartValues(values);
        } else {
        	// 记录结束帧的属性
            captureEndValues(values);
        }
        values.targetedTransitions.add(this);
        capturePropagationValues(values);
        if (start) {
        	// 缓存开始帧的属性到mStartValues中
            addViewValues(mStartValues, view, values);
        } else {
        	// 缓存结束帧的属性到mEndValues中
            addViewValues(mEndValues, view, values);
        }
    }
    if (view instanceof ViewGroup) {
        // 递归遍历所有的children
        ViewGroup parent = (ViewGroup) view;
        for (int i = 0; i < parent.getChildCount(); ++i) {
            captureHierarchy(parent.getChildAt(i), start);
        }
    }
}
复制代码
```

可见`sceneChangeSetup`方法就会触发`Transition`的`captureStartValues` 方法

接下来我们来看看`sceneChangeRunTransition`方法

```
// android.transition.TransitionManager源码
private static void sceneChangeRunTransition(final ViewGroup sceneRoot,
            final Transition transition) {
    if (transition != null && sceneRoot != null) {
        MultiListener listener = new MultiListener(transition, sceneRoot);
        sceneRoot.addOnAttachStateChangeListener(listener);
        // 添加OnPreDrawListener
        sceneRoot.getViewTreeObserver().addOnPreDrawListener(listener);
    }
}

private static class MultiListener implements ViewTreeObserver.OnPreDrawListener,
            View.OnAttachStateChangeListener {
	...
    @Override
    public boolean onPreDraw() {
        removeListeners();
        ...
        // 这里就会触发captureEndValues方法，记录结束帧的属性
        mTransition.captureValues(mSceneRoot, false);
        if (previousRunningTransitions != null) {
            for (Transition runningTransition : previousRunningTransitions) {
                runningTransition.resume(mSceneRoot);
            }
        }
        // 开始执行动画
        // 这里就会调用 Transition的createAnimator方法 和 runAnimators方法
        mTransition.playTransition(mSceneRoot);

        return true;
    }
};
复制代码
```

`Transition`的`playTransition`没啥好看的，至此`TransitionManager`的`beginDelayedTransition`源码分析到这里

上面源码里你可能也看到了`Transition`可以设置目标视图，应用过渡动画, 主要是通过`addTarget`方法实现的，如果没有设置目标视图，默认就会遍历所有的 children 应用在所有的视图上

OverlayView 和 ViewGroupOverlay
------------------------------

`OverlayView`和`ViewGroupOverlay`是`Activity`共享元素动画实现里比较重要的一个类，所以就单独的介绍一下

`OverlayView`是针对`View`的一个顶层附加层 (即遮罩层)，它在 View 的所有内容绘制完成之后 再绘制

`ViewGroupOverlay`是针对`ViewGroup`的, 是`OverlayView`的子类，它在`ViewGroup`的所有内容 (包括所有的 children) 绘制完成之后 再绘制

```
// android.view.View源码
public ViewOverlay getOverlay() {
    if (mOverlay == null) {
        mOverlay = new ViewOverlay(mContext, this);
    }
    return mOverlay;
}
复制代码
```

```
// android.view.ViewGroup源码
@Override
public ViewGroupOverlay getOverlay() {
    if (mOverlay == null) {
        mOverlay = new ViewGroupOverlay(mContext, this);
    }
    return (ViewGroupOverlay) mOverlay;
}
复制代码
```

看上面的源码我们知道，可以直接调用`getOverlay`方法直接获取`OverlayView`或`ViewGroupOverlay`对象, 然后我们就可以在上面添加一些装饰等效果

`OverlayView`只支持添加`drawable`

`ViewGroupOverlay`支持添加`View` 和 `drawable`

注意：如果`View` 的 parent 不为 null, 则会自动先把它从 parent 中 remove 掉，然后添加到`ViewGroupOverlay`中

核心源码如下：

```
// OverlayViewGroup的add方法源码
public void add(@NonNull View child) {
    if (child == null) {
        throw new IllegalArgumentException("view must be non-null");
    }

    if (child.getParent() instanceof ViewGroup) {
        ViewGroup parent = (ViewGroup) child.getParent();
        ...
        // 将child从原来的parent中remove掉
        parent.removeView(child);
        if (parent.getLayoutTransition() != null) {
            // LayoutTransition will cause the child to delay removal - cancel it
            parent.getLayoutTransition().cancel(LayoutTransition.DISAPPEARING);
        }
        // fail-safe if view is still attached for any reason
        if (child.getParent() != null) {
            child.mParent = null;
        }
    }
    super.addView(child);
}
复制代码
```

用法也非常简单，我们来看看一个简单的 demo

```
class OverlayActivity : AppCompatActivity() {

    private lateinit var binding: ActivityOverlayBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityOverlayBinding.inflate(LayoutInflater.from(this))
        setContentView(binding.root)
        addViewToOverlayView()
        addDrawableToOverlayView()
        binding.btClick.setOnClickListener {
        	// 测试一下OverlayView的动画
            TransitionManager.beginDelayedTransition(binding.llOverlayContainer, XYTranslation())
            binding.llOverlayContainer.translationX += 100
        }
    }

    private fun addViewToOverlayView() {
        val view = View(this)
        view.layoutParams = LinearLayout.LayoutParams(100, 100)
        view.setBackgroundColor(Color.parseColor("#ff00ff"))
        // 需要手动调用layout，不然view显示不出来
        view.layout(0, 0, 100, 100)
        binding.llOverlayContainer.overlay.add(view)
    }

    private fun addDrawableToOverlayView() {
        binding.view2.post {
            val drawable = ContextCompat.getDrawable(this, R.mipmap.ic_temp)
            // 需要手动调用setBounds，不然drawable显示不出来
            drawable!!.setBounds(binding.view2.width / 2, 0, binding.view2.width, binding.view2.height / 2)
            binding.view2.overlay.add(drawable)
        }
    }
}
复制代码
```

效果图如下：

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/11dc0e162b054d2f958d44b69b21379b~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

这里唯一需要注意的是，如果是添加`view`，需要手动调用`layout`布局，不然`view`显示不出来；如果添加的是`drawable` 需要手动调用`setBounds`，不然`drawable`也显示不出来

GhostView
---------

`GhostView` 是`Activity`共享元素动画实现里比较重要的一个类，所以就单独的介绍一下

它的作用是在不改变`view`的`parent`的情况下，将`view`绘制在另一个`parent`下

我们先看看`GhostView` 的部分源码

```
public class GhostView extends View {
    private final View mView;
    private int mReferences;
    private boolean mBeingMoved;

    private GhostView(View view) {
        super(view.getContext());
        mView = view;
        mView.mGhostView = this;
        final ViewGroup parent = (ViewGroup) mView.getParent();
        // 这句代码 让mView在原来的parent中隐藏（即不绘制视图）
        mView.setTransitionVisibility(View.INVISIBLE);
        parent.invalidate();
    }

    @Override
    public void setVisibility(@Visibility int visibility) {
        super.setVisibility(visibility);
        if (mView.mGhostView == this) {
        	// 如果view在ghostview中绘制(可见)，则设置在原来的parent不绘制(不可见)
        	// 如果view在ghostview中不绘制(不可见)，则设置在原来的parent绘制(可见)
            int inverseVisibility = (visibility == View.VISIBLE) ? View.INVISIBLE : View.VISIBLE;
            mView.setTransitionVisibility(inverseVisibility);
        }
    }
}

复制代码
```

看源码得知 如果把`View` 添加到`GhostView`里，则默认会调用`view`的`setTransitionVisibility`方法 将`view`设置成在`parent`中不可见, 在`GhostView`里可见；调用`GhostView`的`setVisibility`方法设置 要么在`GhostView`中可见，要么在`parent`中可见

系统内部是使用`GhostView.addGhost`静态方法添加`GhostView`的

我们来看看添加`GhostView`的`addGhost`静态方法源码

```
public static GhostView addGhost(View view, ViewGroup viewGroup, Matrix matrix) {
    if (!(view.getParent() instanceof ViewGroup)) {
        throw new IllegalArgumentException("Ghosted views must be parented by a ViewGroup");
    }
    // 获取 ViewGroupOverlay
    ViewGroupOverlay overlay = viewGroup.getOverlay();
    ViewOverlay.OverlayViewGroup overlayViewGroup = overlay.mOverlayViewGroup;
    GhostView ghostView = view.mGhostView;
    int previousRefCount = 0;
    if (ghostView != null) {
        View oldParent = (View) ghostView.getParent();
        ViewGroup oldGrandParent = (ViewGroup) oldParent.getParent();
        if (oldGrandParent != overlayViewGroup) {
            previousRefCount = ghostView.mReferences;
            oldGrandParent.removeView(oldParent);
            ghostView = null;
        }
    }
    if (ghostView == null) {
        if (matrix == null) {
            matrix = new Matrix();
            calculateMatrix(view, viewGroup, matrix);
        }
        // 创建GhostView
        ghostView = new GhostView(view);
        ghostView.setMatrix(matrix);
        FrameLayout parent = new FrameLayout(view.getContext());
        parent.setClipChildren(false);
        copySize(viewGroup, parent);
        // 设置GhostView的大小
        copySize(viewGroup, ghostView);
        // 将ghostView添加到了parent中
        parent.addView(ghostView);
        ArrayList<View> tempViews = new ArrayList<View>();
        int firstGhost = moveGhostViewsToTop(overlay.mOverlayViewGroup, tempViews);
        // 将parent添加到了ViewGroupOverlay中
        insertIntoOverlay(overlay.mOverlayViewGroup, parent, ghostView, tempViews, firstGhost);
        ghostView.mReferences = previousRefCount;
    } else if (matrix != null) {
        ghostView.setMatrix(matrix);
    }
    ghostView.mReferences++;
    return ghostView;
}
复制代码
```

可见内部的实现最终将`GhostView`添加到了`ViewGroupOverlay`(遮罩层) 里

配合`GhostView`，同时也解决了`ViewGroupOverlay`会将`view`从`parent`中`remove`的问题（即可同时在`ViewGroupOverlay`和原来的`parent`中绘制）

我们来看看一个简单的 demo

```
class GhostViewActivity : AppCompatActivity() {

    private lateinit var binding: ActivityGhostViewBinding
    private var ghostView: View? = null
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = ActivityGhostViewBinding.inflate(LayoutInflater.from(this))
        setContentView(binding.root)
        binding.btClick.setOnClickListener {
        	// 配合动画看看效果
            TransitionManager.beginDelayedTransition(binding.llOverlayContainer, XYTranslation())
            binding.llOverlayContainer.translationX += 100
        }
        binding.btClick2.setOnClickListener {
            val ghostView = ghostView ?: return@setOnClickListener
           	// 测试一下ghostView的setVisibility方法效果
            if (ghostView.isVisible) {
                ghostView.visibility = View.INVISIBLE
            } else {
                ghostView.visibility = View.VISIBLE
            }
            (binding.view1.parent as ViewGroup).invalidate()
        }
        binding.view1.post {
        	// 创建一个GhostView添加到window.decorView的ViewGroupOverlay中
            ghostView = addGhost(binding.view1, window.decorView as ViewGroup)
        }
    }

	// 我们无法直接使用GhostView，只能临时使用反射看看效果
    private fun addGhost(view: View, viewGroup: ViewGroup): View {
        val ghostViewClass = Class.forName("android.view.GhostView")
        val addGhostMethod: Method = ghostViewClass.getMethod(
            "addGhost", View::class.java,
            ViewGroup::class.java, Matrix::class.java
        )
        return addGhostMethod.invoke(null, view, viewGroup, null) as View
    }
}
复制代码
```

效果图如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/09650019541b4e19a9bbb77ecee0321c~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

可见使用`GhostView`并通过`setVisibility`方法，实现的效果是 既可以在`window.decorView`的`ViewGroupOverlay`中绘制，也可以在原来的`parent`中绘制

那怎么同时绘制呢？

只需要在`addGhost`之后强制设置`view`的`setTransitionVisibility`为`View.VISIBLE`即可

```
binding.view1.post {
    ghostView = addGhost(binding.view1, window.decorView as ViewGroup)
    // android 10 之前setTransitionVisibility是hide方法
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
        binding.view1.setTransitionVisibility(View.VISIBLE)
        (binding.view1.parent as ViewGroup).invalidate()
    }
}
复制代码
```

效果图如下：

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/815972a979a1463f8059ed50ed46d3d0~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

`Activity`的共享元素源码分析
-------------------

好的，上面的准备工作做完了之后，下面我们来真正的分析`Activity`的共享元素源码

### 我们先以`ActivityA`打开`ActivityB`为例

先是调用`ActivityOptions.makeSceneTransitionAnimation`创建包含共享元素的`ActivityOptions`对象

```
//android.app.ActivityOptions类的源码
public class ActivityOptions {
	...
	public static ActivityOptions makeSceneTransitionAnimation(Activity activity,
	        Pair<View, String>... sharedElements) {
	    ActivityOptions opts = new ActivityOptions();
	    // activity.mExitTransitionListener是SharedElementCallback对象
	    makeSceneTransitionAnimation(activity, activity.getWindow(), opts,
	            activity.mExitTransitionListener, sharedElements);
	    return opts;
	}
}
复制代码
```

其中`activity`的`mExitTransitionListener`是`SharedElementCallback`对象，默认值是`SharedElementCallback.NULL_CALLBACK`，使用的是默认实现；可以调用`Activity`的`setExitSharedElementCallback`方法设置这个对象, 但是大多数情况下用默认的即可

下面我们来简单介绍下`SharedElementCallback`的一些回调在什么情况下触发

```
public abstract class SharedElementCallback {
	...
    static final SharedElementCallback NULL_CALLBACK = new SharedElementCallback() {
    };
    
    /**
    * 共享元素 开始帧准备好了 触发
    * @param sharedElementNames 共享元素名称
    * @param sharedElements 共享元素view，并且已经将开始帧的属性应用到view里了
    * @param sharedElementSnapshots 调用SharedElementCallback.onCreateSnapshotView方法创建的快照
    **/
    public void onSharedElementStart(List<String> sharedElementNames,
            List<View> sharedElements, List<View> sharedElementSnapshots) {}
            
    /**
    * 共享元素 结束帧准备好了 触发
    * @param sharedElementNames 共享元素名称
    * @param sharedElements 共享元素view，并且已经将结束帧的属性应用到view里了
    * @param sharedElementSnapshots 调用SharedElementCallback.onCreateSnapshotView方法创建的快照
    * 		注意：跟onSharedElementStart方法的sharedElementSnapshots参数是同一个对象
    */
    public void onSharedElementEnd(List<String> sharedElementNames,
        List<View> sharedElements, List<View> sharedElementSnapshots) {}
        
    /**
    * 比如在ActivityA存在，而在ActivityB不存在的共享元素 回调
    * @param rejectedSharedElements 在ActivityB中不存在的共享元素
    **/
    public void onRejectSharedElements(List<View> rejectedSharedElements) {}
    
    /**
    * 需要做动画的共享元素映射关系准备好之后 回调
    * @param names 支持的所有共享元素名称（是ActivityA打开ActivityB时传过来的所有共享元素名称）
    * @param sharedElements 需要做动画的共享元素名称及view的对应关系
    *	 	注意：比如ActivityA打开ActivityB，对于ActivityA中的回调 names和sharedElements的大小基本上是一样的，ActivityB中的回调就可能会不一样
    **/
    public void onMapSharedElements(List<String> names, Map<String, View> sharedElements) {}
    
    /**
    * 将共享元素 view 生成 bitmap 保存在Parcelable中，最终这个Parcelable会保存在sharedElementBundle中
    * 如果是ActivityA打开ActivityB, 则会把sharedElementBundle传给ActivityB
    **/
    public Parcelable onCaptureSharedElementSnapshot(View sharedElement, Matrix viewToGlobalMatrix,
        RectF screenBounds) {
        ...
     }
	 
     /**
     * 根据snapshot反过来创建view
     * 如果是ActivityA打开ActivityB, ActivityB接收到Parcelable对象后，在适当的时候会调用这个方法创建出view对象
     **/
    public View onCreateSnapshotView(Context context, Parcelable snapshot) {
            ...
    }
 	
    /**
    * 当共享元素和sharedElementBundle对象都已经传第给对方的时候触发（表明接下来可以准备执行过场动画了）
    * 比如: ActivityA 打开 ActivityB, ActivityA调用完onCaptureSharedElementSnapshot将信息保存在sharedElementBundle中，然后传给ActivityB，这个时候ActivityA 和 ActivityB的SharedElementCallback都会触发onSharedElementsArrived方法
    **/
    public void onSharedElementsArrived(List<String> sharedElementNames,
        List<View> sharedElements, OnSharedElementsReadyListener listener) {
        listener.onSharedElementsReady();
    }
}
复制代码
```

`SharedElementCallback`的每个回调方法的大致意思是这样的

接下来我门继续往下看源码 `makeSceneTransitionAnimation`

```
//android.app.ActivityOptions类的源码
public class ActivityOptions {
	...
	static ExitTransitionCoordinator makeSceneTransitionAnimation(Activity activity, Window window,
            ActivityOptions opts, SharedElementCallback callback,
            Pair<View, String>[] sharedElements) {
        // activity的window一定要添加Window.FEATURE_ACTIVITY_TRANSITIONS特征
        if (!window.hasFeature(Window.FEATURE_ACTIVITY_TRANSITIONS)) {
            opts.mAnimationType = ANIM_DEFAULT;
            return null;
        }
        opts.mAnimationType = ANIM_SCENE_TRANSITION;

        ArrayList<String> names = new ArrayList<String>();
        ArrayList<View> views = new ArrayList<View>();

        if (sharedElements != null) {
            for (int i = 0; i < sharedElements.length; i++) {
                Pair<View, String> sharedElement = sharedElements[i];
                String sharedElementName = sharedElement.second;
                if (sharedElementName == null) {
                    throw new IllegalArgumentException("Shared element name must not be null");
                }
                names.add(sharedElementName);
                View view = sharedElement.first;
                if (view == null) {
                    throw new IllegalArgumentException("Shared element must not be null");
                }
                views.add(sharedElement.first);
            }
        }
	//创建ActivityA退出时的过场动画核心类
        ExitTransitionCoordinator exit = new ExitTransitionCoordinator(activity, window,
                callback, names, names, views, false);
        //注意 这个opts保存了ActivityA的exit对象，到时候会传给ActivityB的EnterTransitionCoordinator对象
        opts.mTransitionReceiver = exit;
        // 支持的共享元素名称
        opts.mSharedElementNames = names;
        // 是否是返回
        opts.mIsReturning = (activity == null);
        if (activity == null) {
            opts.mExitCoordinatorIndex = -1;
        } else {
        	// 将exit添加到mActivityTransitionState对象中，然后由ActivityTransitionState对象管理和调用exit对象里的方法
            opts.mExitCoordinatorIndex =
                    activity.mActivityTransitionState.addExitTransitionCoordinator(exit);
        }
        return exit;
    }
}

复制代码
```

接下来我们来看看`ExitTransitionCoordinator`这个类的构造函数干了啥

```
// android.app.ActivityTransitionCoordinator源码
abstract class ActivityTransitionCoordinator extends ResultReceiver {
    ...
    public ActivityTransitionCoordinator(Window window,
            ArrayList<String> allSharedElementNames,
            SharedElementCallback listener, boolean isReturning) {
        super(new Handler());
        mWindow = window;
        // activity里的SharedElementCallback对象
        mListener = listener;
        // 支持的所有共享元素名称
        // 比如ActivityA打开ActivityB,则是makeSceneTransitionAnimation方法传过来的共享元素名称
        mAllSharedElementNames = allSharedElementNames;
        // 是否是返回
        mIsReturning = isReturning;
    }
}

// android.app.ExitTransitionCoordinator源码
public class ExitTransitionCoordinator extends ActivityTransitionCoordinator {

	public ExitTransitionCoordinator(ExitTransitionCallbacks exitCallbacks,
            Window window, SharedElementCallback listener, ArrayList<String> names,
            ArrayList<String> accepted, ArrayList<View> mapped, boolean isReturning) {
        super(window, names, listener, isReturning);
        // viewsReady主要有以下作用
        // 1. 准备好需要执行动画的共享元素，并排序 保存在mSharedElementNames和mSharedElements中
        // 2. 准备好需要做退出动画的非共享元素,保存在mTransitioningViews中
        // 3. 这里会触发 SharedElementCallback的onMapSharedElements回调
        viewsReady(mapSharedElements(accepted, mapped));
        // 将mTransitioningViews中的不在屏幕内的非共享元素剔除掉
        stripOffscreenViews();
        mIsBackgroundReady = !isReturning;
        mExitCallbacks = exitCallbacks;
    }
}
复制代码
```

这里比较重要的方法就是`viewsReady`方法, 核心作用就是我上面说的

```
// android.app.ActivityTransitionCoordinator源码
protected void viewsReady(ArrayMap<String, View> sharedElements) {
    // 剔除掉不在mAllSharedElementNames中共享元素
    sharedElements.retainAll(mAllSharedElementNames);
    if (mListener != null) {
        // 执行SharedElementCallback的onMapSharedElements回调
        mListener.onMapSharedElements(mAllSharedElementNames, sharedElements);
    }
    // 共享元素排序
    setSharedElements(sharedElements);
    if (getViewsTransition() != null && mTransitioningViews != null) {
        ViewGroup decorView = getDecor();
        if (decorView != null) {
            // 遍历decorView收集非共享元素
            decorView.captureTransitioningViews(mTransitioningViews);
        }
        // 移除掉其中的共享元素
        mTransitioningViews.removeAll(mSharedElements);
    }
    setEpicenter();
}
复制代码
```

准备好`ActivityOptions`参数后，就可以调用`startActivity(Intent intent, @Nullable Bundle options)`方法了，然后就会调用到`activity`的`cancelInputsAndStartExitTransition`方法

```
// android.app.Activity源码
private void cancelInputsAndStartExitTransition(Bundle options) {
    final View decor = mWindow != null ? mWindow.peekDecorView() : null;
    if (decor != null) {
        decor.cancelPendingInputEvents();
    }
    if (options != null) {
        // 开始处理ActivityA的退场动画
        mActivityTransitionState.startExitOutTransition(this, options);
    }
}
复制代码
```

```
// android.app.ActivityTransitionState源码
public void startExitOutTransition(Activity activity, Bundle options) {
    mEnterTransitionCoordinator = null;
    if (!activity.getWindow().hasFeature(Window.FEATURE_ACTIVITY_TRANSITIONS) ||
            mExitTransitionCoordinators == null) {
        return;
    }
    ActivityOptions activityOptions = new ActivityOptions(options);
    if (activityOptions.getAnimationType() == ActivityOptions.ANIM_SCENE_TRANSITION) {
        int key = activityOptions.getExitCoordinatorKey();
        int index = mExitTransitionCoordinators.indexOfKey(key);
        if (index >= 0) {
            mCalledExitCoordinator = mExitTransitionCoordinators.valueAt(index).get();
            mExitTransitionCoordinators.removeAt(index);
            if (mCalledExitCoordinator != null) {
                mExitingFrom = mCalledExitCoordinator.getAcceptedNames();
                mExitingTo = mCalledExitCoordinator.getMappedNames();
                mExitingToView = mCalledExitCoordinator.copyMappedViews();
                // 调用ExitTransitionCoordinator的startExit
                mCalledExitCoordinator.startExit();
            }
        }
    }
}
复制代码
```

这里`startExitOutTransition`里面就会调用`ExitTransitionCoordinator`的`startExit`方法

```
// android.app.ExitTransitionCoordinator源码
public void startExit() {
    if (!mIsExitStarted) {
        backgroundAnimatorComplete();
        mIsExitStarted = true;
        pauseInput();
        ViewGroup decorView = getDecor();
        if (decorView != null) {
            decorView.suppressLayout(true);
        }
        // 将共享元素用GhostView包裹，然后添加的Activity的decorView的OverlayView中
        moveSharedElementsToOverlay();
        startTransition(this::beginTransitions);
    }
}
复制代码
```

这里的`moveSharedElementsToOverlay`方法比较重要，会使用到最开始介绍的`GhostView`和`OverlayView` ，目的是将共享元素绘制到最顶层

然后开始执行`beginTransitions`方法

```
// android.app.ExitTransitionCoordinator源码
private void beginTransitions() {
	// 获取共享元素的过渡动画类Transition，可以通过window.setSharedElementExitTransition方法设置
	// 一般不需要设置 有默认值
    Transition sharedElementTransition = getSharedElementExitTransition();
    // 获取非共享元素的过渡动画类Transition，也可以通过window.setExitTransition方法设置
    Transition viewsTransition = getExitTransition();
	
	// 将sharedElementTransition和viewsTransition合并成一个 TransitionSet
    Transition transition = mergeTransitions(sharedElementTransition, viewsTransition);
    ViewGroup decorView = getDecor();
    if (transition != null && decorView != null) {
        setGhostVisibility(View.INVISIBLE);
        scheduleGhostVisibilityChange(View.INVISIBLE);
        if (viewsTransition != null) {
            setTransitioningViewsVisiblity(View.VISIBLE, false);
        }
        // 开始采集开始帧和结束帧，执行过度动画
        TransitionManager.beginDelayedTransition(decorView, transition);
        scheduleGhostVisibilityChange(View.VISIBLE);
        setGhostVisibility(View.VISIBLE);
        if (viewsTransition != null) {
            setTransitioningViewsVisiblity(View.INVISIBLE, false);
        }
        decorView.invalidate();
    } else {
        transitionStarted();
    }
}
复制代码
```

这里在`TransitionManager.beginDelayedTransition`的前后都有屌用`setGhostVisibility`和`scheduleGhostVisibilityChange`方法，是为了采集前后帧的属性，执行过度动画，采集完成之后，会显示`GhostView`, 而隐藏原来 parent 里的共享元素`view`

上面的`sharedElementTransition`和`viewsTransition`都添加了监听器，在动画结束之后分别调用`sharedElementTransitionComplete`和`viewsTransitionComplete`方法

```
// android.app.ExitTransitionCoordinator源码
@Override
protected void sharedElementTransitionComplete() {
    // 这里就会采集共享元素当前的属性（大小、位置等），会触发SharedElementCallback.onCaptureSharedElementSnapshot方法
    mSharedElementBundle = mExitSharedElementBundle == null
            ? captureSharedElementState() : captureExitSharedElementsState();
    super.sharedElementTransitionComplete();
}

// android.app.ActivityTransitionCoordinator源码
protected void viewsTransitionComplete() {
    mViewsTransitionComplete = true;
    startInputWhenTransitionsComplete();
}
复制代码
```

然后在`startInputWhenTransitionsComplete`方法里会调用`onTransitionsComplete`方法，最终会调用`notifyComplete`方法

```
// android.app.ExitTransitionCoordinator源码
protected boolean isReadyToNotify() {
    // 1. 调用完sharedElementTransitionComplete后，mSharedElementBundle不为null
    // 2. mResultReceiver是在ActivityB创建完EnterTransitionCoordinator之后，发送MSG_SET_REMOTE_RECEIVER消息 将EnterTransitionCoordinator传给ActivityA之后不为null
    // 3. 看构造函数，一开始就为true
    return mSharedElementBundle != null && mResultReceiver != null && mIsBackgroundReady;
}

protected void notifyComplete() {
    if (isReadyToNotify()) {
        if (!mSharedElementNotified) {
            mSharedElementNotified = true;
            // 延迟发送一个MSG_CANCEL消息，清空各种状态等
            delayCancel();

            if (!mActivity.isTopOfTask()) {
            	//  mResultReceiver是ActivityB的EnterTransitionCoordinator对象
                mResultReceiver.send(MSG_ALLOW_RETURN_TRANSITION, null);
            }

            if (mListener == null) {
                mResultReceiver.send(MSG_TAKE_SHARED_ELEMENTS, mSharedElementBundle);
                notifyExitComplete();
            } else {
                final ResultReceiver resultReceiver = mResultReceiver;
                final Bundle sharedElementBundle = mSharedElementBundle;
                // 触发SharedElementCallback.onSharedElementsArrived
                mListener.onSharedElementsArrived(mSharedElementNames, mSharedElements,
                        new OnSharedElementsReadyListener() {
                            @Override
                            public void onSharedElementsReady() {
                            	// 发送MSG_TAKE_SHARED_ELEMENTS，将共享元素的sharedElementBundle信息传递给ActivityB
                                resultReceiver.send(MSG_TAKE_SHARED_ELEMENTS,
                                        sharedElementBundle);
                                notifyExitComplete();
                            }
                        });
            }
        } else {
            notifyExitComplete();
        }
    }
}
复制代码
```

这里的`notifyComplete`会在特定的条件下不断触发，一旦`isReadyToNotify`为`true`，就会执行方法里的逻辑

这里可能比较关心的是`resultReceiver`到底是什么对象，是怎么赋值的？？？（这里在接下来讲到 ActivityB 的时候会介绍到）

`ActivityA`的流程暂时到这里就没发走下去了

接下来我们来看看`ActivityB`, 当打开了`ActivityB`的时候会执行`Activity`的`performStart`方法

```
// android.app.Activity源码
final void performStart(String reason) {
    dispatchActivityPreStarted();
    // getActivityOptions() 获取到的是在上面ActivityA中创建的ActivityOptions对象
    // 里面有支持的所有的共享元素名称、ActivityA的ExitTransitionCoordinator对象、返回标志等信息
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    mFragments.noteStateNotSaved();
    mCalled = false;
    mFragments.execPendingActions();
    mInstrumentation.callActivityOnStart(this);
    EventLogTags.writeWmOnStartCalled(mIdent, getComponentName().getClassName(), reason);
 	...
    mActivityTransitionState.enterReady(this);
    dispatchActivityPostStarted();
}

复制代码
```

然后就进入到`ActivityTransitionState`的`enterReady`方法

```
// android.app.ActivityTransitionState源码
public void enterReady(Activity activity) {
    if (mEnterActivityOptions == null || mIsEnterTriggered) {
        return;
    }
    mIsEnterTriggered = true;
    mHasExited = false;
    // 获取ActivityA传过来的所有共享元素名称
    ArrayList<String> sharedElementNames = mEnterActivityOptions.getSharedElementNames();
    // 获取ActivityA的ExitTransitionCoordinator
    ResultReceiver resultReceiver = mEnterActivityOptions.getResultReceiver();
    // 获取返回标志
    final boolean isReturning = mEnterActivityOptions.isReturning();
    if (isReturning) {
        restoreExitedViews();
        activity.getWindow().getDecorView().setVisibility(View.VISIBLE);
    }
    // 创建EnterTransitionCoordinator，保存resultReceiver、sharedElementNames等对象
    mEnterTransitionCoordinator = new EnterTransitionCoordinator(activity,
            resultReceiver, sharedElementNames, mEnterActivityOptions.isReturning(),
            mEnterActivityOptions.isCrossTask());
    if (mEnterActivityOptions.isCrossTask()) {
        mExitingFrom = new ArrayList<>(mEnterActivityOptions.getSharedElementNames());
        mExitingTo = new ArrayList<>(mEnterActivityOptions.getSharedElementNames());
    }

    if (!mIsEnterPostponed) { // 是否推迟执行动画，配合postponeEnterTransition方法使用
        startEnter();
    }
}
复制代码
```

上面的`mIsEnterPostponed`, 默认值是 false，可以通过`postponeEnterTransition`和 `startPostponedEnterTransition`控制什么时候执行动画，这个不是重点，我们忽略

我们先来看看`EnterTransitionCoordinator`的构造函数

```
// android.app.EnterTransitionCoordinator源码
EnterTransitionCoordinator(Activity activity, ResultReceiver resultReceiver,
        ArrayList<String> sharedElementNames, boolean isReturning, boolean isCrossTask) {
    super(activity.getWindow(), sharedElementNames,
            getListener(activity, isReturning && !isCrossTask), isReturning);
    mActivity = activity;
    mIsCrossTask = isCrossTask;
    // 保存ActivityA的ExitTransitionCoordinator对象到mResultReceiver中
    setResultReceiver(resultReceiver);
    // 这里会将ActivityB的window背景设置成透明
    prepareEnter();
    
    // 构造resultReceiverBundle，保存EnterTransitionCoordinator对象
    Bundle resultReceiverBundle = new Bundle();
    resultReceiverBundle.putParcelable(KEY_REMOTE_RECEIVER, this);
    // 发送MSG_SET_REMOTE_RECEIVER消息，将EnterTransitionCoordinator对象传递给ActivityA
    mResultReceiver.send(MSG_SET_REMOTE_RECEIVER, resultReceiverBundle);
    ...
}
复制代码
```

这个时候`ActivityA`那边就接收到了`ActivityB`的`EnterTransitionCoordinator`对象

接下来我门看看`ActivityA`是怎么接收`MSG_SET_REMOTE_RECEIVER`消息的

```
// android.app.ExitTransitionCoordinator 源码
@Override
protected void onReceiveResult(int resultCode, Bundle resultData) {
    switch (resultCode) {
        case MSG_SET_REMOTE_RECEIVER:
            stopCancel();
            // 将`ActivityB`的`EnterTransitionCoordinator`对象保存到mResultReceiver对象中
            mResultReceiver = resultData.getParcelable(KEY_REMOTE_RECEIVER);
            if (mIsCanceled) {
                mResultReceiver.send(MSG_CANCEL, null);
                mResultReceiver = null;
            } else {
            	//又会触发notifyComplete(),这个时候isReadyToNotify就为true了，就会执行notifyComplete里的代码
                notifyComplete();
            }
            break;
        ...
    }
}
复制代码
```

这个时候`ActivityA`的共享元素动画的核心逻辑就基本已经走完了

接下来继续看`ActivityB`的逻辑，接来下会执行`startEnter`方法

```
// android.app.ActivityTransitionState源码
private void startEnter() {
    if (mEnterTransitionCoordinator.isReturning()) { // 这个为false
        if (mExitingToView != null) {
            mEnterTransitionCoordinator.viewInstancesReady(mExitingFrom, mExitingTo,
                    mExitingToView);
        } else {
            mEnterTransitionCoordinator.namedViewsReady(mExitingFrom, mExitingTo);
        }
    } else {
    	// 会执行这个逻辑
        mEnterTransitionCoordinator.namedViewsReady(null, null);
        mPendingExitNames = null;
    }

    mExitingFrom = null;
    mExitingTo = null;
    mExitingToView = null;
    mEnterActivityOptions = null;
}
复制代码
```

也就是说接下来会触发`EnterTransitionCoordinator`的`namedViewsReady`方法, 然后就会触发`viewsReady`方法

```
// android.app.EnterTransitionCoordinator源码
public void namedViewsReady(ArrayList<String> accepted, ArrayList<String> localNames) {
    triggerViewsReady(mapNamedElements(accepted, localNames));
}

// android.app.EnterTransitionCoordinator源码
private void triggerViewsReady(final ArrayMap<String, View> sharedElements) {
    if (mAreViewsReady) {
        return;
    }
    mAreViewsReady = true;
    final ViewGroup decor = getDecor();
    // Ensure the views have been laid out before capturing the views -- we need the epicenter.
    if (decor == null || (decor.isAttachedToWindow() &&
            (sharedElements.isEmpty() || !sharedElements.valueAt(0).isLayoutRequested()))) {
        viewsReady(sharedElements);
    } else {
        mViewsReadyListener = OneShotPreDrawListener.add(decor, () -> {
            mViewsReadyListener = null;
            // 触发viewsReady
            viewsReady(sharedElements);
        });
        decor.invalidate();
    }
}
复制代码
```

`EnterTransitionCoordinator`的`viewsReady`代码逻辑 跟 `ExitTransitionCoordinator`的差不多，准备好共享元素和非共享元素等，

```
// android.app.EnterTransitionCoordinator源码
@Override
protected void viewsReady(ArrayMap<String, View> sharedElements) {
    // 准备好共享元素和非共享元素
    super.viewsReady(sharedElements);
    mIsReadyForTransition = true;
    // 隐藏共享元素
    hideViews(mSharedElements);
    Transition viewsTransition = getViewsTransition();
    if (viewsTransition != null && mTransitioningViews != null) {
    	// 将mTransitioningViews当作target设置到viewsTransition中
        removeExcludedViews(viewsTransition, mTransitioningViews);
        // 剔除掉mTransitioningViews中不在屏幕中的view
        stripOffscreenViews();
        // 隐藏非共享元素
        hideViews(mTransitioningViews);
    }
    if (mIsReturning) {
        sendSharedElementDestination();
    } else {
    	// 将共享元素用GhostView包裹，然后添加到ActivityB的decorView的OverlayView中
        moveSharedElementsToOverlay();
    }
    if (mSharedElementsBundle != null) {
        onTakeSharedElements();
    }
}
复制代码
```

一般情况下，这个时候`mSharedElementsBundle`为 null, 所以不会走`onTakeSharedElements`方法

这里的`mSharedElementsBundle`对象是在 ActivityA 的`notifyComplete`中发送的`MSG_TAKE_SHARED_ELEMENTS`消息传过来的

```
// android.app.EnterTransitionCoordinator源码
@Override
protected void onReceiveResult(int resultCode, Bundle resultData) {
    switch (resultCode) {
        case MSG_TAKE_SHARED_ELEMENTS:
            if (!mIsCanceled) {
                mSharedElementsBundle = resultData;
                onTakeSharedElements();
            }
            break;
        ...
    }
}
复制代码
```

可见当`ActivityB`接收到`MSG_TAKE_SHARED_ELEMENTS`消息，赋值完`mSharedElementsBundle`之后，也会执行`onTakeSharedElements`方法

接下来我们来看看`onTakeSharedElements`方法

```
// android.app.EnterTransitionCoordinator源码
private void onTakeSharedElements() {
    if (!mIsReadyForTransition || mSharedElementsBundle == null) {
        return;
    }
    final Bundle sharedElementState = mSharedElementsBundle;
    mSharedElementsBundle = null;
    OnSharedElementsReadyListener listener = new OnSharedElementsReadyListener() {
        @Override
        public void onSharedElementsReady() {
            final View decorView = getDecor();
            if (decorView != null) {
                OneShotPreDrawListener.add(decorView, false, () -> {
                    startTransition(() -> {
                            startSharedElementTransition(sharedElementState);
                    });
                });
                decorView.invalidate();
            }
        }
    };
    if (mListener == null) {
        listener.onSharedElementsReady();
    } else {
    	// 触发SharedElementCallback.onSharedElementsArrived回调
        mListener.onSharedElementsArrived(mSharedElementNames, mSharedElements, listener);
    }
}
复制代码
```

接下来就会执行`startTransition`方法，然后执行`startSharedElementTransition`方法, 开始执行`ActivityB`的动画了

```
//  android.app.EnterTransitionCoordinator源码
private void startSharedElementTransition(Bundle sharedElementState) {
    ViewGroup decorView = getDecor();
    if (decorView == null) {
        return;
    }
    // Remove rejected shared elements
    ArrayList<String> rejectedNames = new ArrayList<String>(mAllSharedElementNames);
    // 过滤出ActivityA存在，ActivityB不存在的共享元素
    rejectedNames.removeAll(mSharedElementNames);
    // 根据ActivityA传过来的共享元素sharedElementState信息，创建快照view对象
    // 这里会触发SharedElementCallback.onCreateSnapshotView方法
    ArrayList<View> rejectedSnapshots = createSnapshots(sharedElementState, rejectedNames);
    if (mListener != null) {
    	// 触发SharedElementCallback.onRejectSharedElements方法
        mListener.onRejectSharedElements(rejectedSnapshots);
    }
    removeNullViews(rejectedSnapshots);
    // 执行渐隐的退出动画
    startRejectedAnimations(rejectedSnapshots);

    // 开始创建共享元素的快照view
    // 这里会再触发一遍SharedElementCallback.onCreateSnapshotView方法
    ArrayList<View> sharedElementSnapshots = createSnapshots(sharedElementState,
            mSharedElementNames);
    // 显示共享元素
    showViews(mSharedElements, true);
    // 添加OnPreDrawListener，在下一帧触发SharedElementCallback.onSharedElementEnd回调
    scheduleSetSharedElementEnd(sharedElementSnapshots);
    // 设置共享元素设置到动画的开始位置，并返回在ActivityB布局中的原始的状态（即结束位置）
    // 这里会触发SharedElementCallback.onSharedElementStart回调
    ArrayList<SharedElementOriginalState> originalImageViewState =
            setSharedElementState(sharedElementState, sharedElementSnapshots);
    requestLayoutForSharedElements();

    boolean startEnterTransition = allowOverlappingTransitions() && !mIsReturning;
    boolean startSharedElementTransition = true;
    setGhostVisibility(View.INVISIBLE);
    scheduleGhostVisibilityChange(View.INVISIBLE);
    pauseInput();
    // 然后就开始采集开始帧和结束帧，执行过度动画
    Transition transition = beginTransition(decorView, startEnterTransition,
            startSharedElementTransition);
    scheduleGhostVisibilityChange(View.VISIBLE);
    setGhostVisibility(View.VISIBLE);

    if (startEnterTransition) {
    	// 添加监听器，动画结束的时候，将window的背景恢复成不透明等
        startEnterTransition(transition);
    }
    // 将共享元素设置到结束的位置（为了TransitionManager能采集到结束帧的值）
    setOriginalSharedElementState(mSharedElements, originalImageViewState);

    if (mResultReceiver != null) {
        // We can't trust that the view will disappear on the same frame that the shared
        // element appears here. Assure that we get at least 2 frames for double-buffering.
        decorView.postOnAnimation(new Runnable() {
            int mAnimations;

            @Override
            public void run() {
                if (mAnimations++ < MIN_ANIMATION_FRAMES) {
                    View decorView = getDecor();
                    if (decorView != null) {
                        decorView.postOnAnimation(this);
                    }
                } else if (mResultReceiver != null) {
                    mResultReceiver.send(MSG_HIDE_SHARED_ELEMENTS, null);
                    mResultReceiver = null; // all done sending messages.
                }
            }
        });
    }
}
复制代码
```

接下来看一下`beginTransition`方法

```
// android.app.EnterTransitionCoordinator源码
private Transition beginTransition(ViewGroup decorView, boolean startEnterTransition,
        boolean startSharedElementTransition) {
    Transition sharedElementTransition = null;
    if (startSharedElementTransition) {
        if (!mSharedElementNames.isEmpty()) {
            // 获取共享元素的过渡动画类Transition，可以通过window.setSharedElementEnterTransition方法设置
            // 一般不需要设置 有默认值
            sharedElementTransition = configureTransition(getSharedElementTransition(), false);
        }
        ...
    }
    Transition viewsTransition = null;
    if (startEnterTransition) {
        mIsViewsTransitionStarted = true;
        if (mTransitioningViews != null && !mTransitioningViews.isEmpty()) {
            // 获取非共享元素的过渡动画类Transition，可以通过window.setEnterTransition方法设置
            // 一般不需要设置 有默认值
            viewsTransition = configureTransition(getViewsTransition(), true);
        }
        ...
    // 合并成TransitionSet 对象
    Transition transition = mergeTransitions(sharedElementTransition, viewsTransition);
    if (transition != null) {
        transition.addListener(new ContinueTransitionListener());
        if (startEnterTransition) {
            setTransitioningViewsVisiblity(View.INVISIBLE, false);
        }
        // 开始采集开始帧和结束帧，执行过度动画
        TransitionManager.beginDelayedTransition(decorView, transition);
        if (startEnterTransition) {
            setTransitioningViewsVisiblity(View.VISIBLE, false);
        }
        decorView.invalidate();
    } else {
        transitionStarted();
    }
    return transition;
}
复制代码
```

到了这里，就会真正的开始执行 `ActivityB`的共享元素和非共享元素的进场动画

当动画执行结束之后就会触发 `onTransitionsComplete`方法

```
// android.app.EnterTransitionCoordinator源码
@Override
protected void onTransitionsComplete() {
    // 将共享元素和GhostView从decorView的OverlayView中remove掉
    moveSharedElementsFromOverlay();
    final ViewGroup decorView = getDecor();
    if (decorView != null) {
        decorView.sendAccessibilityEvent(AccessibilityEvent.TYPE_WINDOW_CONTENT_CHANGED);

        Window window = getWindow();
        if (window != null && mReplacedBackground == decorView.getBackground()) {
            window.setBackgroundDrawable(null);
        }
    }
    if (mOnTransitionComplete != null) {
        mOnTransitionComplete.run();
        mOnTransitionComplete = null;
    }
}

复制代码
```

用非常简单点的话总结共享元素流程是：

*   1.  ActivityA 先执行退场动画
*   2.  ActivityA 将共享元素的结束位置等属性传递给 ActivityB
*   3.  ActivityB 加载自己的布局，在 onStart 生命周期左右去找到共享元素 先定位到 ActivityA 的结束位置
*   4.  ActivityB 开始执行过度动画，过渡到自己布局中的位置

这就是 从 ActivityA 打开 ActivityB 的共享元素动画过程的核心源码分析过程

### `ActivityB`返回`ActivityA`

既然是返回，首先肯定是要调用`ActivityB`的`finishAfterTransition`方法

```
// android.app.Activity 源码
public void finishAfterTransition() {
    if (!mActivityTransitionState.startExitBackTransition(this)) {
        finish();
    }
}
复制代码
```

这里就会调用`ActivityTransitionState`的`startExitBackTransition`方法

```
// android.app.ActivityTransitionState源码
public boolean startExitBackTransition(final Activity activity) {
	// 获取打开ActivityB时 传过来的共享元素名称
    ArrayList<String> pendingExitNames = getPendingExitNames();
    if (pendingExitNames == null || mCalledExitCoordinator != null) {
        return false;
    } else {
        if (!mHasExited) {
            mHasExited = true;
            Transition enterViewsTransition = null;
            ViewGroup decor = null;
            boolean delayExitBack = false;
           ...
           // 创建ExitTransitionCoordinator对象
            mReturnExitCoordinator = new ExitTransitionCoordinator(activity,
                    activity.getWindow(), activity.mEnterTransitionListener, pendingExitNames,
                    null, null, true);
            if (enterViewsTransition != null && decor != null) {
                enterViewsTransition.resume(decor);
            }
            if (delayExitBack && decor != null) {
                final ViewGroup finalDecor = decor;
                // 在下一帧调用startExit方法
                OneShotPreDrawListener.add(decor, () -> {
                    if (mReturnExitCoordinator != null) {
                        mReturnExitCoordinator.startExit(activity.mResultCode,
                                activity.mResultData);
                    }
                });
            } else {
                mReturnExitCoordinator.startExit(activity.mResultCode, activity.mResultData);
            }
        }
        return true;
    }
}

复制代码
```

这个方法首先会获取到需要执行退场动画的共享元素（由`ActivityA`打开`ActivityB`时传过来的），然后会创建`ExitTransitionCoordinator`对象，最后调用`startExit` 执行`ActivityB`的退场逻辑

我们继续看看`ExitTransitionCoordinator`的构造方法，虽然在上面在分析`ActivityA`打开`ActivityB`时已经看过了这个构造方法，但`ActivityB`返回`ActivityA`时有点不一样，`accepted`和`mapped`参数为`null`，`isReturning`参数为`true`

```
// android.app.ExitTransitionCoordinator源码
public ExitTransitionCoordinator(Activity activity, Window window,
        SharedElementCallback listener, ArrayList<String> names,
        ArrayList<String> accepted, ArrayList<View> mapped, boolean isReturning) {
    super(window, names, listener, isReturning);
    // viewsReady方法跟上面介绍的一样，主要是mapSharedElements不一样了
    viewsReady(mapSharedElements(accepted, mapped));
    // 剔除掉mTransitioningViews中不在屏幕内的非共享元素
    stripOffscreenViews();
    mIsBackgroundReady = !isReturning;
    mActivity = activity;
}

// android.app.ActivityTransitionCoordinator源码
protected ArrayMap<String, View> mapSharedElements(ArrayList<String> accepted,
        ArrayList<View> localViews) {
    ArrayMap<String, View> sharedElements = new ArrayMap<String, View>();
    if (accepted != null) {
        for (int i = 0; i < accepted.size(); i++) {
            sharedElements.put(accepted.get(i), localViews.get(i));
        }
    } else {
        ViewGroup decorView = getDecor();
        if (decorView != null) {
            // 遍历整个ActivityB的所有view，找到所有设置了transitionName属性的view
            decorView.findNamedViews(sharedElements);
        }
    }
    return sharedElements;
}
复制代码
```

这里由于`accepted`和`mapped`参数为`null`，所以会遍历整个`decorView`上的所有`view`，找到所有设置了`transitionName`属性的`view`，保存到`sharedElements`

然后`viewsReady`就会根据自己所支持的共享元素名称，从`sharedElements`中删除所有不支持的共享元素，然后对其排序，并保存到`mSharedElements`(保存的`view`对象) 和`mSharedElementNames`(保存的是共享元素名称) 中; 同时也会准备好非共享元素`view`对象，保存在`mTransitioningViews`中

注意`viewReady`会触发`SharedElementCallback.onMapSharedElements`回调

结下来就会执行`ExitTransitionCoordinator` 的`startExit`方法

```
// android.app.ExitTransitionCoordinator源码
public void startExit(int resultCode, Intent data) {
    if (!mIsExitStarted) {
        ...
        // 这里又将ActivityB的共享元素用GhostView包裹一下，添加的decorView的OverlayView中
        moveSharedElementsToOverlay();
        // 将ActivityB的window背景设置成透明
        if (decorView != null && decorView.getBackground() == null) {
            getWindow().setBackgroundDrawable(new ColorDrawable(Color.TRANSPARENT));
        }
        final boolean targetsM = decorView == null || decorView.getContext()
                .getApplicationInfo().targetSdkVersion >= VERSION_CODES.M;
        ArrayList<String> sharedElementNames = targetsM ? mSharedElementNames :
                mAllSharedElementNames;
        //这里创建options对象，保存ExitTransitionCoordinator、sharedElementNames等对象
        ActivityOptions options = ActivityOptions.makeSceneTransitionAnimation(mActivity, this,
                sharedElementNames, resultCode, data);
        // 这里会将ActivityB改成透明的activity，同时会将options对象传给ActivityA
        mActivity.convertToTranslucent(new Activity.TranslucentConversionListener() {
            @Override
            public void onTranslucentConversionComplete(boolean drawComplete) {
                if (!mIsCanceled) {
                    fadeOutBackground();
                }
            }
        }, options);
        startTransition(new Runnable() {
            @Override
            public void run() {
                startExitTransition();
            }
        });
    }
}
复制代码
```

这个方法的主要作用是

*   使用`GhostView` 将共享元素`view`添加到最顶层`decorView`的`OverlayView`中
*   然后创建一个`ActivityOptions` 对象，把`ActivityB`的`ExitTransitionCoordinator`对象和支持的共享元素集合对象传递给`ActivityA`
*   将 ActivityB 改成透明背景

然后就会执行`startExitTransition`方法

```
// android.app.ExitTransitionCoordinator源码
private void startExitTransition() {
    // 获取ActivityB的非共享元素退场的过渡动画Transition对象
    // 最终会调用getReturnTransition方法获取Transition对象
    Transition transition = getExitTransition();
    ViewGroup decorView = getDecor();
    if (transition != null && decorView != null && mTransitioningViews != null) {
        setTransitioningViewsVisiblity(View.VISIBLE, false);
        // 开始执行非共享元素的退场动画
        TransitionManager.beginDelayedTransition(decorView, transition);
        setTransitioningViewsVisiblity(View.INVISIBLE, false);
        decorView.invalidate();
    } else {
        transitionStarted();
    }
}
复制代码
```

看到这里我们就知道了，这里会单独先执行非共享元素的退场动画

`ActivityB`的退场流程暂时就走到这里了，结下来就需要`ActivityA`的配和，所以接下来我们来看看`ActivityA`的进场逻辑

`ActivityA`进场时，会调用`performStart`方法

```
// android.app.Activity 源码
final void performStart(String reason) {
    dispatchActivityPreStarted();
    // 这里的getActivityOptions()获取到的就是上面说的`ActivityB`传过来的对象
    mActivityTransitionState.setEnterActivityOptions(this, getActivityOptions());
    mFragments.noteStateNotSaved();
    mCalled = false;
    mFragments.execPendingActions();
    mInstrumentation.callActivityOnStart(this);
    EventLogTags.writeWmOnStartCalled(mIdent, getComponentName().getClassName(), reason);
    ...
    // ActivityA准备执行进场逻辑
    mActivityTransitionState.enterReady(this);
    dispatchActivityPostStarted();
}

// android.app.ActivityTransitionState 源码
public void enterReady(Activity activity) {
    // mEnterActivityOptions对象就是`ActivityB`传过来的对象
    if (mEnterActivityOptions == null || mIsEnterTriggered) {
        return;
    }
    mIsEnterTriggered = true;
    mHasExited = false;
    // 共享元素名称
    ArrayList<String> sharedElementNames = mEnterActivityOptions.getSharedElementNames();
    // ActivityB的ExitTransitionCoordinator对象
    ResultReceiver resultReceiver = mEnterActivityOptions.getResultReceiver();
    // 返回标志 true
    final boolean isReturning = mEnterActivityOptions.isReturning();
    if (isReturning) {
        restoreExitedViews();
        activity.getWindow().getDecorView().setVisibility(View.VISIBLE);
    }
    // 创建ActivityA的EnterTransitionCoordinator对象
    mEnterTransitionCoordinator = new EnterTransitionCoordinator(activity,
            resultReceiver, sharedElementNames, mEnterActivityOptions.isReturning(),
            mEnterActivityOptions.isCrossTask());
    if (mEnterActivityOptions.isCrossTask()) {
        mExitingFrom = new ArrayList<>(mEnterActivityOptions.getSharedElementNames());
        mExitingTo = new ArrayList<>(mEnterActivityOptions.getSharedElementNames());
    }

    if (!mIsEnterPostponed) {
        startEnter();
    }
}
复制代码
```

`ActivityA`进场时，会在`performStart`里获取并保存`ActivityB`传过来的对象，然后创建`EnterTransitionCoordinator`进场动画实现的核心类，然后调用 startEnter 方法

```
// android.app.ActivityTransitionState 源码
private void startEnter() {
    if (mEnterTransitionCoordinator.isReturning()) {
    	// 这里的mExitingFrom、mExitingTo、mExitingToView是ActivityA打开ActivityB的时候保存下载的对象
    	// 所以一般情况下都不为null
        if (mExitingToView != null) {
            mEnterTransitionCoordinator.viewInstancesReady(mExitingFrom, mExitingTo,
                    mExitingToView);
        } else {
            mEnterTransitionCoordinator.namedViewsReady(mExitingFrom, mExitingTo);
        }
    } else {
        mEnterTransitionCoordinator.namedViewsReady(null, null);
        mPendingExitNames = null;
    }

    mExitingFrom = null;
    mExitingTo = null;
    mExitingToView = null;
    mEnterActivityOptions = null;
}
复制代码
```

接下来就会执行`EnterTransitionCoordinator`的`viewInstancesReady`方法

```
// android.app.EnterTransitionCoordinator 源码
public void viewInstancesReady(ArrayList<String> accepted, ArrayList<String> localNames,
        ArrayList<View> localViews) {
    boolean remap = false;
    for (int i = 0; i < localViews.size(); i++) {
        View view = localViews.get(i);
        // view的TransitionName属性有没有发生变化，或者view对象没有AttachedToWindow
        if (!TextUtils.equals(view.getTransitionName(), localNames.get(i))
                || !view.isAttachedToWindow()) {
            remap = true;
            break;
        }
    }
    if (remap) {// 如果发生了变化，则会调用mapNamedElements遍历decorView找到所有设置了TransitionName的view
        triggerViewsReady(mapNamedElements(accepted, localNames));
    } else { // 一般会执行这个else
        triggerViewsReady(mapSharedElements(accepted, localViews));
    }
}
复制代码
```

接下来就会执行 `triggerViewsReady`，里面就会调用`viewsReady`方法，`viewsReady`在上面介绍过，唯一有点不一样的是 这里的`mIsReturning`是`true`, 所以会执行`sendSharedElementDestination`方法

```
// android.app.EnterTransitionCoordinator源码
@Override
protected void viewsReady(ArrayMap<String, View> sharedElements) {
    // 准备好共享元素和非共享元素
    super.viewsReady(sharedElements);
    mIsReadyForTransition = true;
    // 隐藏共享元素
    hideViews(mSharedElements);
    Transition viewsTransition = getViewsTransition();
    if (viewsTransition != null && mTransitioningViews != null) {
    	// 将mTransitioningViews当作target设置到viewsTransition中
        removeExcludedViews(viewsTransition, mTransitioningViews);
        // 剔除掉mTransitioningViews中不在屏幕中的view
        stripOffscreenViews();
        // 隐藏非共享元素
        hideViews(mTransitioningViews);
    }
    if (mIsReturning) {
        sendSharedElementDestination();
    } else {
        moveSharedElementsToOverlay();
    }
    if (mSharedElementsBundle != null) {
        onTakeSharedElements();
    }
}
复制代码
```

```
// android.app.EnterTransitionCoordinator源码
private void sendSharedElementDestination() {
    boolean allReady;
    final View decorView = getDecor();
    if (allowOverlappingTransitions() && getEnterViewsTransition() != null) {
        allReady = false;
    } else if (decorView == null) {
        allReady = true;
    } else {
        allReady = !decorView.isLayoutRequested();
        if (allReady) {
            for (int i = 0; i < mSharedElements.size(); i++) {
                if (mSharedElements.get(i).isLayoutRequested()) {
                    allReady = false;
                    break;
                }
            }
        }
    }
    if (allReady) {
    	// 捕获共享元素当前的状态, 会触发SharedElementCallback.onCaptureSharedElementSnapshot方法
        Bundle state = captureSharedElementState();
        // 将共享元素view 添加的最顶层（decorView的OverlayView中）
        moveSharedElementsToOverlay();
        // 给ActivityB发送MSG_SHARED_ELEMENT_DESTINATION，将共享元素的状态传给ActivityB
        mResultReceiver.send(MSG_SHARED_ELEMENT_DESTINATION, state);
    } else if (decorView != null) {
        OneShotPreDrawListener.add(decorView, () -> {
            if (mResultReceiver != null) {
                Bundle state = captureSharedElementState();
                moveSharedElementsToOverlay();
                mResultReceiver.send(MSG_SHARED_ELEMENT_DESTINATION, state);
            }
        });
    }
    if (allowOverlappingTransitions()) {
    	// 执行非共享元素的进场动画
        startEnterTransitionOnly();
    }
}
复制代码
```

`sendSharedElementDestination`方法主要有以下三个作用

*   获取`ActivityA`当前共享元素的状态 传给`ActivityB`，当作过度动画结束位置的状态（即结束帧）
*   将共享元素添加到最顶层（decorView 的 OverlayView 中）
*   给`ActivityB`发送`MSG_SHARED_ELEMENT_DESTINATION`消息传递`state`
*   优先开始执行`ActivityA`的非共享元素的进场动画

到这里`ActivityA`的逻辑暂时告一段落，接下来我们来看看`ActivityB`接收到`MSG_SHARED_ELEMENT_DESTINATION`时干了些什么

```
// android.app.ExitTransitionCoordinator
@Override
protected void onReceiveResult(int resultCode, Bundle resultData) {
    switch (resultCode) {
        ...
        case MSG_SHARED_ELEMENT_DESTINATION:
            // 保存ActivityA传过来的共享元素状态
            mExitSharedElementBundle = resultData;
            // 准备执行共享元素退出动画
            sharedElementExitBack();
            break;
        ...
    }
}
复制代码
```

接下来我们来看看`sharedElementExitBack`方法

```
// android.app.ExitTransitionCoordinator
private void sharedElementExitBack() {
    final ViewGroup decorView = getDecor();
    if (decorView != null) {
        decorView.suppressLayout(true);
    }
    if (decorView != null && mExitSharedElementBundle != null &&
            !mExitSharedElementBundle.isEmpty() &&
            !mSharedElements.isEmpty() && getSharedElementTransition() != null) {
        startTransition(new Runnable() {
            public void run() {
            	// 会执行这个方法
                startSharedElementExit(decorView);
            }
        });
    } else {
        sharedElementTransitionComplete();
    }
}
复制代码
```

接下来就会执行`startSharedElementExit`方法

```
// android.app.ExitTransitionCoordinator
private void startSharedElementExit(final ViewGroup decorView) {
    // 获取共享元素的过度动画的Transition对象,里面最终会调用`getSharedElementReturnTransition`方法获取该对象
    Transition transition = getSharedElementExitTransition();
    transition.addListener(new TransitionListenerAdapter() {
        @Override
        public void onTransitionEnd(Transition transition) {
            transition.removeListener(this);
            if (isViewsTransitionComplete()) {
                delayCancel();
            }
        }
    });
    // 根据ActivityA传过来的状态，创建快照view对象
    // 这里会触发SharedElementCallback.onCreateSnapshotView方法
    final ArrayList<View> sharedElementSnapshots = createSnapshots(mExitSharedElementBundle,
            mSharedElementNames);
    OneShotPreDrawListener.add(decorView, () -> {
    	// 在下一帧触发，将共享元素的属性设置到开始状态(ActivityA中共享元素的状态)
    	// 这里会触发SharedElementCallback.onSharedElementStart回调
        setSharedElementState(mExitSharedElementBundle, sharedElementSnapshots);
    });
    setGhostVisibility(View.INVISIBLE);
    scheduleGhostVisibilityChange(View.INVISIBLE);
    if (mListener != null) {
    	// 先触发SharedElementCallback.onSharedElementEnd回调
        mListener.onSharedElementEnd(mSharedElementNames, mSharedElements,
                sharedElementSnapshots);
    }
    // 采集开始帧和结束帧，并执行动画
    TransitionManager.beginDelayedTransition(decorView, transition);
    scheduleGhostVisibilityChange(View.VISIBLE);
    setGhostVisibility(View.VISIBLE);
    decorView.invalidate();
}
复制代码
```

看到上面的方法你可能会发现，先触发了`onSharedElementEnd`方法，然后再触发`onSharedElementStart`，这可能是因为`ActivityB`返回到`ActivityA`时, `google`工程师定义为是从结束状态返回到开始状态吧，即`ActivityB`的状态为结束状态，`ActivityA`的状态为开始状态

至于`setGhostVisibility`和`scheduleGhostVisibilityChange`主要的作用是为`TransitionManager`采集开始帧和结束帧执行动画用的

到这里`ActivityB`就开始执行共享元素的退出动画了

当`ActivityB`共享元素动画执行结束之后，就会调用`sharedElementTransitionComplete`方法，然后就会调用`notifyComplete`方法

```
@Override
protected void sharedElementTransitionComplete() {
    // 这里又会获取ActivityB共享元素的状态(之后会传给ActivityA)
    // 可能会触发ActivityB的SharedElementCallback.onCaptureSharedElementSnapshot回调
    mSharedElementBundle = mExitSharedElementBundle == null
            ? captureSharedElementState() : captureExitSharedElementsState();
    super.sharedElementTransitionComplete();
}
复制代码
```

这里为什么要再一次获取`ActivityB`的共享元素的状态，因为需要传给`ActivityA`, 然后`ActivityA`再根据条件判断 共享元素的状态是否又发生了变化，然后交给`ActivityA`自己去执行共享元素动画

至于最后会执行`notifyComplete`，源码就没什么好看的了，上面也都介绍过，这里面主要是给`ActivityA`发送了`MSG_TAKE_SHARED_ELEMENTS`消息，将`ActivityB`的共享元素的状态对象 (`mSharedElementBundle`) 传递给`ActivityA`

到这里`ActivityB`退场动画基本上就结束了，至于最后的状态清空等处理 我们就不看了

接下来我们继续看`ActivityA`接收到`MSG_TAKE_SHARED_ELEMENTS`消息后做了什么

```
@Override
protected void onReceiveResult(int resultCode, Bundle resultData) {
    switch (resultCode) {
        case MSG_TAKE_SHARED_ELEMENTS:
            if (!mIsCanceled) {
            	//  保存共享元素状态对象
                mSharedElementsBundle = resultData;
                // 准备执行共享元素动画
                onTakeSharedElements();
            }
            break;
    	...
    }
}
复制代码
```

结下来就会执行`onTakeSharedElements`方法，这些方法的源码上面都介绍过，我就不在贴出来了，这里面会触发`SharedElementCallback.onSharedElementsArrived`回调，然后执行`startSharedElementTransition`

```
//  android.app.EnterTransitionCoordinator源码
private void startSharedElementTransition(Bundle sharedElementState) {
    ViewGroup decorView = getDecor();
    if (decorView == null) {
        return;
    }
    // Remove rejected shared elements
    ArrayList<String> rejectedNames = new ArrayList<String>(mAllSharedElementNames);
    // 过滤出ActivityB存在，ActivityA不存在的共享元素
    rejectedNames.removeAll(mSharedElementNames);
    // 根据ActivityB传过来的共享元素sharedElementState信息，创建快照view对象
    // 这里会触发SharedElementCallback.onCreateSnapshotView方法
    ArrayList<View> rejectedSnapshots = createSnapshots(sharedElementState, rejectedNames);
    if (mListener != null) {
    	// 触发SharedElementCallback.onRejectSharedElements方法
        mListener.onRejectSharedElements(rejectedSnapshots);
    }
    removeNullViews(rejectedSnapshots);
    // 执行渐隐的退出动画
    startRejectedAnimations(rejectedSnapshots);

    // 开始创建共享元素的快照view
    // 这里会再触发一遍SharedElementCallback.onCreateSnapshotView方法
    ArrayList<View> sharedElementSnapshots = createSnapshots(sharedElementState,
            mSharedElementNames);
    // 显示共享元素
    showViews(mSharedElements, true);
    // 添加OnPreDrawListener，在下一帧触发SharedElementCallback.onSharedElementEnd回调
    scheduleSetSharedElementEnd(sharedElementSnapshots);
    // 设置共享元素设置到动画的开始位置，并返回在ActivityA布局中的原始的状态（即结束位置）
    // SharedElementCallback.onSharedElementStart回调
    ArrayList<SharedElementOriginalState> originalImageViewState =
            setSharedElementState(sharedElementState, sharedElementSnapshots);
    requestLayoutForSharedElements();

    boolean startEnterTransition = allowOverlappingTransitions() && !mIsReturning;
    boolean startSharedElementTransition = true;
    setGhostVisibility(View.INVISIBLE);
    scheduleGhostVisibilityChange(View.INVISIBLE);
    pauseInput();
    // 然后就开始采集开始帧和结束帧，执行过度动画
    Transition transition = beginTransition(decorView, startEnterTransition,
            startSharedElementTransition);
    scheduleGhostVisibilityChange(View.VISIBLE);
    setGhostVisibility(View.VISIBLE);

    if (startEnterTransition) {// 这里为false，不会执行, 因为非共享元素动画执行单独执行了
        startEnterTransition(transition);
    }
    // 将共享元素设置到结束的位置（为了TransitionManager能采集到结束帧的值）
    setOriginalSharedElementState(mSharedElements, originalImageViewState);

    if (mResultReceiver != null) {
        // We can't trust that the view will disappear on the same frame that the shared
        // element appears here. Assure that we get at least 2 frames for double-buffering.
        decorView.postOnAnimation(new Runnable() {
            int mAnimations;

            @Override
            public void run() {
                if (mAnimations++ < MIN_ANIMATION_FRAMES) {
                    View decorView = getDecor();
                    if (decorView != null) {
                        decorView.postOnAnimation(this);
                    }
                } else if (mResultReceiver != null) {
                    mResultReceiver.send(MSG_HIDE_SHARED_ELEMENTS, null);
                    mResultReceiver = null; // all done sending messages.
                }
            }
        });
    }
}
复制代码
```

这里要特别说明的是

*   这里没有执行`ActivityA`的非共享元素的进场动画，因为在之前已经优先调用了非共享元素的进场动画
*   虽然这里调用了`ActivityA`的共享元素动画，但是基本上并不会创建动画对象去执行，因为`ActivityB`传过来的状态 跟 `ActivityA`当前的状态是一模一样的，除非你在某种情况下并在执行动画之前 强制改变 `ActivityA`的当前状态；所以你所看到的共享元素的退场动画其实是`ActivityB`的共享元素退场动画，而不是`ActivityA`的

最后`ActivityA`的共享元素动画结束之后 会就调用`onTransitionsComplete`（不需要执行动画，就会立马触发），将`ActivityA`的共享元素 view 从从 decorView 的 ViewGroupOverlay 中 remove 掉

到这里由`ActivityB`返回`ActivityA`的退场动画到这里基本上就结束了，至于最后的`cancel`等状态清理就不介绍了

到这里我也用非常简单点的大白话总结一下`ActivityB`返回`ActivityA`的退场动画：

*   1.  将`ActivityB`的 window 背景设置成透明, 并执行非共享元素的退场动画
*   2.  返回到 ActivityA 时，将会执行到 performStart 方法，并执行非共享元素的进场动画
*   3.  `ActivityB`接收到`ActivityA`传过来的共享元素状态，开始执行共享元素的退场动画
*   4.  `ActivityA`接收到`ActivityB`的共享元素状态，继续执行共享元素动画 (但由于两个状态没有变化，所以并不会执行动画，会立马直接动画结束的回调)

### SharedElementCallback 回调总结

最后我们在总结以下`SharedElementCallback`回调的顺序，因为你有可能会自定义这个类 做一些特定的逻辑处理

当是 ActivityA 打开 ActivityB 时

```
ActivityA: ==Exit, onMapSharedElements
ActivityA: ==Exit, onCaptureSharedElementSnapshot
ActivityA: ==Exit, onCaptureSharedElementSnapshot
ActivityB: ==Enter, onMapSharedElements
ActivityA: ==Exit, onSharedElementsArrived
ActivityB: ==Enter, onSharedElementsArrived
ActivityB: ==Enter, onCreateSnapshotView
ActivityB: ==Enter, onRejectSharedElements
ActivityB: ==Enter, onCreateSnapshotView
ActivityB: ==Enter, onSharedElementStart
ActivityB: ==Enter, onSharedElementEnd
复制代码
```

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f882fba6bee14140ba1ddde1e86e95d8~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

当是 ActivityB 返回到 ActivityA 时

```
ActivityB: ==Enter, onMapSharedElements
ActivityA: ==Exit, onMapSharedElements
ActivityA: ==Exit, onCaptureSharedElementSnapshot
ActivityB: ==Enter, onCreateSnapshotView
ActivityB: ==Enter, onSharedElementEnd
ActivityB: ==Enter, onSharedElementStart
ActivityB: ==Enter, onSharedElementsArrived
ActivityA: ==Exit, onSharedElementsArrived
ActivityA: ==Exit, onRejectSharedElements
ActivityA: ==Exit, onCreateSnapshotView
ActivityA: ==Exit, onSharedElementStart
ActivityA: ==Exit, onSharedElementEnd
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e9c16b4e43a0480dab0976a232e95979~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

好了，到这里 我所要介绍的内容已经结束了，上面的源码是针对 Android30 和 Android31 分析的 (我在不同的时间段用不同的笔记本写的，所以上面的源码有的是 Android30 的源码，有的是 Android31 的源码)

最后再附上一张 Activity 共享元素动画的全程时序图

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/05998d87f7e64f5fa45ae42d16be3773~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)