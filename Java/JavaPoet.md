## javapoet介绍
javapoet，是square公司的开源库。正如其名，java诗人，通过注解来生成java源文件，通常要使用javapoet这个库与Filer配合使用。主要和注解配合用来干掉那些重复的模板代码(如butterknife和databinding所做的事情)，当然你也可以使用这个技术让你的代码更加的炫酷。


## 简单使用
使用之前肯定是需要引入这个库的
```
compile 'com.squareup:javapoet:1.7.0'
```
使用javapoet前需要了解4个常用类

* MethodSpec 代表一个构造函数或方法声明。
* TypeSpec 代表一个类，接口，或者枚举声明。
* FieldSpec 代表一个成员变量，一个字段声明。
* JavaFile包含一个顶级类的Java文件。

国际惯例先自动生成一个helloWorld类
定义一个编译期注解
```java
@Retention(RetentionPolicy.CLASS)
@Target(ElementType.TYPE)
public @interface clazz_hello {
    String value();
}
```
然后看下helloworld的注解处理器
```java
@AutoService(Processor.class)
public class HelloWorldProcess extends AbstractProcessor {

    private Filer filer;

    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        filer = processingEnv.getFiler();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        for (TypeElement element : annotations) {
            if (element.getQualifiedName().toString().equals(clazz_hello.class.getCanonicalName())) {
                // 构造方法
                MethodSpec main = MethodSpec.methodBuilder("main")              // 方法名为main
                        .addModifiers(Modifier.PUBLIC, Modifier.STATIC)         // public static
                        .returns(void.class)                                    // return void
                        .addParameter(String[].class, "args")                   // 参数类型： String[] args
                        .addStatement("$T.out.println($S)", System.class, "Hello, JavaPoet!")       // 方法体中的内容:
                        .build();
                TypeSpec helloWorld = TypeSpec.classBuilder("HelloWorld")       // 类的名字: HelloWorld
                        .addModifiers(Modifier.PUBLIC, Modifier.FINAL)          // public final
                        .addMethod(main)                                        // 添加一个方法 main(上面定义的)
                        .build();

                try {
                    JavaFile javaFile = JavaFile.builder("com.whyalwaysmea", helloWorld)                // 文件名，
                            .addFileComment(" This codes are generated automatically. Do not modify!")  // 文件中的注解
                            .build();
                    javaFile.writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return true;
    }

    @Override
    public Set<String> getSupportedAnnotationTypes() {
        Set<String> annotations = new LinkedHashSet<>();
        annotations.add(clazz_hello.class.getCanonicalName());
        return annotations;
    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }
}
```
最后编译一下，再看看生成的文件
```java
//  This codes are generated automatically. Do not modify!
package com.whyalwaysmea;

import java.lang.String;
import java.lang.System;

public final class HelloWorld {
  public static void mainnnn(String[] args) {
    System.out.println("Hello, JavaPoet!");
  }
}
```

## 相关链接
[JavaPoet使用指南](https://gold.xitu.io/post/584d4b5b0ce463005c5dc444)
