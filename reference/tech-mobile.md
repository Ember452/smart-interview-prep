# 移动 端
> 面试高频问题与答案要点。覆盖 iOS / Android / Flutter / RN 通用考点。

---

## 1. iOS

### 1.1 内存管理
- **ARC vs MRC** — ARC 编译器自动插入 retain/release/autorelease，规则：strong 引用+1 / weak 不增加引用计数且释放后置 nil / assign 不增加也不置 nil（野指针风险）；MRC 手动管理保留/释放
- **循环引用与解决** — 两个对象互相 strong 引用导致都无法释放。方案：weak（obj1 强引用 obj2，obj2 弱引用 obj1）；unowned（类似 weak 但不置 nil，适用生命周期相同场景）；闭包用 [weak self] 捕获列表
- **NSTimer 循环引用** — Timer 被 RunLoop 强持有 → Target 被 Timer 强持有 → Target 持有 Timer。解决方案：iOS 10+ 用 block API；中间代理对象弱引用 target 转发消息

### 1.2 RunLoop
- **RunLoop 核心机制** — 事件循环，线程保活同时不忙等。Source0（触摸/performSelector）→ Source1（Mach port 内核消息）→ Timer → Observer（监听 RunLoop 状态变化）。Mode 决定处理哪些 Source/Timer/Observer
- **RunLoop 与 AutoreleasePool 的关系** — Observer 监听 kCFRunLoopEntry 时创建 AutoReleasePool，kCFRunLoopBeforeWaiting 时释放旧池创建新池（降低内存峰值）；主线程默认开启了 RunLoop（UIApplicationMain）

### 1.3 启动优化
- **冷启动三阶段** — T1：加载 Mach-O/dynamic linker 动态库加载+rebase/binding；T2：main() 前（+load 方法/C++ static 初始化/ObjC runtime 注册）；T3：main() 后（didFinishLaunching）。优化：减少动态库数量、+load 换成 +initialize、延迟非首屏初始化
- **二进制重排** — 按 launch 调用顺序排列代码段（Order File），减少 Page Fault 次数，降低 T1 耗时

### 1.4 离屏渲染
- **触发条件** — cornerRadius + masksToBounds（同时设置）、shadow（除非指定 shadowPath）、mask 蒙层、group opacity、复杂文字绘制。正常渲染通过 GPU 直接合成；离屏渲染需额外创建缓冲区 → 渲染 → 合成，性能代价高
- **圆角优化** — 图片圆角：Core Graphics 提前裁剪成圆角图 → 缓存；仅背景色圆角可只设 cornerRadius（不设 masksToBounds）；或贝塞尔曲线遮罩

---

## 2. Android

### 2.1 Activity 生命周期
- **典型场景流转** — 启动 onCreate → onStart → onResume。切后台：onPause → onStop。切回：onRestart → onStart → onResume。横竖屏切换/配置变更默认重建（onDestroy → onCreate），可声明 configChanges 接管
- **onSaveInstanceState vs onRestoreInstanceState** — 系统因内存不足杀进程前保存状态（onSaveInstanceState）→重建时从 Bundle 恢复（onCreate/onRestoreInstanceState）。轻量 Serializable/Parcelable 数据，大对象不行

### 2.2 Handler 消息机制
- **核心组件** — Handler（发送+处理消息）、Looper（从 MessageQueue 取消息分发给 Handler）、MessageQueue（单链表消息队列、按时间排序）。一个线程只一个 Looper
- **消息延时实现** — MessageQueue.enqueueMessage() 按 when 插入合适位置；Looper.loop() 取队首消息，若未到时间则 nativePollOnce() 阻塞指定时间
- **内存泄漏** — 非静态内部类 Handler 持有外部 Activity 引用 → 消息延时未处理 → Activity 无法回收。解决：静态内部类 + WeakReference；onDestroy 中 removeCallbacksAndMessages

### 2.3 ANR 与卡顿
- **ANR 触发阈值** — 输入事件 5s / BroadcastReceiver 10s（前台）/60s（后台）/ Service 20s（前台）/200s（后台）无响应
- **排查工具** — traces.txt（ANR 各线程堆栈）→ 主线程 BlockCanon（Looper 消息耗时监控）→ Systrace/Perfetto 跟踪系统调用执行时间

---

## 3. Flutter

### 3.1 渲染管线
- **三棵树** — Widget Tree（配置/不可变/频繁重建）→ Element Tree（桥接/管理生命周期/可变）→ RenderObject Tree（布局+绘制+命中测试）。setState → Widget 重建 → Element 根据 key/type 判断复用或新建 → RenderObject 收到新约束重新布局/绘制
- **布局约束模型** — 父级向下传递 Constraints（min/max 宽高），子级向上返回 Size。类似 iOS AutoLayout 的约束传递、但无方程求解

### 3.2 状态管理
- **Provider 原理** — InheritedWidget 依赖注入，ChangeNotifier 发出通知 → Consumer/Selector 重建对应子组件。相比 Bloc 更轻量，适合中小型项目
- **Bloc/Cubit 原理** — 事件驱动 + Stream：Event → Bloc 处理 → yield 新 State → BlocBuilder 重建。优点：业务逻辑与 UI 彻底分离，可测试性强；适合中大型项目

### 3.3 性能优化
- **Widget 重建控制** — const 构造函数标记常量组件（永不重建）；RepaintBoundary 隔离重绘区域（子组件重绘不触发父级）；Selector 替代 Consumer 减少不必要的重绘范围
- **列表优化** — ListView.builder 按需构建可见 item（回收不可见 Widget）；固定高度 item 减少测量开销；避免列表内复杂布局/Clip（需离屏渲染）
- **包体积治理** — 移除未使用的 package；ProGuard/R8 代码混淆+裁剪；图片用 WebP 格式；使用 deferred components 延迟加载

---

## 4. React Native

### 4.1 Bridge 机制
- **异步桥接流程** — JS 线程调用 NativeModules.xxx → 参数 JSON 序列化到 Bridge Queue → Native 线程处理 → 结果 JSON 回 Bridge → JS 线程回调。高频通信（动画/手势）时 JSON 编码/解码瓶颈 → 丢帧
- **新架构 JSI/Fabric/TurboModules** — JSI：JS 可直接调用 Native（C++ 接口），消除 Bridge 序列化开销；Fabric：新渲染器，Shadow Tree 跨线程共享且可按优先级渲染；TurboModules：模块按需加载而非启动时全注册
- **Hermes 引擎** — 专为 RN 优化的 JS 引擎。AOT 编译为字节码（减小 APK 体积/加快启动），内存占用比 JSC 低，Fiber 渲染配合效果更好

### 4.2 性能痛点
- **列表长列表优化** — FlatList 虚拟化（windowSize/maxToRenderPerBatch/removeClippedSubviews）；getItemLayout 提供固定高度跳过测量；keyExtractor 稳定 key。避免列表内匿名函数每次都创建新的 props 引用
- **图片加载** — FastImage（SDWebImage/Glide 磁盘缓存+预加载）；resizeMode=cover 而非 contain 减少计算；渐进式/缩略图优先加载

---

## 5. 移动端通用

- **弱网优化** — 请求合并/接口聚合、离线缓存优先展示（缓存+异步刷新）、图片分级加载（低清→高清）、按网络类型自适应（WiFi 全量加载/4G 低清）
- **灰度发布** — 按设备 ID/地域/用户分组逐步放量（5%→20%→50%→100%），实时监控 Crash 率/核心接口成功率/性能指标异常即时回滚
- **热修复方案对比** — Android：Tinker（补丁包整包替换，兼容性好）；iOS：JSPatch（JS 调用 OC runtime，已被 App Store 限制）→ 目前主流用内建配置开关/降级
- **内存泄漏排查** — Android：LeakCanary（弱引用 + GC 后检查引用队列）；Profiler Memory Dump；iOS：Xcode Memory Graph 调试器；Instruments Leaks/Allocations

---

## 6. 启动性能与深度优化

- **冷启动 vs 温启动 vs 热启动** — 冷启动：进程未创建，全量初始化；温启动：进程已杀但 Activity 状态被保存，重建 UI 较快；热启动：进程仍在后台，直接切回前台。Android 优化：Application.onCreate 延迟初始化（懒加载 SDK）+ ContentProvider 初始化拆分 + Splash Screen API；iOS 优化：见上方冷启动三阶段
- **包体积优化实战** — Android：APK Analyzer 分析大文件占比、AAB 格式 Google Play 按 ABI 拆分、so 动态下发（ReLinker 加载）；iOS：LinkMap 分析符号大小、__TEXT 段超 400MB 则 App Thinning 失效、资源 Asset Catalog 优化
- **Deep Link 与 Universal Links** — URL Scheme（myapp://path，系统弹框确认，已安装才可跳，安全性低）；Universal Links（iOS）/ App Links（Android）：HTTPS 链接直接打开 App，未安装则跳 Web。核心配置：apple-app-site-association 文件/assetlinks.json 验证域名归属

---

## 7. 跨平台方案选型

| 维度 | Flutter | React Native | 原生 |
|------|---------|-------------|------|
| 性能 | 接近原生（Skia/Impeller 自绘） | JS Bridge 有开销，新架构 JSI 大幅改善 | 最佳 |
| 热更新 | 不支持（需发版） | 支持（CodePush/热更新） | 不支持 |
| 包体积 | 较大（引擎约 4-6MB） | 中等（JS Bundle） | 最小 |
| 适用场景 | 多端一致 UI、新应用 | 已有 Web 团队复用、需热更 | 性能敏感、平台特性深度使用 |