# 工具如何使用很快就可以学完，但背后的涉及到知识点花点时间久点



# 下面所有内容的简述，你要是不懂就跳到对应位置看

# 知识点汇总简述

## 一、性能监控方式

1. **控制台测速**：`console.time`、`performance.mark` / `performance.measure`，用于代码块耗时埋点，打印到控制台。
2. Chrome DevTools工具
   - **Performance面板**：录制页面行为，查看FPS、CPU、NET、主线程Main火焰图；识别FP/FCP/LCP标记、红色三角**Long Task长任务（>50ms，主线程阻塞元凶）**，勾选Screenshots查看页面截图；可限制CPU来模拟手机弱机。
   - **Lighthouse**：一键生成性能报告，输出核心指标量化值并给出优化建议。
   - **Memory面板**：排查内存泄漏。
   - **Network面板**：查看TTFB等网络耗时。

## 二、前端性能指标（含全称+含义）

1. FP（First Paint，首次绘制）：浏览器第一次渲染任意像素。

2. FCP（First Contentful Paint，首次内容绘制）：首次渲染出文字/图片等有效内容，用来诊断白屏。

3. LCP（Largest Contentful Paint，最大内容绘制）：首屏最大内容渲染完成，**Core Web Vitals核心指标**。

4. TTI（Time to Interactive，可交互时间）：页面完全稳定响应用户操作的时间。

5. TBT（Total Blocking Time，总阻塞时间）：FCP到TTI之间，长任务超出50ms的阻塞时长总和，反映主线程阻塞程度。

6. INP（Interaction to Next Paint，交互到下一次绘制）：用户交互到下一次页面绘制耗时，**Core Web Vitals，衡量交互卡顿**。

7. CLS（Cumulative Layout Shift，累积布局偏移）：页面元素意外跳动的分数，**Core Web Vitals**。

8. TTFB（Time To First Byte，首字节时间）：收到服务器第一个字节的耗时，代表后端网络响应速度。

   > 旧指标FID已废弃，被INP替代。

## 三、浏览器渲染原理

浏览器流水线：DOM+CSSOM → Layout(重排) → Paint(重绘) → Composite(合成)

- 开销：Layout > Paint > Composite
- `transform` / `opacity`动画走GPU合成，不触发重排重绘；避免读写DOM布局属性交替造成强制同步布局。

## 四、性能优化四大方向

1. **JS优化**：拆分长任务；防抖节流；记忆化缓存；代码分割、路由懒加载、Tree-shaking；清理定时器/事件监听，防止内存泄漏。
2. **渲染优化**：减少重排重绘；大数据列表使用虚拟列表；合理使用`will-change`。
3. **资源加载优化**：图片压缩、懒加载；关键CSS内联；脚本加`defer/async`；preload/preconnect预加载。
4. **网络优化**：开启资源压缩；配置缓存；减少请求数量与体积；接口缓存、延迟非首屏请求。

## 五、性能排查完整流程

1. 复现卡顿场景；
2. Performance录制，定位问题类型（JS/渲染/网络）；
3. Lighthouse获取指标分数；
4. 使用`performance.mark`埋点精准定位代码；
5. 优化后**再次录制对比数据，验证优化效果**（不能凭主观感受）。



# 费曼

测试前端性能有两种方式：代码埋点和浏览器 DevTools。 代码里可以用 `performance.mark` 做精细埋点，简单测速就用`console.time`。但排查页面卡顿最好用 Performance 面板：开启录制，复现卡顿操作，录制结束看火焰图，放大找到长任务，点击任务就能定位源码。（火焰图就是 Performance 录制结束之后，在面板中间 `Main（主线程）` 那块区域看到的图。）

性能指标分核心指标和辅助指标： 三大核心指标是 LCP、INP、CLS。 FCP 是辅助指标，代表首次渲染有效内容，可以在 Performance 顶部看到； INP 衡量交互响应快慢；CLS 衡量页面元素跳动程度，CLS 的具体数值靠 Lighthouse 查看。 TTFB 是首字节时间，用来评估后端网络响应速度。

浏览器渲染顺序：DOM、CSSOM → Layout 重排 → Paint 重绘 → Composite 合成。Layout 开销最大，所以要尽量减少 DOM 尺寸、位置修改。动画优先使用 transform 和 opacity，只触发合成。

优化主要四个方向：JS 脚本优化、图片静态资源优化、渲染优化、网络优化。JS 可以用 defer/async 避免阻塞 HTML 渲染；图片做懒加载、压缩、缓存。





# 一、控制台怎么监控性能（别人常用的两种方式）

## 方式1：Console简单计时（写代码埋点，在Console面板打印）

`performance.mark` / `performance.measure` 是浏览器原生API，**可以在Performance面板看到标记，也能在控制台打印耗时**，这就是很多演示里看到的监控代码。

```ts
// 1. 打标记：起点
performance.mark('start-calc');

// 这里放你要测试的代码
let arr = [];
for(let i=0; i<100000; i++) arr.push(i);

// 2. 终点标记
performance.mark('end-calc');

// 3. 计算两个标记之间耗时
performance.measure('计算循环耗时', 'start-calc', 'end-calc');

// 取出结果打印到控制台
const measure = performance.getEntriesByName('计算循环耗时')[0];
console.log('代码执行耗时：', measure.duration, 'ms');

// 清理标记（防止多次执行堆积）
performance.clearMarks();
performance.clearMeasures();
```

简单临时测试也可以用 `console.time`，更简短：

```ts
console.time('test');
// 待测代码
console.timeEnd('test'); // 控制台直接输出耗时
```

> 缺点：只能测**代码块耗时**，看不到渲染、重排、FPS、CPU占用。适合小函数单元测速。

## 方式2：Chrome DevTools Performance面板（最核心！排查页面卡顿首选）

### 先看视频：https://m.bilibili.com/video/BV1o42mYXE2f

1. F12打开开发者工具，切到 **Performance**

2. 左上角录制按钮（圆点），录制前建议：

   - CPU Throttling：限制CPU（比如4x减速，模拟手机弱性能）
   - 勾选Screenshots，录制页面截图

3. 点击录制，操作页面（点击、滚动、加载），操作完点停止

4. 看懂图表：

   - **FPS**：**帧率，目标60fps，红色条 = 掉帧卡顿；1帧≈16.6ms，任务超过16.6ms就会卡**

   - **CPU**：满彩条=CPU跑满，主线程阻塞

   - **NET**：网络加载情况

   - Main（主线程火焰图）

     ：最重要！

     - 黄色：JS执行（Scripting）
     - 紫色：重排Layout（布局）
     - 绿色：重绘Paint（绘制）
     - 灰色：空闲

   - 放大火焰图，点长Task，看Bottom-Up/Call Tree，定位

     哪个函数耗时长

     > 核心原则：**主线程长任务（>50ms）就是INP交互卡顿元凶**

### 其他配套面板

- **Lighthouse**：一键跑页面综合性能评分，输出LCP、FCP、CLS三大核心Web指标，给出优化建议（首屏加载首选）
- **Memory面板**：看JS堆内存，判断内存泄漏（反复操作页面，内存持续上涨不回落就是泄漏）
- **Performance monitor**：实时监视器，实时看FPS、CPU、DOM节点、每秒Layout/Paint数量

# 二、前端性能指标（要监控什么）

**Core Web Vitals（谷歌核心指标）**

1. LCP 最大内容绘制：首屏最大元素加载完成时间，理想 <2.5s
2. INP 交互响应时间：点击/输入到页面响应，理想 <200ms
3. CLS 累积布局偏移：页面加载过程元素跳动，理想 <0.1

附加指标： FCP 首次内容绘制、TTI 可交互时间、FPS帧率、长任务Long Task、资源加载耗时

# 三、性能优化，分4大类（定位问题后针对性优化）

## 1. JS执行优化（解决主线程长任务、交互卡顿）

1. 拆分长任务：超过50ms的代码拆成小块，`requestIdleCallback` / `setTimeout` 分片执行，避免阻塞主线程
2. 防抖debounce、节流throttle：滚动、输入、resize高频事件
3. 避免不必要计算：缓存计算结果（记忆化），不要在循环里频繁操作DOM
4. 代码分割：动态import()，路由懒加载，首屏只加载必要JS，减少bundle体积
5. 移除冗余代码：Tree-shaking，删除未使用代码；去掉大的第三方库（换轻量版）
6. 内存管理：及时解绑事件监听、销毁定时器，防止内存泄漏

## 2. 渲染优化（减少Layout重排、Paint重绘，高频动画重点）

浏览器渲染流水线：DOM+CSSOM → **Layout布局** → Paint绘制 → Composite合成

> 最贵：Layout（重排），其次Paint，合成最省

- 尽量只触发 **Composite合成**：用 `transform`、`opacity` 做动画，这两个属性走GPU合成，**不触发重排重绘**
- 不要频繁读取DOM布局属性再修改（读写交替，强制同步布局，浏览器反复计算Layout）

```
// ❌坏例子：读一次改一次，强制布局
for(let i=0;i<els.length;i++){
  const top = els[i].offsetTop; // 读
  els[i].style.top = top + 1 + 'px'; // 写
}
// ✅好例子：一次性全部读，再一次性全部写
const tops = [];
for(let i=0;i<els.length;i++) tops.push(els[i].offsetTop);
for(let i=0;i<els.length;i++) els[i].style.top = tops[i]+1+'px';
```

- 减少DOM节点数量，列表大数据用**虚拟列表**
- CSS：减少复杂选择器，避免通配符；动画元素开启GPU加速 `will-change: transform`（慎用）

## 3. 资源加载优化（解决首屏白屏、LCP慢）

1. 图片优化：webp/avif、响应式图片、懒加载 `loading="lazy"`，大图片预加载
2. 静态资源：开启Gzip/Brotli压缩；打包带hash文件名，配置强缓存
3. 脚本：`<script defer>` / `<script async>`，避免JS阻塞HTML解析；非关键JS延迟加载
4. 关键CSS内联，非关键CSS异步加载，防止CSS阻塞渲染
5. 预加载：`preload` 预加载首屏关键资源；`preconnect` 提前建立跨域连接

## 4. 网络优化

- 减少请求数量：小图雪碧图/inline base64（慎用）、合并请求
- 减少请求体积，资源压缩
- 接口优化：非首屏接口延迟请求，接口缓存，减少重复请求

# 四、完整排查流程（标准工作流）

1. 先复现卡顿场景
2. Performance录制，看FPS、长Task，定位是JS执行 / Layout / 资源加载慢
3. Lighthouse跑首屏指标，看LCP/FCP问题
4. 用performance.mark埋点，精准定位是哪一段代码
5. 优化后，**再次录制对比数据**，验证优化前后指标变化（不能凭感觉）





# FCP = First Contentful Paint ｜**首次内容绘制**

一句话：**页面开始加载 → 屏幕上第一次渲染出「有实际意义的内容」的耗时**

> 内容才算：文本、图片、SVG、有内容的canvas ❌ 不算：只是空白页面、背景色、空的div边框（只有底色不算FCP）

## FP 和 FCP 的区别（很容易搞混）

- **FP First Paint（首次绘制）**：浏览器画了任意不一样的像素就行，比如页面背景由白变灰，就算FP。此时页面上**没有文字图片**。

- FCP

  ：FP之后，直到

  第一个DOM内容（文字/图）渲染出来

  ，才触发FCP。

  > FCP一定 ≥ FP

## FCP 和 LCP 的区别（重点！）

- **FCP：第一个内容出现的时间**（页面终于不是空白了，用户看到一点点东西）

- LCP：首屏最大那块内容渲染完成时间

  （用户看到主要大块内容）

  > ✅ 重要规则：**LCP永远不会比FCP更早**，FCP慢，LCP基本一定慢。 FCP**不是谷歌三大核心Web指标**（三大是 LCP / INP / CLS），但是**非常重要的诊断指标**，用来定位首屏白屏原因

## 参考标准（75分位真实用户数据）

- ✅ 良好：**≤1.8s**
- ⚠️ 需要优化：1.8s ~ 3.0s
- ❌ 差：>3.0s

## 怎么在控制台/DevTools看FCP

1. Lighthouse 直接跑完报告，里面会输出FCP
2. Performance录制，在时间轴上能看到标记：`First Contentful Paint`
3. JS代码捕获（web-vitals库，线上埋点）

```
import {onFCP} from 'web-vitals';
onFCP(metric => {
  console.log('FCP耗时：', metric.value, 'ms');
})
```

## FCP为什么会慢？（最常见原因）

1. **阻塞渲染资源**：没有defer/async的JS、外部大CSS，浏览器要等解析完才能渲染内容（头号元凶）
2. HTML返回太慢（TTFB首字节延迟高）
3. 关键CSS没有内联，浏览器下载完CSS才开始渲染文字
4. JS大量阻塞主线程，延迟DOM生成

## FCP优化思路

- CSS：**关键CSS内联，非关键CSS异步加载**
- JS：首屏非必要脚本加上 `defer` / `async`，避免阻塞HTML解析
- 缩短TTFB，后端尽快吐出HTML
- 减少HTML里首屏之前的大JS，减少长任务阻塞DOM构建

> 简单理解用户体感：FCP就是**用户从输入网址后，第一次看到页面出现文字/图片，不再是纯白屏的等待时长**。

要不要我顺便把 FP、FCP、LCP、TTI、TBT 一次性做个简短对照表给你，方便你看Performance面板时对照？





# 前端性能指标对照表

| 缩写 | 英文全称                  | 中文名称         | 核心含义                                                     | 理想阈值 | 备注                                                         |
| ---- | ------------------------- | ---------------- | ------------------------------------------------------------ | -------- | ------------------------------------------------------------ |
| FP   | First Paint               | 首次绘制         | 浏览器第一次在屏幕上渲染**任何像素**（页面背景变色就算，不一定有文字图片） | ≤1.5s    | 只是页面不再纯白，**没有有效内容**，不是核心指标             |
| FCP  | First Contentful Paint    | 首次内容绘制     | 页面第一次渲染出**有实际内容**（文字、图片、SVG等）          | ≤1.8s    | 代表用户首次看到文字/图像，诊断白屏很有用，不属于Core Web Vitals |
| LCP  | Largest Contentful Paint  | 最大内容绘制     | 首屏**最大块内容元素**渲染完成时间                           | ≤2.5s    | ✅ 三大核心指标之一；衡量首屏主内容加载速度                   |
| TTI  | Time to Interactive       | 可交互时间       | 页面完全加载完毕，**能稳定响应用户点击输入**的时间           | ≤3.8s    | 页面看起来加载完，但JS还在跑，此时点击无响应，TTI就是解决这个衡量 |
| TBT  | Total Blocking Time       | 总阻塞时间       | FCP到TTI之间，所有**长任务超出50ms的阻塞时长总和**           | ≤200ms   | 衡量主线程阻塞程度，TBT越大，交互越卡顿，和INP强相关         |
| INP  | Interaction to Next Paint | 交互到下一次绘制 | 用户交互（点击/输入）到浏览器完成下一次绘制的耗时            | ≤200ms   | ✅ 三大核心指标，替代旧的FID，衡量交互卡顿                    |
| CLS  | Cumulative Layout Shift   | 累积布局偏移     | 页面加载过程元素意外跳动的总分                               | ≤0.1     | ✅ 三大核心指标；图片没设宽高、动态加载内容会造成布局抖动     |
| FID  | First Input Delay         | 首次输入延迟     | 用户第一次交互到浏览器可以处理的延迟                         | ≤100ms   | **已废弃**，旧核心指标，现已被INP取代                        |
| TTFB | Time To First Byte        | 首字节时间       | 浏览器发起请求，到收到服务器返回第一个字节耗时               | ≤600ms   | 衡量后端/网络响应速度，TTFB高，后面所有指标都会跟着变差      |





# 各指标在Chrome DevTools Performance面板 查找指南

> 打开F12 → Performance，点击录制，操作页面后停止录制。 指标标记会显示在**时间轴上方的蓝色小竖线 + 文字标签**；火焰图看主线程任务。

| 缩写                          | 在哪里看                                                    | 怎么识别                                                     | 补充说明                                                     |
| ----------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| FP First Paint                | 顶部时间轴标签                                              | 标签文字：`First Paint`                                      | 页面第一次绘制像素。一般在最靠前的一批标记里，早于FCP。      |
| FCP First Contentful Paint    | 顶部时间轴标签                                              | 标签文字：`First Contentful Paint`                           | 第一个文字/图片渲染完成。**FCP线一定在FP后面**。             |
| LCP Largest Contentful Paint  | 顶部时间轴标签                                              | 标签文字：`Largest Contentful Paint`                         | 首屏最大元素渲染完成。 ⚠️ 注意：LCP有可能在页面加载后期才出现，不是一定紧跟FCP。 |
| TTI Time to Interactive       | **不在Performance顶部自带标记**，需要Lighthouse             | Performance面板本身**不会自动标出TTI**。 获取方式：切到Lighthouse面板跑检测，报告会输出TTI数值。 | 页面稳定可交互的时间。Performance只能手动看主线程长任务辅助判断，没有原生标记。 |
| TBT Total Blocking Time       | **不在Performance顶部自带标记**，Lighthouse输出             | TBT = FCP ~ TTI之间，所有长任务超过50ms部分累加。 获取方式：Lighthouse报告直接给出TBT数值。 | 你可以在Main主线程火焰图，找到红色三角的**Long Task（长任务>50ms）**，自己估算阻塞时长。 |
| INP Interaction to Next Paint | 两种查看方式： 1. Lighthouse 2. Performance录制，看交互事件 | 录制时做点击/输入操作。在Main主线程找到你的交互（Click/Input），看从交互触发到下一次Paint之间的耗时。 Lighthouse报告会直接给出INP分数。 | 衡量交互卡顿。火焰图里长任务会拉高INP。                      |
| CLS Cumulative Layout Shift   | Lighthouse报告；Performance看截图                           | Performance录制开启Screenshots。看时间轴上页面截图，观察元素跳动。CLS数值在Lighthouse报告。 | 布局跳动，Performance只能肉眼观察跳动现象，**数值由Lighthouse给出**。 |
| TTFB Time To First Byte       | Network面板（不是Performance）                              | Network面板，选中HTML请求 → Timing标签里的`Waiting for server response`。 | 请求发出到收到后端第一个字节。**属于网络指标**。             |

## 🔥 重点名词：Long Task（长任务，TBT/INP的根源）

1. 在Performance的Main主线程火焰图
2. 任务块右上角有**红色小三角** = Long Task
3. 定义：**任务执行时间 >50ms**
4. 原理：主线程被JS长时间占用，用户点击页面没有响应，INP变差、TBT升高

## ✅ 快速上手步骤（照着袁进老师视频里的操作）

1. F12 → Performance，勾选 **Screenshots**
2. CPU Throttling 选 4x（模拟低配手机，更容易看出卡顿）
3. 点录制圆点，刷新页面，页面加载完点停止
4. 看顶部蓝色标记：FP、FCP、LCP，横向坐标就是时间
5. 放大Main主线程火焰图，找红色三角长任务
6. 再跑Lighthouse，拿到TBT、INP、CLS、TTI的量化分数

## 小坑提醒

1. Performance面板**不会自动计算TBT、TTI、CLS**，只能看到现象；量化数值要靠Lighthouse
2. LCP会更新：页面加载过程中，最大元素可能发生变化，LCP标记是**最后一次更新**的那个时间点
3. FP/FCP/LCP标记只有**页面首次加载**会出现；滚动、点击等后续操作不会再触发这几个标记