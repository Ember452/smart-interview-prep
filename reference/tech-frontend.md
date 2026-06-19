# 前端技术知识库

> 面试高频问题与答案要点。面试官按子题灵活选题，结合候选人项目场景追问。

---

## 1. 框架核心机制

### 1.1 响应式原理
- **Vue 响应式原理** — Vue2 Object.defineProperty 劫持 getter/setter + 依赖收集（Dep + Watcher）；Vue3 用 Proxy 代理整个对象，直接拦截 13 种操作，无初始递归遍历开销，支持数组索引和新增属性
- **React 状态更新与渲染** — setState 异步批量合并（合成事件/生命周期中），将多个 setState 合并为一次更新；更新触发 render → Virtual DOM diff → 最小化 DOM 操作
- **Vue3 ref vs reactive** — ref 包装基本类型（.value 访问），reactive 包装对象（直接访问属性）；ref 内部对对象值会转 reactive；模板解包仅对顶层 ref 生效

### 1.2 Virtual DOM / Diff
- **Virtual DOM 核心思想** — JavaScript 对象模拟 DOM 树，通过最小量 DOM 操作更新视图；首次渲染慢于 innerHTML，但增量更新更快
- **Vue Patch/Diff** — 双端对比（头头/尾尾/头尾/尾头），最大复用；同级比较 O(n)；无 key 时按索引复用可能出错；key 为唯一 id 时精准匹配
- **React Fiber 解决什么问题** — 旧 reconciler 递归不可中断，长任务阻塞主线程；Fiber 可中断/恢复/优先级调度（requestIdleCallback），Concurrent Mode 中高优更新可打断低优

### 1.3 Hooks / Composition API
- **React Hooks 设计动机** — 类组件中逻辑复用困难（HOC 嵌套地狱）、生命周期中不相关逻辑混杂、this 绑定；Hooks 让函数组件有状态和副作用，逻辑可按关注点分离
- **useEffect 和 useLayoutEffect 区别** — useEffect 异步执行（渲染后），不阻塞渲染；useLayoutEffect 同步执行（DOM 变更后浏览器绘制前），适合读取布局数据
- **Vue3 Composition API 优势** — 逻辑按功能组织而非选项分割；比 Mixin 更清晰（命名冲突/来源不明问题）；更好的 TS 支持

---

## 2. 性能优化

### 2.1 渲染性能
- **重绘与回流区别** — 回流（Reflow）改几何属性（宽高/位置），重新布局 → 重绘 → 合成，代价高；重绘（Repaint）只改外观（颜色/阴影），不重新布局。优化方向：批量修改 DOM（documentFragment/fastdom）、用 transform 代替 top/left（跳过布局）、对频繁操作节流/防抖
- **React 渲染优化** — React.memo 避免 props 未变的子组件重渲染；useMemo/useCallback 缓存计算值和函数引用；虚拟列表（react-window）只渲染可视区，万级列表不卡顿

### 2.2 打包优化
- **Tree Shaking** — 基于 ES Module 静态分析，找出未引用的代码并在打包时删除。前提：ES Module、production mode、sideEffects 配置排除无副作用模块
- **Code Split / 懒加载** — Webpack `import()` 动态导入，Vue Router `component: () => import()`，React `React.lazy + Suspense`。按路由拆分（route-based）最常用，首屏只加载必需代码
- **首屏性能指标体系** — FCP（首次内容绘制/≤1.8s）、LCP（最大内容绘制/≤2.5s）、TTI（可交互时间/≤3.8s）、CLS（累计布局偏移/≤0.1）。优化方向：关键 CSS 内联、JS defer/async、预加载/预连接、压缩/缓存

### 2.3 缓存策略
- **前端缓存层级** — Service Worker（Cache API 拦截请求）→ 浏览器 HTTP 缓存（强缓存/协商缓存）→ Memory Cache（页面内图/样式等）→ 应用层缓存（LocalStorage/IndexedDB/Redux cache）
- **强缓存 vs 协商缓存** — 强缓存：Expires/Cache-Control（max-age），不请求服务器；协商缓存：ETag/If-None-Match、Last-Modified/If-Modified-Since，304 返回

---

## 3. 状态管理

- **Redux 核心三原则** — 单一数据源（Single store）、State 只读（行动触发 reducer 创建新 state）、纯函数 reducer 返回新状态。数据流：dispatch(action) → reducer → new state → 订阅组件重渲染
- **Pinia vs Vuex** — Pinia 无 mutations（只有 state/getters/actions）、模块化自动按文件组织、完整 TS 支持、可多 store 并存、devtools 支持更好
- **跨组件状态方案对比** — Props drilling（小项目简单）/ Context（React 16.3+，适合中低频更新）/ Redux（有中间件，复杂状态流）/ Zustand（极简，类似 Pinia）/ Recoil/Jotai（原子化状态，精确更新）
- **状态库选型判断** — 页面级状态用 Props/Context→应用级全局状态（用户信息/主题等）用状态库→服务端缓存状态（API 响应）优先用 React Query/SWR 而非自己管理

---

## 4. 工程化

### 4.1 模块化
- **ESM vs CommonJS** — ESM 静态分析（编译时确定依赖，可 Tree Shaking）、异步加载；CJS 动态（运行时 require）、同步加载、值的拷贝而非引用
- **组件设计原则** — 单一职责（一个组件做一件事）、高内聚低耦合、Props 向下/Events 向上、组合优于继承

### 4.2 测试
- **前端测试金字塔** — 静态分析/ESLint（基底）→ 单元测试（Jest/Vitest 最大量）→ 组件测试（Testing Library 适中）→ E2E（Cypress/Playwright 最少）。覆盖核心业务逻辑，不追求 100%覆盖率
- **快照测试的局限性** — 仅检测"是否有变化"，不能验证"是否应该变化"；大面积快照更新会导致无视真正的回归

### 4.3 监控
- **前端监控体系** — 错误监控（Sentry/fundebug 捕获 JS 错误/Promise rejection/资源加载失败）、性能监控（Web Vitals API/PerformanceObserver）、用户行为（埋点/热力图/录屏）
- **白屏如何监控** — MutationObserver 监听 DOM + 关键节点检查；采样对比预期 DOM 结构；无 DOM 生成则判定白屏

---

## 5. CSS 与布局

- **Flex vs Grid 选择** — Flex 一维布局（单行或单列对齐和空间分配），适合导航栏/列表项/卡片内排版；Grid 二维（同时控制行和列），适合页面整体布局/仪表盘
- **BFC（块格式化上下文）** — 独立渲染区域，内部布局不影响外部。解决：margin 重叠、浮动元素高度塌陷、覆盖浮动元素。触发方式：overflow:hidden、float、position:absolute、display:flow-root
- **CSS-in-JS vs CSS Modules** — CSS Modules 编译时生成唯一类名，无运行时开销，兼容性好，适合通用项目；CSS-in-JS（styled-components/Emotion）运行时注入，动态样式强，与组件耦合紧密

---

## 6. 跨端方案

- **React Native 核心原理** — 异步桥接：JS 线程 → Bridge（JSON 序列化）→ Native 线程。瓶颈在 Bridge 高频通信（滚动/动画）会丢帧。新架构 JSI 替代 Bridge，允许 JS 直接调用 Native（减少序列化），Fabric 渲染器线程优先级更高
- **Flutter 渲染管线** — Skia 引擎原生绘制（不依赖 OEM Widget），三棵树（Widget → Element → RenderObject），Widget 不可变且频繁重建。性能瓶颈：Widget 重建过多→build 耗时（用 const/RepaintBoundary 隔离）
- **跨端选型考量** — 纯展示+高频迭代选 RN/Flutter；强性能+复杂动画选 Flutter；团队有原生储备需要部分跨端选 RN；已有 H5 想快速上 App 选混合方案（WebView/JsBridge）

---

## 7. SSR 与微前端

- **CSR vs SSR vs SSG vs ISR** — CSR（客户端渲染：浏览器下载 JS 后渲染，首屏白屏长，SEO 差）；SSR（服务端渲染：服务器生成 HTML 返回，首屏快+SEO 友好，但服务器压力大）；SSG（构建时生成静态 HTML，极快但内容静态）；ISR（增量静态再生：Next.js 特色，SSG + 后台定时重新生成，动态与性能兼顾）
- **Next.js 核心机制** — App Router（服务端组件 RSC + 客户端组件混合）、Streaming SSR（分块传输渲染）、Server Actions（服务端函数直接调用）、Image Optimization（自动 WebP/AVIF/响应式）。性能优势：首屏 TTFB 快（边缘缓存）+ SEO 完整
- **微前端架构** — 解决的问题：巨型前端应用拆分独立部署、技术栈共存、团队并行开发。方案：Module Federation（Webpack 5，运行时共享模块，轻量）；qiankun（阿里，基于 single-spa，沙箱隔离+样式隔离）；iframe（完全隔离但通信复杂）。挑战：样式冲突/全局状态共享/路由冲突/构建复杂度上升