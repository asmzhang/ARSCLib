# ARSCLib
## Android 二进制资源（Binary Resources）读 / 写 Java 库
该库基于 **AOSP 中 **`**androidfw/ResourceTypes.h**`** 的结构实现**，目标是 **完全替代 aapt / aapt2**。

---

## 功能概览
### 支持读取、写入、修改、创建：
+ 资源表（`resources.arsc`）
+ 二进制 XML 文件
    - `AndroidManifest.xml`
    - 资源 XML（binary xml）

---

## JSON ↔ 二进制 资源转换（适用于已混淆资源）
+ 将资源解码为**可读的 JSON**
+ 将 JSON 格式的资源**重新编码 / 构建为二进制资源**

👉 主要用于 **资源混淆 / 反混淆 / 自动化处理**

---

## XML ↔ 二进制 资源转换（适用于未混淆资源）
+ 将资源解码为**源码级 XML**
+ 将 XML 源码**重新编码 / 构建为二进制资源**

---

## ⚠ 注意事项（非常重要）
### 1️⃣
**将资源解码为 XML 的前提是：**

+ 所有资源名称必须是：
    - 未混淆的
    - 合法的
    - 可读的

否则无法保证正确还原为 XML。

---

### 2️⃣
**该库假设使用者具备扎实的 Android XML 源码知识**

因此在 **编码 / 构建阶段：**

+ ❌ 不会像 aapt / aapt2 那样严格校验 XML 语法
+ ❌ 很多错误不会抛异常
+ ❌ 即使值明显不规范，也可能构建成功

举例：

```plain
<manifest
    package="Wrong 😂 (package) name!" />
```

+ 使用 ARSCLib：**可以成功构建**
+ Android 设备：**可能可以正常安装 / 运行**
+ aapt/aapt2：**直接拒绝**

👉 你需要自己清楚：  
**哪些值 Android 系统能接受，哪些只是工具链不接受。**

---

## 示例应用
下面这个工具就是基于 ARSCLib 开发的：

👉 [https://github.com/REAndroid/APKEditor](https://github.com/REAndroid/APKEditor)

---

## 平台支持
+ Android
+ Linux
+ macOS
+ Windows

**所有支持 Java 的平台均可运行**

---

## 引入方式
### Maven
```plain
repositories {
    mavenCentral()
}

dependencies {
    implementation("io.github.reandroid:ARSCLib:+")
}
```

---

### 本地 Jar
```plain
dependencies {
    implementation(files("$rootProject.projectDir/libs/ARSCLib.jar"))
}
```

---

## 构建 Jar 包
```plain
git clone https://github.com/REAndroid/ARSCLib.git
cd ARSCLib
./gradlew jar
# 构建完成后，Jar 位于：
# ./build/libs/ARSCLib-x.x.x.jar
```

---

## 使用示例
### Java 示例
```plain
import com.reandroid.apk.AndroidFrameworks;
import com.reandroid.apk.ApkModule;
import com.reandroid.apk.FrameworkApk;
import com.reandroid.archive.ByteInputSource;
import com.reandroid.arsc.chunk.PackageBlock;
import com.reandroid.arsc.chunk.TableBlock;
import com.reandroid.arsc.chunk.xml.AndroidManifestBlock;
import com.reandroid.arsc.chunk.xml.ResXmlAttribute;
import com.reandroid.arsc.chunk.xml.ResXmlElement;
import com.reandroid.arsc.coder.EncodeResult;
import com.reandroid.arsc.coder.ValueCoder;
import com.reandroid.arsc.value.Entry;

import java.io.File;
import java.io.IOException;

public class ARSCLibExample {

    public static void createNewApk() throws IOException {

        // 创建 APK 模块
        ApkModule apkModule = new ApkModule();

        // 创建资源表和 Manifest（二进制）
        TableBlock tableBlock = new TableBlock();
        AndroidManifestBlock manifest = new AndroidManifestBlock();

        apkModule.setTableBlock(tableBlock);
        apkModule.setManifest(manifest);

        // 初始化 Android Framework（用于资源 ID / 系统资源）
        FrameworkApk framework = apkModule.initializeAndroidFramework(
                AndroidFrameworks.getLatest().getVersionCode());

        // 创建资源包（packageId = 0x7f）
        PackageBlock packageBlock = tableBlock.newPackage(0x7f, "com.example");

        // drawable/ic_launcher
        Entry appIcon = packageBlock.getOrCreate("", "drawable", "ic_launcher");
        EncodeResult color = ValueCoder.encode("#006400");
        appIcon.setValueAsRaw(color.valueType, color.value);

        // 默认语言字符串
        Entry appNameDefault = packageBlock.getOrCreate("", "string", "app_name");
        appNameDefault.setValueAsString("My Application");

        // 德语
        Entry appNameDe = packageBlock.getOrCreate("-de", "string", "app_name");
        appNameDe.setValueAsString("Meine Bewerbung");

        // 俄语（俄罗斯）
        Entry appNameRu = packageBlock.getOrCreate("-ru-rRU", "string", "app_name");
        appNameRu.setValueAsString("Мое заявление");

        // Manifest 基本信息
        manifest.setPackageName("com.example");
        manifest.setVersionCode(100);
        manifest.setVersionName("1.0.0");
        manifest.setIconResourceId(appIcon.getResourceId());

        manifest.setCompileSdkVersion(framework.getVersionCode());
        manifest.setCompileSdkVersionCodename(framework.getVersionName());
        manifest.setPlatformBuildVersionCode(framework.getVersionCode());
        manifest.setPlatformBuildVersionName(framework.getVersionName());

        // 权限
        manifest.addUsesPermission("android.permission.INTERNET");
        manifest.addUsesPermission("android.permission.READ_EXTERNAL_STORAGE");

        // 设置 Application label（注意：所有语言的 app_name 共享同一个 resourceId）
        manifest.setApplicationLabel(appNameDefault.getResourceId());

        // 创建主 Activity
        ResXmlElement mainActivity =
                manifest.getOrCreateMainActivity("android.app.Activity");

        ResXmlAttribute labelAttribute = mainActivity
                .getOrCreateAndroidAttribute(
                        AndroidManifestBlock.NAME_label,
                        AndroidManifestBlock.ID_label);

        labelAttribute.setValueAsString("Hello World");

        // Android 要求 base APK 至少有一个 dex
        ByteInputSource dummyDex =
                new ByteInputSource(new byte[0], "classes.dex");
        apkModule.add(dummyDex);

        // 输出 APK
        File outFile = new File("test_out.apk");
        apkModule.writeApk(outFile);

        // 后续自行签名并安装
    }
}
```