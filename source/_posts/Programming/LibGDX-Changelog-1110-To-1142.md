---
title: libGDX引擎从1.11.0到1.14.2的变更整理
toc: true
categories:
  - Programming
tags: [Java, libGDX, Changelog]
date: 2026-08-18 20:39:00
updated: 2026-08-18 20:39:00
---

本文整理了 [libGDX](https://libgdx.com/) 游戏开发引擎从 1.11.0 到 1.14.2 的主要变更，内容来源于[官方变更日志](https://libgdx.com/news/changelog/)。

我整理这些内容是因为我的开源项目 [ArkPets](https://github.com/isHarryh/Ark-Pets) 将在近期（2026 年 8 月）针对所使用的 libGDX 版本进行升级。

<!-- more -->

## 变更整理

### 通用收益

核心 API 变更以及对全平台都有收益的改进。

| 版本   | 内容                                                                                                                                              |
| ------ | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.12.0 | 新增 Haptics API：4 个 `Input#vibrate()` 重载                                                                                                     |
| 1.12.0 | 触摸取消事件（锁屏等场景）改为调用 `InputProcessor#touchCancelled` 而非 `touchUp`                                                                 |
| 1.12.0 | 新增 `Texture3D` 支持                                                                                                                             |
| 1.12.0 | `Stage#actorRemoved`：actor 移除前触发 exit 事件                                                                                                  |
| 1.12.0 | 新增 `CheckBox#getImageDrawable`、`I18NBundle#keys()`、`JsonReader#stop()`、`OrientedBoundingBox`                                                 |
| 1.12.0 | `TextureAtlas` 改用 `FileHandle#reader`，外部可控制字符集                                                                                         |
| 1.12.0 | 修复 `Intersector#isPointInTriangle`、`DataInput#readString` 非 ASCII、`MathUtils.lerpAngle` 极端输入；三角函数性能优化；修复 `RenderBuffer` 泄漏 |
| 1.12.0 | Skin 序列化支持 `useIntegerPositions`；`scaledSize` 按 float 处理                                                                                 |
| 1.12.0 | Tiled 地图视差因子支持；六边形地图 stagger index 修复                                                                                             |
| 1.12.0 | 各类 Scene2D 修复与改进                                                                                                                           |
| 1.12.1 | 新增 `Vector4` 类                                                                                                                                 |
| 1.12.1 | `TiledDrawable` 支持 `setAlign` 对齐；`draw` 提供静态方法免实例化                                                                                 |
| 1.12.1 | 修复 `ModelBatch#addMesh` 位运算优先级、ParticleEditor/Flame 崩溃                                                                                 |
| 1.13.0 | `Sprite` / `SpriteBatch` 多项性能优化：缓存 sprite 颜色、GL30 默认模式免更新索引                                                                  |
| 1.13.0 | 帧缓冲多重采样支持（`GL31FrameBufferMultisampleTest` 示例）                                                                                       |
| 1.13.0 | 原生文本输入 API                                                                                                                                  |
| 1.13.0 | `JsonReader` 改进，新增 `JsonSkimmer`、`JsonString`                                                                                               |
| 1.13.0 | `Timer#stop` 记录停止时长、重启后续期；修复已取消任务仍执行                                                                                       |
| 1.13.0 | 修复 GlyphLayout 固定宽度字形偏移、scene2d.ui 小数位置布局                                                                                        |
| 1.13.0 | 新增 `Color#CLEAR_WHITE`、`Color#set(Color, float)`；`OrthographicCamera#update` 精度保护                                                         |
| 1.13.0 | `Screen` 继承 `Disposable`；`ScreenUtils` 抗锯齿标志；粒子发射器小改进                                                                            |
| 1.13.0 | HTTP 请求状态检测；`SelectBox` 弹出不超出 Stage 右缘；`DragListener` 仅响应正确 touch up                                                          |
| 1.13.1 | Tiled 地图的大量改进                                                                                                                              |
| 1.13.1 | 新增 Tiled JSON 地图加载器；`class` MapProperty、Layer TintColor、Image Layer repeat x/y、group 内 imagelayer 解析                                |
| 1.13.1 | `XmlReader#getChildren` / `replaceChild`；`Array#replaceFirst` / `replaceAll`                                                                     |
| 1.13.1 | 修复 `ShortArray#lastIndexOf`（char→short）、`LongArray#lastIndexOf`（char→long）                                                                 |
| 1.13.1 | `LongArray#pop`/`peek` 空数组时抛异常；多边形交集去重顶点并重置变换                                                                               |
| 1.13.1 | `FPSLogger#setBound`；Hiero 字体名校验；depth shader 回收修复                                                                                     |
| 1.13.5 | Tiled "Text Objects" 支持（`TextMapObject`）                                                                                                      |
| 1.13.5 | 全系列集合新增 `replaceFirst` / `replaceAll`（Boolean/Byte/Char/Float/Int/Long/Short/Array/DelayedRemoval/Snapshot）                              |
| 1.13.5 | `Drawable` 默认接口方法；`BitmapFontCache#getPageCount`；`TextField#drawBackground`                                                               |
| 1.13.5 | Box2D 原生 `ContactFilter`（`World#setContactFilter(null)`）性能提升                                                                              |
| 1.13.5 | 修复 `Container#setCullingArea`                                                                                                                   |
| 1.14.0 | 新增 `PoolManager`（免反射对象池，需预注册类）                                                                                                    |
| 1.14.0 | 通用 Tiled 地图加载器（Universal Tiled Map Loader）；Tiled 类与模板对象支持                                                                       |
| 1.14.0 | `JsonValue#toJson(Writer)`、`JsonMatcher` 模式匹配取值                                                                                            |
| 1.14.0 | `ShaderProgram#setUniform_iv` 整型数组 uniform                                                                                                    |
| 1.14.0 | 多重采样 FBO 支持降至 OpenGL ES 3.0+（原 3.1+）                                                                                                   |
| 1.14.0 | `Vector.One` 静态字段；FreeType 升级至 2.13.3                                                                                                     |
| 1.14.0 | 修复 `JsonValue#setChild` 子为 null、六边形地图图层偏移、空 TiledMap 的渲染器构造                                                                 |
| 1.14.0 | scene2d.ui `getProgrammaticChangeEvents()`；`FileTextureArrayData` 接受 TextureData 数组                                                          |
| 1.14.1 | 新增 `LongSet`、`IdentitySet`                                                                                                                     |
| 1.14.1 | GlyphLayout 与 Label 支持 Justify 两端对齐                                                                                                        |
| 1.14.1 | `ScrollPane#smoothScroll()` 控制平滑滚动速度                                                                                                      |
| 1.14.1 | `Input#isTextInputFieldOpened`；`KeyboardHeightObserver` 扩展 `onKeyboardShow/Hide`                                                               |
| 1.14.1 | scene2d 支持 NativeInput（`TextField#DEFAULT_ONSCREEN_KEYBOARD = new NativeOnscreenKeyboard()`）                                                  |
| 1.14.1 | TextField 移动端独占选项：`setAutocompleteOptions`、`setKeyboardType`、`preventAutoCorrection`                                                    |
| 1.14.1 | `GeometryUtils` 修复与改进；`XmlReader.Element` 增删时正确维护 parent                                                                             |
| 1.14.1 | 修复翻转 `TextureAtlas` 创建的 `NinePatch`、Octree 碰撞边界检查、负 delta time                                                                    |
| 1.14.1 | Skin 内部资源改用 `IdentityMap`；`InputProcessor#removeProcessor` 增加返回值                                                                      |
| 1.14.2 | `DelaunayTriangulator` 性能提升约 2 倍，新增 `ShewchukExactPredicates`，可处理全部非退化输入                                                      |
| 1.14.2 | 各 Map 新增 `putMissing()`；TimSort/ComparableTimSort 行为改进与清理                                                                              |
| 1.14.2 | 修复 `ClickListener` 取消时 over 状态卡住、`BitmapFontCache#clear` 未重置字形计数                                                                 |
| 1.14.2 | `InputMultiplexer` 与 `addAll` 返回类型回滚（见破坏性变更章节）                                                                                   |
| 1.14.2 | `TextField` 默认密码字符改为 Unicode 圆点（U+2022）                                                                                               |

### Windows 平台收益

| 版本   | 内容                                                                                                              |
| ------ | ----------------------------------------------------------------------------------------------------------------- |
| 1.12.0 | **音频切换 API**：`Audio#switchOutputDevice` / `getAvailableOutputDevices`；设备拔出（如断开耳机）音频不再中断    |
| 1.12.0 | **LWJGL3 支持 OpenGL ES 3.1 & 3.2**（为计算着色器、几何/细分着色器铺路）                                          |
| 1.12.0 | LWJGL 3.3.1 → **3.3.2**；修复 MP3 `setPosition()`；HdpiMode.Logical 下 `glViewport` 修复                          |
| 1.12.0 | 无边框全屏支持负显示器坐标；`setCursorPosition` 后立即更新鼠标坐标；`LwjglGraphics.setupDisplay` 不再选错显示模式 |
| 1.12.0 | 立体声可在单声道输出设备播放（改善 downmix/upmix）                                                                |
| 1.12.1 | LWJGL 3.3.2 → **3.3.3**                                                                                           |
| 1.12.1 | **修复 ANGLE GLES 渲染器崩溃**（同时修复了 1.11.0 natives 双份打包问题）                                          |
| 1.12.1 | **`Sound` 的 Ogg 解码改用 STBVorbis，大幅提速**                                                                   |
| 1.12.1 | 操作系统中切换音频设备时自动跟随切换                                                                              |
| 1.12.1 | GLIBC 要求降至 2.17（重新支持旧 Linux 系统）；任务栏在左/上时无边框全屏修复；`setCursor` 不再释放已捕获光标       |
| 1.13.0 | 新增 `Lwjgl3ApplicationConfiguration#pauseWhenMinimized` / `pauseWhenLostFocus`                                   |
| 1.13.0 | 支持 8/32/64-bit PCM 与 MP3 WAV；环绕声文件改进                                                                   |
| 1.13.0 | 修复桌面端 ANGLE GLES 渲染器                                                                                      |
| 1.13.0 | 移除多余 `window.makeCurrent()` 调用提升性能；窗口缩放时 delta time 不更新修复                                    |
| 1.13.1 | **LWJGL 从 3.3.4/3.3.5 回退至 3.3.3**（规避杀毒误报与 OpenAL 日志刷屏）                                           |
| 1.13.1 | `pauseWhenLostFocus` 行为修复；Wayland 下窗口位置/图标设置兼容                                                    |
| 1.13.1 | gl2/gles2.0 下 `SpriteBatch` 默认改用 VBO（原 VertexArray）                                                       |
| 1.13.5 | `OpenALLwjgl3Audio#registerSound/registerMusic` 签名改用方法引用（破坏性，见下章）                                |
| 1.14.0 | 新增 `Lwjgl3ApplicationConfiguration#useGlfwAsync()` 简写（替代 macOS 手动 `GLFW_LIBRARY_NAME` 设置）             |
| 1.14.1 | 新增 `setRGBABits` / `setDepthBits` / `setStencilBits` / `setSamples` 独立配置                                    |
| 1.14.1 | 修复 ANGLE gles 兼容模式下 `glMipMapGeneration` 检测                                                              |
| 1.14.2 | **修复 LWJGL3 OGG 音效内存泄漏**                                                                                  |
| 1.14.2 | LWJGL2：避免持锁分配 posted runnable 堆栈（LWJGL2 已不适用，仅记录）                                              |

### 其他平台收益

| 版本   | 内容                                                                                                                                                                                                                                                                         |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.12.0 | **iOS**：新增 MobiVM MetalANGLE 后端（OpenGL→Metal 转译，应对 Apple 弃用 OpenGL）                                                                                                                                                                                            |
| 1.12.0 | **iOS**：MobiVM 2.3.16 → 2.3.19（iOS 16 绑定）                                                                                                                                                                                                                               |
| 1.12.0 | **GWT**：支持 WebGL 2（`GwtApplicationConfiguration#useGL30`），全部后端至此支持 GLES 3.0                                                                                                                                                                                    |
| 1.12.0 | **Android**：光标可捕获；gdx-setup 升级 AGP 7.2.2 / SDK 32                                                                                                                                                                                                                   |
| 1.12.1 | **iOS**：MobiVM → 2.3.20                                                                                                                                                                                                                                                     |
| 1.12.1 | **Android**：因副作用回退 1.12.0 的鼠标捕获；修复三指触控状态不一致                                                                                                                                                                                                          |
| 1.12.1 | **GWT**：新增 `GwtGL20 & GwtGL30#glGetFramebufferAttachmentParameteriv`                                                                                                                                                                                                      |
| 1.13.0 | **Linux**：新增 RISC-V 支持，`gdx-xxx-natives-desktop.jar` 包含该架构原生库                                                                                                                                                                                                  |
| 1.13.0 | **iOS**：MobiVM → 2.3.21；iOS 后端实现 `AudioDevice`（`audioDeviceBufferSize/Count` 配置）；preferred FPS 逻辑改进                                                                                                                                                           |
| 1.13.0 | **GWT**：音频设备切换（需 `fetchAvailableOutputDevices`）；`AssetManager` 不再停滞；正确的 `glTexImage2D`                                                                                                                                                                    |
| 1.13.0 | **Android**：预测性返回手势（需 manifest 开关）；刘海屏下渲染选项                                                                                                                                                                                                            |
| 1.13.1 | **iOS**：MobiVM → 2.3.22；修复 iOS 18.1 模拟器 `Gdx.openURI()`                                                                                                                                                                                                               |
| 1.13.1 | **Android**：libGDX 自带 `androidx.core:core` 依赖（无需手动添加）；键盘自动纠错改进；`setContentView` 手动调用崩溃修复                                                                                                                                                      |
| 1.13.1 | **GWT**：classpath 文件加载修复；大文件复制时计算 MD5 避免报错                                                                                                                                                                                                               |
| 1.13.5 | **iOS**：MobiVM → 2.3.23；MetalANGLE 后端替换为 MetalANGLEKit（更新 ANGLE 版本）；修复 TextField `setText` 后无法删除字符                                                                                                                                                    |
| 1.13.5 | **Android**：minSDK 升至 21 后不再需要 Multidex 配置；修复 `onDestroy` 偶发 NPE                                                                                                                                                                                              |
| 1.14.0 | **Android**：修复读取软按键栏高度崩溃；`AndroidGraphics#createGraphics` 可提取自定义实例；替换 `AndroidAudioDevice/AndroidCursor` 弃用方法                                                                                                                                   |
| 1.14.1 | **iOS**：MobiVM → 2.3.24                                                                                                                                                                                                                                                     |
| 1.14.1 | **Android**：`KeyboardHeightObserver` 不再重复上报；优化 insets 的计入规则；`AsynchronousSound` 补齐 soundId API；返回键/手势崩溃修复；Android 35+ 不再设置 SHORT_EDGES cutout；`AndroidDaydream`/`updateSafeAreaInsets` NPE 修复；`AndroidGraphics#destroy` 死锁（ANR）修复 |
| 1.14.1 | **GWT**：修复 `AtlasTmxMapLoader` / `AtlasTmjMapLoader` 兼容问题                                                                                                                                                                                                             |
### 破坏性变更

破坏性的变更、弃用或移除。

| 版本   | 内容                                                                                                                                                                                                                                                             |
| ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1.12.0 | `InputProcessor` 新增 `touchCancelled`（接口实现需补充；旧行为可在其内调 `touchUp`）                                                                                                                                                                             |
| 1.12.0 | **Android**：沉浸模式默认开启（`useImmersiveMode` 默认 true）                                                                                                                                                                                                    |
| 1.12.0 | **Desktop**：`AudioDevice#getLatency()` 返回值单位改为采样数（原毫秒）                                                                                                                                                                                           |
| 1.12.0 | **Linux**：放弃旧 libc（改在 Ubuntu 20.04 构建）                                                                                                                                                                                                                 |
| 1.12.0 | **iOS**：最低版本 11.0；不再支持 armv7 32 位；改用动态框架；`hideHomeIndicator` 默认 false；`screenEdgesDeferringSystemGestures` 默认 `UIRectEdge.All`；preferred FPS 默认不设上限；`create` 与首个 `resize` 移入 `didFinishLaunching`（长时间阻塞会被系统终止） |
| 1.12.0 | Scene2D：`Actor#localToAscendantCoordinates` 非祖先时抛异常；`WidgetGroup#hit` 先校验布局；`Cell` getter 返回对象包装类型                                                                                                                                        |
| 1.12.0 | 3D：`MeshPartBuilder#lastIndex` 返回 int；骨骼最大权重默认限 4（可配置）；`Mesh#getVerticesBuffer` 等 4 个方法弃用（换带 boolean 参数版本）；`Mesh#bind/unbind` 新增实例属性位置参数                                                                             |
| 1.13.0 | **Android**：minAPI 升至 19（4.4）；后端依赖 AndroidX（需 `android.useAndroidX=true`）；移除 1.9.10 弃用的 `setCatchBackKey/isCatchBackKey/setCatchMenuKey/isCatchMenuKey`（改用 `setCatchKey/isCatchKey`）                                                      |
| 1.13.0 | **Android & iOS**：`Gdx.app.postRunnable()` 中 Runnable 抛出的异常不再吞掉，会直接崩溃                                                                                                                                                                           |
| 1.13.0 | **GWT**：升级 2.11.0，html 子项目需添加 `com.google.jsinterop:jsinterop-annotations:2.0.2:sources`                                                                                                                                                               |
| 1.13.0 | **iOS**：最低版本 12.0                                                                                                                                                                                                                                           |
| 1.13.0 | 移除 `gdx-lwjgl3-glfw-awt-macos` 扩展（macOS 改用 `Configuration.GLFW_LIBRARY_NAME.set("glfw_async")`）                                                                                                                                                          |
| 1.13.1 | AndroidX core 依赖改由 libGDX 自带（1.13.0 遗留适配项收敛）                                                                                                                                                                                                      |
| 1.13.5 | Gradle：SNAPSHOT 发布迁移至 `https://central.sonatype.com/repository/maven-snapshots/`（OSSRH 于 2025-06 退役）                                                                                                                                                  |
| 1.13.5 | Gradle：`publish` 任务改用 jreleaser，`RELEASE_REPOSITORY_URL` 环境变量不再生效                                                                                                                                                                                  |
| 1.13.5 | **Android**：minSDK 升至 21；移除 `AndroidApplicationConfiguration#touchSleepTime`；Proguard 需新增 `boolean getUseDefaultContactFilter();`                                                                                                                      |
| 1.13.5 | Skin `setEnabled` 仅对实现 `Styleable` 的 actor 生效（旧行为用 `setEnabledReflection` 或为 actor 实现 `Styleable`）                                                                                                                                              |
| 1.13.5 | LWJGL3：`OpenALLwjgl3Audio#registerSound/registerMusic` 签名改为方法引用（`registerMusic("myMusic", MyMusic::new)`）                                                                                                                                             |
| 1.13.5 | `HorizontalGroup` / `VerticalGroup` 默认 `transform = false`                                                                                                                                                                                                     |
| 1.13.5 | 弃用 `ReflectionPool` 及相关 `Pools#get/obtain`，改用 `DefaultPool`（支持 `Pools.get(MyClass::new)`）                                                                                                                                                            |
| 1.13.5 | 弃用 `Queue/ArrayMap/Array/ParallelArray.ChannelDescriptor` 中接受 `Class` 参数的构造器，改用 `ArraySupplier` 构造器                                                                                                                                             |
| 1.14.0 | 1.13.5 的 `Pools` 变更部分回滚：`Pools#get/obtain` 恢复要求 `Class` 参数；`Pools` 类整体弃用，改用 `PoolManager`（需预注册类以避免反射）                                                                                                                         |
| 1.14.0 | `JsonSkimmer` API 改用 `JsonToken` 参数                                                                                                                                                                                                                          |
| 1.14.0 | `JsonValue#last` 新增（O(1) append），改动 `next` 的代码可能需同步改动 `last`                                                                                                                                                                                    |
| 1.14.0 | `JsonValue#get` 不再忽略大小写（改用 `getIgnoreCase()`）                                                                                                                                                                                                         |
| 1.14.0 | `StringBuilder` 类删除，方法并入 `CharArray`                                                                                                                                                                                                                     |
| 1.14.1 | **iOS**：移除 `IOSAudio#willTerminate`，改由新 `Audio#dispose` 替代                                                                                                                                                                                              |
| 1.14.1 | `open/closeTextInputField` API 大改写：新增 `NativeInputCloseCallback`；关闭时不再自动发 "ENTER" 事件；`maxTextLength` 不可为 null；`TextInputWrapper#setText/setPosition/shouldClose` 移除                                                                      |
| 1.14.1 | `TextField.OnscreenKeyboard` 重构：`show(boolean)` 拆分为 `show(TextField)` 与 `close()`                                                                                                                                                                         |
| 1.14.1 | `TextField.next` 返回类型变化（覆写需重写）                                                                                                                                                                                                                      |
| 1.14.1 | `Json#ignoreUnknownField()` 参数变更（旧值用 `object.getClass()` 与 `value.name`）                                                                                                                                                                               |
| 1.14.1 | `ObjectSet/IntSet/OrderedSet` 的 `addAll` 返回 boolean（1.14.2 已回滚，见下）                                                                                                                                                                                    |
| 1.14.2 | 撤销 1.14.1 中 `InputMultiplexer` 与 `addAll` 返回类型的破坏性变更                                                                                                                                                                                               |
### 杂项变更

构建系统、工具链与生态相关。

| 版本   | 内容                                                                                                   |
| ------ | ------------------------------------------------------------------------------------------------------ |
| 1.12.0 | libGDX 本体改用 Java 11 构建（新 Android 要求）；项目仍可用 JDK 8 构建、运行时兼容性不受影响           |
| 1.12.0 | jnigen 升级至 2.4.1；javadoc 生成增加 `-use` 标志                                                      |
| 1.13.0 | jnigen 升级至 2.5.2                                                                                    |
| 1.13.0 | libGDX 改用 Java 17 构建（Gradle 8 要求）；gdx-setup 新项目使用 Gradle 8.4 + AGP 8.1.2（至少 Java 17） |
| 1.13.0 | 官方提醒：部分杀毒软件误报 LWJGL 3 natives（LWJGL#1005 / gdx-liftoff#207）                             |
| 1.13.1 | LWJGL 3.3.4/3.3.5 因杀毒误报与 OpenAL 日志问题回退至 3.3.3                                             |
| 1.13.1 | 版本字符串不再内联进 class（`no longer inline the version string`）                                    |
| 1.13.5 | 1.13.2 ~ 1.13.4 发布失败跳版；Sonatype OSSRH 退役（2025-06），迁移至 Central Portal + jreleaser 发布   |
| 1.13.5 | 官方提醒：因 R8 与 Supplier API 冲突，1.13.5 不建议用于 Android（建议 1.14.0 或停留在 1.13.1）         |
| 1.14.0 | 各类 javadoc 改进                                                                                      |
| 1.14.1 | `PoolManager` 兼容 R8 full mode；`ModelBuilder` 相关字段改 protected                                   |

## 后记

我在 ArkPets 3.12.0 版本尝试将 libGDX 从 1.11.0 升级至 1.14.2，并在我的 Windows 设备上进行了简单的性能测试。

测试方法是使用同一角色模型在新旧两版本对称运行若干轮次。设置了最大 FPS 为 60（预期帧间隔 16.7 ms）每个版本各采集了 1.7 万帧的帧级数据和 680 个进程样本点数据。

| 渲染指标       | 1.11.0 (before)    | 1.14.2 (after)    |
| -------------- | ------------------ | ----------------- |
| **帧耗时 max** | 87.6 ± 9.3 ms      | 7.5 ± 3.1 ms      |
| **帧耗时 p99** | 61.6 ms (1/16.2 s) | 2.53 ms (1/395 s) |
| 帧耗时 p90     | 1.46 ms            | 1.49 ms           |
| 帧耗时 p50     | 0.293 ms           | 0.338 ms          |
| 帧间隔 p99     | 63.5 ms            | 18.1 ms           |
| **有效 FPS**   | 55.9 ± 2.62        | 60.1 ± 0.10       |

可以发现，升级后，帧耗时 Top 1% 的表现有了显著的改善，极大程度地**优化了 low 帧及流畅度表现**。

| 性能指标    | 1.11.0 (before) | 1.14.2 (after) |
| ----------- | --------------- | -------------- |
| CPU 占用    | 12.5%           | 14.1%          |
| 工作集      | 440 MB          | 457 MB         |
| 私有内存    | 823 MB          | 837 MB         |
| 已用 JVM 堆 | 58.9 MB         | 64.6 MB        |

性能指标方面，CPU 占用的略微上升主要是由于多渲染的帧导致的；内存占用的略微上升则影响较小，可以忽略。

总体而言，将 libGDX 从 1.11.0 升级至 1.14.2 是推荐的做法，但需要妥善处理 API 变更。
