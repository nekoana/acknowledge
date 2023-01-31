> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7144527505394368525)

我报名参加金石计划 1 期挑战——瓜分 10 万奖池，这是我的第 2 篇文章，[点击查看活动详情](https://s.juejin.cn/ds/jooSN7t "https://s.juejin.cn/ds/jooSN7t")

上一篇文章 [Android Span 原理解析](https://juejin.cn/post/7141730415399665671 "https://juejin.cn/post/7141730415399665671") 介绍了 Span 的原理。这一篇文章将介绍 Span 的应用，使用 Span 来给 App 添加自定义表情。

原理
--

添加自定义表情的原理其实很简单，就是使用 ImageSpan 对文字进行替换。代码如下：

```
ImageSpan imageSpan = new ImageSpan(this, R.drawable.emoji_kelian);
SpannableStringBuilder spannableStringBuilder = new SpannableStringBuilder("哈哈哈哈[可怜]");
spannableStringBuilder.setSpan(imageSpan, 4, spannableStringBuilder.length(), Spannable.SPAN_EXCLUSIVE_EXCLUSIVE);
textView.setText(spannableStringBuilder);
复制代码
```

上面的代码把 `[可怜]` 文字替换成了对应的表情图片。效果如下图，可以看到图片的大小不符合预期，这是因为 ImageSpan 会显示成图片原来的大小。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b810147cf4e94110935bf868765bf339~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

ImageSpan 的继承关系图如下，出现了 `ReplacementSpan` 和 `DynamicDrawableSpan` 两个新的类，先来看一下它们。`MetricAffectingSpan` 和 `CharacterStyle` 接口在 [Android Span 原理解析](https://juejin.cn/post/7141730415399665671 "https://juejin.cn/post/7141730415399665671") 介绍了，这里就不赘述了。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3aed360091fd444a8feafc976fc7c02a~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### ReplacementSpan 接口

`ReplacementSpan` 是一个接口，看名字是用来替换文字的。它里面定义了两个方法，如下所示。

```
public abstract int getSize(@NonNull Paint paint,
                                    CharSequence text,
                                    @IntRange(from = 0) int start,
                                    @IntRange(from = 0) int end,
                                    @Nullable Paint.FontMetricsInt fm);
复制代码
```

返回替换后 Span 的宽，上面的例子中就是返回图片的宽度，参数作用如下：

*   paint: Paint 的实例
*   text: 当前文本，上面的例子中它的值是是 **哈哈哈哈 [可怜]**
*   start: Span 的开始位置，这里是 4
*   end: Span 的结束位置，这里是 8
*   fm: FontMetricsInt 的实例

`FontMetricsInt` 是描述给定文本大小的字体的各种度量的类。内部属性代表的含义如下图：

*   Top：图中紫线的位置
*   Ascent： 图中绿线的位置
*   Descent： 图中蓝线的位置
*   Bottom： 图中黄线的位置
*   Leading： 未在图中标出，是指上一行的 Bottom 与下一行的 Top 之间的距离。

图片来源 [Meaning of top, ascent, baseline, descent, bottom, and leading in Android's FontMetrics](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F27631736%2Fmeaning-of-top-ascent-baseline-descent-bottom-and-leading-in-androids-font "https://stackoverflow.com/questions/27631736/meaning-of-top-ascent-baseline-descent-bottom-and-leading-in-androids-font")

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/91210f489218426e809df79d2ff93fbf~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

Baseline 是文字绘制的基准线。它不定义在 `FontMetricsInt` 中，但可以通过 `FontMetricsInt` 的属性获取。

上面讲到 `getSize` 方法只返回宽度，那高度是怎么确定的呢？其实它是通过 `FontMetricsInt` 来控制，**不过这里有个坑**，后面会说到。

```
public abstract void draw(@NonNull Canvas canvas,
                                    CharSequence text,
                                    @IntRange(from = 0) int start,
                                    @IntRange(from = 0) int end,
                                    float x,
                                    int top,
                                    int y,
                                    int bottom,
                                    @NonNull Paint paint);
复制代码
```

在 Canvas 中绘制 Span。参数如下：

*   canvas：Canvas 实例
*   text：当前文本
*   start：Span 的开始位置
*   end：Span 的结束位置
*   x：**[可怜]** 的 x 坐标位置
*   top：当前行的 “Top“ 属性值
*   y：当前行的 Baseline
*   bottom: 当前行的 ”Bottom“ 属性值
*   paint：Paint 实例，可能为 null

这里需要特殊注意 **Top** 和 **Bottom**，跟上面说的有点不同这里先记住，后面会一起介绍。

### DynamicDrawableSpan

`DynamicDrawableSpan` 实现了 `ReplacementSpan` 接口的方法。同时它是一个抽象类，定义了 `getDrawable` 抽象方法，由 `ImageSpan` 实现来获取 Drawable 实例。源码如下：

```
@Override

public int getSize(@NonNull Paint paint, CharSequence text,
    @IntRange(from = 0) int start, @IntRange(from = 0) int end,
    @Nullable Paint.FontMetricsInt fm) {
    
    Drawable d = getCachedDrawable();
    Rect rect = d.getBounds();

    //设置图片的高
    if (fm != null) {
    fm.ascent = -rect.bottom;
    fm.descent = 0;
    fm.top = fm.ascent;
    fm.bottom = 0;
    }
    return rect.right;
}

@Override

public void draw(@NonNull Canvas canvas, CharSequence text,
    @IntRange(from = 0) int start, @IntRange(from = 0) int end, float x,
    int top, int y, int bottom, @NonNull Paint paint) {

    Drawable b = getCachedDrawable();
    canvas.save();

    int transY = bottom - b.getBounds().bottom;
    //设置对齐方式，有三种分别是
    //ALIGN_BOTTOM 底部对齐，默认
    //ALIGN_BASELINE 基线对齐
    //ALIGN_CENTER 居中对齐
    if (mVerticalAlignment == ALIGN_BASELINE) {
        transY -= paint.getFontMetricsInt().descent;
    } else if (mVerticalAlignment == ALIGN_CENTER) {
        transY = top + (bottom - top) / 2 - b.getBounds().height() / 2;
    }

    canvas.translate(x, transY);
    b.draw(canvas);
    canvas.restore();
}

public abstract Drawable getDrawable();
复制代码
```

`DynamicDrawableSpan` 有两个坑需要特别注意。

第一个坑就是在 `getSize` 中的 `Paint.FontMetricsInt` 对象和 `draw` 方法中通过 `paint.getFontMetricsInt()` 获取的**不是一个对象**。也就是说，无论我们在 `getSize` 的 `Paint.FontMetricsInt` 中设置什么值，都不会影响到 `paint.getFontMetricsInt()` 获取对象中的值。它影响的是 **top** 和 **bottom** 的值，这也是刚才介绍参数时给 Top 和 Bottom 打引号的原因。

第二个坑是 `ALIGN_CENTER` 在**图片大小超过文字大小**时 “不起作用”。如下图所示，为了方便显示我加了辅助线，白线是代表参数 top，bottom，但是 bottom 被其它颜色覆盖了。可以看到，图片是居中的，是文字没有居中让我们看上去 `ALIGN_CENTER` 没有效果一样。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/945f596f17c041f59ec4f2535ce15124~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

去掉辅助线后，看上去更明显一些。

![](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ed5d14ec13d45e5a45ab8c6bdbfb7fc~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

### ImageSpan

`ImageSpan` 就简单多了，它只实现了 `getDrawable()` 方法来获取 Drawable 实例，代码如下：

```
@Override
public Drawable getDrawable() {

    Drawable drawable = null;
    if (mDrawable != null) {

        drawable = mDrawable;

    } else if (mContentUri != null) {

        Bitmap bitmap = null;
        try {
            InputStream is = mContext.getContentResolver().openInputStream(
            mContentUri);
            bitmap = BitmapFactory.decodeStream(is);
            drawable = new BitmapDrawable(mContext.getResources(), bitmap);
            drawable.setBounds(0, 0, drawable.getIntrinsicWidth(),
            drawable.getIntrinsicHeight());
            is.close();
        } catch (Exception e) {
            Log.e("ImageSpan", "Failed to loaded content " + mContentUri, e);
        }

    } else {
        try {
            drawable = mContext.getDrawable(mResourceId);
            drawable.setBounds(0, 0, drawable.getIntrinsicWidth(),
            drawable.getIntrinsicHeight());
        } catch (Exception e) {
            Log.e("ImageSpan", "Unable to find resource: " + mResourceId);
        }
    }
    return drawable;
}

复制代码
```

这里代码很简单，我们唯一需要关注的就是获取 Drawable 时，需要设置它的宽高，让它别超过文字的大小。

实现
--

说完前面的原理后，实现起来就非常简单了。我们只需要继承 `DynamicDrawableSpan`，实现 `getDrawable()` 方法，让图片的宽高别超过文字的大小就行了。效果如下图所示：

```
public class EmojiSpan extends DynamicDrawableSpan {

    @DrawableRes
    private int mResourceId;

    private Context mContext;

    private Drawable mDrawable;

    public EmojiSpan(@NonNull Context context, int resourceId) {
        this.mResourceId = resourceId;
        this.mContext = context;
    }

    @Override

    public Drawable getDrawable() {

        Drawable drawable = null;

        if (mDrawable != null) {

            drawable = mDrawable;

        } else {
            try {
                drawable = mContext.getDrawable(mResourceId);
                drawable.setBounds(0, 0, 48, 48);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
        return drawable;
    }
}
复制代码
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/6ffbe2ffd6814102adbdb2f1d06b8235~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

上面看上去很完美，但是事情没有那么简单。因为我们只是写死了图片的大小，并没有改变图片位置绘制的算法。如果其他地方使用了 `EmojiSpan` , 但是文字的大小小于图片大小时还是会出问题。如下图，当文字的 textsize 为 10sp 时的情况。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e60c24bfbf694e55b7ea32f3a61802d4~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

实际上，文字大于图片大小时也有问题。如下图所示，多行的情况下，只有表情的行间距明显小于其他行的间距。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/3db8ada85fc146c5a5d8368a1d2e2fd3~tplv-k3u1fbpfcp-zoom-in-crop-mark:3024:0:0:0.awebp?)

如果大家对这个的解决办法感兴趣的话，点赞 + 收藏数 >= 40，我就复刻一下 B 站的自定义表情，加上会动的自定义表情（实际上是 Gif 图）。

参考
--

*   [Meaning of top, ascent, baseline, descent, bottom, and leading in Android's FontMetrics](https://link.juejin.cn?target=https%3A%2F%2Fstackoverflow.com%2Fquestions%2F27631736%2Fmeaning-of-top-ascent-baseline-descent-bottom-and-leading-in-androids-font "https://stackoverflow.com/questions/27631736/meaning-of-top-ascent-baseline-descent-bottom-and-leading-in-androids-font")