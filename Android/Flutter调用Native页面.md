## Flutter调用Native页面

以下描述都是基于最新stable版本

**Flutter 1.12.13+hotfix.5 • channel stable • https://github.com/flutter/flutter.git**

### io.flutter.facade.*

#### 通过使用 Flutter.createView:
这种方式已经作废。

```
		//通过FlutterView引入Flutter编写的页面
        val flutterView: View = Flutter.createView(
                this,
                lifecycle,
                "route1"
        )
        val layout = FrameLayout.LayoutParams(600, 800)
        layout.leftMargin = 100
        layout.topMargin = 200
        addContentView(flutterView, layout)
```

#### 通过使用 Flutter.createFragment:
```
FragmentTransaction tx = getSupportFragmentManager().beginTransaction();
tx.replace(R.id.someContainer, Flutter.createFragment("route1"));
tx.commit();
```
### io.flutter.embedding.android.*

#### 通过 FlutterView(继承自FrameLayout)

未找到对应实现，应该是作废

```
实例化 FlutterView 嵌入 Native
FlutterView flutterView = new FlutterView(this);
FrameLayout frameLayout = findViewById(R.id.framelayout);
frameLayout.addView(flutterView);
//创建一个 FlutterView 就可以了，这个时候还不会渲染。
//调用下面代码后才会渲染
flutterView.attachToFlutterEngine(flutterEngine);

```

### 最新方式  io.flutter.embedding.android.*

#### 通过FlutterActivity打开：

首先需要在AndroidManifestxml中注册：

```
<activity
   android:name="io.flutter.embedding.android.FlutterActivity"
   android:configChanges="orientation|keyboardHidden|keyboard|screenSize|locale|layoutDirection|fontScale|screenLayout|density|uiMode"
   android:hardwareAccelerated="true"
   android:theme="@style/Theme.AppCompat.Light.NoActionBar"
   android:windowSoftInputMode="adjustResize" />
```
通过调用startactivity打开，默认打开的是在路由中注册为“/”的widget

```
startActivity(
    FlutterActivity.createDefaultIntent(this)
  )
```

也可以通过一下方法打开指定的页面：

```
startActivity(
    FlutterActivity
      .withNewEngine()
      .initialRoute("/my_route")
      .build(this)
  )
```

通过 FlutterFragment 打开

通过 xml

```
<fragment
    android:id="@+id/flutterfragment"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:name="io.flutter.embedding.android.FlutterFragment"
    />
```

通过实例化：

```
class FlutterActivity : AppCompatActivity() {

    companion object {
        // Define a tag String to represent the FlutterFragment within this
        // Activity's FragmentManager. This value can be whatever you'd like.
        private const val TAG_FLUTTER_FRAGMENT = "flutter_fragment"
    }


    private var mFrameLayout: FrameLayout? = null
    private var mFlutterFragment: FlutterFragment? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_flutter_layout)
        initView()
    }

    private fun initView() {

        val fragmentManager: FragmentManager = supportFragmentManager
        mFlutterFragment = fragmentManager.findFragmentByTag(TAG_FLUTTER_FRAGMENT) as FlutterFragment?
        if (mFlutterFragment == null) {
            val flutterFragment = FlutterFragment.createDefault()
            mFlutterFragment = flutterFragment
            fragmentManager.beginTransaction()
                    .add(R.id.fl_content, flutterFragment, TAG_FLUTTER_FRAGMENT)
                    .commit()
        }
    }
}

```


关于FlutterEngine的缓存，[异步中文官网](https://flutter.cn/docs/development/add-to-app/debugging)

[Flutter官网](https://flutter.dev/docs/development/add-to-app/android/add-flutter-screen)
