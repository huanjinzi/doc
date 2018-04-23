pull 解析 xml 是基于事件驱动的方式解析 XML 文件，pull 开始解析时，我们可以先通过`getEventType()`方法获取当前解析事件类型，并且通过`next()`方法获取下一个解析事件类型。pull 解析器提供了`START_DOCUMENT`（开始文档）、`END_DOCUMENT`（结束文档）、`START_TAG`（开始标签）、`END_TAG`（结束标签）四种事件解析类型。当处于某个元素时，可以调用`getAttributeValue()`方法获取属性的值，也可以通过`nextText()`方法获取本节点的文本值。下面通过一个例子来进行解析。

`inflate.xml`

```xml
<?xml version="1.0" encoding="utf-8"?> <!--START_DOCUMENT -->
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:id="@+id/depth0"
>  <!--START_TAG-->

    <!--TEXT-->
    <View 
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/depth1_view"
    /> <!--START_TAG 和 END_TAG-->

    <!--TEXT-->
    <View
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/depth1_view2"
    /> <!--START_TAG 和 END_TAG-->

<!--TEXT-->
</LinearLayout> <!--END_TAG 和 END_DOCUMENT-->
```
> 在 PULL 解析中，必须要相应的标签结束，才会产生相应的事件。

例如：收到 `START_DOCUMENT` 事件，说明
```xml
<?xml version="1.0" encoding="utf-8"?> <!--START_DOCUMENT -->
```
解析完成，接着 `START_TAG` 事件，说明
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:id="@+id/depth0"
>  <!--START_TAG-->
```
解析完成，接着 `TEXT` 事件，说明
```xml

    <!--TEXT-->
```
解析完成，只是这里的`TEXT`为空，接着`START_TAG`和`END_TAG`事件，说明
```xml
<View 
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/depth1_view"
    /> <!--START_TAG 和 END_TAG-->
```
解析完成，接着 `TEXT` 事件，说明
```xml

    <!--TEXT-->
```
解析完成，只是这里的`TEXT`为空，接着`START_TAG`和`END_TAG`事件，说明
```xml
<View
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:id="@+id/depth1_view2"
    /> <!--START_TAG 和 END_TAG-->
```
解析完成，接着 `TEXT` 事件，说明
```xml

    <!--TEXT-->
```
解析完成，只是这里的`TEXT`为空，接着`END_TAG` `END_DOCUMENT`,说明
```xml
</LinearLayout>
```
解析完成，整个 xml 解析完成


