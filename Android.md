### gradle安装位置

![image-20260112171408944](Android.assets/image-20260112171408944.png)

### setting.gradle.kts

```
pluginManagement {
    repositories {
        // 阿里云镜像优先，适配Gradle 8.7
        maven("https://maven.aliyun.com/repository/google")
        maven("https://maven.aliyun.com/repository/gradle-plugin")
        maven("https://maven.aliyun.com/repository/public")
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
    repositories {
        maven("https://maven.aliyun.com/repository/google")
        maven("https://maven.aliyun.com/repository/public")
        mavenCentral()
    }
}

rootProject.name = "CAR"
include(":app")
```

### build.gradile.kts (app)

```
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
}

android {
    // 替换成你项目的实际包名（比如com.xxx.car）
    namespace = "com.example.car"
    compileSdk = 34 // AGP 8.4.0适配的推荐版本（Gradle 8.7要求）

    defaultConfig {
        applicationId = "com.example.car" // 替换成你的应用ID
        minSdk = 21
        targetSdk = 34
        versionCode = 1
        versionName = "1.0"

        testInstrumentationRunner = "androidx.test.runner.AndroidJUnitRunner"
    }

    buildTypes {
        release {
            isMinifyEnabled = false
            proguardFiles(
                getDefaultProguardFile("proguard-android-optimize.txt"),
                "proguard-rules.pro"
            )
        }
    }
    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17 // Gradle 8.7要求Java 17
        targetCompatibility = JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation("androidx.core:core-ktx:1.12.0")
    implementation("androidx.appcompat:appcompat:1.6.1")
    implementation("com.google.android.material:material:1.12.0")
    implementation("androidx.constraintlayout:constraintlayout:2.1.4")

    testImplementation("junit:junit:4.13.2")
    androidTestImplementation("androidx.test.ext:junit:1.1.5")
    androidTestImplementation("androidx.test.espresso:espresso-core:3.5.1")
}
```

### build.gradile.kts (project)

```
// Gradle 8.7推荐用plugins块声明插件（替代旧的buildscript），无版本冲突
plugins {
    id("com.android.application") version "8.4.0" apply false
    id("org.jetbrains.kotlin.android") version "1.9.0" apply false
}

// 修复buildDir弃用，适配Gradle 8.x
tasks.register("clean", Delete::class) {
    delete(layout.buildDirectory.get().asFile)
}
```