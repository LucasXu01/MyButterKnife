# 前言

目的：写个简易版 ButterKnife，借手写 ButterKnife 去了解如何实现注解、annotationProcessor 的等使用。

先看下butterknife的结构：

<img src="https://img-blog.csdnimg.cn/6a988a718c7e4b86a8ea8ea038ef7c3b.png" style="zoom:33%;" />

# 源码地址

[https://github.com/LucasXu01/MyButterKnife](https://github.com/LucasXu01/MyButterKnife)

# ButterKnife的使用

在`build.gradle`添加依赖：

```java
android {
  ...
  // Butterknife requires Java 8.
  compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
  }
}
dependencies {
  implementation 'com.jakewharton:butterknife:10.2.3'
  annotationProcessor 'com.jakewharton:butterknife-compiler:10.2.3'
}
```

在`MainActivity`中书写：

```java
public class MainActivity extends AppCompatActivity {
    @BindView(R.id.tv_content)
    TextView tvContent;
    @Override
    protected void onCreate(Bundle savedInstanceState) {
		···
        ButterKnife.bind(this);
        tvContent.setText("修改成功！");
    }
}
```

分析可得，手写一个简易的ButterKnife主要分为两步：**注解定义；注解解析**；

# 手写简易ButterKnife

## 注解定义

新建注解`MyBindView.java`：

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface MyBindView {
    int viewId();
}
```

`int viewId();用于绑定哪个view的 id，我们要在注解上加上变量，这样赋值过去，我们就知道为哪个 id 绑定了：

```java
    @MyBindView(viewId = R.id.tv_content)
    TextView tvContent;
```

改进：

viewId 改为 value，假如名称为 value 的话，编译器会自动帮我们赋值，所以我们要稍微改下：

```java
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.SOURCE)
public @interface MyBindView {
    int value();
}
```

```java
@MyBindView(R.id.tv_content)
TextView tvContent;
```

## 注解解析

ButterKnife 是通过以下代码开始解析的：

```java
ButterKnife.bind(this);
```

我们也新建个类，要在 bind() 方法里面写控件的绑定代码：

- 获取该 Activity 的全部成员变量
- 判断这个成员变量是否被 MyBindView 进行注解
- 注解符合的话，就对该成员变量进行 findViewById 赋值

```java
public class MyButterKnife {
    public static void bind(Activity activity){
        //获取该 Activity 的全部成员变量
        for (Field field : activity.getClass().getDeclaredFields()) {
            //判断这个成员变量是否被 MyBindView 进行注解
            MyBindView myBindView = field.getAnnotation(MyBindView.class);
            if (myBindView != null) {
                try {
                    //注解符合的话，就对该成员变量进行 findViewById 赋值
                    //相当于 field = activity.findViewById(myBindView.value())
                    field.set(activity, activity.findViewById(myBindView.value()));
                } catch (IllegalAccessException e) {
                    e.printStackTrace();
                }
            }
        }
    }
}
```

上面的功能实现上是没有问题的，但是，每个控件的绑定都要依靠反射，这太耗性能，一个还好，但是正常的 Activity 都不止一个 View，随着 View 的增加，执行的时间越长，所以，我们必须寻找新的出路，那就是`AnnotationProcessor`。

最不消耗性能的方式，不就是直接使用 findViewById 去绑定 View 吗？既然如此，那么有没有什么方式能够做到在编译阶段就生成好 findViewById 这些代码，到时使用时，直接调用就行了。

我们要创建新类，这个类名有固定的形式，那就是`原Activity名字+Binding`：

模拟自动生成的文件：

```java
public class MainActivityBinding {
    public MainActivityBinding(MainActivity activity) {
        activity.tvContent = activity.findViewById(R.id.tv_content);
    }
}
```

修改后的 MyButterKnife：

```java
public class MyButterKnife {
    public static void bind(Activity activity) {
        try {
            //获取"当前的activity类名+Binding"的class对象
            Class bindingClass = Class.forName(activity.getClass().getCanonicalName() + "Binding");
            //获取class对象的构造方法，该构造方法的参数为当前的activity对象
            Constructor constructor = bindingClass.getDeclaredConstructor(activity.getClass());
            //调用构造方法
            constructor.newInstance(activity);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

所以，我们现在只剩下一个问题了，那就是如何动态生成 MainActivityBinding 这个类了。

这时，就是真正需要用到 AnnotationProcessor。

AnnotationProcessor 是一种处理注解的工具，能够在源代码中查找出注解，并且根据注解自动生成代码。

## AnnotationProcessor使用

新建一个 module。

Android Studio --> File --> New Module --> Java or Kotlin Library --> Next --> Finish 。

在新建的module中创建`MyBindingProcessor` 继承 AbstractProcessor，

- `process()`：里面存放着自动生成代码的代码。
- `getSupportedAnnotationTypes()`：返回所支持注解类型。

```java
@AutoService(Processor.class)
public class MyBindingProcessor extends AbstractProcessor {
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
      //测试输出
        System.out.println("配置成功！");  
        return false;
    }
    @Override
    public Set<String> getSupportedAnnotationTypes() {
        return super.getSupportedAnnotationTypes();
    }
}
```

注意该模块添加`auto-service`依赖，可自动配置执行，省去传统配置META-INF文件夹的麻烦：

```
    implementation 'com.squareup:javapoet:1.12.1'
    // AS 4.3.1 ->  4.0.1 没有问题
    // As-3.4.1  +  gradle-5.1.1-all + auto-service:1.0-rc4
    compileOnly 'com.google.auto.service:auto-service:1.0-rc4'
    annotationProcessor 'com.google.auto.service:auto-service:1.0-rc4'
```

到这一步，配置好各个module依赖，打开 Terminal，输入`./gradlew :app:compileDebugJava`，查看输出有没有`配置成功！`，有的话，就是配置成功了！

接下来就是在`MyBindingProcessor`的process方法里添加具体的生成代码了：

```java
    @Override
    public boolean process(Set<? extends TypeElement> set, RoundEnvironment roundEnvironment) {
        //获取全部的类
        for (Element element : roundEnvironment.getRootElements()) {
            //获取类的包名
            String packageStr = element.getEnclosingElement().toString();
            //获取类的名字
            String classStr = element.getSimpleName().toString();
            //构建新的类的名字：原类名 + Binding
            ClassName className = ClassName.get(packageStr, classStr + "Binding");
            //构建新的类的构造方法
            MethodSpec.Builder constructorBuilder = MethodSpec.constructorBuilder()
                    .addModifiers(Modifier.PUBLIC)
                    .addParameter(ClassName.get(packageStr, classStr), "activity");
            //判断是否要生成新的类，假如该类里面 MyBindView 注解，那么就不需要新生成
            boolean hasBuild = false;
            //获取类的元素，例如类的成员变量、方法、内部类等
            for (Element enclosedElement : element.getEnclosedElements()) {
                //仅获取成员变量
                if (enclosedElement.getKind() == ElementKind.FIELD) {
                    //判断是否被 MyBindView 注解
                    MyBindView bindView = enclosedElement.getAnnotation(MyBindView.class);
                    if (bindView != null) {
                        //设置需要生成类
                        hasBuild = true;
                        //在构造方法中加入代码
                        constructorBuilder.addStatement("activity.$N = activity.findViewById($L)",
                                enclosedElement.getSimpleName(), bindView.value());
                    }
                }
            }
            //是否需要生成
            if (hasBuild) {
                try {
                    //构建新的类
                    TypeSpec builtClass = TypeSpec.classBuilder(className)
                            .addModifiers(Modifier.PUBLIC)
                            .addMethod(constructorBuilder.build())
                            .build();
                    //生成 Java 文件
                    JavaFile.builder(packageStr, builtClass)
                            .build().writeTo(filer);
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
        return false;
    }
```

运行看看 build 目录有没有文件生成：

<img src="https://img-blog.csdnimg.cn/90627a22a3084129a45f948bb6e9fdd2.png" style="zoom:50%;" />


