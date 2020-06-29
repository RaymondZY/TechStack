# Modular

##  Android Gradle Plugin基础

### config.gradle

在项目级文件夹下创建`config.gradle`文件控制全局的变量。

```java
ext {

    // 定义是否以模块化方式构建项目
    isModular = true

    // 统一设置android类下的参数
    androidConfig = [
            compileSdkVersion  : 29,
            buildToolsVersion  : "29.0.3",
            minSdkVersion      : 16,
            targetSdkVersion   : 29,
            // 为不同的module定义不同的applicationId
            applicationId      : [
                    app     : "zhaoyun.teckstack.android.modular",
                    order   : "zhaoyun.teckstack.android.modular.order",
                    personal: "zhaoyun.teckstack.android.modular.personal"
            ],
            versionCode        : 1,
            versionName        : "1.0",
            sourceCompatibility: 1.8,
            targetCompatibility: 1.8
    ]

    // 统一设置库的依赖
    libraries = [
            "androidx.appcompat:appcompat:1.1.0",
            "androidx.constraintlayout:constraintlayout:1.1.3"
    ]

    testLibraries = [
            "junit:junit:4.13"
    ]

    androidTestLibraries = [
            "androidx.test.ext:junit:1.1.1",
            "androidx.test.espresso:espresso-core:3.2.0"
    ]
}
```

并且在项目级`build.gradle`中引用`config.gradle`。

```java
// 引用config.gradle
apply from: "config.gradle"

buildscript {
    
    repositories {
        google()
        jcenter()
        
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:3.6.3'
    }
}

allprojects {
    repositories {
        google()
        jcenter()
        
    }
}

task clean(type: Delete) {
    delete rootProject.buildDir
}
```

### app/build.gradle

```java
apply plugin: 'com.android.application'

def androidConfig = rootProject.ext.androidConfig
def libraries = rootProject.ext.libraries
def testLibraries = rootProject.ext.testLibraries
def androidTestLibraries = rootProject.ext.androidTestLibraries

android {
    compileSdkVersion androidConfig.compileSdkVersion
    buildToolsVersion androidConfig.buildToolsVersion

    defaultConfig {
        applicationId androidConfig.applicationId.app
        minSdkVersion androidConfig.minSdkVersion
        targetSdkVersion androidConfig.targetSdkVersion
        versionCode androidConfig.versionCode
        versionName androidConfig.versionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        // 向注解处理器发送option参数
        // 约束注解声明的路由路径与module名称一致
        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [module: project.name]
            }
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility = androidConfig.sourceCompatibility
        targetCompatibility = androidConfig.targetCompatibility
    }

    // 约束资源名称的前缀，避免合并打包时资源名冲突
    resourcePrefix "${project.name}_"
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 以更简便的方式添加依赖
    libraries.each { library -> implementation library }
    testLibraries.each { library -> testImplementation library }
    androidTestLibraries.each { library -> androidTestImplementation library }

    // 基础库依赖
    implementation project(":common")
    // 注解依赖
    implementation project(':router-annotation')
    // 注解处理器依赖
    annotationProcessor project(':router-processor')

    // 注解依赖
    if (!isModular) {
        implementation project(":order")
        implementation project(":personal")
    }
}
```

### module/build.gradle

```java
//noinspection GroovyConditionalWithIdenticalBranches

// 模块化构建时使用com.android.application，以便作为单独App运行
// 统一构建时使用com.android.library，以便作为android library合并打包
apply plugin: isModular ? 'com.android.application' : 'com.android.library'

def androidConfig = rootProject.ext.androidConfig
def libraries = rootProject.ext.libraries
def testLibraries = rootProject.ext.testLibraries
def androidTestLibraries = rootProject.ext.androidTestLibraries

android {
    compileSdkVersion androidConfig.compileSdkVersion
    buildToolsVersion androidConfig.buildToolsVersion

    defaultConfig {
        applicationId isModular ? androidConfig.applicationId.order : null
        minSdkVersion androidConfig.minSdkVersion
        targetSdkVersion androidConfig.targetSdkVersion
        versionCode androidConfig.versionCode
        versionName androidConfig.versionName

        testInstrumentationRunner "androidx.test.runner.AndroidJUnitRunner"

        javaCompileOptions {
            annotationProcessorOptions {
                arguments = [module: project.name]
            }
        }
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility = androidConfig.sourceCompatibility
        targetCompatibility = androidConfig.targetCompatibility
    }

    sourceSets {
        main {
            if (isModular) {
                // 模块化构建时指定一份单独的AndroidManifest，以便作为单独App运行
                manifest.srcFile "src/main/debug/AndroidManifest.xml"
            } else {
                // 统一构建时使用原始的AndroidManifest
                manifest.srcFile "src/main/AndroidManifest.xml"
                // 统一构建时删除模块化测试用的代码
                java {
                	exclude "**/debug/**"
            	}
            }
        }
    }
    
    // 约束资源名称的前缀，避免合并打包时资源名冲突
    resourcePrefix "${project.name}_"
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])

    // 以更简便的方式添加依赖
    libraries.each { library -> implementation library }
    testLibraries.each { library -> testImplementation library }
    androidTestLibraries.each { library -> androidTestImplementation library }

    // 基础库依赖
    implementation project(":common")
    // 注解依赖
    implementation project(':router-annotation')
    // 注解处理器依赖
    annotationProcessor project(':router-processor')
}
```



## 路由架构

### 注解

#### Router注解

```java
// 作用于类
@Target(ElementType.TYPE)
// 编译期注解
@Retention(RetentionPolicy.CLASS)
public @interface Router {

    // 定义路由路径
    String path();
}
```

### 注解处理器

#### router-annotation/build.gradle

```java
// 声明为Java library
apply plugin: 'java-library'

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.jar'])
    implementation project(':router-annotation')
    implementation "com.squareup:javapoet:1.13.0"
    // 依赖google库处理注解
    compileOnly "com.google.auto.service:auto-service-annotations:1.0-rc7"
    annotationProcessor "com.google.auto.service:auto-service:1.0-rc7"
}

tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

sourceCompatibility = "8"
targetCompatibility = "8"
```

#### RouterProcessor类

```java
// 定义为注解处理器
@AutoService(Processor.class)
// 支持的Java版本为1.8
@SupportedSourceVersion(SourceVersion.RELEASE_8)
// 支持的注解，传递全路径
@SupportedAnnotationTypes("zhaoyun.teckstack.android.modular.router.annotation.Router")
// 支持的运行参数，从gradle中配置传递
@SupportedOptions("module")
public class RouterProcessor extends AbstractProcessor {

    // 用于处理Element
    private Elements mElementUtils;
    // 用于输出日志
    private Messager mMessager;
    // 用于生成源码文件
    private Filer mFiler;

    // 初始化
    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        mElementUtils = processingEnv.getElementUtils();
        mMessager = processingEnv.getMessager();
        mFiler = processingEnv.getFiler();

        // 获取运行参数
        mModule = processingEnv.getOptions().get("module");
        mMessager.printMessage(Diagnostic.Kind.NOTE, "module = " + mModule);
    }

    // 处理注解
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        // 获取被标注的Element
        Set<? extends Element> elements = roundEnv.getElementsAnnotatedWith(Router.class);
        // 获取包名
        String packageName = mElementUtils.getPackageOf(element).getQualifiedName().toString();
        // 获取类名
        String className = element.getSimpleName().toString();
    }
}
```

