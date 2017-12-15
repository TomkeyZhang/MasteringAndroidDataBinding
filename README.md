#  Android Data Binding [![Build Status](https://travis-ci.org/LyndonChin/MasteringAndroidDataBinding.svg)](https://travis-ci.org/LyndonChin/MasteringAndroidDataBinding)

修改自
https://github.com/LyndonChin/MasteringAndroidDataBinding
根据最新的官方文档[Data Binding Guide](https://developer.android.com/tools/data-binding/guide.html) 对内容进行了补充和调整

Data Binding 解决了 Android UI 编程的一个痛点，官方**原生支持** MVVM 模型可以让我们在不改变既有代码框架的前提下，非常容易地使用这些新特性。

Data Binding 框架如果能够推广开来，也许 *RoboGuice、ButterKnife* 这样的依赖注入框架会慢慢失去市场，因为在 Java 代码中直接使用 `View` 变量的情况会越来越少。

## 准备

新建一个 Project，确保 [Android 的 Gradle 插件](build.gradle#L8)版本不低于 **1.5.0-alpha1**：

```groovy
classpath 'com.android.tools.build:gradle:1.5.0'
```

Android Studio 不低于1.3，然后修改对应模块（Module）的 [build.gradle](app/build.gradle#L7-L9)：

```groovy
dataBinding {
    enabled true
}
```
<strong>注意：</strong>如果有一个依赖library使用了data binding，那么主项目也需要配置data binding
## 一、基础用法

工程创建完成后，我们通过一个最简单的例子来说明 Data Binding 的基本用法。

### 1.1 布局文件

使用 Data Binding 之后，xml 的布局文件就不再用于单纯地展示 UI 元素，还需要定义 UI 元素用到的变量。所以，它的根节点不再是一个 `ViewGroup`，而是变成了 `layout`，并且新增了一个节点 `data`。

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"/>
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"/>
   </LinearLayout>
</layout>
```
data中定义的User变量可以方便的被下面的布局引用，引用的方式@{}.

### 1.2 数据对象

添加一个 POJO 类, 以下3种写法是等价的

```java
public class User {
   public final String firstName;
   public final String lastName;
   public User(String firstName, String lastName) {
       this.firstName = firstName;
       this.lastName = lastName;
   }
}
```
```java
public class User {
    private final String firstName;
    private final String lastName;

    public User(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    public String getFirstName() {
        return firstName;
    }

    public String getLastName() {
        return lastName;
    }
}
```
```java
public class User {
    private final String firstName;
    private final String lastName;

    public User(String firstName, String lastName) {
        this.firstName = firstName;
        this.lastName = lastName;
    }

    public String firstName() {
        return firstName;
    }

    public String lastName() {
        return lastName;
    }
}
```
稍后，我们会新建一个 `User` 类型的变量，然后把它跟布局文件中声明的变量进行绑定。

### 1.3 绑定数据

默认情况下，Android Studio会根据xml文件的名称生成一个继承自`ViewDataBinding`的绑定类，例如main_activity.xml会生成一个MainActivityBinding类，这个类中会生成xml中的variable变量，View对象以及数据绑定的逻辑。创建这个类对象最简单的方式就是在加载布局的时候

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
   super.onCreate(savedInstanceState);
   MainActivityBinding binding = DataBindingUtil.setContentView(this, R.layout.main_activity);
   User user = new User("Test", "User");
   binding.setUser(user);
}
```

其中这一句

```java
   MainActivityBinding binding = DataBindingUtil.setContentView(this, R.layout.main_activity);
```

也可以换成这两句：

```java
MainActivityBinding binding = MainActivityBinding.inflate(getLayoutInflater());
setContentView(binding.getRoot());
```
在ListView或者RecylerView的adapter中可以这样用：

```java
ListItemBinding binding = ListItemBinding.inflate(layoutInflater, viewGroup, false);
//或者
ListItemBinding binding = DataBindingUtil.inflate(layoutInflater, R.layout.list_item, viewGroup, false);
```

### 1.4 事件处理
Data Binding中可以使用表达式来处理View中的事件（如：OnClick），这些xml中事件的名称一般都是跟View中相应的listener中的方法一致，比如View.OnLongClickListener中有`onLongClick()`方法，所以xml中会有事件`android:onLongClick`，目前有两种方式来处理一个事件：

* Method References：这种方式你的方法参数要跟监听器的参数完全一致。当表达式表示的是一个方法引用，Data Binding会把这个方法包装在一个listener中，然后把这个listener设置到目标View上。


* Listener Bindings：这种方式Data Binding同样也会把lambda表达式的逻辑包装到一个listener里面去，在事件发生时执行你的lambda表达式。

####1.4.1 Method References(示例)

```java
public class MyHandlers {
    public void onClickFriend(View view) { ... }
}
```
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="handlers" type="com.example.MyHandlers"/>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"
           android:onClick="@{handlers::onClickFriend}"/>
   </LinearLayout>
</layout>
```
注意：onClickFriend方法的参数必须跟View.OnClickListener中的onClick方法参数完全一致。这一点跟我们将`android:onClick`指向Activity中的某个方法是一样的。如果参数不一致，会在编译的时候报错。

####1.4.2 Listener Bindings(示例)
在Method References中，方法的参数必须跟listener中的方法参数完全匹配，但是在Listener Bindings中只需要返回值跟listener中方法的返回值一致就可以。

```java
public class Presenter {
    public void onSaveClick(Task task){}
}
```

```xml
<?xml version="1.0" encoding="utf-8"?>
  <layout xmlns:android="http://schemas.android.com/apk/res/android">
      <data>
          <variable name="task" type="com.android.example.Task" />
          <variable name="presenter" type="com.android.example.Presenter" />
      </data>
      <LinearLayout android:layout_width="match_parent" android:layout_height="match_parent">
          <Button android:layout_width="wrap_content" android:layout_height="wrap_content"
          android:onClick="@{() -> presenter.onSaveClick(task)}" />
      </LinearLayout>
  </layout>
```
这两者最重要的区别是给View设置listener的时机不一样，Method References是只有在绑定好handler对象（handler不为null）后才会给View设置listener，而Listener Binding会一直挂一个listener到View上，只有handler对象不为null时才会执行相应方法。

这种方法更加灵活，你可以忽略view参数，也可以声明一个view参数，可以将此参数往下传，也可以不传

```xml
 android:onClick="@{(view) -> presenter.onSaveClick(task)}"
```
如果需要view参数

```java
public class Presenter {
    public void onSaveClick(View view, Task task){}
}
```

```xml
 android:onClick="@{(theView) -> presenter.onSaveClick(theView,task)}"
```
也可以这样：

```java
public class Presenter {
    public void onCompletedChanged(Task task, boolean completed){}
}
```

```xml
   <CheckBox android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:onCheckedChanged="@{(cb, isChecked) -> presenter.completeChanged(task, isChecked)}" />
```
如果事件的方法需要一个返回值，比如onLongClickListener，onLongClick方法必须返回一个boolean的值

```java
public class Presenter {
    public boolean onLongClick(View view, Task task){}
}
```

```xml
   android:onLongClick="@{(theView) -> presenter.onLongClick(theView, task)}"
```
如果因为事件处理对象（presenter）为null，导致表达式无法执行，框架会返回java类型的默认值。如：对象类型返回null，int类型返回0，boolean类型返回false等等。


####1.4.3 避免复杂的 Listeners
写好Listener表达式可以让你的代码简洁易读，但是另一方面如果在其中写很复杂的表达式会使你的布局变得难以阅读和维护，建议只在表达式中调用某个类的方法就可以了，不要有逻辑判断。
附：某些`View`存在特定的`click`事件处理器，为了解决跟`android:onClick`的冲突，新增了一些属性来区分

| Class        | Listener Setter           | Attribute  |
| ------------- |:-------------:| -----:|
| SearchView	     |setOnSearchClickListener(View.OnClickListener) | android:onSearchClick |
| ZoomControls     | setOnZoomInClickListener(View.OnClickListener)    |  android:onZoomIn |
| ZoomControls |setOnZoomOutClickListener(View.OnClickListener)      |    android:onZoomOut |
		
				


## 二、布局
### 2.1 Imports标签
在data标签中可以引用0到多个import标签，这样在布局中就可以使用标签中的属性和方法，就像在java代码中使用一样

```xml
<data>
    <import type="android.view.View"/>
</data>
<TextView
   		android:text="@{user.lastName}"
   		android:layout_width="wrap_content"
   		android:layout_height="wrap_content"
   		android:visibility="@{user.isAdult ? View.VISIBLE : View.GONE}"／>
```
如果class的名字冲突，可以定义“别名”

```xml
<import type="android.view.View"/>
<import type="com.example.real.estate.View"
        alias="Vista"/>
```

定义一个`List<User>`对象可以这样写

```xml
<data>
    <import type="com.example.User"/>
    <import type="java.util.List"/>
    <variable name="user" type="User"/>
    <variable name="userList" type="List&lt;User&gt;"/>
</data>
```
如果User里面有一个connection属性也是User，在xml里面可以这样引用这个对象的属性

```xml
<TextView
   android:text="@{((User)(user.connection)).lastName}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```
引用某个类的静态方法可以这么写：

```xml
<data>
    <import type="com.example.MyStringUtils"/>
    <variable name="user" type="com.example.User"/>
</data>
…
<TextView
   android:text="@{MyStringUtils.capitalize(user.lastName)}"
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

在整个xml文件中，java.lang.*包中的类是被自动导入的。比如要使用String类是不需要import的。

### 2.2 Variable标签
每个Variable元素表示一个可以被引用的属性

```xml
<data>
    <import type="android.graphics.drawable.Drawable"/>
    <variable name="user"  type="com.example.User"/>
    <variable name="image" type="Drawable"/>
    <variable name="note"  type="String"/>
</data>
```
如果要针对不同的屏幕模式配置不同的布局，Android Studio生成Binding类的时候会先将这些xml中的Variable合并放到一个基类中，然后不同模式的Binding类会继承这个类。如果这样写将编译出错

layout-land

```xml
<data>
    <variable name="user"  type="com.example.User"/>
</data>
```
layout-port

```xml
<data>
	<variable name="user"  type="com.example.tomkeyzhang.mytestapplication.User"/>
</data>
```

每个Variable会在生成的Binding类生成setter和getter方法，在未赋值之前它们会有一个java的默认值：对象类型-null，int－0，boolean－false等等

默认情况下在布局中有一个隐藏属性context可以使用，这个属性是取自getContext()方法。如果你又定义了一个名为context的Variable，这个属性会被覆盖。

### 2.3 自定义Binding Class的名字和路径
默认情况下是根据xml文件名称来生成Binding Class，具体的规则是首字母大写，去除下划线并将后面的第一个字母大写，然后最后以`Binding`结尾。生成的这个类会被放在包名`（com.example.tomkeyzhang.mytestapplication）`下面的`databinding`包中。

如果要修改Binding Class的名称，可以这样

```xml
<data class="ActivityDataBinding">
    ...
</data>
```
如果希望生成的类直接放在包名目录下，可以这样写

```xml
<data class=".ActivityDataBinding">
    ...
</data>
```
当然，也可以完全自定义路径

```xml
<data class="com.example.ActivityDataBinding">
    ...
</data>
```
### 2.4 Include标签
使用include标签的时候我们可以直接把Variable传递过去

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </LinearLayout>
</layout>
```
同时注意在被include的布局中也需要定义一个user的Variable。
目前Data Binding还不支持在merge下直接放一个include标签，如下是不被支持的

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android"
        xmlns:bind="http://schemas.android.com/apk/res-auto">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <merge>
       <include layout="@layout/name"
           bind:user="@{user}"/>
       <include layout="@layout/contact"
           bind:user="@{user}"/>
   </merge>
</layout>
```
### 2.5 表达式语言
#### 2.5.1 通用特性
目前支持[`这些特性`](https://developer.android.com/topic/libraries/data-binding/index.html#expression_language)，例如：

```xml
android:text="@{String.valueOf(index + 1)}"
android:visibility="@{age < 13 ? View.GONE : View.VISIBLE}"
android:transitionName='@{"image_" + id}'
```

目前还不支持[`这些操作`](https://developer.android.com/topic/libraries/data-binding/index.html#expression_language)

#### 2.5.2 空接合运算符
这个运算符`(??)`表示如果左边不为null就取左边的值，否则取右边的值

```xml
android:text="@{user.displayName ?? user.lastName}"
```
上面的写法等价于

```xml
android:text="@{user.displayName != null ? user.displayName : user.lastName}"
```

#### 2.5.3 属性引用
这个比较简单，之前也用过，不管是在class中定义的是普通属性，`getter`方法，还是`Observable`属性，引用方式都是一样的

```xml
android:text="@{user.lastName}"
```
#### 2.5.4 空值保护
使用表达式任何时候都不会抛空指针异常，生成的Binding类会检查每个属性是否为空，空的话会取默认值

#### 2.5.5 集合对象
可以使用`[]`来访问集合（如：Array、List，Map等）的元素

```xml
<data>
    <import type="android.util.SparseArray"/>
    <import type="java.util.Map"/>
    <import type="java.util.List"/>
    <variable name="list" type="List&lt;String&gt;"/>
    <variable name="sparse" type="SparseArray&lt;String&gt;"/>
    <variable name="map" type="Map&lt;String, String&gt;"/>
    <variable name="index" type="int"/>
    <variable name="key" type="String"/>
</data>
…
android:text="@{list[index]}"
…
android:text="@{sparse[index]}"
…
android:text="@{map[key]}"
```

#### 2.5.6 字符串常量
如果要取map中某一个常量key的值，可以这么写

```xml
android:text='@{map["firstName"]}'
```

```xml
 android:text="@{map[`firstName`]}"
```

但是不能这么谢，虽然文档中说可以。。

```xml
 android:text="@{map['firstName']}"
```

#### 2.5.7 资源
可以在表达式中跟外面使用一样的语法来引用资源

```xml
 android:padding="@{large? @dimen/largePadding : @dimen/smallPadding}"
```
格式化的`string`和复数可以通过参数来表示

```xml
android:text="@{@string/nameFormat(firstName, lastName)}"
android:text="@{@plurals/banana(bananaCount)}"
```
部分[`资源类型`](https://developer.android.com/topic/libraries/data-binding/index.html#expression_language)需要显示声明类型，例如：

```xml
<array name="array">
    <item>1</item>
    <item>2</item>
</array>
```

```xml
android:text="@{``+@intArray/array}"
```
    
## 三、数据对象
任何的POJO对象都可以被绑定到xml上，但是修改一个对象的属性（比如firstName）正常并不会自动更新UI。data binding真正强大之处是它可以在属性变化时自动通知界面更新，目前有三种方式可以实现这个效果。

### 3.1 Observable Objects（观察者对象）
要实现 Observable Binding，首先得有一个 implement 了接口 android.databinding.Observable 的类，为了方便，Android 原生提供了已经封装好的一个类 - BaseObservable，并且实现了监听器的注册机制。

我们可以直接继承 BaseObservable.

```java
public class ObservableUser extends BaseObservable {
    private String firstName;
    private String lastName;

    @Bindable
    public String getFirstName() {
        return firstName;
    }

    @Bindable
    public String getLastName() {
        return lastName;
    }

    public void setFirstName(String firstName) {
        this.firstName = firstName;
        notifyPropertyChanged(BR.firstName);
    }

    public void setLastName(String lastName) {
        this.lastName = lastName;
        notifyPropertyChanged(BR.lastName);
    }
}

```
`BR` 是编译阶段生成的一个类，功能与 `R.java` 类似，用 `@Bindable` 标记过 `getter` 方法会在 `BR` 中生成一个 `entry`。

通过代码可以看出，当数据发生变化时还是需要手动发出通知。 通过调用 `notifyPropertyChanged(BR.firstName)` 可以通知系统 `BR.firstName` 这个 `entry` 的数据已经发生变化，需要更新 `UI`。

### 3.2 ObservableFields（观察者属性）
这种方式无需继承`BaseObservable `直接直接对每个类属性进行设置即可。

```java
private static class User {
   public final ObservableField<String> firstName =
       new ObservableField<>();
   public final ObservableField<String> lastName =
       new ObservableField<>();
   public final ObservableInt age = new ObservableInt();
}
```
然后可以这样使用

```java
user.firstName.set("Google");
int age = user.age.get();
```
对于对象类型的属性可以使用`ObservableField`，对于基本类型，可以直接使用系统给我们封装好的，比如：`ObservableInt`,`ObservableBoolean`等，这样在访问的时候可以避免装箱和拆箱。

### 3.3 Observable Collections（观察者集合）
一些`app`会使用跟动态的方来来存储数据，比如需要用Map和List来存储数据，在集合中的数据发生变化的时候也要更新UI，`Data Binding`给我们提供ObservableArrayMap和ObservableArrayList来处理集合数据的更新。

```java
ObservableArrayMap<String, Object> user = new ObservableArrayMap<>();
user.put("firstName", "Google");
user.put("lastName", "Inc.");
user.put("age", 17);
```

```xml
<data>
    <import type="android.databinding.ObservableMap"/>
    <variable name="user" type="ObservableMap&lt;String, Object&gt;"/>
</data>
…
<TextView
   android:text='@{user["lastName"]}'
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
<TextView
   android:text='@{String.valueOf(1 + (Integer)user["age"])}'
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```

`ObservableArrayList`使用示例

```java
ObservableArrayMap<String, Object> user = new ObservableArrayMap<>();
user.put("firstName", "Google");
user.put("lastName", "Inc.");
user.put("age", 17);
```

```xml
<data>
    <import type="android.databinding.ObservableMap"/>
    <variable name="user" type="ObservableMap&lt;String, Object&gt;"/>
</data>
…
<TextView
   android:text='@{user["lastName"]}'
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
<TextView
   android:text='@{String.valueOf(1 + (Integer)user["age"])}'
   android:layout_width="wrap_content"
   android:layout_height="wrap_content"/>
```
## 四、Binding Class的使用
上面咱们知道Binding Class的名称和包名都是可以被自定义的，所以的Binding Class都是直接或间接的继承自`ViewDataBinding`
### 4.1 创建Binding Class
为了把布局的控制权完全交给`Binding Class`,`Binding`的操作应该紧跟在布局被`inflate`之后，创建`Binding Class`主要有这几种方式

```java
MyLayoutBinding binding = MyLayoutBinding.inflate(layoutInflater);
MyLayoutBinding binding = MyLayoutBinding.inflate(layoutInflater, viewGroup, false);
```

如果`view`已经被`inflate`过了，可以直接把view传进来

```java
MyLayoutBinding binding = MyLayoutBinding.bind(viewRoot);
```

有时候我们并不知道要绑定哪一个布局

```java
ViewDataBinding binding = DataBindingUtil.inflate(LayoutInflater, layoutId,
    parent, attachToParent);
ViewDataBinding binding = DataBindingUtil.bindTo(viewRoot, layoutId);
```

### 4.2 获取View对象
这样在`xml`中给View声明了一个id，就会在`Binding Class`中生成一个相应属性

```xml
<layout xmlns:android="http://schemas.android.com/apk/res/android">
   <data>
       <variable name="user" type="com.example.User"/>
   </data>
   <LinearLayout
       android:orientation="vertical"
       android:layout_width="match_parent"
       android:layout_height="match_parent">
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.firstName}"
   android:id="@+id/firstName"/>
       <TextView android:layout_width="wrap_content"
           android:layout_height="wrap_content"
           android:text="@{user.lastName}"
  android:id="@+id/lastName"/>
   </LinearLayout>
</layout>
```

```
public final TextView firstName;
public final TextView lastName;
```

### 4.3 变量
每个`xml`中的`Variable`都会对应在`Binding Class`中生成`getter and setter`方法


```xml
<data>
    <import type="android.graphics.drawable.Drawable"/>
    <variable name="user"  type="com.example.User"/>
    <variable name="image" type="Drawable"/>
    <variable name="note"  type="String"/>
</data>
```

```
public abstract com.example.User getUser();
public abstract void setUser(com.example.User user);
public abstract Drawable getImage();
public abstract void setImage(Drawable image);
public abstract String getNote();
public abstract void setNote(String note);

```

### 4.4 处理ViewStubs
`ViewStub`只是一个占位符，当真正的`View`加载后它是会被替换掉的. 注意到咱们上面看到`Binding Class`中的`View`都是`final`的，所以对于`ViewStub`，会生成一个`ViewStubProxy`对象来与之对应，用法：

```java
 binding.viewStub.getViewStub().inflate();
 binding.viewStub.setOnInflateListener(new ViewStub.OnInflateListener() {
            @Override
            public void onInflate(ViewStub stub, View inflated) {
                ViewStubBinding stubBinding = DataBindingUtil.bind(inflated);
                stubBinding.setUser(new User("222", "xxx", false));
            }
        });
```
`ViewStub`执行`inflate`会触发`ViewStub.OnInflateListener`，这时候可以创建`Binding Class`，并将数据对象传递进去。

### 4.5 Binding高级特性

## 五、属性设置

## 六、转换器

## 七、双向绑定

