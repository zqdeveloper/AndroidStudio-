Gradle是⼀一种基于Groovy语法的项⽬目构建⼯工具，其运⾏行行在JVM上，借鉴了了脚本语⾔言诸多特性，兼容Java，可以直接使⽤用Java各种类库。下⾯面我们来看⼀一些Gradle相关实⽤用技巧。

* Gradle Task。Task（任务）是Gradle中的一个核⼼心概念，每一个声明的任务都可以看作是⼀个任务对象，可以拥有⾃自⼰己的属性和⽅方法（默认类型是DefaultTask）,同Java中java.lang.Object类似。在任务之间可以相互依赖，使⽤用关键字dependsOn,还可以通过doFirst(closure)和doLast(closure)等在任务执⾏行行⽣生命周期中插⼊入具体业务逻辑，常⻅见的任务类型有⽤用于复制的Copy、⽤用于打包的Jar、⽤用于执⾏行行JavaExec等，最常⻅见的Task如下
    * gradle assemble,⽣生成所有渠道的Debug和Release包。
    * gradle assembleAndroidTest,⽣生成所有渠道的测试包。
    * gradle assembleDebug,⽣生成所有渠道的Debug包。
    * gradle assembleRelease,⽣生成所有渠道的Release包。
    * gradle assemble XXX，⽣生成某个渠道的Debug和Release包。
* gradle加速
    * 常规设置。如开启Gradle daemon进程等（gradle.properties文件，建议使用全局配置），代码如下。
        ~~~ 
        开启gradle守护进程
        org.gradle.daemon=true
        #jvm内存
        org.gralde.jvmargs=-Xmx2048m -XX:MaxPerSize=512m -XX:+HeapDumpOnOutOfMemoryError
        #并行项目执行（多module依赖复杂慎用）
        org.gradle.parallel=true
        org.gradle.configureondemand=true
        ~~~
    * 开启增量编译，在对应的module的build.gradle文件中，按如下设置
        ~~~
        dexOptions{
            incremental true
        }
        ~~~
    * 屏蔽不需要的Task。屏蔽不需要的Task或特定的Task，按如下设置
    ~~~
    //屏蔽系统Task
    tasks.whenTaskAdded {task->
        if (task.name.contains("lint")//跳过lint检查
        ||task.name.equals("clean")//如果instant run不生效，把clean这行去掉
        ||task.name.contains("Aidl")//如果项目中有用到aidl，则不可以舍弃这个任务
        ||task.name.contains("mockableAndroidJar")//用不到测试的时候，就可以先关闭
        ||task.name.contains("UnitTest")
        ||task.name.contains("AndroidTest")
        ||task.name.contains("Ndk")//用不到Ndk和JNI的也关闭掉
        ||task.name.contains("Jni")
        )
            task.enabled=false
    }
    ~~~
    * 代理设置。在根目录的gradle.properties中配置，代码如下。
    ~~~
    systemPro.http.proxyHost=127.0.0.1
    systemPro.http.proxyPort=1010
    systemPro.https.proxyHost=127.0.0.1
    systemPro.https.proxyPort=1010
    ~~~
* Gradle加速的实用建议
    * 通过productFlavors设置build variant,针对不同的product保留对应的配置信息，加速构建，类似多渠道打包。
    * 避免编译不必要的资源。如dev包通过设置resConfigs "en" "xxhdpi",只实用英文string资源和xxhdpi的屏幕密度资源，代码如下：
    ~~~
    productFlavors{
        dev{
            resConfigs "en","xxhdpi"
        }
    }
    ~~~
    * 配置debug构建的Crushlytics为Disable(Crushlytics 为崩溃上报分析工具，Debug阶段可能并不需要)，如果Debug期间需要开始Crushlytics，那也alwaysUpdateBuild为false,避免每次都更新ID,代码如下：
    ~~~
    buildTypes {
        debug{
            ext.enableCrashlytics=false
            ext.alwaysUpdateBuildId=false
        }
    }
    ~~~
    * 用静态的构建配置来构建你的Debug版，避免在Debug下使用动态配置（如version codes,version names,resources等），类似下面要阐述的版本号/依赖统一管理。
    * 用静态的版本依赖，避免使用+号，代码如下：
    ~~~
    classpath 'com.android.tools.build:gradle:3.+'//动态依赖
    classpath 'com.android.tools.build:gradle:3.4.0'//静态依赖
    ~~~
    * 配置on demand为enable状态，指定Gradle仅能配置你想要构建的Modules。Android Studio路径为：File-->Settings-->Build-->Compiler-->check Configure on demand.
    * 建议使用library模块，模块化代码抽离。
    * 当你的构建消耗时间过长时，如果存在较复杂和独立的构建逻辑，考虑将其创建为独立的Tasks(自定义gradle插件)，按需使用。
    * 配置dexOptions(Android Studio 2.2新增)和开启library pre-dexing(DEX预处理)。两者都是针对DEX构建优化，dexOptions可以配置包裹preDexLibraries、maxProcessCount和javaMaxHeapSize,代码如下
    ~~~
    android {
        dexOptions {
            preDexLibraries true
            maxProcessCount 8
        }
    }
    ~~~
    * 增加Gradle堆大小（开启Dex-in-process）。Dex-in-process默认允许多个DEX进程运行在一个单独的VM中，所以可以通过分配足够的内存来开启这个特性（Android Studio 2.1+）。
    * 将图片转换成Webp格式，不用在构建时做压缩。WebP是一种具备JPEG类似的有损压缩和PNG的透明支持的高压缩质量的图片格式，同时可以减少包Size。
    * 禁止使用PNG crunching。也是一种禁止构建时默认压缩图片的方法。
    ~~~
    aaptOptions{
        cruncherEnabled false
    }
    ~~~
    * 使用Instant Run
    * 使用构建缓存，Android Gradle插件2.3.0+默认开启来构建缓存。
    * 避免使用注解处理器，使用注解处理器时将导致增量构建不可用。
    * Profile your buil。这条主要针对那些超级App(拥有大量自定义构建逻辑等)，需要知晓每个阶段/每个Task的时间消耗来优化那些耗时逻辑，build profile的生成通过在Android Studio的命令行中操作（View——>Tool Windows——>Terminal），具体如下。
        * 清除：gradlew clean(Windows)或(./gradlew clean(Mac))
        * 构建：gradlew——>profile——>recompile-scipts——>offline——>return-tasks assembleFlavorDebug(其中:profile表示开启profiling;offline表示禁止gradle获取离线依赖，防止gradle更新数据影响报告;return-tasks表示强制Gradle返回所有Task并忽略任何Task的优化；recopile-scripts表示强制脚本重新编译跳过cache)
        * 查看：找到project-root/build/reports/profile/目录下的profile_timestamp.html文件，在浏览器中打开即可呈现完整时间消耗的构建报告。
        * 多渠道打包。鉴于国内Android App应用市场的百花齐放，多渠道打包是gradle中讨论最早，用的最多的，其本质是productFlavors的使用，结合占位符与AndroidManifest的使用，可以为不同渠道设置不同包名。
        ~~~
        productFlavors {
            dev {
                resConfigs "en", "xxhdpi"
                applicationIdSuffix ".debug"//不同包名设置，便于线上和开发包安装同一手机
            }
            googleplay {}
            qihu360 {}
            xiaomi {}
            tencent {}
            productFlavors.all { flavor ->
                flavor.manifestPlaceholders = [UMENG_CHANNEL_VALUE: name]
            }
        }
        ~~~
* Gradle通用技巧

    * Log开关控制。定义动态编译生成对象，通过buildConfigFile控制，然后在java代码中通过控制BuildConfig.enableLog来获取，代码如下
        ~~~
        buildTypes {
            debug {
                buildConfigField("boolean","enableLog","true")
            }
            release {
                buildConfigField("boolean","enableLog","true")
            }
        }
        ~~~
    * 版本号/依赖统一管理。建立独立的gradle(config.gradle),然后apply from当前gradle,通过设置project.ext,再通过rootProject.ext进行引用，一下代码为Xknife-Android的global_config.gradle文件的一部分。
        ~~~
        ext {
        isApplication = true
        def retrofitVersion = '2.5.0'
        def rxJava = '2.0.2'
        def supportVersion = '28.0.0'
        def JPUSH = '3.1.8'
        def glide = "4.8.0"
        roomVersion = '1.1.1'
        archLifecycleVersion = '1.1.1'
        guavaVersion = '27.0.1-android'
        android = [
        applicationId : "com.rrc.footballwealth",
        compileSdkVersion: 28,
        minSdkVersion : 16,
        targetSdkVersion : 28,
        versionCode : 4,
        versionName : "1.1.1",
        buildToolsVersion: "28.0.3"
        ]
        dependencies = [
        appcompat_v7 : "com.android.support:appcompat-v7:$supportVersion",
        constraint_layout : "com.android.support.constraint:constraint-layout:1.1.3",
        recyclerview : "com.android.support:recyclerview-v7:$supportVersion",
        retrofit : "com.squareup.retrofit2:retrofit:$retrofitVersion",
        adapter_rxjava : "com.squareup.retrofit2:adapter-rxjava2:$retrofitVersion",
        converter_gson : "com.squareup.retrofit2:converter-gson:$retrofitVersion",
        converter_wire : "com.squareup.retrofit2:converter-wire:$retrofitVersion",
        converter_scalars : "com.squareup.retrofit2:converter-scalars:$retrofitVersion",
        rxjava : "io.reactivex.rxjava2:rxjava:$rxJava",
        design : "com.android.support:design:$supportVersion",
        eventbus : "org.greenrobot:eventbus:3.1.1",
        gson : "com.google.code.gson:gson:2.2.1",
        guava : "com.google.guava:guava:$guavaVersion",
        support_annotations: "com.android.support:support-annotations:$supportVersion",
        ]
        }
        //使用
        applicationId rootProject.ext.android.applicationId
        ~~~
    * 另外，还可以在gradle.properties文件中定义一些统一的编译常量（如定义常量XX=1,然后需在需要的module中通过project.XX引用）

* APK输出名字定制化。定制化APK输出名字，自动加上版本号、时间等信息，避免手动重命名，代码如下：
~~~
 applicationVariants.all { variant ->
    variant.outputs.each{output->
        output.outputFile=new File(
        output.outputFile.parent+"/${variant.buildType.name}","XXX-${variant.buildType.name}-${variant.versionName}-${variant.productFlavors[0].name}.apk".toLowerCase()
        )
    }
}
~~~
* 构建不同的名称、版本好和App Id等,代码如下。
~~~
buildTypes {
 debug {
 applicationIdSuffix ".Debug"
 versionNameSuffix "-debug"
 resValue "string","app_name","XXX(debug)"
 }
 release {
 resValue "string","app_name","XXX"
 }
}
~~~
* 修改默认的Build配置文件名（Settings.gradle文件），代码如下：
~~~
rootProject.buildFileName='XX.gradle'
Java版本设置。在Gradle中设置Java版本，代码如下
compileOptions {
    sourceCompatibility JavaVersion.VERSION_1_8
    targetCompatibility JavaVersion.VERSION_1_8
}
~~~