### Tools attributes reference

Android Studio支持tools命名空间中的各种XML属性，这些属性支持设计时功能（例如，在fragment中显示哪种布局）或编译时行为（例如应用于XML资源的缩小模式）。 构建应用程序时，构建工具会删除这些属性，因此不会影响APK大小或运行时行为。

要使用这些属性，请将tools命名空间添加到您要使用它们的每个XML文件的根元素，如下所示：


```
<RootTag xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools" >
```

---

### Tools属性功能介绍

一、错误处理属性

以下属性用于suppress lint warning messages

##### tools:ignore

Intended for: Any element

Used by: Lint

This attribute accepts a comma-separated list of lint issue ID's that you'd like the tools to ignore on this element or any of its decendents.
此属性接受以逗号分隔的lint问题ID列表，您希望工具在此元素或其任何后代上忽略这些ID。

例如，你可以使用tools来忽略MissingTranslation错误：

```
<string name="show_all_apps" tools:ignore="MissingTranslation">All</string>
```

##### tools:targetApi

Intended for: Any element

Used by: Lint

This attribute works the same as the @TargetApi annotation in Java code: it lets you specify the API level (either as an integer or as a code name) that supports this element.
此属性与Java代码中的@TargetApi注释相同：它允许您指定支持此元素的API级别（作为整数或代码名称）。

This tells the tools that you believe this element (and any children) will be used only on the specified API level or higher. This stops lint from warning you if that element or its attributes are not available on the API level you specify as your minSdkVersion.
这告诉工具您认为此元素（以及任何子元素）将仅在指定的API级别或更高级别上使用。 这会阻止lint警告您，如果该元素或其属性在您指定为minSdkVersion的API级别上不可用。

例如，您可以使用此方法，因为GridLayout仅在API级别14及更高级别上可用，但您知道此布局不用于任何较低版本：


```
<GridLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    tools:targetApi="14" >
```

但是，这只是举例子。。你应该使用support库中的GridLayout。

##### tools:locale

Intended for: <resources>

Used by: Lint, Android Studio editor

This tells the tools what the default language/locale is for the resources in the given <resources> element (because the tools otherwise assume English) in order to avoid warnings from the spell checker. The value must be a valid locale qualifier.
这告诉工具给定<resources>元素中的资源的默认语言/语言环境是什么（因为工具另外假定为英语），以避免来自拼写检查器的警告。 该值必须是有效的区域设置限定符。

例如，您可以将其添加到values / strings.xml文件（默认字符串值），以指示用于默认字符串的语言是西班牙语而不是英语：


```
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:locale="es">
```

---

二、设计时预览属性

以下属性定义的布局特征仅在Android Studio布局预览时可见。

##### tools: instead of android:

Intended for: <View>

Used by: Android Studio layout editor

You can insert sample data in your layout preview by using the tools: prefix instead of android: with any <View> attribute from the Android framework. This is useful when the attribute's value isn't populated until runtime but you want to see the effect beforehand, in the layout preview.
您可以使用工具：前缀而不是android：在Android框架中使用任何<View>属性在布局预览中插入示例数据。 当在运行时之前未填充属性的值但您希望在布局预览中预先看到效果时，这非常有用。

例如：TextView的文字需要在运行时赋值，预览时可使用tools:text="hello world!"来预览布局，而不会影响编译后的显示；同理，tools:visibility="invisible"

##### tools:context

Intended for: Any root <View>

Used by: Lint, Android Studio layout editor

This attribute declares which activity this layout is associated with by default. This enables features in the editor or layout preview that require knowledge of the activity, such as what the layout theme should be in the preview and where to insert onClick handlers when you make those from a quickfix.
此属性声明默认情况下此布局与哪个活动相关联。 这使得编辑器或布局预览中的功能需要activity的信息，例如布局主题在预览中应该是什么以及在从quickfix创建时插入onClick处理程序的位置。

// TODO 插图 tools-attribute-context_2x.png

Quickfix for the onClick attribute works only if you've set tools:context
只有设置了context之后，onClick属性的quickfix才有效

##### tools:itemCount

Intended for: <RecyclerView>

Used by: Android Studio layout editor

For a given RecyclerView, this attribute specifies the number of items the layout editor should render in the Preview window.
对于给定的RecyclerView，此属性指定布局编辑器应在“预览”窗口中呈现的项目数。


```
<android.support.v7.widget.RecyclerView
    android:id="@+id/recyclerView"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:itemCount="3"/>
```

##### tools:layout

Intended for: <fragment>

Used by: Android Studio layout editor

This attribute declares which layout you want the layout preview to draw inside the fragment (because the layout preview cannot execute the activity code that normally applies the layout).
此属性声明您希望布局预览在片段内绘制的布局（因为布局预览无法执行通常应用布局的活动代码）。


```
<fragment android:name="com.example.master.ItemListFragment"
    tools:layout="@layout/list_content" />

```

##### tools:listitem / tools:listheader / tools:listfooter

Intended for: <AdapterView> (and subclasses like <ListView>)

Used by: Android Studio layout editor

These attributes specify which layout to show in the layout preview for a list's items, header, and footer. Any data fields in the layout are filled with numeric contents such as "Item 1" so that the list items are not repetitive.
这些属性指定在列表的项目，页眉和页脚的布局预览中显示的布局。 布局中的任何数据字段都填充了数字内容，例如“Item 1”，以便列表项不重复。


```
<ListView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@android:id/list"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:listitem="@layout/sample_list_item"
    tools:listheader="@layout/sample_list_header"
    tools:listfooter="@layout/sample_list_footer" />
```

##### tools:showIn

Intended for: Any root <View> in a layout that's referred to by an <include>

Used by: Android Studio layout editor

This attribute allows you to point to a layout that uses this layout as an include, so you can preview (and edit) this file as it appears while embedded in its parent layout.
此属性允许您指向将此布局用作包含的布局，因此您可以预览（和编辑）此文件，因为它在嵌入其父布局时显示。


```
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:text="@string/hello_world"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:showIn="@layout/activity_main" />
```

##### tools:menu

Intended for: Any root <View>

Used by: Android Studio layout editor

This attribute specifies which menu the layout preview should show in the app bar. The value can be one or more menu IDs, separated by commas (without @menu/ or any such ID prefix and without the .xml extension)
此属性指定布局预览应在应用栏中显示的菜单。 该值可以是一个或多个菜单ID，以逗号分隔（不带@ menu /或任何此类ID前缀且不带.xml扩展名）

```
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:menu="menu1,menu2" />
```

##### tools:minValue / tools:maxValue

Intended for: <NumberPicker>

Used by: Android Studio layout editor

These attributes set minimum and maximum values for a NumberPicker view.
这些属性设置NumberPicker视图的最小值和最大值。


```
<NumberPicker xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/numberPicker"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    tools:minValue="0"
    tools:maxValue="10" />
```

##### tools:openDrawer

Intended for: <DrawerLayout>

Used by: Android Studio layout editor

This attribute allows you to open a DrawerLayout in the Preview pane of the layout editor. You can also modify how the layout editor renders the layout by passing one of the following values:
此属性允许您在布局编辑器的“预览”窗格中打开DrawerLayout。 您还可以通过传递以下值之一来修改布局编辑器呈现布局的方式：

| Constant    | Value    |  Description  |
| --------   | -----:   | :----: |
| end        | 800005      |   Push object to the end of its container, not changing its size.    |
| left        | 3     |   Push object to the left of its container, not changing its size.    |
| right        | 5      |   Push object to the right of its container, not changing its size.    |
| start        | 800003      |   Push object to the start of its container, not changing its size.    |


```
<android.support.v4.widget.DrawerLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/drawer_layout"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:openDrawer="start" />
```

##### "@tools:sample/*" resources

Intended for: Any view that supports UI text or images.

Used by: Android Studio layout editor

This attribute allows you to inject placeholder data or images into your view. For example, if you want to test how your layout behaves with text, but you don't yet have finalized UI text for your app, you can use placeholder text as follows:
此属性允许您将占位符数据或图像注入视图。 例如，如果要测试布局在文本中的行为方式，但尚未为应用程序确定最终的UI文本，则可以使用占位符文本，如下所示：

```
<TextView xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    tools:text="@tools:sample/lorem" />
```

|Attribute value | Description of placeholder data
| --------   | :----: |
|@tools:sample/full_names	| Full names that are randomly generated from the combination of @tools:sample/first_names and @tools:sample/last_names.
|@tools:sample/first_names	|Common first names.
|@tools:sample/last_names	|Common last names.
|@tools:sample/cities	|Names of cities from across the world.
|@tools:sample/us_zipcodes	|Randomly generated US zipcodes.
|@tools:sample/us_phones	|Randomly generated phone numbers with the following format: (800) 555-xxxx.
|@tools:sample/lorem	|Placeholder text that is derived from Latin.
|@tools:sample/date/day_of_week	|Randomized dates and times for the specified format.
|@tools:sample/date/ddmmyy
|@tools:sample/date/mmddyy
|@tools:sample/date/hhmm
|@tools:sample/date/hhmmss
|@tools:sample/avatars	|Vector drawables that you can use as profile avatars.
|@tools:sample/backgrounds/scenic	|Images that you can use as backgrounds.

---

三、资源收缩属性

以下属性允许您启用严格引用检查，并声明在使用资源收缩时是保留还是丢弃某些资源。

要启用资源收缩，请在build.gradle文件中将shrinkResources属性设置为true（与minifyEnabled一起用于代码收缩）。 例如：


```
android {
    ...
    buildTypes {
        release {
            shrinkResources true
            minifyEnabled true
            proguardFiles getDefaultProguardFile('proguard-android.txt'),
                    'proguard-rules.pro'
        }
    }
}
```

##### tools:shrinkMode

Intended for: <resources>

Used by: Build tools with resource shrinking

This attribute allows you to specify whether the build tools should use "safe mode" (play it safe and keep all resources that are explicitly cited and that might be referenced dynamically with a call to Resources.getIdentifier()) or "strict mode" (keep only the resources that are explicitly cited in code or in other resources).
此属性允许您指定构建工具是否应使用“安全模式”（保护安全并保留所有明确引用的资源，并且可以通过调用Resources.getIdentifier（）动态引用）或“严格模式”（ 仅保留代码或其他资源中明确引用的资源。

The default is to use safe mode (shrinkMode="safe"). To instead use strict mode, add shrinkMode="strict" to the <resources> tag as shown here:
默认是使用安全模式（shrinkMode =“safe”）。 要改为使用严格模式，请将shrinkMode =“strict”添加到<resources>标记:

```
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:shrinkMode="strict" />
```

When you enable strict mode, you may need to use tools:keep to keep resources that were removed but that you actually want, and use tools:discard to explicitly remove even more resources.
当您启用严格模式时，您可能需要使用工具：保留以保留已删除但实际需要的资源，并使用工具：discard以显式删除更多资源。

##### tools:keep

Intended for: <resources>

Used by: Build tools with resource shrinking

When using resource shrinking to remove unused resources, this attribute allows you to specify resources to keep (typically because they are referenced in an indirect way at runtime, such as by passing a dynamically generated resource name to Resources.getIdentifier()).
shrinkMode为strict时，此属性允许您指定要保留的资源（通常是因为它们在运行时以间接方式引用，例如通过将动态生成的资源名称传递给Resources.getIdentifier（））。

To use, create an XML file in your resources directory (for example, at res/raw/keep.xml) with a <resources> tag and specify each resource to keep in the tools:keep attribute as a comma-separated list. You can use the asterisk character as a wild card. 
要使用，请在资源目录中创建XML文件（例如，在res / raw / keep.xml中），并使用<resources>标记并指定要保留在工具中的每个资源：将属性保留为以逗号分隔的列表。 您可以将星号字符用作通配符。

```
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:keep="@layout/used_1,@layout/used_2,@layout/*_3" />
```

##### tools:discard

Intended for: <resources>

Used by: Build tools with resource shrinking

When using resource shrinking to strip out unused resources, this attribute allows you to specify resources you want to manually discard (typically because the resource is referenced but in a way that does not affect your app, or because the Gradle plugin has incorrectly deduced that the resource is referenced).
shrinkMode为strict时，此属性允许您指定要手动丢弃的资源（通常是因为资源被引用但不会影响您的应用，或者因为Gradle插件错误地推断出 资源被引用）。

To use, create an XML file in your resources directory (for example, at res/raw/keep.xml) with a <resources> tag and specify each resource to keep in the tools:discard attribute as a comma-separated list. You can use the asterisk character as a wild card. 

```
<?xml version="1.0" encoding="utf-8"?>
<resources xmlns:tools="http://schemas.android.com/tools"
    tools:discard="@layout/unused_1" />
```

---

#### 参考资料

[Tools attributes reference-developers][1]

[1]:https://developer.android.google.cn/studio/write/tool-attributes