## 简单使用
1.在你app级别的build.gradle文件中加入DataBinding元素，来在你的应用重配置DataBinding。
```
android {
    ....
    dataBinding {
        enabled = true
    }
}
```
2.DataBinding的布局文件
根布局是一个layout标签，
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <LinearLayout
        android:id="@+id/activity_main"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:id="@+id/first_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <TextView
            android:id="@+id/last_name"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>
</layout>
```
3.简单的数据绑定
```java
private User mUser = new User("Why", "alwayse me a ");

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ActivityMainBinding mainBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);

    mainBinding.firstName.setText(mUser.getFirstName());
    mainBinding.lastName.setText(mUser.getLastName());
}
```
4.UI数据绑定
```xml
private User mUser = new User("Why", "alwayse me a ");

@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ActivityMainBinding mainBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
    mainBinding.setUser(mUser);
//  mainBinding.setVariable(BR.user, mUser);
}
```
5.事件绑定
方法引用：绑定表达式必须和所对应的回调方法有一样的参数
```xml
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="user"
            type="com.whyalwaysmea.databinding.User" />
        <variable
            name="event"
            type="com.whyalwaysmea.databinding.MainActivity.Event" />
    </data>

    <LinearLayout
        android:id="@+id/activity_main"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <TextView
            android:onClick="@{event.clickFisrName}"
            android:text="@{user.firstName}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <TextView
            android:text="@{user.lastName}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>
</layout>
```
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ActivityMainBinding mainBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
    mainBinding.setUser(mUser);
    mainBinding.setEvent(new Event());
}

public class Event {
    public void clickFisrName(View view) {
        Toast.makeText(MainActivity.this, "onclick", Toast.LENGTH_SHORT).show();
    }
}
```

### include使用
```xml
<!-- include_demo.xml-->
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <data>
        <variable
            name="user"
            type="com.whyalwaysmea.databinding.User" />
        <variable
            name="event"
            type="com.whyalwaysmea.databinding.MainActivity.Event" />
    </data>

    <LinearLayout
        android:background="@color/colorPrimary"
        android:id="@+id/activity_main"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="wrap_content">

        <TextView
            android:text="@{user.firstName}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

        <TextView
            android:text="@{user.lastName}"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>
    </LinearLayout>
</layout>

<!-- activity_main.xml-->
<include
    layout="@layout/include_demo" bind:user="@{user}"/>
```

### ViewStub
```xml
<!-- ViewStub-->
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
              android:orientation="vertical"
              android:layout_width="wrap_content"
              android:layout_height="wrap_content">

    <ImageView
        android:src="@mipmap/ic_launcher"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"/>

</LinearLayout>


<!-- activity_main.xml-->
<?xml version="1.0" encoding="utf-8"?>
<layout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    xmlns:bind="http://schemas.android.com/apk/res-auto">

    <data>
        <variable
            name="user"
            type="com.whyalwaysmea.databinding.User" />
        <variable
            name="event"
            type="com.whyalwaysmea.databinding.MainActivity.Event" />
    </data>

    <LinearLayout
        android:id="@+id/activity_main"
        android:orientation="vertical"
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <ViewStub
            android:id="@+id/view_stub"
            android:layout="@layout/view_stub"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

            ...

        <include
            layout="@layout/include_demo" bind:user="@{user}"/>
    </LinearLayout>
</layout>
```
```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    ActivityMainBinding mainBinding = DataBindingUtil.setContentView(this, R.layout.activity_main);
    mainBinding.viewStub.getViewStub().inflate();
}
```
