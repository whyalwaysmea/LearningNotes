# IntentFilter的匹配规则
我们知道，启动Activity分为两种，显示调用和隐式调用。
显示调用需要明确地指定被启动对象的组件信息，包括包名和类名；
隐式调用则需要明确指定组件信息。
原则上一个Intent不应该即是显示调用又是隐式调用，如果二者共存的话以显示调用为主。
```xml
<activity
   android:name="com.Activity">
   <intent-filter>
       <action android:name="android.intent.action.SEND" />
       <category android:name="android.intent.category.DEFAULT" />
       <data android:mimeType="image/*" />
   </intent-filter>

   <intent-filter>
       <action android:name="com.abc" />
       <action android:name="com.abcd" />

       <category android:name="com.category.abc" />
       <category android:name="com.category.abcd" />
       <category android:name="android.intent.category.DEFAULT" />

       <data android:mimeType="text/plain" />
   </intent-filter>
</activity>
```
一个Activity中可以有多个intent-filter，一个Intent只要能够匹配任何一组intent-filter即可成功启动对应的Activity。
intent-filter中，action和category是必须存在的。 data按需求而定。


## action的匹配规则
action的匹配要求Intent中的action存在且必须和过滤规则中的其中一个action相同。   
一个Intent只能设置一个action

## category的匹配规则
category的匹配要求Intent中如果含有category，那么所有的category都必须和过滤规则中的其中一个category相同。   

换句话说，Intent中如果出现了category，不管有几个category，对于每个category来说，它必须是过滤规则中已经定义了的category。  

当然，如果在intent-filter中设置了`<category android:name="android.intent.category.DEFAULT" />`，那么Intent不设置category也可以匹配的。因为在调用 startActivity() 方法的时候会自动将这 category 添加到 Intent 中(查看所有系统源码，也都带上了这个 category)

`<category android:name="android.intent.category.DEFAULT" />`这个是必须存在的，因为不含有DEFAULT这个category的Activity是无法接收隐式的Intent的。

## data的匹配规则
data的匹配规则和action类似，如果过滤规则中定义了data，那么Intent中必须也要定义可匹配的data。
如果要为Intent指定完整的data，必须要调用`setDataAndType`方法，不能先调用`setData`再调用`setType`,因为这两个方法会彼此清除对方的值。

在这里浅显介绍一下系统添加 data 下面的标签都有哪些。<data>标签中主要可以配置以下内容：

1.android:scheme   
用于指定数据的协议部分。

2.android:host    
用于指定数据的主机名部分。

3.android:port   
用于指定数据的端口部分，一般紧随在主机名之后。

4.android:path   
用于指定主机名和端口之后的部分，如一段网址中跟在域名之后的内容。
5.android:mimeType   
用于指定可以处理的数据类型，允许使用通配符的方式进行指定。  

只有<data>标签中指定的内容和 Intent 中携带的 Data 完全一致时，当前活动才能够响应该 Intent。不过一般在<data>标签中都不会指定过多的内容，常见的是 mimeType 和 scheme。

## 判断
当我们通过隐式方式启动一个Activity的时候，可以做一下判断，看是否有Activity能够匹配我们的隐式Intent，如果不做判断就有可能crash。
判断方法有两种，采用`PackageManager`的`resolveActivity`方法或者`Intent`的`resolveActivity`方法，如果它们找不到匹配的Activity就会返回null。
