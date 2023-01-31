> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7154560903537491975)

App 中搜索功能是必不可少的，搜索功能可以帮助用户快速获取想要的信息。对此，Android 提供了一个搜索框架，本文介绍如何通过搜索框架实现搜索功能。

搜索框架简介
------

Android 搜索框架提供了搜索弹窗和搜索控件两种使用方式。

*   搜索弹窗：系统控制的弹窗，激活后显示在页面顶部，输入的内容提交后会通过`Intent`传递到指定的搜索`Activity`中处理，可以添加搜索建议。
    
*   搜索控件（`SearchView`）：系统实现的搜索控件，可以放在任意位置（可以与`Toolbar`结合使用），默认情况下与`EditText`类似，需要自己添加监听处理用户输入的数据，通过配置可以达到与搜索弹窗一致的行为。
    

使用搜索框架实现搜索功能
------------

### 可搜索配置

在 res/xml 目录下创建`searchable.xml`（必须用此命名），如下：

```
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:hint="@string/search_hint"
    android:label="@string/app_name" />
复制代码
```

`android:label`是此配置文件必须配置的属性，通常配置为 App 的名字，`android:hint`配置用户未输入内容时的提示文案，官方建议格式为`“搜索${content or product}”`。

更多可搜索配置包含的语法和用法可以看[官方文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Ftopics%2Fsearch%2Fsearchable-config%3Fhl%3Dzh-cn "https://developer.android.google.cn/guide/topics/search/searchable-config?hl=zh-cn")。

### 搜索页面

配置一个单独的`Activity`用于显示搜索内容，用户可能会在搜索完一个内容后立刻搜索下一个内容，所以建议把搜索页面设置为`SingleTop`，避免重复创建搜索页面。

在`AndroidManifest`中配置搜索页面，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        ... 
        >

        <activity
            android:
            android:exported="false"
            android:launchMode="singleTop">

            <meta-data
                android:
                android:resource="@xml/searchable" />

            <intent-filter>
                <action android: />
            </intent-filter>
        </activity>
    </application>
</manifest>
复制代码
```

在`Activity`中处理搜索数据，代码如下：

```
class SearchActivity : AppCompatActivity() {

    private lateinit var binding: LayoutSearchActivityBinding

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.layout_search_activity)
        // 当搜索页面第一次打开时，获取搜索内容
        getQueryKey(intent)
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        // 更新Intent数据
        setIntent(intent)
        // 当搜索页面多次打开，并仍在栈顶时，获取搜索内容
        getQueryKey(intent)
    }

    private fun getQueryKey(intent: Intent?) {
        intent?.run {
            if (Intent.ACTION_SEARCH == action) {
                // 用户输入的内容
                val queryKey = getStringExtra(SearchManager.QUERY) ?: ""
                if (queryKey.isNotEmpty()) {
                    doSearch(queryKey)
                }
            }
        }
    }

    private fun doSearch(queryKey: String) {
        // 根据用户输入内容执行搜索操作
    }
}
复制代码
```

### 使用 SearchView

`SearchView`可以放在页面的任意位置，本文与`Toolbar`结合使用，如何在`Toolbar`中创建菜单项在[上一篇文章](https://juejin.cn/post/7149494902139650079 "https://juejin.cn/post/7149494902139650079")中介绍过，此处省略。要使`SearchView`与搜索弹窗保持一致的行为需要在代码中进行配置，如下：

```
class SearchActivity : AppCompatActivity() {

    private lateinit var binding: LayoutSearchActivityBinding

    override fun onCreateOptionsMenu(menu: Menu?): Boolean {
        menuInflater.inflate(R.menu.example_seach_menu, menu)
        menu?.run {
            val searchManager = getSystemService(Context.SEARCH_SERVICE) as SearchManager
            val searchView = findItem(R.id.action_search).actionView as SearchView
            //设置搜索配置  
            searchView.setSearchableInfo(searchManager.getSearchableInfo(componentName))
        }
        return true
    }
    
    ...
}
复制代码
```

### 使用搜索弹窗

在`Activity`中使用搜索弹窗，如果`Activity`已经配置为搜索页面则无需额外配置，否则需要在`AndroidManifest`中添加配置，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        ... 
        >

        <activity
            android:>

            <!--为当前页面指定搜索页面-->
            <!--如果所有页面都使用搜索弹窗，则将此meta-data移到applicaion标签下-->
            <meta-data
                android:
                android:value=".search.SearchActivity" />
        </activity>
    </application>
</manifest>
复制代码
```

在`Activity`中通过`onSearchRequested`方法来调用搜索弹窗，如下：

```
class SearchExampleActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: LayoutSearchExampleActivityBinding = DataBindingUtil.setContentView(this, R.layout.layout_search_example_activity)
        binding.btnSearchDialog.setOnClickListener { onSearchRequested() }
    }
}
复制代码
```

#### 搜索弹窗对 Activity 生命周期的影响

搜索弹窗的显示隐藏，不会像其他弹窗一样触发`Activity`的`onPause`、`onResume`方法。如果在搜索弹窗显示隐藏的同时需要对其他功能进行处理，可以通过`onSearchRequested`和`OnDismissListener`来实现，代码如下：

```
class SearchExampleActivity : AppCompatActivity() {

    override fun onSearchRequested(): Boolean {
        // 搜索弹窗显示，可以在此处理其他功能
        return super.onSearchRequested()
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: LayoutSearchExampleActivityBinding = DataBindingUtil.setContentView(this, R.layout.layout_search_example_activity)
        binding.btnSearchDialog.setOnClickListener { onSearchRequested() }
        val searchManager = getSystemService(Context.SEARCH_SERVICE) as SearchManager
        searchManager.setOnDismissListener {
            // 搜索弹窗隐藏，可以在此处理其他功能
        }
    }
}
复制代码
```

#### 附加额外的参数

使用搜索弹窗时，如果需要附加额外的参数用于优化搜索查询的过程，例如用户的性别、年龄等，可以通过如下代码实现：

```
// 配置额外参数
class SearchExampleActivity : AppCompatActivity() {

    override fun onSearchRequested(): Boolean {
        val appData = Bundle()
        appData.putString("gender", "male")
        appData.putInt("age", 24)
        startSearch(null, false, appData, false)
        // 返回true表示已经发起了查询
        return true
    }

    ...
}

// 在搜素页面中获取额外参数
class SearchActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)  
        intent?.run {
            if (Intent.ACTION_SEARCH == action) {
                // 用户输入的内容
                val queryKey = getStringExtra(SearchManager.QUERY) ?: ""
                // 额外参数
                val appData = getBundleExtra(SearchManager.APP_DATA)
                appData?.run {
                    val gender = getString("gender") ?: ""
                    val age = getInt("age")
                }
            }
        }
    }
}
复制代码
```

### 语音搜索

语音搜索让用户无需输入内容就可进行搜索，要开启语音搜索，需要在`searchable.xml`增加配置，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:hint="@string/search_hint"
    android:label="@string/app_name"
    android:voiceSearchMode="showVoiceSearchButton|launchRecognizer" />
复制代码
```

语音搜索必须配置`showVoiceSearchButton`用于显示语音搜索按钮，配置`launchRecognizer`指定语音搜索按钮启动一个语音识别程序用于识别语音转录为文本并发送至搜索页面。

更多语音搜索配置包含的语法和用法可以看[官方文档](https://link.juejin.cn?target=https%3A%2F%2Fdeveloper.android.google.cn%2Fguide%2Ftopics%2Fsearch%2Fsearchable-config%3Fhl%3Dzh-cn "https://developer.android.google.cn/guide/topics/search/searchable-config?hl=zh-cn")。

注意，语音识别后的文本会直接发送至搜索页面，无法更改，需要进行完备的测试确保语音搜索功能适合你的 App。

### 搜索记录

用户执行过搜索后，可以将搜索的内容保存下来，下次要搜索相同的内容时，输入部分文字后就会显示匹配的搜索记录。

要实现此功能，需要完成下列步骤：

#### 创建 SearchRecentSuggestionsProvider

自定义 RecentSearchProvider 继承`SearchRecentSuggestionsProvider`，代码如下：

```
class RecentSearchProvider : SearchRecentSuggestionsProvider() {

    companion object {
        // 授权方的名称（建议设置为文件提供者的完整名称）
        const val AUTHORITY = "com.chenyihong.exampledemo.search.RecentSearchProvider"
        // 数据库模式 
        // 必须配置 DATABASE_MODE_QUERIES 
        // 可选配置 DATABASE_MODE_2LINES，为搜索记录提供第二行文本，可用于作为详情补充
        const val MODE: Int = DATABASE_MODE_QUERIES or DATABASE_MODE_2LINES
    }

    init {
        // 设置搜索授权方的名称与数据库模式
        setupSuggestions(AUTHORITY, MODE)
    }
}
复制代码
```

在`AndroidManifest`中配置`Provider`，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        ... 
        >
        
        <!--android:authorities的值与RecentSearchProvider中的AUTHORITY一致-->
        <provider
            android:
            android:authorities="com.chenyihong.exampledemo.search.RecentSearchProvider"
            android:exported="false" />
    </application>
</manifest>
复制代码
```

#### 修改可搜索配置

在`searchable.xml`增加配置，如下：

```
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:hint="@string/search_hint"
    android:label="@string/app_name"
    android:voiceSearchMode="showVoiceSearchButton|launchRecognizer"
    android:searchSuggestAuthority="com.chenyihong.exampledemo.search.RecentSearchProvider"
    android:searchSuggestSelection=" ?"/>
复制代码
```

`android:searchSuggestAuthority` 的值与 RecentSearchProvider 中的 AUTHORITY 保持一致。`android:searchSuggestSelection`的值必须为`" ?"`，该值为数据库选择参数的占位符，自动由用户输入的内容替换。

#### 在搜索页面中保存查询

获取到用户输入的数据时保存，代码如下：

```
class SearchActivity : BaseGestureDetectorActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        intent?.run {
            if (Intent.ACTION_SEARCH == action) {
                val queryKey = getStringExtra(SearchManager.QUERY) ?: ""
                if (queryKey.isNotEmpty()) {
                    // 第一个参数为用户输入的内容
                    // 第二个参数为第二行文本，可为null，仅当RecentSearchProvider.MODE为DATABASE_MODE_QUERIES or DATABASE_MODE_2LINES时有效。
                    SearchRecentSuggestions(this@SearchActivity, RecentSearchProvider.AUTHORITY, RecentSearchProvider.MODE)
                        .saveRecentQuery(queryKey, "history $queryKey")
                }
            }
        }
    }
}
复制代码
```

#### 清除搜索历史

为了保护用户的隐私，官方的建议是 App 必须提供清除搜索记录的功能。请求搜索记录可以通过如下代码实现：

```
class SearchActivity : BaseGestureDetectorActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        SearchRecentSuggestions(this, RecentSearchProvider.AUTHORITY, RecentSearchProvider.MODE)
            .clearHistory()
    }
}
复制代码
```

示例
--

整合之后做了个示例 Demo，代码如下：

```
// 可搜索配置
<?xml version="1.0" encoding="utf-8"?>
<searchable xmlns:android="http://schemas.android.com/apk/res/android"
    android:hint="@string/search_hint"
    android:label="@string/app_name"
    android:searchSuggestAuthority="com.chenyihong.exampledemo.search.RecentSearchProvider"
    android:searchSuggestSelection=" ?"
    android:voiceSearchMode="showVoiceSearchButton|launchRecognizer" />

// 清单文件
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        ...
        >
        
        <activity
            android:
            android:screenOrientation="portrait">

            <!--为当前页面指定搜索页面-->
            <meta-data
                android:
                android:value=".search.SearchActivity" />
        </activity>

        <activity
            android:
            android:exported="false"
            android:launchMode="singleTop"
            android:parentActivity
            android:screenOrientation="portrait">

            <meta-data
                android:
                android:value="com.chenyihong.exampledemo.search.SearchExampleActivity" />

            <meta-data
                android:
                android:resource="@xml/searchable" />

            <intent-filter>
                <action android: />
            </intent-filter>
        </activity>

        <provider
            android:
            android:authorities="com.chenyihong.exampledemo.search.RecentSearchProvider"
            android:exported="false" />
    </application>
</manifest>

// 示例Activity
class SearchExampleActivity : BaseGestureDetectorActivity() {

    override fun onSearchRequested(): Boolean {
        val appData = Bundle()
        appData.putString("gender", "male")
        appData.putInt("age", 24)
        startSearch(null, false, appData, false)
        return true
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val binding: LayoutSearchExampleActivityBinding = DataBindingUtil.setContentView(this, R.layout.layout_search_example_activity)
        binding.btnSearchView.setOnClickListener { startActivity(Intent(this, SearchActivity::class.java)) }
        binding.btnSearchDialog.setOnClickListener { onSearchRequested() }
        val searchManager = getSystemService(Context.SEARCH_SERVICE) as SearchManager
        searchManager.setOnDismissListener {
            runOnUiThread { Toast.makeText(this, "Search Dialog dismiss", Toast.LENGTH_SHORT).show() }
        }
    }
}

class SearchActivity : BaseGestureDetectorActivity() {

    private lateinit var binding: LayoutSearchActivityBinding

    private val textDataAdapter = TextDataAdapter()

    private val originData = ArrayList<String>()

    private var lastQueryValue = ""

    override fun onCreateOptionsMenu(menu: Menu?): Boolean {
        menuInflater.inflate(R.menu.example_seach_menu, menu)
        menu?.run {
            val searchManager = getSystemService(Context.SEARCH_SERVICE) as SearchManager
            val searchView = findItem(R.id.action_search).actionView as SearchView
            searchView.setOnCloseListener {
                textDataAdapter.setNewData(originData)
                false
            }
            searchView.setSearchableInfo(searchManager.getSearchableInfo(componentName))
            if (lastQueryValue.isNotEmpty()) {
                searchView.setQuery(lastQueryValue, false)
            }
        }
        return true
    }

    override fun onOptionsItemSelected(item: MenuItem): Boolean {
        if (item.itemId == R.id.action_clear_search_histor) {
            SearchRecentSuggestions(this, RecentSearchProvider.AUTHORITY, RecentSearchProvider.MODE)
                .clearHistory()
        }
        return super.onOptionsItemSelected(item)
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        binding = DataBindingUtil.setContentView(this, R.layout.layout_search_activity)
        setSupportActionBar(binding.toolbar)
        supportActionBar?.run {
            title = "SearchExample"
            setHomeAsUpIndicator(R.drawable.icon_back)
            setDisplayHomeAsUpEnabled(true)
        }
        binding.rvContent.adapter = textDataAdapter
        originData.add("test data qwertyuiop")
        originData.add("test data asdfghjkl")
        originData.add("test data zxcvbnm")
        originData.add("test data 123456789")
        originData.add("test data /.,?-+")
        textDataAdapter.setNewData(originData)
        // 获取搜索内容
        getQueryKey(intent, false)
    }

    override fun onNewIntent(intent: Intent?) {
        super.onNewIntent(intent)
        // 更新Intent数据
        setIntent(intent)
        // 获取搜索内容
        getQueryKey(intent, true)
    }

    private fun getQueryKey(intent: Intent?, newIntent: Boolean) {
        intent?.run {
            if (Intent.ACTION_SEARCH == action) {
                val queryKey = getStringExtra(SearchManager.QUERY) ?: ""
                if (queryKey.isNotEmpty()) {
                    SearchRecentSuggestions(this@SearchActivity, RecentSearchProvider.AUTHORITY, RecentSearchProvider.MODE)
                        .saveRecentQuery(queryKey, "history $queryKey")
                    if (!newIntent) {
                        lastQueryValue = queryKey
                    }
                    val appData = getBundleExtra(SearchManager.APP_DATA)
                    doSearch(queryKey, appData)
                }
            }
        }
    }

    private fun doSearch(queryKey: String, appData: Bundle?) {
        appData?.run {
            val gender = getString("gender") ?: ""
            val age = getInt("age")
        }
        val filterData = originData.filter { it.contains(queryKey) } as ArrayList<String>
        textDataAdapter.setNewData(filterData)
    }
}
复制代码
```

[ExampleDemo github](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2FChenYiHong930921%2Fexample-demo "https://github.com/ChenYiHong930921/example-demo")

[ExampleDemo gitee](https://link.juejin.cn?target=https%3A%2F%2Fgitee.com%2FChenYhong0921%2Fexample-demo "https://gitee.com/ChenYhong0921/example-demo")

效果如图：

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/1eea255c01274a9ba68e74c50e25a3a3~tplv-k3u1fbpfcp-zoom-in-crop-mark:4536:0:0:0.image?)