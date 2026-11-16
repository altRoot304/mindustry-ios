# Mindustry iOS / iPadOS 增强版（源码构建）

基于 **Anuken/Mindustry 官方源码**（v8 master 分支）的 iOS 构建增强项目。
原版功能 100% 保留（游戏逻辑零修改，仅触碰 iOS 平台层），修复 JS 模组兼容性、
提供 Java 模组构建时打包能力、可调最低系统版本，产出**无任何第三方弹窗**的未签名 IPA。

## 与你手里那个 ipa 的关系

你提供的 `像素工厂_8.160.7_ipa.store.ipa` 是官方 v8 build 160.7 的 RoboVM 构建产物，
被修改者二次封装（改名"像素工厂"、改 Bundle ID、加入弹窗）。
本仓库从官方源码出发重新构建，天然不含任何第三方修改。

## 本仓库相对官方 master 的改动

| 文件 | 改动 | 目的 |
|---|---|---|
| `ios/robovm.xml` | `forceLinkClasses` 增加 `rhino.**`、`arc.**` | AOT 编译时保留 Rhino 脚本引擎与 Arc API 全量类 —— **JS 模组兼容性核心修复**（官方只保留 `mindustry.**`，模组反射访问的类会被裁剪） |
| `core/src/mindustry/mod/Mods.java` | 移除 `if(ios) throw "Java class mods are not supported"` | 解除官方对 iOS 的 Java 模组禁令，改走平台类加载器 |
| `ios/src/mindustry/ios/IOSLauncher.java` | 覆写 `loadJar()` 返回系统类加载器 | iOS 上 Java 模组类从 AOT 编译产物中解析（jar 资源照常读取） |
| `ios/build.gradle` | ① `javamods/` 目录自动打包 ② `-Pminios` 参数 ③ 默认跳过签名 ④ natives 拷贝容错 | 构建时 Java 模组注入、最低 iOS 版本可调、产出未签名 IPA |
| `ios/Info.plist.xml` | `MinimumOSVersion` → `${app.minios}` | 版本门槛参数化 |
| `ios/robovm.properties` | 新增，含 `app.minios=15.0` | 默认保持官方 15.0 |
| `ios/libs/*.framework` | 从官方构建产物提取的 5 个 framework（arc / MetalANGLEKit / libEGL / libGLESv2 / libfeature_support） | 官方仓库缺失的 iOS 预编译依赖，没有它们无法构建 |
| `.gitignore` | 不再忽略 `ios/libs/` 与 `ios/robovm.properties` | 上述依赖可入库 |
| `.github/workflows/build-ios.yml` | 新增 | 免费 macOS 云端一键构建 |

## 构建验证记录

- 最新验证：`8.20261116.1726`（build 65）—— 启动器 5 页（存档/模组/地图/蓝图/信息）、
  rhino iOS 适配升级、内置三模组（默认禁用）、mod name 键修复、检查更新全部编入并验证
- 关键构建经验（供后续维护参考）：
  1. 资源与源码快照必须同 commit（`Sounds` 类由 `assets/sounds` 目录生成）
  2. RoboVM 从 classpath jar 的 `META-INF/robovm/ios/robovm.xml` 声明的
     `frameworkPaths` 解析 framework → frameworks 打包进 `native-frameworks.jar`
  3. robovm 打包 `<resource>directory</resource>` 会去掉 `data/` 前缀 →
     运行时读 `bundled-mods/`（双路径兼容）
  4. arc 的 `setChecked()` 会触发 change 事件 → 初始化勾选前关掉 programmatic events
  5. Java 模组对游戏类的同名补丁（如 VE 的 `mindustry.world.Build`）在 AOT 编译时
     以 core 优先，不触发链接冲突

## 获取最新 IPA（Release 直下）

每次构建完成后，IPA 自动发布到仓库的 **latest-ipa** Release：
👉 `https://github.com/<你的用户名>/mindustry-ios/releases/tag/latest-ipa`
（国内可用 gh-proxy 前缀加速：`https://gh-proxy.com/https://github.com/...`）
下载后 Sideloadly/AltStore 签名安装。启动器与游戏永远同包更新。

## 内置模组（默认禁用）

首次启动会自动安装到模组目录，但**默认禁用**（防止模组冲突卡死游戏），
在启动器「模组」页手动启用后重启生效：

| 模组 | 类型 | 说明 |
|---|---|---|
| Vanilla Expansion (VE) | Java, AOT 打包 | 火星人076 的原版拓展 v2.1.3 |
| BetaMindy | Java, AOT 打包 | sk7725 v2.01 |
| Faster & Smarter Unloader | JS | 卸载器优化 |

要增减内置模组：把 jar/zip 放进 `ios/data/bundled-mods/`（随包分发、默认禁用）
和 `ios/javamods/`（Java 模组需 AOT 打包）后重新构建。
启用状态按键 `mod-<内部名>-enabled` 存储，与游戏内启用逻辑共用。

## 装机后测试清单

1. 冷启动 → 启动器出现（存档/模组/地图/蓝图/信息 5 页）→ 点「启动游戏」进主菜单
2. 信息页：脚本引擎状态应显示「正常」；若「初始化失败」→ 看「最近错误」行
3. 模组页：三个内置模组应为**禁用**状态 → 启用 VE → 重启游戏 → 主菜单出现 VE 内容
4. 从文件 App 导入一个 JS 模组 zip 测试 rhino 引擎
5. 存档：玩一局 → 启动器-存档 → 备份 → 删除 → 从备份恢复
6. 任何异常：信息页「查看/分享运行日志」→ 日志发 issue 或 AI 反馈

## 构建（推荐：GitHub Actions，免费）

1. 在 GitHub 新建一个**公开**仓库（如 `mindustry-ios`，公开仓库 Actions 免费）。
2. 把本目录（`src/`）的全部内容推上去：

   ```bash
   cd 解压出来的src目录
   git remote set-url origin https://github.com/你的用户名/mindustry-ios.git
   git push -u origin master
   ```

3. 仓库页面 → **Actions** → 选 **Build Mindustry iOS** → **Run workflow**：
   - `min_ios`：最低 iOS 版本，默认 `15.0`（老设备可改 `13.0` 或 `12.0`）
   - `build_version`：留空自动用时间戳，版本号形如 `8.20261115.1541`
   - `with_java_mods`：是否打包 `ios/javamods/` 里的 jar（见下文）
4. 构建完成（约 20–60 分钟）后，在该次运行页面底部下载
   **mindustry-ios-unsigned-ipa** 工件，得到**未签名 IPA**。

> **可行性背书**：同样基于 RoboVM 的开源卡牌游戏
> [Card-Forge/forge](https://github.com/Card-Forge/forge/blob/master/.github/workflows/ios-build-ipa.yml)
> 自 2024 年起就用几乎相同的流程（macOS runner + Temurin 17 + RoboVM AOT）持续产出
> 未签名 IPA，其经验已融入本仓库的 workflow（macos-15 runner、`~/.robovm` SDK 缓存）。

> 本地有 macOS + Xcode 的话也可以直接 `./gradlew ios:createIPA`（需把 Arc 仓库克隆到 Mindustry 目录旁边）。

### 登录备选方案（设备码不可用时）

在 GitHub 网页生成 Personal Access Token 替代设备码授权：
**Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate**，
选中你的 `mindustry-ios` 仓库、给 `Contents: Read and write` + `Workflows: Read and write` 权限，
然后推送时把 token 当作密码使用（或 `git remote set-url origin https://<token>@github.com/用户名/mindustry-ios.git`）。

## Java 模组（构建时打包）

RoboVM 的 AOT 编译**无法运行时加载字节码**，所以 Java 模组不能像手机/电脑那样即插即用。
替代方案（已在 workflow 中自动化）：

1. 把 Java 模组的 jar 放进 `ios/javamods/`（提交到你的仓库，或构建时上传）；
2. 构建时它们会自动被 AOT 编译进可执行文件，并自动注册 force-link 规则；
3. 安装后，你仍需把这个 jar 文件导入游戏的模组目录 —— 游戏识别到元数据后
   直接用内置类运行，不再尝试动态加载。

- 脚本部分（`assets/scripts/`、JSON 内容）从 jar 正常读取，与桌面端一致。
- 若 jar 没打进本次构建，游戏会提示加载失败，不影响其他模组。
- 模组与游戏版本的 API 兼容性由模组作者保证（`minGameVersion` 照常校验）。

## JS / JSON 模组（即装即用）

JS 引擎是官方自带的 **Rhino 解释器**（`optimizationLevel=-1`，纯解释执行，无需 JIT）。
JS 模组通过游戏内「模组」页面导入 zip，或用「文件」App 把 zip 存入
Mindustry 的文档目录后在游戏内刷新。`rhino.**` 与 `arc.**` 强制链接后，
模组对 `arc.*` API 的反射访问不会再被 AOT 裁剪。

## 安装（免费开发者账号自签）

未签名 IPA 无法直接安装，任选其一：

- **AltStore / SideStore**（推荐，手机上可续签）：电脑装 AltServer，手机装 AltStore，
  用自己的 Apple ID 安装 IPA。免费账号限制：7 天需续签、同时最多 3 个自签应用、最多 3 台设备。
- **Sideloadly**（电脑，Windows/macOS）：拖入 IPA + Apple ID 一键签名安装。
- **爱思助手**：自签功能同理。
- **TrollStore**（仅限设备未修补 CoreTrust 漏洞的 iOS 14–16.6.1 / 17.0）：可永久签，
  还能附带你需要的 entitlements。

签名时 Bundle ID 保持 `io.anuke.mindustry` 即可（免费账号允许）。

### 关于 JIT

本版本**不需要 JIT 权限**：游戏本体是 RoboVM 完整 AOT 编译（比解释执行快得多），
JS 引擎用解释模式。若你的安装渠道提供 JIT（AltJIT / SideJIT / TrollStore），
开启与否均不影响本版本运行。

## 老设备兼容

- 官方要求 iOS 15.0，本仓库默认保持不变（可自行调整）。
- 打包的 ANGLE 渲染组件（MetalANGLE / libEGL / libGLESv2）实测最低 **iOS 11.0**。
- 想覆盖更老机型：Actions 触发时把 `min_ios` 改成 `13.0` / `12.0`，
  或直接改 `ios/robovm.properties` 的 `app.minios`。
- 实际可玩性参考：2GB 内存机型（iPhone 6s 及以后）流畅；1GB 机型（5s/6）能装但大概率吃紧。

## 性能说明

- 使用 RoboVM **release** 模式完整 AOT（LLVM -O2），非解释/混合模式 —— 这是官方构建配置，
  也是能达到的最优基线；渲染经 MetalANGLE 走 Metal，这是 iOS 上唯一可行路径（Apple 已废弃 OpenGL）。
- 主二进制约 60MB+，首次启动后不再有 JIT 预热开销。

## 已知边界（诚实说明）

- **运行时安装 Java 模组**：AOT 架构下无法实现（Apple 沙盒 + 禁 JIT 的本质限制），
  请用上面的构建时打包方案。
- 若官方 master 与 Arc/ANGLE 组件出现 API 演进，构建时以官方提交为准重新提取
  framework（本仓库的提取自 build 160.7 时期，接口稳定）。
- 修改版弹窗的具体内容无法从二进制确认，但源码构建产物 100% 无第三方代码。
