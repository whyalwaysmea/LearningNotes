## ButterKnife简单使用
我们需要先添加依赖
```
dependencies {
  compile 'com.jakewharton:butterknife:8.4.0'
  annotationProcessor 'com.jakewharton:butterknife-compiler:8.4.0'
}
```
然后就可以使用了
```java
class ExampleActivity extends Activity {
  @BindView(R.id.user) EditText username;
  @BindView(R.id.pass) EditText password;

  @BindString(R.string.login_error) String loginErrorMessage;

  @OnClick(R.id.submit) void submit() {
    // TODO call server...
  }

  @Override public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.simple_activity);
    ButterKnife.bind(this);
    // TODO Use fields...
  }
}
```
ButterKnife的使用相信大家也都不会陌生的，不过在讲解源码之前需要一些注解相关的知识。
如果对于注解和apt不了解的可以先看看这两个文章。
虽然android-apt已经不再维护了，今天分析的ButterKnife8.4它是利用Android studio的官方插件annotationProcessor实现的。但是annotationProcessor和apt在表现上看起来也是差不多的

[APT使用](https://github.com/whyalwaysmea/LearningNotes/blob/master/Android/APT.md)

[Java注解](https://github.com/whyalwaysmea/LearningNotes/blob/master/Java/%E6%B3%A8%E8%A7%A3.md)

## Annotation


## 相关链接

[ButterKnife源码分析](http://www.jianshu.com/p/1c449c1b0fa2#)

[Android注解使用之通过annotationProcessor注解生成代码实现自己的ButterKnife框架](http://www.cnblogs.com/whoislcj/p/6168641.html#commentform)

[深入理解 ButterKnife，让你的程序学会写代码](http://dev.qq.com/topic/578753c0c9da73584b025875#rd)
