# YouLong Shield Plugin（游龙护盾 · 插件分发）

> 本仓库只包含**编译产物**（aar + 文档），不含任何源码。源码仓库保持私密。
> 产物由 GitHub Actions 从源码自动构建（commit `49ef264`）。

## 文件清单

| 文件 | 用途 |
|---|---|
| `shieldlib-plugin.aar` | 插件核心：`ShieldKit` 统一 API + 12 个能力类 + 双 ABI native so |
| `shieldlib-plugin-aar-bundle.zip` | 完整包：核心 aar + 免 root 特权内核 9 个 aar（api/server/manager 等） |
| `INTEGRATION.md` | 完整集成与调用指南（含坑位说明） |

## Release 直链（匿名可直接下载）

- 核心 aar：
  https://github.com/Null-Byte01/youlong-shield-plugin/releases/download/v0.2.0/shieldlib-plugin.aar
- 完整包（含特权内核）：
  https://github.com/Null-Byte01/youlong-shield-plugin/releases/download/v0.2.0/shieldlib-plugin-aar-bundle.zip
- 本 README 原文：
  https://raw.githubusercontent.com/Null-Byte01/youlong-shield-plugin/main/README.md
- 完整指南原文：
  https://raw.githubusercontent.com/Null-Byte01/youlong-shield-plugin/main/INTEGRATION.md

## 给 AI 的集成指令（把下面整段复制给任何 AI 助手即可）

请帮我把 YouLong Shield 安全插件集成进我当前的 Android 工程，步骤如下：

1. 从这个直链下载插件（无需登录）：
   https://github.com/Null-Byte01/youlong-shield-plugin/releases/download/v0.2.0/shieldlib-plugin.aar
2. 把下载的 `shieldlib-plugin.aar` 放进我的 app 模块 `libs/` 目录。
3. 在 `app/build.gradle` 的 dependencies 里添加：
   implementation files('libs/shieldlib-plugin.aar')
   implementation 'androidx.core:core:1.17.0'
   implementation 'androidx.documentfile:documentfile:1.0.1'
4. 在我的 Application 类 onCreate() 里调用一次：
   com.youlong.shield.ShieldKit.init(this);
5. 之后即可在任意位置调用安全能力：
   - ShieldKit.isTampered()               是否被 Frida/Xposed 注入
   - ShieldKit.isTraced()                 是否被调试器跟踪（误报率高，勿据此崩溃）
   - ShieldKit.hasSignatureBinding()      签名绑定是否生效
   - ShieldKit.isValidPackageName(pkg)    包名合法性校验（防 shell 注入）
   - ShieldKit.startSentinel(mainPid, guardPid, dir) / stopSentinel()   双进程守护
   - ShieldKit.decryptAsset(bytes)        AES-256-GCM 资源解密
   注意：以上全部不需要网络权限。

如需免 root 特权（shell/uid 2000 执行命令）：
- 还要下载完整包：
  https://github.com/Null-Byte01/youlong-shield-plugin/releases/download/v0.2.0/shieldlib-plugin-aar-bundle.zip
- 解压后把里面全部 11 个 aar 一并放进 libs/，依赖改为：
  implementation fileTree(dir: 'libs', include: ['*.aar'])
- 完整步骤与 maven 依赖清单、硬性要求，见：
  https://raw.githubusercontent.com/Null-Byte01/youlong-shield-plugin/main/INTEGRATION.md
- 关键硬性要求：宿主 applicationId 必须是 com.youlong.hd；宿主需
  aaptOptions { noCompress 'dex' }；激活界面通过显式 Intent 拉起
  roro.stellar.manager.MainActivity 完成无线调试配对。

重要约束：
- ShieldKit.init 必须在 Application.onCreate 调用一次（重复调用安全）。
- ShieldKit.runPrivileged 会阻塞，只能在后台线程调用，失败返回 "ERROR:..."。
- 我的工程如果开启了混淆，无需额外 keep 规则（aar 自带 consumer proguard）。

## 快速接入（3 步，人工操作参考）

```groovy
// 1. app/build.gradle
dependencies {
    implementation files('libs/shieldlib-plugin.aar')
    implementation 'androidx.core:core:1.17.0'
    implementation 'androidx.documentfile:documentfile:1.0.1'
}
```

```java
// 2. Application.onCreate
com.youlong.shield.ShieldKit.init(this);
```

```java
// 3. 随处调用
boolean tampered = ShieldKit.isTampered();
boolean ok = ShieldKit.isValidPackageName("com.example.app");
```

## 说明与许可

- 基于 AGPL-3.0 开源项目 youlong-security（github.com/iill392/youlong-security）的二改产物，仅供个人使用，不分发、不商用。
- 免 root 特权由内嵌的 Stellar 内核提供（MPL-2.0 / Apache-2.0）。
- 本分发仓库不含源码；如需源码请自行按 AGPL-3.0 条款向上游获取。
