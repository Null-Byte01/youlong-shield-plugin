# shieldlib 插件集成与调用指南

> 版本 0.2.0 ｜ 对应产物：`shieldlib-plugin.aar`（核心）+ `shieldlib-plugin-aar-bundle.zip`（含特权内核全套）

## 0. 先分清三个东西

| 东西 | 是什么 | 什么时候用 |
|---|---|---|
| `youlong-shield-app-launcher-debug.apk` | 独立 APP（桌面图标已开） | 直接装手机用 |
| `shieldlib-plugin.aar` | **插件本体**：无界面的能力库（自保护/风险扫描/资源加解密） | 塞进你自己的 App |
| `shieldlib-plugin-aar-bundle.zip` | 插件 + 免 root 特权内核全套 aar | 要"免 root 执行 shell 命令"才需要 |

---

## 1. 快速接入（3 步，自保护 + 风险扫描 + 资源加解密）

**第 1 步**：把 `shieldlib-plugin.aar` 复制到宿主工程的 `app/libs/` 目录（没有 libs 就新建）。

**第 2 步**：宿主的 `app/build.gradle` 里加：

```groovy
dependencies {
    implementation files('libs/shieldlib-plugin.aar')
    implementation 'androidx.core:core:1.17.0'
    implementation 'androidx.documentfile:documentfile:1.0.1'
}
```

**第 3 步**：在你自己的 Application 里初始化，然后到处调用：

```java
public class MyApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
        com.youlong.shield.ShieldKit.init(this);   // 必须调一次
    }
}
```

---

## 2. API 速查表（全部通过 ShieldKit 调用）

| 方法 | 作用 | 线程要求 |
|---|---|---|
| `ShieldKit.init(context)` | 初始化（必须在 Application 调一次） | 主线程 |
| `ShieldKit.isNativeReady()` | native 加解密层是否加载成功 | 任意 |
| `ShieldKit.isTampered()` | 是否被 Frida/Xposed 注入 | 任意 |
| `ShieldKit.isTraced()` | 是否被调试器跟踪（误报率高，别据此崩溃） | 任意 |
| `ShieldKit.hasSignatureBinding()` | 签名绑定是否生效 | 任意 |
| `ShieldKit.isValidPackageName(pkg)` | 校验包名合法（防 shell 命令注入） | 任意 |
| `ShieldKit.riskDbSize(ctx)` | 本地病毒库条目数（开源版恒为 0） | 任意 |
| `ShieldKit.startSentinel(mainPid, guardPid, dir)` | 启动双进程守护哨兵 | 任意 |
| `ShieldKit.stopSentinel()` | 停止哨兵 | 任意 |
| `ShieldKit.decryptAsset(bytes)` | AES-256-GCM 解密内置资源 | 任意 |
| `ShieldKit.privilegeAvailable()` | 特权服务是否在线 | 任意（内部已防卡死） |
| `ShieldKit.hasPrivilege()` | 是否已获特权授权 | 任意 |
| **`ShieldKit.runPrivileged(cmd, timeoutMs)`** | **免 root 执行 shell 命令，返回输出** | **必须后台线程！会阻塞** |
| `ShieldKit.requestPrivilegeReconnect()` | 请求特权服务重投 Binder | 任意 |

调用示例：

```java
// 自保护
if (ShieldKit.isTampered()) { /* 被注入了，自行决定策略 */ }

// 风险扫描
boolean ok = ShieldKit.isValidPackageName("com.tencent.mm");  // true
boolean bad = ShieldKit.isValidPackageName("com.x; rm -rf");  // false

// 免 root 特权（必须后台线程！）
new Thread(() -> {
    String out = ShieldKit.runPrivileged("pm list packages -3", 15000L);
    // out = 第三方应用包名列表
}).start();
```

---

## 3. 完整版：接入"免 root 特权"（要跑 shell 命令才需要）

特权功能依赖 Stellar 内核（9 个 aar），核心 aar 里不带。步骤：

**第 1 步**：解压 `shieldlib-plugin-aar-bundle.zip`，把里面 **全部 10 个 aar** 复制到 `app/libs/`：
（shieldlib-release、api、aidl、shared、provider、server、userservice、shizuku-aidl、shizuku-api、manager）

**第 2 步**：`app/build.gradle` 依赖改成文件树方式 + 补齐内核需要的 maven 依赖：

```groovy
android {
    // ⚠️ 硬性要求 1：特权服务从宿主 APK 直接 mmap dex，dex 不能压缩，否则服务进程被系统杀
    aaptOptions { noCompress 'dex' }
    packagingOptions {
        resources { excludes += ['META-INF/versions/9/OSGI-INF/MANIFEST.MF'] }
    }
    // ⚠️ 硬性要求 2：applicationId 必须是 com.youlong.hd
    //    （Stellar 内核按包名识别"管理器"，改包名特权授权会失败）
    defaultConfig {
        applicationId "com.youlong.hd"
    }
    compileOptions {
        sourceCompatibility JavaVersion.VERSION_21
        targetCompatibility JavaVersion.VERSION_21
    }
    kotlinOptions { jvmTarget = '21' }   // 宿主需启用 Kotlin 插件
}

dependencies {
    implementation fileTree(dir: 'libs', include: ['*.aar'])
    compileOnly 'dev.rikka.hidden:stub:4.4.0'
    implementation 'dev.rikka.hidden:compat:4.4.0'
    implementation 'androidx.appcompat:appcompat:1.7.1'
    implementation 'com.google.android.material:material:1.13.0'
    implementation 'androidx.core:core-ktx:1.17.0'
    implementation 'org.lsposed.hiddenapibypass:hiddenapibypass:6.1'
    implementation 'org.conscrypt:conscrypt-android:2.5.2'
    implementation 'io.github.vvb2060.ndk:boringssl:20250114'
    implementation 'org.lsposed.libcxx:libcxx:27.0.12077973'
    implementation 'com.github.topjohnwu.libsu:core:6.0.0'
    implementation platform('androidx.compose:compose-bom:2026.01.01')
    implementation 'androidx.compose.ui:ui'
    implementation 'androidx.compose.foundation:foundation'
    implementation 'androidx.compose.material3:material3'
    implementation 'androidx.activity:activity-compose:1.12.3'
    implementation 'androidx.navigation:navigation-compose:2.9.7'
    implementation 'androidx.lifecycle:lifecycle-viewmodel-compose:2.10.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.10.2'
    implementation 'com.google.code.gson:gson:2.13.1'
    implementation 'androidx.room:room-runtime:2.7.1'
    implementation 'androidx.work:work-runtime-ktx:2.10.0'
    implementation 'androidx.multidex:multidex:2.0.1'
}
```

**第 3 步**：`settings.gradle` 的 repositories 里加 jitpack（libsu 需要）：

```groovy
maven { url 'https://jitpack.io' }
```

**第 4 步**：激活特权。激活界面没进 aar（它在 APP 版里），宿主直接拉起内置 Stellar 管理器完成"无线调试配对"：

```java
Intent i = new Intent();
i.setClassName(getPackageName(), "roro.stellar.manager.MainActivity");
startActivity(i);
// 在打开的界面里完成无线调试配对，配对成功后 ShieldKit.hasPrivilege() 变 true
```

之后就能 `runPrivileged()` 了。注意服务端是独立进程，App 被杀后 Binder 会失效，调 `requestPrivilegeReconnect()` 可让它重投。

---

## 4. 常见坑

1. **忘了 `ShieldKit.init()`** → `runPrivileged` 重连广播发不出去（不会崩，但功能弱）。
2. **主线程调 `runPrivileged`** → 返回 `ERROR:特权服务不可用（主线程不做等待）`，不是 bug，是防卡死保护。
3. **改了 applicationId** → 特权授权永远失败（内核按包名认管理器）。
4. **漏了 `noCompress 'dex'`** → 特权服务进程被系统 SIGKILL，表现为"永远未连接"。
5. **混淆**：aar 自带 `proguard.txt`（keep 了 JNI 类），宿主开混淆不用额外配置。

---

## 5. 最省心方案（如果不折腾 aar）

直接把私有源码仓库（不公开）整个当你的工程底座：`shieldlib/` 已经在里面，你的业务代码往 `app/` 里加，特权、依赖、配置全是现成的——这是源码级集成，一次配好永远不踩坑。
