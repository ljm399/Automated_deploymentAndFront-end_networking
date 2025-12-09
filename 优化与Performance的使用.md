# 优化

下面我将详细拆解这个方案的原理、代码逻辑以及它如何优化白屏时间。

### 一。白屏优化

#### 1.问题原因

在前端（Vue/React）中，如果你在一个循环里（`v-for`）一次性渲染 100 个 `<heavy-comp>`（重组件），浏览器的主线程（Main Thread）会经历以下过程：

1.  **JS 执行**：Vue 创建 100 个组件实例，初始化数据，生成虚拟 DOM。
2.  **DOM 操作**：将 100 个组件对应的真实节点插入页面。
3.  **渲染**：浏览器计算布局（Layout）并绘制（Paint）。

**问题在于**：在第 3 步绘制完成之前，主线程一直被第 1 和第 2 步霸占着。用户看到的就是一片空白（白屏），或者是上一页的残留画面，且无法交互。如果这个过程耗时 2 秒，用户就得死等 2 秒。

#### 2. 解决方案：requestAnimationFrame 的工作原理

**利用帧（Frame）的概念，每一帧只画一部分**。

#### 代码逻辑

1.  **`const count = ref(0)`**:
    *   创建一个响应式变量 `count`，初始值为 0。这个变量控制当前允许渲染到第几个组件。

2.  **`requestAnimationFrame(update)`**:
    *   这是一个核心 API。`requestAnimationFrame` 告诉浏览器：“在下一帧绘制之前，请执行这个 `update` 函数”。
    *   通常屏幕刷新率是 60Hz，意味着大约每 **16.6ms** 执行一次。

3.  **`function update()`**:
    *   `count.value++`: 每一帧，计数器加 1。
    *   `requestAnimationFrame(update)`: 递归调用，确保持续不断地在每一帧更新计数器，直到不再需要（虽然代码里没写停止逻辑，实际项目中通常加个 `if (count.value > max) return`）。

4.  **`return function (n)`**:
    *   这是暴露给模板使用的判断函数。
    *   `return count.value >= n`: 只有当当前的帧计数 `count` 大于等于组件的索引 `n` 时，该组件才会被渲染。

#### 模板结合逻辑：

```html
<div v-for="n in 100">
  <!-- 关键点：v-if="defer(n)" -->
  <heavy-comp v-if="defer(n)"></heavy-comp>
</div>
```

*   **第 1 帧**：`count` 为 0。`defer(1)` 到 `defer(100)` 都返回 `false`。**页面渲染 0 个组件**（或者渲染骨架屏/Loading），速度极快，首屏瞬间出现。
*   **第 2 帧**：`update` 执行，`count` 变为 1。Vue 检测到响应式数据变化，触发更新。`defer(1)` 变为 `true`。**第 1 个组件开始渲染**。
*   **第 3 帧**：`count` 变为 2。`defer(2)` 变为 `true`。**第 2 个组件开始渲染**。
*   ...以此类推。

#### 3. 它是如何优化白屏的？

1.  **即时响应（FPS 优先）**：
    *   如果不优化：JS 执行 500ms -> 白屏 500ms -> 一次性显示所有内容。
    *   优化后：JS 执行 5ms -> **显示空容器/Loading** -> 第 16ms 显示第 1 个 -> 第 32ms 显示第 2 个... -> 用户看到内容在不断增加。

2.  **避免长任务阻塞（Yield to Main Thread）**：
    *   `requestAnimationFrame` 允许浏览器在两次 JS 执行之间进行 **UI 绘制**。
    *   这意味着即使后 99 个组件还在排队渲染，用户已经可以看到前几个组件，并且页面是可以滚动的，按钮是可以点击的。

#### 4. 进阶优化：分批次处理 (Batching)

你提供的代码是“一帧渲染一个”。如果组件非常多（比如 1000 个），或者组件其实没那么重，一帧渲染一个可能太慢了（100 个需要 1.6 秒才能完全显示）。

**优化建议**：一帧渲染一组。

修改 `useDefer` 代码：

```javascript
export function useDefer(maxCount = 100) {
  const count = ref(0);
  
  function update() {
    // 每次更新增加的数量，比如一次渲染 10 个
    // 这样 100 个组件只需要 10 帧（约 160ms）就能渲染完
    count.value += 10; 
    
    if (count.value < maxCount) {
       requestAnimationFrame(update);
    }
  }
  
  update();
  
  return function (n) {
    return count.value >= n;
  };
}
```





### 二.虚拟滚动 (Virtual Scroll / Windowing)

这是解决由数据量巨大（如 1 万条以上）导致的页面卡顿的**终极方案**。`useDefer` 解决了初始化慢的问题，但随着你一直往下滚，页面上的 DOM 节点会越来越多（100 -> 1000 -> 10000），页面会越来越卡，内存占用越来越高。

#### 🔴 之前的问题
*   **DOM 爆炸**：渲染 10,000 条数据 = 浏览器里有 10,000 个 `<div>`。
*   **后果**：
    *   浏览器内存飙升。
    *   滚动时 FPS 极低（浏览器重绘压力大）。
    *   DOM 操作极慢。

#### 🟢 优化后 (只渲染可视区域)
*   **原理**：无论你有 10 万条数据，我屏幕一次只能装下 20 条。那我永远只渲染这 20 条（加上缓冲区可能共 30 条）。
*   **实现**：
    1.  计算出一个巨大的“占位容器”高度（比如 10000 * 50px = 500,000px），让滚动条看起来像是真的。
    2.  监听滚动事件 `scroll`。
    3.  根据 `scrollTop` 计算当前应该显示第几条到第几条（比如第 500-520 条）。
    4.  通过 `transform: translateY` 把这 20 条内容定位到可视窗口的位置。

*   **带来的优化**：
    *   **恒定的 DOM 数量**：永远只有几十个 DOM 节点。
    *   **极佳的性能**：无论数据量是一千还是一亿，滚动流畅度几乎没区别。

#### 💻 代码简易实现 (Vue 3 概念版)

```html
<template>
  <!-- 视口容器，有固定高度和 overflow-y: scroll -->
  <div class="viewport" @scroll="onScroll" ref="viewportRef">
    <!-- 幽灵占位层，负责把滚动条撑开到正确高度 -->
    <div class="phantom" :style="{ height: totalHeight + 'px' }"></div>
    
    <!-- 真实渲染列表，通过 absolute 或 transform 定位到可视区 -->
    <div class="list-area" :style="{ transform: `translateY(${offset}px)` }">
      <div v-for="item in visibleData" :key="item.id" class="item">
        {{ item.text }}
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue';

const itemHeight = 50; // 假设每行 50px
const list = ref(new Array(10000).fill(0).map((_, i) => ({ id: i, text: `Item ${i}` })));
const scrollTop = ref(0);

// 总高度
const totalHeight = computed(() => list.value.length * itemHeight);

// 计算当前可视区该显示哪些数据
const visibleData = computed(() => {
  const start = Math.floor(scrollTop.value / itemHeight);
  const end = start + 20; // 假设屏幕能放20个，实际需加 Buffer
  return list.value.slice(start, end); 
});

// 计算偏移量，让列表始终跟着视口走
const offset = computed(() => Math.floor(scrollTop.value / itemHeight) * itemHeight);

const onScroll = (e) => {
  scrollTop.value = e.target.scrollTop;
};
</script>
```

---





### 三. `v-memo` (Vue 3的 新特性)

这是一个类似于 React `useMemo` 的指令，用于手动控制渲染缓存，**这是优化长列表更新性能的利器**。

#### 🔴 之前的问题
假设你有一个 1000 条的列表。当你选中第 1 条时（`selectedId` 变了），虽然只有第 1 条的样式变了，但 Vue 的 Diff 算法默认还是会遍历对比这 1000 个组件的 Props，确认它们是不是真的没变。这个**Diff 的过程**也是有 JS 成本的。

#### 🟢 优化后
告诉 Vue：“只要 `item.id` 和 `selectedId` 这两个值没变，你就完全别管这个 DOM 树，Diff 都不要做，直接跳过”。

*   **带来的优化**：在处理大数据量的交互（如选中、高亮）时，将 JS 运算时间减少 90% 以上。

#### 💻 代码对比

```html
<template>
  <div v-for="item in bigList" :key="item.id">
    
    <!-- 🟢 优化：只有当 item 数据变了，或者选中状态变了，才重新 Diff -->
    <!-- 如果不加 v-memo，每次 bigList 更新，Vue 都要遍历所有子节点做 Diff -->
    <div v-memo="[item.id === selectedId, item.title]">
      <heavy-component 
        :data="item" 
        :is-selected="item.id === selectedId"
        @click="select(item.id)"
      />
    </div>
    
  </div>
</template>
```





### 四. **纯展示类大数据**最有效的优化手段

#### vue2使用冻结的对象 (Object.freeze) 

这是 Vue 2 中针对**纯展示类大数据**最有效的优化手段，在 Vue 3 中依然有效但机制略有不同。

*   **🔴 之前的问题（性能浪费）**
    Vue 的核心机制是响应式（Reactivity）。当你把一个巨大的数组（比如 10 万条新闻列表）交给 Vue 的 `data` 时，Vue 会深度遍历这个对象所有的属性，给它们都加上 `getter` 和 `setter`（Vue 2 使用 `Object.defineProperty`，Vue 3 使用 `Proxy`）。
    *   **痛点**：如果这 10 万条数据只是拿来展示，永远不会修改，那么 Vue 做的这些劫持工作纯属浪费 CPU 和内存。

*   **🟢 解决了什么**
    `Object.freeze()` 是 JS 原生 API，它能让一个对象“冻结”，禁止修改。Vue 在遍历数据时，如果发现对象被冻结了，就会**跳过响应式转换**的过程。

*   **⚡ 带来的优化**
    *   **初始化速度大幅提升**：在大数据量场景下，页面渲染速度可能提升 5-10 倍。
    *   **内存占用大幅降低**：减少了大量的 Observer 实例。

*   **💻 代码对比**

```javascript
// ❌ 之前的写法：Vue 会递归劫持 list 中的每一个属性，哪怕你只是用来读
export default {
  data() {
    return {
      list: []
    }
  },
  async created() {
    const users = await api.getHugeList(); // 假设返回 5万 条数据
    this.list = users; // 🔴 性能卡顿点：Vue 开始疯狂工作进行劫持
  }
}

// ✅ 优化后的写法：告诉 Vue "别碰这些数据，我保证不改它"
export default {
  data() {
    return {
      list: []
    }
  },
  async created() {
    const users = await api.getHugeList();
    // 🟢 冻结对象，Vue 直接跳过劫持，瞬间完成赋值
    this.list = Object.freeze(users); 
  }
}
```

### vue3使用 `shallowRef` 替代 `Object.freeze`**

#### 1. 为什么 Vue 3 还需要优化纯展示数据？

虽然 Vue 3 使用了 **Proxy** 替代了 Vue 2 的 `Object.defineProperty`，性能已经有了质的飞跃（Proxy 是懒代理，只有访问到了才去劫持），但在处理**巨量数据**（如 echarts 坐标点、大屏可视化数据、几十万条列表）时，Vue 3 默认的**深度代理（Deep Reactivity）** 依然会有性能损耗：

1.  **内存占用**：Vue 需要为数据中的每一层对象创建 Proxy 实例。
2.  **初始化开销**：虽然是懒代理，但如果你的组件一上来就读取了深层数据进行渲染，Vue 还是要递归地把它们全变成 Proxy。

#### 2. Vue 3 的更优解：`shallowRef`

`shallowRef`（浅层 Ref）的作用是：**只监听 `.value` 的变化，不劫持 `.value` 内部对象的深层属性。**

#### 💻 代码对比

假设我们有一个从后端拿回来的巨大列表 `hugeList`。

**❌ 写法 1：普通的 `ref` (性能浪费)**

```javascript
import { ref } from 'vue';

// Vue 会把 hugeList 里的每一个对象、每一个属性都变成 Proxy
const list = ref(hugeList); 

// 修改深层属性，触发视图更新（但如果是纯展示，我们根本不需要这个特性）
list.value[0].name = 'New Name'; // 触发重绘
```

**✅ 写法 2：Vue 3 推荐 `shallowRef` (性能极佳)**

```javascript
import { shallowRef } from 'vue';

// Vue 只监听 list.value 这个“壳”的变化
// list.value 内部的数据全是原生 JS 对象，没有 Proxy
const list = shallowRef(hugeList); 

// 🟢 触发更新：只有替换整个 .value 时，视图才会更新
list.value = newHugeList; 

// 🟡 不触发更新：修改内部属性，Vue 不会理你（这正是我们要的）
list.value[0].name = 'New Name'; // 视图不更新，省资源
```

---

#### 3. `Object.freeze` 在 Vue 3 中还有用吗？

**有用，但有副作用。**

在 Vue 3 中，如果你把一个被 `Object.freeze` 的对象传给 `ref()` 或 `reactive()`，Vue 内部会判断：“这个对象不可扩展，我没法给它套 Proxy”。于是 Vue 会放弃代理，直接使用原始对象。

**它和 `shallowRef` 的核心区别在于“可变性”：**

*   **`shallowRef`**：数据**不再响应式**，但数据本身**依然可以修改**（只是改了界面不刷新）。
*   **`Object.freeze`**：数据**不再响应式**，且数据**彻底锁死，不能修改**（修改会报错）。

#### 💻 代码场景对比

**场景：使用 ECharts 渲染地图数据**

```javascript
import { ref, shallowRef, onMounted } from 'vue';

// 假设这是一份 10MB 的 GeoJSON 数据
const heavyData = { type: 'FeatureCollection', features: [...] };

// 方案 A: 使用 Object.freeze (Vue 2 习惯)
const mapDataA = ref(Object.freeze(heavyData));

// 方案 B: 使用 shallowRef (Vue 3 推荐)
const mapDataB = shallowRef(heavyData);

onMounted(() => {
  // 🔴 区别来了：
  
  // 如果我想给数据临时打个标记（比如 visited = true）
  
  // 方案 A 会报错！因为对象被冻结了，JS 不允许修改
  // mapDataA.value.features[0].visited = true; // ❌ Error: Cannot assign to read only property
  
  // 方案 B 允许修改！虽然视图不会自动更新，但数据改成功了，ECharts 读的时候能读到
  mapDataB.value.features[0].visited = true; // ✅ OK
  
  // 初始化图表
  initChart(mapDataB.value);
});
```

#### 案例：Vue 2 如何高效使用大数据的 ECharts

在 Vue 2 中，由于没有 `shallowRef` 这种原生 API，解决大数据量（如 ECharts、高德地图、长列表）导致的卡顿问题，主要靠 **`Object.freeze`** 和 **“非响应式实例属性”** 这两招。

针对 **Vue 2 如何高效使用 ECharts**，这里有一个标准的最佳实践指南，分为 **数据层** 和 **实例层** 两个维度来优化。

---

#### 1. 数据层优化：使用 `Object.freeze`

这是 Vue 2 处理纯展示大数据的核心手段。

*   **原理**：Vue 2 初始化 `data` 时，会递归遍历所有属性调用 `Object.defineProperty`。如果检测到对象被 `Object.freeze` 冻结了，Vue 就会**跳过**这个遍历过程。
*   **效果**：对于几万条的图表数据，初始化时间从 几百毫秒 -> 几毫秒。

#### ❌ 错误写法 (Vue 2)

```javascript
export default {
  data() {
    return {
      chartData: [] // 准备放大量数据
    }
  },
  methods: {
    async loadData() {
      const res = await api.getData(); // 假设返回 10MB 的数据
      // 🔴 灾难现场：Vue 试图把这 10MB 数据的每一层都加上 getter/setter
      // 页面会卡死几秒钟
      this.chartData = res; 
    }
  }
}
```

#### ✅ 正确写法 (Vue 2)

```javascript
export default {
  data() {
    return {
      chartData: []
    }
  },
  methods: {
    async loadData() {
      const res = await api.getData();
      // 🟢 优化：冻结对象。Vue 直接放弃劫持，性能瞬间起飞
      // 注意：this.chartData 依然可以被重新赋值，但 res 内部属性不能改
      this.chartData = Object.freeze(res); 
    }
  }
}
```

---

#### 2. 实例层优化：ECharts 实例千万别放 `data` 里

这是新手在 Vue 2 中使用 ECharts **最容易踩的坑**，会导致内存溢出或浏览器崩溃。

*   **问题**：ECharts 的实例对象（`echarts.init` 返回的那个对象）非常巨大，且内部包含大量循环引用和 DOM 节点引用。
*   **后果**：如果你把它放在 `data` 里，Vue 2 会试图递归劫持这个庞然大物，导致：
    1.  初始化极慢。
    2.  报错（RangeError: Maximum call stack size exceeded）。
    3.  内存泄漏。

#### ❌ 错误写法 (致命)

```javascript
export default {
  data() {
    return {
      myChart: null // 🔴 把它声明在 data 里，试图让它响应式
    }
  },
  mounted() {
    // Vue 会试图深度劫持这个复杂的实例
    this.myChart = echarts.init(document.getElementById('main')); 
  }
}
```

#### ✅ 正确写法 (挂载到 `this`)

**直接挂载到组件实例上 (`this.xxx`)，而不是 `data` 里。** 这样它就是普通的 JS 属性，完全脱离 Vue 的响应式系统。

```javascript
export default {
  data() {
    return {
      // 只放业务数据
    }
  },
  mounted() {
    // 🟢 正确：直接挂在 this 上
    // 这样 myChart 只是一个普通变量，Vue 不会去碰它
    this.myChart = echarts.init(this.$refs.chartRef);
    
    this.initChart();
  },
  beforeDestroy() {
    // 记得销毁，防止内存泄漏
    if (this.myChart) {
      this.myChart.dispose();
      this.myChart = null;
    }
  },
  methods: {
    initChart() {
      // 使用
      this.myChart.setOption({...});
    }
  }
}
```

---

#### 总结 Vue 2 优化策略

1.  **数据层**：只要这堆数据是**“只读”**的（比如接口拿回来直接塞给 ECharts，或者只用来 `v-for` 展示），无脑套上 **`Object.freeze()`**。
2.  **实例层**：所有第三方库的复杂实例（ECharts 实例、地图实例、富文本编辑器实例），**永远不要写在 `data` 里**。直接在 `created` 或 `mounted` 里 `this.xxx = xxx` 赋值。









### 五. 使用计算属性 (Computed)

*   **🔴 之前的问题（重复计算）**
    有些开发者习惯直接在模板里写逻辑，或者用 `methods`。
    *   **痛点**：模板内的表达式或 `methods`，在**每次页面重绘时都会重新执行**。哪怕数据没变，它也要再算一遍。

*   **🟢 解决了什么**
    `computed` 基于它们的**响应式依赖进行缓存**。只有相关的数据发生了改变，它才会重新求值。

*   **⚡ 带来的优化**
    *   **减少 CPU 消耗**：对于复杂的逻辑（比如过滤庞大的数组、复杂的数学计算），避免了大量无意义的重复运算。

*   **💻 代码对比**

```javascript
// ❌ 性能较差：每次页面有任何变化（哪怕是其他无关组件变化），这个函数都会执行
methods: {
  getFilteredList() {
    console.log('我又计算了一次'); // 疯狂打印
    return this.hugeList.filter(item => item.active);
  }
}

// ✅ 优化写法：只有当 hugeList 发生变化时，才会重新计算
computed: {
  filteredList() {
    console.log('数据变了我才算'); // 打印次数很少
    return this.hugeList.filter(item => item.active);
  }
}
```







### 六。 非实时绑定的表单项 (v-model.lazy)

*   **🔴 之前的问题（频繁触发）**
    默认的 `v-model` 监听的是 `input` 事件。
    *   **痛点**：用户在输入框打字，每敲一下键盘，数据就更新一次，视图就可能重绘一次。如果这个输入框关联了一个复杂的搜索逻辑，用户还没输完，后台已经发起了 10 次搜索请求。

*   **🟢 解决了什么**
    `.lazy` 修饰符把监听事件改成了 `change`（即失去焦点或回车时才更新）。

*   **⚡ 带来的优化**
    *   **减少逻辑执行频率**：从“每输入一个字执行一次”变为“输完了一口气执行一次”。
    *   **提升输入流畅度**：避免输入过程中因复杂渲染导致的卡顿。

*   **💻 代码对比**

```html
<!-- ❌ 默认：搜 "vue" -->
<!-- 输v -> 触发搜索 -> 输u -> 触发搜索 -> 输e -> 触发搜索 -->
<input v-model="searchQuery" />

<!-- ✅ 优化： -->
<!-- 输 "vue" -> 回车/点空白处 -> 触发一次搜索 -->
<input v-model.lazy="searchQuery" />
```





### 七. 保持对象引用稳定

这是一个比较高级的优化点，主要涉及子组件的无效更新。

*   **🔴 之前的问题（Props 引用变动）**
    在父组件给子组件传参时，如果直接在模板里写对象或数组字面量。
    *   **痛点**：每次父组件更新，都会创建一个**新的**对象 `{ color: 'red' }`。虽然内容一样，但内存地址（引用）变了。子组件会认为“Props 变了”，从而触发不必要的重新渲染。

*   **🟢 解决了什么**
    把变量定义在 `data` 或 `const` 中，保证引用地址不变。

*   **⚡ 带来的优化**
    *   配合 `v-memo` 或 Vue 2 的纯组件，可以彻底避免子组件的无意义渲染。

*   **💻 代码对比**

```html
<!-- ❌ 糟糕的写法 -->
<!-- 每次 Parent 更新，style 都是一个新的对象 { color: 'red' } -->
<!-- Child 组件会检测到 props 变化，被迫更新 -->
<ChildComponent :style="{ color: 'red' }" />

<!-- ✅ 优化写法 -->
<script>
const staticStyle = { color: 'red' }; // 永远是同一个引用

export default {
  data() {
    return {
      staticStyle // 或者放在 data 里
    }
  }
}
</script>

<template>
  <ChildComponent :style="staticStyle" />
</template>
```

#### 详细解释

#### 1. 核心概念：值 vs 引用

在 JS 中，对象（Object）和数组（Array）是引用类型。

* **字面量每次都是新的**：
  当你写 `{ a: 1 }` 时，JS 引擎会在内存里开辟一块新空间。

  ```javascript
  console.log({ a: 1 } === { a: 1 }); // false
  // 虽然内容一样，但它们住在内存的不同“地址”
  ```

* **保持引用稳定**：

  ```javascript
  const obj = { a: 1 };
  console.log(obj === obj); // true
  // 地址没变
  ```

---

#### 2. 🔴 灾难现场：隐形性能杀手

假设你有一个渲染非常耗时的子组件 `<HeavyChart />`。

#### 场景还原

```html
<template>
  <div class="parent">
    <!-- 触发父组件更新的按钮 -->
    <button @click="count++">点击刷新父组件 {{ count }}</button>

    <!-- 🔴 错误写法：直接在模板里写对象字面量 -->
    <!-- 每次父组件 count 变化引发重绘时， -->
    <!-- 都会凭空创建一个新的 { theme: 'dark' } 对象传给子组件 -->
    <HeavyChart :config="{ theme: 'dark' }" />
  </div>
</template>
```

#### 此时发生了什么？

1.  用户点击按钮，`count` 变了，**父组件触发更新（Re-render）**。
2.  父组件的 `render` 函数重新执行。
3.  执行到 `<HeavyChart>` 这一行时，JS 引擎遇到了 `{ theme: 'dark' }`。
4.  **JS 引擎：** “这是一个对象字面量，我要在内存地址 `0x1001` 创建一个新对象。”
5.  **Vue 对比子组件 Props：**
    *   旧 Props：`config` 指向地址 `0x0099`。
    *   新 Props：`config` 指向地址 `0x1001`。
    *   **判断结果：** `0x0099 !== 0x1001`，**Props 变了！**
6.  **后果：** `<HeavyChart>` 被迫强制重新渲染，虽然它的配置内容压根没变。

---

#### 3. 🟢 优化方案：保持引用稳定

我们要做的就是把这个对象“提出来”，让它有一个固定的“户口地址”。

#### 写法 A：如果是纯静态配置（提至组件外）

如果这个配置永远不会变，直接扔到 `script` 外面或者是 `const` 里。

```javascript
<script>
// 🟢 只有一个内存地址，永远不变
const staticConfig = { theme: 'dark' };

export default {
  data() {
    return { count: 0, staticConfig } // 挂载到 data 上以便模板访问
  }
}
</script>

<template>
  <!-- 无论父组件怎么刷新，staticConfig 的引用地址永远是同一个 -->
  <HeavyChart :config="staticConfig" />
</template>
```

#### 写法 B：如果依赖 data（使用 Computed）

如果配置依赖组件内的变量，使用 `computed`。

```javascript
<script>
export default {
  data() {
    return { count: 0, theme: 'dark' }
  },
  computed: {
    // 🟢 只有当 this.theme 变化时，chartConfig 才会重新生成新对象
    // 如果只是 this.count 变化，chartConfig 会返回缓存的旧引用
    chartConfig() {
      return { theme: this.theme };
    }
  }
}
</script>

<template>
  <HeavyChart :config="chartConfig" />
</template>
```

---

#### 4. ⚡ 为什么说要配合 `v-memo` 或纯组件？

这里有一个 Vue 机制的细节点：

在 Vue 2/3 的默认组件更新策略中，**只要父组件更新了，子组件通常也会被动进入 update 流程**（去检查 props 变没变）。

*   **如果引用不稳定（错误写法）**：子组件检查 Props 发现变了 -> **肯定更新**。
*   **如果引用稳定（优化写法）**：子组件检查 Props 发现没变 -> **尝试跳过更新**。

但是，为了做到**“彻底”**跳过 Diff 过程（连检查都省了），我们需要更强的手段。

#### 在 Vue 3 中配合 `v-memo`

`v-memo` 允许我们手动告诉 Vue：“只要这个数组里的依赖没变，这块 DOM（包括子组件）连 Diff 都不要做，直接用上次的缓存”。

```html
<template>
  <div class="parent">
    <button @click="count++">刷新 {{ count }}</button>

    <!-- 🟢 终极优化 -->
    <!-- 依赖项是 staticConfig -->
    <!-- 因为 staticConfig 的引用永远没变，Vue 直接跳过这段 DOM 的 Diff -->
    <!-- 子组件连 "Props是否变化" 的检查步骤都省了 -->
    <div v-memo="[staticConfig]">
      <HeavyChart :config="staticConfig" />
    </div>
  </div>
</template>
```

**如果这里你用了 `{ theme: 'dark' }` (错误写法) 配合 `v-memo` 会怎样？**

*   `v-memo` 检查依赖数组：`[ { theme... } ]`。
*   因为是新对象，`v-memo` 认为依赖变了，缓存失效。
*   **优化失败**。所以说，`v-memo` 生效的前提是 **引用稳定**。

---

#### 总结

**一句话口诀：**
**“父组件模板里，别随手写花括号 `{}` 传对象，那是给子组件投毒。”**





### 八. 使用 keep-alive

*   **🔴 之前的问题（状态丢失与重复挂载）**
    在做 Tab 切换或路由跳转时，组件会被销毁。
    *   **痛点**：用户切到“页面B”，再切回“页面A”。页面A 之前的滚动位置、填写的表单全都丢了，而且还需要重新请求接口、重新渲染 DOM，体验差且慢。

*   **🟢 解决了什么**
    `keep-alive` 是 Vue 的内置组件，它能将包裹在其中的组件实例**缓存在内存中**，而不是销毁它们。

*   **⚡ 带来的优化**
    *   **秒开**：再次进入页面时，直接从内存读取 DOM 状态，无需重新执行 `created`、`mounted` 等生命周期，也无需重新请求网络。
    *   **体验好**：保留了用户的操作状态（如滚动位置）。

*   **💻 代码对比**

```html
<!-- ❌ 普通路由：每次切换 /home 和 /about，组件都会销毁重建 -->
<router-view></router-view>

<!-- ✅ 优化：组件会被缓存 -->
<keep-alive>
  <router-view></router-view>
</keep-alive>
```



### 九. vue2中使用函数式组件 

只针对 Vue 2 项目。

*   **🔴 之前的问题（组件实例的开销）**
    在 Vue 2 中，创建一个普通的组件（Stateful Component）是非常“昂贵”的。
    
    *   **痛点**：即使你只是写一个简单的标签展示组件（比如 `<my-heading>标题</my-heading>`），Vue 内部也要为它做全套服务：
        1.  创建 Vue 实例（`this`）。
        2.  初始化生命周期钩子（`created`, `mounted` 等）。
        3.  初始化响应式数据监听（`data`, `computed`）。
    *   **后果**：如果你在一个长列表里渲染 1000 个这种简单组件，就等于创建了 1000 个 Vue 实例，内存和 CPU 瞬间爆炸。
    
*   **🟢 解决了什么**
    函数式组件是一种**没有状态（无 `data`）**、**没有实例（无 `this`）**的组件。它仅仅是一个接受 `props` 并返回 VNode 的函数。

*   **⚡ 带来的优化**
    
    *   **渲染速度飙升**：因为跳过了实例初始化过程，在 Vue 2 中，函数式组件的渲染开销比普通组件低得多。
    *   **内存占用极大减少**：没有了数以千计的 Vue 实例对象。
    
    > **注意**：在 **Vue 3** 中，普通组件已经经过了底层重写和极大优化，函数式组件的性能优势已经微乎其微，甚至官方推荐直接用普通组件。所以这主要是一个 Vue 2 的优化技巧。
    
    
    
    函数式组件是 Vue 2 中一种非常**硬核**且**古老**的性能优化写法。虽然在 Vue 3 中已经不再需要这样写了，但理解它对于看懂 Vue 2 的老项目源码或 UI 组件库（如 ElementUI 源码）至关重要。
    
    ### 1. 核心概念：什么是“函数式组件”？
    
    普通组件（Stateful Component）是有“灵魂”的：它有 `data`（状态）、有 `computed`、有生命周期（`created`, `mounted`），还有一个指向自己的 `this` 实例。
    
    **函数式组件（Functional Component）是“没灵魂”的工具人：**
    
    *   **没有 `this`**：它没有组件实例。
    *   **没有状态**：没有 `data`，数据全靠外部传入。
    *   **没有生命周期**：没有 `created`, `mounted` 等钩子。
    
    **它的本质就是一个函数**：`view = f(props)`。给你什么数据，你就渲染什么 DOM，干完就走，不留痕迹。
    
    ---
    
    ### 2. 代码逐行详解
    
    ```html
    <!-- 1. functional 关键字 -->
    <template functional>
      
      <!-- 2. 事件绑定：listeners.click -->
      <div class="btn" @click="listeners.click">
        
        <!-- 3. 插槽 -->
        <slot></slot>
      
      </div>
    </template>
    ```
    
    #### ① `<template functional>`
    *   **含义**：告诉 Vue 编译器，“这只是一个用来渲染 HTML 的函数，别给我创建 Vue 实例（Vue Instance）”。
    *   **优化点**：在 Vue 2 中，创建一个组件实例的开销是很大的（初始化响应式系统、设置事件中心等）。加上 `functional` 后，渲染开销极大降低，**渲染速度通常快 30%~50%**。
    
    #### ② `@click="listeners.click"` (最难理解的点)
    在普通组件里，我们通常写 `@click="$emit('click')"`。为什么这里变了？
    
    *   **普通组件**：父组件绑定 `@click` -> 放在子组件的 `_events` 里 -> 子组件调用 `$emit` 触发。
    *   **函数式组件**：没有实例，所以**没有 `$emit` 方法**！
        *   父组件传进来的所有事件监听器（`@click`, `@hover`），都会被打包到一个叫 **`listeners`** 的对象里传进来。
        *   `listeners.click` 其实就是父组件传进来的那个函数。
        *   `@click="listeners.click"` 的意思是：当 div 被点击时，**直接执行**父组件传进来的那个 click 函数。
    
    #### ③ Props 和数据去哪了？
    虽然这段代码里没写 Props，但在函数式组件的模板中，你可以直接访问以下变量（Vue 自动注入的）：
    *   **`props`**: 包含所有传递的属性。
    *   **`listeners`**: 包含所有绑定的事件。
    *   **`data`**: 包含 `class`、`style`、`attrs` 等数据对象。
    *   **`slots`**: 插槽内容。
    
    ---
    
    ### 3. 普通组件 vs 函数式组件 (代码对比)
    
    假设我们要封装一个简单的标题组件。
    
    #### 🔴 子普通组件 (Standard)
    开销大，功能全。
    
    ```html
    <template>
      <div @click="handleClick">
        {{ title }}
      </div>
    </template>
    
    <script>
    export default {
      props: ['title'],
      // Vue 内部要执行 new VueComponent()，初始化 data, computed, watch...
      methods: {
        handleClick() {
          this.$emit('click'); // 依赖 this 上下文
        }
      }
    }
    </script>
    ```
    
    #### 🟢 子函数式组件 (Functional)
    开销极小，只负责渲染。
    
    ```html
    <template functional>
      <!-- 没有 this，直接从参数里拿 props 和 listeners -->
      <div @click="listeners.click">
        {{ props.title }}
      </div>
    </template>
    
    <script>
    export default {
      // 必须显式声明 props，否则会在 data.attrs 里
      props: ['title']
    }
    </script>
    ```
    
    ---
    
    ### 4. 为什么现在（Vue 3）不怎么提了？
    
    *   **在 Vue 2 中**：函数式组件是**性能优化的神器**。凡是纯展示的组件（Icon、Button、Tag），都建议写成函数式。
    *   **在 Vue 3 中**：**普通组件已经被优化得非常快了**。
    



#### 只负责展示 UI 这句话的真正意思

这是一个非常敏锐且关键的问题！你触碰到了 **“展示型组件（UI 组件）”** 和 **“业务型组件（容器组件）”** 之间最核心的区别。

**一句话回答：**
**“负责展示 UI” 并不代表它是“死”的。它依然可以响应用户操作，关键在于它自己不处理业务逻辑，只是把操作“转告”给父组件。**

这里的 `@click="listeners.click"` **不是在处理逻辑，而是在“传声”**。

让我们详细拆解一下这个误区：

##### 1. 什么是“只负责展示 UI”？

**误区：** “展示 UI” = 静态的 HTML，不能动，不能点。
**真相：** “展示 UI” = **无状态（Stateless）**。即组件内部不持有数据，不修改数据。

一个按钮（Button）是典型的 UI 组件。
*   它长什么样？（红色、圆角） -> **这是 UI**。
*   它能被点击吗？（能） -> **这也是 UI 的一部分（交互）**。
*   点击后发生什么？（是提交表单？还是弹窗？还是跳转？） -> **这是业务逻辑**。

##### 2. 为什么函数式组件里可以写点击事件？

请注意看代码里的 `listeners.click` 到底做了什么：

```html
<!-- 函数式组件 MyButton.vue -->
<template functional>
  <div @click="listeners.click"> <!-- 👈 关键在这里 -->
    {{ props.title }}
  </div>
</template>
```

*   **它做了什么？** 它只是当了一个**“二传手”**。用户点了 `div`，它立刻执行父组件传进来的 `click` 方法。
*   **它没做什么？** 它没有修改 `this.count`，没有发起 `axios` 请求，没有改变自己的颜色。它甚至不知道点击后会发生什么。

**因为它没有“私自”处理任何事情，只是把用户的动作原封不动地交给了父亲，所以它依然是纯粹的“展示组件”。**

##### 3. 对比：什么样的点击事件**不能**用函数式组件？

如果这个点击事件包含了**组件内部的状态变更**，那它就不能是函数式组件了。

##### ❌ 场景：计数器 (这是业务逻辑)
这个组件自己要在内部维护 `count`，点击一次自己加 1。函数式组件**做不到**，因为它没有 `this`，也没有 `data`。

```html
<!-- 普通组件 (不能用 functional) -->
<template>
  <div @click="count++"> <!-- 👈 只有普通组件能做，因为它有 data -->
    当前数字：{{ count }}
  </div>
</template>
<script>
export default {
  data() { return { count: 0 } } // 函数式组件没有 data
}
</script>
```

##### ✅ 场景：删除按钮 (这是 UI 交互)
这个组件只是一个漂亮的垃圾桶图标。点击它，具体删哪个文件？删完怎么提示？都是父组件的事。

```html
<!-- 函数式组件 -->
<template functional>
  <div class="trash-icon" @click="listeners.delete"> <!-- 👈 只是通知父组件 -->
    🗑️ 删除
  </div>
</template>
```





### 十. 打包体积优化 (Bundle Size Optimization)

*   **🟢 解决了什么**
    通过 **路由懒加载 (Code Splitting)** 和 **Tree Shaking**，把巨石打碎。
    *   **按需加载**：只有当用户真正点击了“个人中心”的路由时，才去下载那部分代码。

#### 场景 A：路由懒加载

```javascript
// ❌ 之前的写法：静态导入
// 无论用户是否访问 Dashboard，这段代码都会被打包进主包
import Dashboard from './views/Dashboard.vue';

const routes = [
  { path: '/dashboard', component: Dashboard }
]


// ✅ 优化后的写法：动态导入 (Dynamic Import)
// Webpack/Vite 遇到 import() 会自动把这个文件单独切割成一个 js 文件
const routes = [
  { 
    path: '/dashboard', 
    component: () => import('./views/Dashboard.vue') 
  }
]
```

#### 场景 B：第三方库按需引入 (以 ECharts 或组件库为例)

```javascript
// ❌ 之前的写法：引入整个库
// 哪怕你只画了一个饼图，它把地图、K线图、雷达图的所有代码都打包进来了
import * as echarts from 'echarts'; 
// 结果：包体积 + 2MB 😱


// ✅ 优化后的写法：按需引入 (Tree Shaking)
// 只有被引用的模块才会被打包
import { use } from 'echarts/core';
import { CanvasRenderer } from 'echarts/renderers';
import { PieChart } from 'echarts/charts';

use([CanvasRenderer, PieChart]);
// 结果：包体积 + 50KB 🥳
```

---





# 控制台的Performance 

【使用Chrome Performance进行性能调优】 https://www.bilibili.com/video/BV1Pr4y1N7QZ/?share_source=copy_web&vd_source=e123aa4d7cba8e52abde344187dda6f0

### 主要是用里面的main 和  Call Tree

### 操作过程

- 先点左中间那个原点，开始记录
- 然后自己执行动画
- 停止，不用等进度条加载完，直接点击停止就行
- 运行时建议开启无痕模式， 因为浏览器的插件产生的性能也算

### 里面参数介绍

- 一次事件循环中，只会触发一次重排（大概率）

- Task 是 当前任务队列 ----- 一般是main的最上面的灰色样式
  - 执行到宏任务 代码，会开启另一个task，微任务不会
