> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7188057359754166331)

前言
--

**HashMap** 是 `Java` 中一个很常用的容器，不过也是面试的重灾区，问题的方式多种多样。

本文着重讲述 **HashMap** 在`JDK 1.7` 和 `Jdk 1.8` 下的原理以及一些面试可能会被问到的问题。

现在先来看看自己能否回答得上这些问题，答案能得满分吗？

`1.为什么要用 HashMap，你知道其底层原理吗？`

`2.HashMap 中的 hash 值有什么用？`

`3.为什么 HashMap 中的数组长度必须是 2 的幂次方？`

`4.为什么建议用 String Integer 这样的包装类作为 HashMap 的键？`

`5.JDK 1.7中为什么会形成死循环？`

`6.红黑树的概念，它的原理是什么，为什么要用它？`

`...`

如果对于上面这些问题还不清楚，别急，和我一起往下看。

✔️ 本文阅读时长约为: `20min`

为什么要用到 HashMap
--------------

在 `Java` 中我们经常用 `ArrayList` 作为容器来储存一些数据，它的底层是由顺序表实现的，**自然查询快并且是随机访问，但是在其中间增删效率就很慢了**。那它的兄弟 `LinkedList` 怎么样呢？`LinkedList` 底层是由链表实现的，链表我们知道 **增删效率高，但是查找效率慢，每次查询都要从头节点开始向后查找** 。难道没有一种容器能综合它们的优点吗？诶有，这时候该我们的 `HashMap` 登场了。

⭐⭐ `HashMap` 通过键值对进行存储，通过键进行查询，在 `JDK 1.7及以前` 底层实现是一个 **数组 + 链表** 的形式，而在 `JDK 1.7以后` 底层是 **数组 + 链表 + 红黑树** 。

JDK 1.7 及以前的 HashMap
--------------------

```
// 数组结构
transient Entry<K,V>[] table;


// 这是链表结构中的节点，用于解决 hash碰撞，加入一个 next 记录下一个节点
Entry(int h, K k, V v, Entry<K,V> n) {
    value = v;
    next = n;
    key = k ;
    hash = h;
}
复制代码
```

`JDK 1.7及以前` -> **数组 + 链表**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c9820919c5a34b889aac4e00277615c9~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

**HashaMap HashMap 除了 map 总和 hash 有关** ，那么这个 `hash` 到底是什么？我们知道 `Object` 类有一个 `native` 方法叫做 `hashCode`，返回的是一个 `int` 类型的值，那它又有什么用呢？没错，它在我们的 `HashMap` 中体现出来了。我们来看看 **HashMap** 中的 `hash` 方法。

```
final int hash(Object k) {
    // k 是作为当前键的对象
    int h = 0;
    if (useAltHashing) {
        if (k instanceof String) {
            return sun.misc.Hashing.stringHash32 ((string) k);
        }
    h = hashSeed;
    }
    // 每个对象都有一个 hashCode 值
    h ^= k.hashCode() ;
    h ^= (h >>> 20) ~ (h >>> 12);
    // 通过这些位运算来得到 HashMap 中的 hash
    return h ^ (h >>> 7) ^ (h >>> 4);
}
复制代码
```

在 `HashMap` 中通过 `hash()` 可以得到一个值，而这个值与后续存放数据的下标有关。

```
// 确定当前键值对存放在数组中的下标
static int indexFor(int h, int length) {
    // 用计算出的 hash 按位与 上(length - 1)，算出对应数组的下标
    return h & (length - 1);
}
复制代码
```

这里的 `length` 是有一个隐藏的机制，它最开始默认是 `16`，后面遇到扩容都会 `<< 1`，也就是说 `length` 永远是 **2 的幂次方** 。那这 ... 是为什么呢？

⛅ 我们再来看看这个方法返回的是什么。`h & (length - 1)` ，这句代码目的是求出数组中的下标，用来存放当前这对键值对，那求出的值必须在 `0 - (lenght - 1)` 之内，因此我们可以大胆猜测，这就是一个求模运算！可是 **& 和 % 肯定是不同的操作，按位与怎么就相当于求模运算了呢？** 这正是因与 `length` 特殊的限定有关。因为 `length` 是一个 **2 的幂次方的数** ，因此减去 1 后，低位每一位都是 1。比如 `length = 16`，减去以 1 后就是 `00001111`，这时候再与 `h` 按位与就相当于 `length` 和 `h` 求模运算了。因此 **length 是 2 的幂次方** ，此外转化成二进制后，`1` 分布得很均匀，与 `h` 按位与时得出的结果在 `length` 上也是均匀的，从而在数组中也是均匀分布，所以有效减少了 **hash 碰撞** ，而且位运算的效率是高于求模运算的。

```
public v put(K key, V value) {
    if (key == null)
        return putForNullKey(value);
    // 通过 key 计算hash值
    int hash = hash(key);
    // 通过 key 找到数组中待存放的下标
    int i = indexFor(hash, table.length);
    // 链表
    for (Entry<K,V> e = table[i]; e != null; e = e.next) {
        object k;
        // 如果这个数组下标中有元素，开始遍历链表
        if (e.hash == hash && ((k = e.key) == key || key .equals(k))) {
            // 如果两个 hash 值相同，或者键相同，则修改 value
            V oldValue = e.value;
            e.value = Value;
            e.recordAccess(this);
            return oldValue;
        }
    }
    modCount++;
    // 可能两个 hash 值不同，但计算出的 index 一样，这就发生了 hash碰撞
    // 使用头插法，将其放在链表头部
    addEntry (hash, key, value, i)
    return null;
}
复制代码
```

`addEntry()` 内部调用 `createEntry()`。

```
void createEntry (int hash, K key, v value, int bucketIndex) {
    Entry<K,V> e = table[bueketIndex];
    // 注意这里将 e 传进去
    // 也就是说这里使用的是 头插法
    table[bucketIndex] = new Entry<>(hash, key, value, e);
    size++;
}
复制代码
```

⚡ `JDK 1.7` 用的是 **头插法** ，这是造成 **HashMap** 形成死循环的原因之一。⚡

我们刚刚看了 `put()` 的源码，做个总结，到这里我们知道 `HashMap` 底层是一个 **数组 + 链表** 的一个结构。它会通过计算 `key` 的 **hash 值** ，进而与 `length - 1` 进行按位与算出数组所在的下标。如果算出的 `index` 和其它元素重复，我们称之为 **哈希碰撞** ，这时会将新来的 `键值对` 放在其对应数组下标的链表的第一位 (头插法) 。

现在我们从宏观上来看看 `put()` 的整个过程。

_PS：做动画的时候想成尾插了，这里大家注意一下，不要被误导，JDK 1.7 就是头插法_

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ae947ba0eeab4792ac6aee5315bead70~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

前两个节点算出来的 `index` 都是 `4`，发生了 **hash 碰撞**，因此第二个节点通过 **头插法** 的方式，放到了链表的头部，第三个节点的 `index` 是 `1`，放在了数组下标为 `1` 的地方。

好了，`put()` 我们讲完后，`get()` 就不在话下了，它的原理还是和 `put()` 一样，先通过 `hash` 计算出对应数组的下标，然后就去链表中遍历，最后返回相应的结果。现在来讲讲几个重要的变量。

```
// 初始大小
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // 向左位移4位，也就是16

// 加载因子
static final float DEFAULT_LOAD_FACTOR = 0.75f;
复制代码
```

`HashMap` 默认大小就是 `16`，这个加载因子与它的大小乘积叫作 `threshold 阈值`，一旦超过了这个值，`HashMap` 就会进行一次扩容。不过请大家思考一下，**HashMap 中扩容会造成什么问题？**

没错，一旦扩容 `HashMap` 中的数组长度就会增加，而我们计算下标正是通过 **hash 和 (length - 1) 进行按位与** ，因此扩完容后下标肯定会变，那原来储存的数据就会被打乱。在 `JDK 1.7` 中的处理办法是扩完容后 (创建一个新的数组，大小是原来的 2 倍)，会遍历每一个 `entry` ，进行 **rehash** ，再次计算 **hash 和 数组下标，然后放到新的数组中去。**

```
// 如果当前 size > 阈值，扩容之后就会调用这个方法，进行 rehash
void transfer(Entry[] newTable, boolean rehash) { 
    int newCapacity = newTable.length; 
    for (Entry<K,V> e : table) { 
        while(null != e) {
            // 得到当前节点的下一个节点 next
            // 留意一下这段代码，HashMap形成死循环时会讲到
            Entry<K,V> next = e.next; 
            if (rehash) { 
                // 重新计算hash
                e.hash = null == e.key ? 0 : hash(e.key); 
            } 
            // 每一个 entry 重新计算数组所对应的下标
            int i = indexFor(e.hash, newCapacity;
            e.next = newTable[i]; 
            // 放到新的数组中去
            newTable[i] = e;
            e = next; 
        } 
    } 
}
复制代码
```

✈️ **HashMap** 的`插入`， `查找`， `删除` 的核心原理都一样，这里做个**总结**。

<table><thead><tr><th>操作</th><th>原理</th></tr></thead><tbody><tr><td><strong>插入</strong></td><td>先计算 <code>key</code> 的 <code>hash</code> 值，根据 <code>hash</code> 值找到数组位置，再往链表中添加元素</td></tr><tr><td><strong>查找</strong></td><td>先计算 <code>key</code> 的 <code>hash</code> 值，根据 <code>hash</code> 值找到数组位置，再遍历链表，找到对应的节点</td></tr><tr><td><strong>删除</strong></td><td>先计算 <code>key</code> 的 <code>hash</code> 值，根据 <code>hash</code> 值找到数组位置，再从链表中找到对应节点，然后删除节点</td></tr></tbody></table>

### 为什么说 `HashMap` 会造成死循环？

在多线程下扩容 `HashMap` 会形成死循环，我们重新回顾 `transfer()` 后再看看正常情况下的扩容。

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cbddc6982aa14b4d88e6c088729c121a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

成功扩容后，我们发现链表被倒置了，现在再来看看 **多线程** 下对 **HashMap** 进行扩容形成环形数据结构的情况。

假设现在有两个线程在同时扩容 `HashMap`，当 `线程A` 执行到 `Entry<K,V> next = e.next` 时被挂起，待到 `线程B` 扩容完毕后，`线程A` 重新拿到 **CPU 时间片** 。

因为 `线程A` 在扩容前执行过 `Entry<K,V> next = e.next`，因此现在 `线程A` 的 `e` 和 `next` 分别指向 `key("cofbro")` 和 `key("cc")`。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/9071976b047744dabee71eef9060ef3a~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

现在 `线程A` 开始扩容，首先执行 `newTab[i] = e`，将 `entry` 插入到数组中，随后按照顺序执行 `e = next`，然后进入下一个循环 `Entry<K,V> next = e.next`，**因为此时 HashMap 已经被 `线程B` 扩容完成，所以此时的 `next = key("cofbro")`** 。

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/b64d229a17514693b1d6aa1b03c87642~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

现在继续执行上个操作流程，不过由于 `key("cofbro").next` 没有节点了，因此 `next` 是 `null`。 ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f9e371878c634f2b8b2d129fc34c2617~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

我们看到这个时候又会将 `key("cofbro")` 插到链表的头部，死循环就这样产生了。 ![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a1a86cd5edad4405a0d95928dc648b6c~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

JDK 1.7 以后的 HashMap
-------------------

我们现在知道 `HashMap` 结合了 `ArrayList` 的查询效率高的特点以及 `LinkedList` 插入效率高的特点，但是如果我们要存储的数据过于庞大，肯定会造成很多次的 **哈希冲突**，这样一来，链表上的节点会堆积得过多，在做查询的时候效率又变得很低，又失去了 `HashMap` 本来的特点。

那么 `JDK 1.8` 就做出了改变，使用 **数组 + 链表 + 红黑树** 的结构。当节点数不大于 8 时，还是一个链表结构，只不过插入节点时变成了 **尾插法** ，当节点数大于 8 后，将从链表结构转化成红黑树结构，复杂度也从 `O(n)` 变成 `O(logn)`。

### 什么是红黑树

讲解 **HashMap** 原理之前，先来说说 **什么是红黑树**。

**红黑树** 其实是一颗特殊的 `二叉排序树`，它有以下限定：

`1.每一个节点都有一个颜色，非黑即红`

`2.所有 null节点都是叶子节点，并且和根节点一样，都是黑色`

`3.所有红色节点的子节点都是黑色`

`4.从任一节点到其叶子节点的所有路径上都包含相同数目的黑节点`

上面这些限定的主要作用其实是保证 **从根节点开始向下查找到叶子节点，其最长路径不多于最短路径的 2 倍**，也就是说我们去查询一个数据，最坏情况只比最好情况差一倍，这样提高了整体的查询速度。而 **红黑树** 本身就是一颗特殊的 **二叉排序树**，因此它的查询复杂度从链表的 `O(n) -> O(logn)` 。

♻️ 为了保持红黑树上述特性，我们每次插入、删除都需要对其进行一定的处理，才能使得这课红黑树一直保持这它的特点，这里我以插入来举例。

### 红黑树的变色

由于有 **性质 4** 的存在，因此每次插入的节点都先是 **红色**。

1️⃣ `情况1：`插入的新节点位于树的根上、其父节点是黑色

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e12fbd1115c941b08a9dc903ba632bed~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

2️⃣ `情况2：`如果插入节点的父节点和父兄节点都是红色，父节点，父兄节点、祖父节点都要变色。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/613251e6492b46bc8b2a12f2b7c0ba42~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

### 红黑树的平衡

为了保持 **红黑树** 的相对平衡以及维护红黑树特性，每次插入时会判断是否进行旋转。

1️⃣ `情况1：`当插入节点的父节点是红色、父兄节点是黑色且插入节点是左子树，这时进行一次右旋转。

**步骤：`插入节点7`，将根节点的左子树改为新插入节点的兄弟节点，再将根节点作为新插入节点的兄弟节点，最后把新插入节点的父节点改为根节点。**

![](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/7bb8bf59b9c84426ae545034b202f017~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

有点懵的话我们再来看看动画:

![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ba0a8703be1e4e7f85dc2b2ff0e20706~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

2️⃣ `情况2：`当插入节点的父节点是红色、父兄节点是黑色且插入节点是右子树，这时进行一次左旋转。

**步骤：`插入节点30`，将根节点的右子树改为新插入节点的兄弟节点，再将根节点作为新插入节点的兄弟节点，最后把新插入节点的父节点改为根节点。** ![](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/d9dc7ec59f9f40899475d24f1dc2d5ef~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.awebp?)

有些复杂的平衡往往需要经历两次旋转才能完成，比如 **左右旋转（先左再右）** 或者 **右左旋转（先右再左）**，不过都是基于上述两种变换完成的，搞懂一次旋转后，二次旋转是没有问题的，这里就不赘述了。

### 新版 HashMap 原理

新版的 **HashMap** 将原来的 `Entry` 改了个名字变成 `Node` 了，并且新增了 `TreeNode` 节点，专门为红黑树指定的。

```
// 就是原来的 Entry<K,V>
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;

    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }   
}

// 红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;
    // 这里有 next 是因为转化成红黑树结构后，仍然维护着链表结构
    // 并不会因为转化成红黑树后而将链表结构删除掉
    TreeNode(int hash, K key, V val, Node<K,V> next) {
        super(hash, key, val, next);
    }

   
    final TreeNode<K,V> root() {
        for (TreeNode<K,V> r = this, p;;) {
            if ((p = r.parent) == null)
                return r;
            r = p;
        }
    }
}
复制代码
```

同样的，我们看原理还是从 `put()` 入手，跟着我来一起看一遍 `put()` 工作原理。

```
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // table为空或者length等于0，就调用resize方法进行初始化（第一次放元素就会第一次调用resize）
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 通过hash值计算下标，将下标位置的头节点赋值给p，如果p为空则在数组对应位置添加一个节点
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
        // 如果数组的当前下标位置不为 null，则进行遍历
        // 如果当前节点的key和hash值和之前储存的一样，代表找到了目标节点，将其赋值给 e
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果 p 节点类型是 TreeNode，则调用红黑树的putTreeVal方法查找目标节点
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            for (int binCount = 0; ; ++binCount) {
                // 这里就是调用链表结构的查找方式
                // 如果p的next节点为空时，则新增一个节点并插入链表尾部，注意这里从头插法改为尾插法
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // 如果节点数查过8，调用treeifyBin()转为红黑树结构
                    if (binCount >= TREEIFY_THRESHOLD - 1)
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果e节点的hash值和key值都与传入的相同，代表找到了目标节点
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
                // 将p指向下一个节点
                p = e;  
            }
        }
        // 找到目标节点后，用新值替换旧值并返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;
    // 如果插入节点后节点数超过阈值，则调用resize方法进行扩容
    if (++size > threshold)
        resize();
    afterNodeInsertion(evict);
    return null;
}
复制代码
```

现在再来看看是如何构建红黑树的：

```
final void treeify(Node<K,V>[] tab) {
    TreeNode<K,V> root = null;
    for (TreeNode<K,V> x = this, next; x != null; x = next) {
        next = (TreeNode<K,V>)x.next;   // 获得下一个节点 next
        x.left = x.right = null;    // 初始化红黑树
        if (root == null) {  // 初始化根节点
            x.parent = null;
            x.red = false;
            root = x;
        }
        else {
            K k = x.key; // 得到每一个 key
            int h = x.hash; // 得到每一个 hash值
            Class<?> kc = null;
            // 从根节点遍历，寻找插入位置
            // 根据红黑树的特性，向左越小，向右越大
            for (TreeNode<K,V> p = root;;) {
                int dir, ph;
                K pk = p.key;
                if ((ph = p.hash) > h)
                    // 向左查找
                    dir = -1;
                else if (ph < h)
                    // 向右查找 
                    dir = 1;
                // 如果两个hash值相同，就比较key
                else if ((kc == null && 
                          (kc = comparableClassFor(k)) == null) ||
                         (dir = compareComparables(kc, k, pk)) == 0)
                    // 比较当前这个节点和遍历到的节点的大小
                    dir = tieBreakOrder(k, pk);
                TreeNode<K,V> xp = p;
                // dir <= 0 则向p左边查找,否则向p右边查找
                // 如果为null，则当前位置就是即将插入的位置
                if ((p = (dir <= 0) ? p.left : p.right) == null) {
                    x.parent = xp;
                    // 向左边查找
                    if (dir <= 0)
                        xp.left = x;
                    // 向右边查找
                    else
                        xp.right = x;
                    // 这里就是前面讲到的，对二叉树进行平衡和变色以保持红黑树特性
                    // 就是之前讲到的原理
                    root = balanceInsertion(root, x);
                    break;
                }
            }
        }
    }
    // 如果root节点不是节点, 就将其调整为头节点
    moveRootToFront(tab, root);
}
复制代码
```

在前面我们看过 `JDK 1.7` 的 **HashMap** , 由于原理还是差的不多，因此还是能轻易看懂的。在新版的 **HashMap** 中增加了 `TREEIFY_THRESHOLD` 变量，目的就是用来标识是否将链表转化成红黑树。此外还有变化就是 `resize(扩容时)` **不再像 JDK 1.7 那样重新计算 hash 了。**

✔️ **JDK 1.8 是这么优化的：将扩容后的长度 - 1 按位与 上原来计算出的 key 的 hash 值**，这样的好处是：

`1.HashMap中计算出的 hash 转化成二级制后 0 1可以看成是均匀分布的，这样与 (length - 1) 按位与计算出来的值在数组中也还是均匀分布的`

`2.因此进行按位与会有两种结果。某个 node 的hash对应的 (length - 1) 最高位的数如果是1，新数组下标就是 oldIndex + 旧数组的大小，如果是0，新数组下标就是 oldIndex，就是原索引位置`

`3.相比上个版本的 HashMap 少了一次重新计算hash的过程，提高了一定的效率`

```
// JDK 1.8 的 扩容resize
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 如果旧表的大小比 Integer.MAX_VALUE 还大这就没办法扩容了，直接返回旧表
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 扩容，将newCap变成oldCap的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 新的阈值改为原来的两倍
            newThr = oldThr << 1;
    }
    else if (oldThr > 0)
        newCap = oldThr;
    else {
        // 这时将阈值和容量设置成初始值，因为初次创建 HashMap 就会调用一次 resize
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 通过 加载因子 * 新表容量 计算出新表的阈值 
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    // 遍历旧表所有节点进行重新填装
    if (oldTab != null) {
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果e.next为空, 说明旧表中该位置只有1个节点，根据新的索引，放进新表中
                if (e.next == null)
                    // 这就是刚刚讲到的新版 HashMap 中计算下标的方法，很精妙
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    // 有哈希冲突，红黑树中进行重hash
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    // 有哈希冲突，链表中进行重hash
                    // 下标仍然是 原来位置的 节点 
                    Node<K,V> loHead = null, loTail = null;
                    // 下标是 原位置下标 + oldCap 的节点
                    Node<K,V> hiHead = null, hiTail = null; 
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 如果e的hash值相对旧表的容量的最高位不是1，则扩容后的索引和原来一样
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 如果e的hash值相对旧表的容量的最高位不是1，则扩容后的索引是原索引 + oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 旧表的数据重新填充到新表上 原索引 的节点
                    // 将最后一个节点的next设为空
                    // 并将新表上索引位置为 原索引 的节点设置为头节点
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 旧表的数据重新填充到新表上 原索引+oldCap 的节点
                    // 将最后一个节点的next设为空
                    // 并将新表上索引位置为 原索引+oldCap 的节点设置为头节点
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    // 返回扩容后的新表
    return newTab;
}
复制代码
```

😀 **新版的 HashMap 的 put() 原理到这里就讲完了**，查询和删除还是之前说的，只要懂了 `put()`，再去看它们源码的时候也是很轻松了，到这里我已经写了太多了，就不再拿出来讲了。

接下来我们回答一下开头提出的问题。

问题总结
----

```
为什么要用HashMap，你知道底层原理吗？
复制代码
```

_答：HashMap 结合了查询效率快以及增删效率快的优点，作为一个容器是非常可贵的。HashMap 存的都是一个个键值对，我们通过键值对对其进行修改和访问，在 JDK 1.7 及以前底层是一个数组 + 链表的结构，当发生哈希冲突时，就将节点采用头插法的方式放在链表头部。在 JDK1.7 以后底层是一个数组 + 链表 + 红黑树的结构，同样的，发生哈希冲突时，依旧是插到链表里去，只不过现在是尾插法，这样做就是为了避免出现循环数据，另外当链表节点大于 8 时，会转成红黑树进行存储，但这并不代表删除了链表结构，链表结构依然存在，当节点数量重新小于 8 后，红黑树又会重新变成链表结构。(再说说 get(),put() 方法原理就差不多了)。_

```
2.HashMap 中的 hash 值有什么用？
复制代码
```

_HashMap 中的 hash 值是由 hash 函数产生的，它是保证 HashMap 能正常运转的基石，所有键值对进行存储时的位置都是靠它和 (length - 1) 进行位运算得出的，我们进行插入、访问、删除都必须先得到对应的 hash 值，再通过它算出对应的索引值，最后才能操作对应的键值对。_

```
3.为什么 HashMap 中的数组长度必须是 2 的幂次方？
复制代码
```

_因为数组长度减一按位与上 hash()，从结果上来看等同于 `h % length` ，恰好在我们的数组范围内。此外，数组长度是 2 的幂次方保证了在与 hash() 进行按位与时，低位的每一位都是 1。又因为我们通过 hash() 计算出的值可以认为是均匀分布的，因此二者进行按位与后产生的值，也就是索引值，在数组当中就是分布均匀的。另外，在 JDK1.8 后，因为有上述的特点，当扩容时我们不用再重新计算 hash 值，只需要判断当前节点对应的最高位上是否是 1，如果是 1，就代表新索引位置为 oldIndex + oldCap，如果是 0，则新索引位置还是 oldIndex。从而减少一次 hash 计算，提高效率_

```
4.为什么建议用 String Integer 这样的包装类作为 HashMap 的键？
复制代码
```

_首先，这些类内部重写了 hashCode()，equal() 方法，本身就遵循 HashMap 中的规范，可以有效减小 hash 碰撞的概率。其次，这些类都是 final 类型，也就是说 key 是不可变的，避免同一个对象的计算出的 hash 值不一样。除了上面所说的，很多容器都不能用基本类型并且也不能存 null 值也是一个原因。_

```
5.JDK 1.7中为什么会形成死循环？
复制代码
```

_JDK1.7 中在链表中添加节点采用的是头插法，这就导致两个线程同时去扩容 HashMap 的时候会出现一个线程执行到一少半，比如执行得到 e 和 next，这时另一个线程抢占了 CPU 时间片并且将 HashMap 扩容完成，此时 HashMap 中链表是一个倒序。这个时候第一个线程再去扩容时，第一次得到的 e 就会去访问两次，在链表中插入两次，这就会导致循环数据的产生，从而访问时形成死循环。虽然说 JDK1.8 解决了这个问题，但它依然是线程不安全的，并发场景下还是要使用 ConcurrentHashMap。_

```
6.红黑树的概念，它的原理是什么，为什么要用它？
复制代码
```

_红黑树是一颗特殊的二叉查找树，它有每一个节点都有一个颜色，非黑即红、所有 null 节点都是叶子节点，并且和根节点一样，都是黑色等特点。引入它的意义就是为了解决 hash 碰撞次数过多，导致链表长度太大，查询耗时的问题。有了红黑树，我们查询效率就提升至 O(logn)，这就类似二分查找。(红黑树原理就是上面讲的，总结一下)_

写在最后
----

篇幅很长，能看到最后很不容易，给自己一个 **大大的赞** 吧！👏👏

### 如果觉得写的不错😀😀，就给个赞再走吧~

**创作实属不易，你的肯定是我创作的动力，下次见！**

_如果本篇博客有任何错误的地方，请大家批评指正，不胜感激。_