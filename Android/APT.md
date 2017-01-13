## APT
APT(Annotation Processing Tool 的简称)，可以在代码编译期解析注解，并且生成新的 Java 文件，减少手动的代码输入。现在有很多主流库都用上了 APT，比如 Dagger2, ButterKnife, EventBus3 等，
使用APT来处理annotation的流程

1. 定义注解（如@automain）
2. 定义注解处理器
3. 在处理器里面完成处理方式，通常是生成java代码。
4. 注册处理器
5. 利用APT完成工作内容。

编译期解析注解基本原理：
在某些代码元素上（如类型、函数、字段等）添加注解，在编译时编译器会检查AbstractProcessor的子类，并且调用该类型的process函数，然后将添加了注解的所有元素都传递到process函数中，使得开发人员可以在编译器进行相应的处理，例如，根据注解生成新的Java类，这也就是ButterKnife等开源库的基本原理。



## 注解处理器
注解处理器是 javac 自带的一个工具，用来在编译时期扫描处理注解信息。你可以为某些注解注册自己的注解处理器。如果你还不了解注解可以看这里[注解](https://github.com/whyalwaysmea/LearningNotes/blob/master/Java/%E6%B3%A8%E8%A7%A3.md)

注解处理器在 Java 5 的时候就已经存在了，但直到 Java 6 （发布于2006看十二月）的时候才有可用的API。过了一段时间java的使用者们才意识到注解处理器的强大。所以最近几年它才开始流行。

一个特定注解的处理器以 java 源代码（或者已编译的字节码）作为输入，然后生成一些文件（通常是.java文件）作为输出。
那意味着什么呢？你可以生成 java 代码！
这些 java 代码在生成的.java文件中。因此你不能改变已经存在的java类，例如添加一个方法。
这些生成的 java 文件跟其他手动编写的 java 源代码一样，将会被 javac 编译。


让我们来看一下处理器的 API。所有的处理器都继承了AbstractProcessor，如下所示：
```java
public class MyProcessor extends AbstractProcessor {

    // 这类似于每个处理器的main()方法。
    // 可以在这个方法里面编码实现扫描，处理注解，生成 java 文件。
	@Override
	public boolean process(Set<? extends TypeElement> annoations,
			RoundEnvironment env) {
		return false;
	}

    // 在这个方法里面你必须指定哪些注解应该被注解处理器注册。
    // 它的返回值是一个String集合，包含了你的注解处理器想要处理的注解类型的全称。
    // 换句话说，你在这里定义你的注解处理器要处理哪些注解。
	@Override
	public Set<String> getSupportedAnnotationTypes() {
		Set<String> annotataions = new LinkedHashSet<String>();
	    annotataions.add("com.example.MyAnnotation");
	    return annotataions;
	}

    // 用来指定你使用的 java 版本。通常你应该返回SourceVersion.latestSupported()
	@Override
	public SourceVersion getSupportedSourceVersion() {
		return SourceVersion.latestSupported();
	}

    // 所有的注解处理器类都必须有一个无参构造函数。
    // 然而，有一个特殊的方法init()，它会被注解处理工具调用，以ProcessingEnvironment作为参数。
    // ProcessingEnvironment 提供了一些实用的工具类Elements, Types和Filer。我们在后面将会使用到它们。
	@Override
	public synchronized void init(ProcessingEnvironment processingEnv) {
		super.init(processingEnv);
	}

}
```

## 注册处理器
由于处理器是javac的工具，因此我们必须将我们自己的处理器注册到javac中
在以前我们需要提供一个.jar文件，打包你的注解处理器到此文件中，并且，在你的jar中，需要打包一个特定的文件 javax.annotation.processing.Processor到META-INF/services路径下把MyProcessor.jar放到你的builpath中，javac会自动检查和读取javax.annotation.processing.Processor中的内容，并且注册MyProcessor作为注解处理器。
不过现在谷歌提供了一个AutoService注解，你只需要引入这个依赖，然后在你的解释器第一行加上
>@AutoService(Processor.class)

然后就可以自动生成META-INF/services/javax.annotation.processing.Processor文件的。省去了打jar包这些繁琐的步骤。

## APT实战
### 注解模块实现
创建module di-annotation ，类型为 Java Library，定义项目所需要的注解。
```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.FIELD)
public @interface DIActivity  {

}

@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface DIView {
    int value() default 0;
}
```


### 注解处理器的实现
创建module ioc-compiler ，类型为 Java Library，用于编写注解处理器。
在刚才介绍注解处理器的时候也看到了，我们会使用一个google提供的库来帮我们生成相关信息，所以我们先进行库的依赖
```
dependencies {
    compile fileTree(include: ['*.jar'], dir: 'libs')
    compile project(':di-annotation')
    compile 'com.squareup:javapoet:1.7.0'
    compile 'com.google.auto.service:auto-service:1.0-rc2'
}
```
* 因为要用到前面定义的注解,所以需要依赖ioc-annotation
* javapoet 是方块公司出的又一个好用到爆炸的裤子，提供了各种 API 让你用各种姿势去生成 Java 代码文件，避免了徒手拼接字符串的尴尬。
* auto-service 是 Google 家的裤子，主要用于注解 Processor，对其生成 META-INF 配置信息。

关于javapoet的使用可以参考这个[JavaPoet](https://github.com/whyalwaysmea/LearningNotes/blob/master/Java/JavaPoet.md)
这里在介绍`AbstractProcessor`中的几个方法，帮助理解
#### 常用Element子类
1. TypeElement：类
2. ExecutableElement：成员方法
3. VariableElement：成员变量

#### 通过包名和类名获取TypeName
```java
TypeName targetClassName = ClassName.get(“PackageName”, “ClassName”);
```

#### 通过Element获取TypeName
```java
TypeName type = TypeName.get(element.asType());
```
#### 获取TypeElement的包名
```java
String packageName = processingEnv.getElementUtils().getPackageOf(type).getQualifiedName().toString();
```

#### 获取TypeElement的所有成员变量和成员方法
```java
List<? extends Element> members = processingEnv.getElementUtils().getAllMembers(typeElement);
```

```java
// 用 @AutoService 来注解这个处理器，可以自动生成配置信息。
@AutoService(Processor.class)
public class DIProcessor extends AbstractProcessor {

    //文件相关的辅助类
    private Filer mFileUtils;
    //元素相关的辅助类
    private Elements mElementUtils;
    //日志相关的辅助类
    private Messager mMessager;

    /**
     * 在 init() 可以初始化拿到一些实用的工具类。
     * @param processingEnv
     */
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv){
        super.init(processingEnv);
        mFileUtils = processingEnv.getFiler();
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
    }

    /**
     * @return   返回所要处理的注解的集合。
     */
    @Override
    public Set<String> getSupportedAnnotationTypes(){
        Set<String> annotationTypes = new LinkedHashSet<String>();
        annotationTypes.add(DIActivity.class.getCanonicalName());
        return annotationTypes;
    }

    /**
     * @return 返回 Java 版本
     */
    @Override
    public SourceVersion getSupportedSourceVersion(){
        return SourceVersion.latestSupported();
    }

    /**
     * 关键的处理类。 可以认为两个大步骤：
     * 1、收集信息
     * 2、生成编译时生成的类
     * @param annotations
     * @param roundEnv
     * @return
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv){
		System.out.println("DIProcessor");
		// 获取标注了@DIActivity 的元素
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(DIActivity.class);
        for (Element element : elements) {
            // 判断是否Class
            TypeElement typeElement = (TypeElement) element;
            List<? extends Element> members = mElementUtils.getAllMembers(typeElement);
			// 生成方法
            MethodSpec.Builder bindView = MethodSpec.methodBuilder("bindView")
                    .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
                    .returns(TypeName.VOID)
                    .addParameter(ClassName.get(typeElement.asType()), "activity");
            for (Element item : members) {
                DIView diView = item.getAnnotation(DIView.class);
                if (diView == null){
                    continue;
                }
                bindView.addStatement(String.format("activity.%s = (%s) activity.findViewById(%s)",item.getSimpleName(),ClassName.get(item.asType()).toString(),diView.value()));
            }

			// 生成类
            TypeSpec typeSpec = TypeSpec.classBuilder("DI" + element.getSimpleName())
                    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                    .addMethod(bindView.build())
                    .build();
            JavaFile javaFile = JavaFile.builder(getPackageName(typeElement), typeSpec).build();
            try {
                javaFile.writeTo(processingEnv.getFiler());
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
        return true;
    }
}
```

### 使用编译时注解
想要使用我们写的简单的DI框架，需要配置一下apt相关的东西
配置项目根目录的build.gradle
```
dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.8'
}
```
配置app的build.gradle
```
apply plugin: 'com.android.application'
apply plugin: 'com.neenbedankt.android-apt'
//...
dependencies {
    //..
    compile project(':di-annotation')
    apt project(':di-compiler')
}
```
编译使用
```java
@DIActivity
public class MainActivity extends AppCompatActivity {

    @DIView(R.id.hello)
    TextView helloDi;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        DIMainActivity.bindView(this);

        helloDi.setText("hello APT !!!");
    }
}
```
可能有人会一下懵逼了，因为`DIMainActivity`我们根本就没有创建过啊，怎么会有那么个方法呢。
其实是一开始我只写了`@DIActivity`，然后编译一下，才生成了DIMainActivity，`DIMainActivity`在app/build/generated/source/apt里面
DIMainActivity是怎么生成的呢？ 它就是通过DIProcessor来生成的。

这就有点像Dagger2了，因为它的就是类似这样的，需要编译一下，生成对应的方法。

当然，我们还可以做进一步的处理，就像ButterKnife一样，预先就把API方法给自己写了，相关的处理也是我们自己来做。在app层就只管调用就好啦


## 相关链接
[Android 如何编写基于编译时注解的项目](http://blog.csdn.net/lmj623565791/article/details/51931859)

[Annotation-Processing-Tool详解](http://qiushao.net/2015/07/07/Annotation-Processing-Tool%E8%AF%A6%E8%A7%A3/)

[Android 利用 APT 技术在编译期生成代码](http://brucezz.itscoder.com/use-apt-in-android#)
