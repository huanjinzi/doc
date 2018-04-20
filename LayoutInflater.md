LayoutInflater 用来从 xml 文件加载 View

基本用法：

```java
LayoutInflater layoutInflater = LayoutInflater.from(context);
```

```java
LayoutInflater layoutInflater = (LayoutInflater) context  
        .getSystemService(Context.LAYOUT_INFLATER_SERVICE);
```
简单使用
```java
Button button = layoutInflater.inflate(R.layout.button, null);
```
现在需要从`linearlayout.xml`解析出`LinearLayout`对象，从`button.xml`中解析出`Button`对象，并
将`Button`对象添加到`LinearLayout`对象中，那到底怎么从 xml 中加载 View 呢？

*`linearlayout.xml`*
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">
</LinearLayout>
```
*`button.xml`*
```xml
<?xml version="1.0" encoding="utf-8"?>
<Button xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:text="button"/>
```

```java
public View inflate(@LayoutRes int resource, @Nullable ViewGroup root, boolean attachToRoot) {
        // 获取 Context ，创建 LayoutInflater 时会传递一个 Context 对象进来
        final Resources res = getContext().getResources();
        if (DEBUG) {
            Log.d(TAG, "INFLATING from resource: \"" + res.getResourceName(resource) + "\" ("
                    + Integer.toHexString(resource) + ")");
        }

        // Resources#getLayout() 会调用 Resources#loadXmlResourceParser(resource, "layout");
        final XmlResourceParser parser = res.getLayout(resource);
        try {
            return inflate(parser, root, attachToRoot);// 重要
        } finally {
            parser.close();// 释放资源
        }
    }

```

```java
XmlResourceParser loadXmlResourceParser(@AnyRes int id, @NonNull String type) // type = "layout"
            throws NotFoundException {
        // TypedValue 像 Message 一样，用来传递信息，可以回收利用
        final TypedValue value = obtainTempTypedValue();
        try {
            final ResourcesImpl impl = mResourcesImpl;
            // 在 ResourcesImpl#getValue() 内部会调用 AssetManager#getResourceValue()
            // 会根据传进来的 id 给 value 赋值，后面就可以从 value 中获取资源信息
            impl.getValue(id, value, true);
            if (value.type == TypedValue.TYPE_STRING) {
                // value.type 需要为 TypedValue.TYPE_STRING，value.string.toString()为需要解析的 xml 路径
                // 先在 XmlBlock[] 缓存中查找 XmlBlock，否则由传入的 file[value.string.toString()]创建新的
                // XmlBlock，调用 XmlBlock#newParser() 返回 XmlResourceParser
                return impl.loadXmlResourceParser(value.string.toString(), id,
                        value.assetCookie, type);
            }
            throw new NotFoundException("Resource ID #0x" + Integer.toHexString(id)
                    + " type #0x" + Integer.toHexString(value.type) + " is not valid");
        } finally {
            releaseTempTypedValue(value);// 回收TypedValue
        }
    }
```
```java
public View inflate(XmlPullParser parser, @Nullable ViewGroup root, boolean attachToRoot) {
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "inflate");
        final Context inflaterContext = mContext;
        final AttributeSet attrs = Xml.asAttributeSet(parser);
        View result = null;

        // Look for the root node.
        int type;
        while ((type = parser.next()) != XmlPullParser.START_TAG &&
                type != XmlPullParser.END_DOCUMENT) {
            // Empty
            // 循环调用 parser.next() 直到找到 XmlPullParser.START_TAG
            // 或到 XmlPullParser.END_DOCUMENT
        }
        final String name = parser.getName();//节点标签名
        // Temp is the root view that was found in the xml
        // 反射调用创建 View
        final View temp = createViewFromTag(root, name, inflaterContext, attrs);

        // Inflate all children under temp against its context.
        rInflateChildren(parser, temp, attrs, true);

        // return the top view found in xml.
        result = temp;
        Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        return result;
    }
```

```java
void rInflate(XmlPullParser parser, View parent, Context context,
            AttributeSet attrs, boolean finishInflate) throws XmlPullParserException, IOException {

        final int depth = parser.getDepth();
        int type;
        while (((type = parser.next()) != XmlPullParser.END_TAG ||
                parser.getDepth() > depth) && type != XmlPullParser.END_DOCUMENT) {

            final String name = parser.getName();
            final View view = createViewFromTag(parent, name, context, attrs);
            final ViewGroup viewGroup = (ViewGroup) parent;
            final ViewGroup.LayoutParams params = viewGroup.generateLayoutParams(attrs);
            rInflateChildren(parser, view, attrs, true);
            viewGroup.addView(view, params);
        }
    }
```
```java
final void rInflateChildren(XmlPullParser parser, View parent, AttributeSet attrs,
            boolean finishInflate) throws XmlPullParserException, IOException {
        rInflate(parser, parent, parent.getContext(), attrs, finishInflate);
    }
```
其中 LayoutInflater#rInflate() 中会调用 LayoutInflater#rInflateChildren ，LayoutInflater#rInflateChildren 又会调用 LayoutInflater#rInflate,
形成递归调用，下面是一个xml解析 View 的实例，`inflate.xml`放在 `assets/`下面
*`inflate.xml`*
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:id="@+id/depth0">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:orientation="vertical"
        android:id="@+id/depth1">

        <View  
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:id="@+id/depth2_view" />
        <View  
            android:layout_width="match_parent"
            android:layout_height="match_parent"
            android:id="@+id/depth2_view2" />

    </LinearLayout>

    <View  
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/depth1_view" />

</LinearLayout>
```



