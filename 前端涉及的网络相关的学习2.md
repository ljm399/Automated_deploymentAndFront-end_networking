## 一。`filename: '[name].[contenthash].js'`   和 强缓存

---

### 一、 历史痛点：固定文件名时代 (没它之前)

在早期的 Web 开发中，打包出来的文件名通常是固定的，比如 `bundle.js` 或 `style.css`。

#### 1. 场景还原
假设你的服务器（Nginx）为了让网页加载快，开启了 **强缓存 (Cache-Control: max-age=31536000)**。这意味着：浏览器会把 `bundle.js` 存在本地硬盘里，一年内都不会再去向服务器请求。

#### 2. 灾难发生
你今天修复了一个紧急的 Bug，重新打包发布了，服务器上的文件确实更新了（代码变了），但文件名依然叫 `bundle.js`。

*   **老用户访问**：浏览器一看：“我要请求 `bundle.js`？我有缓存啊，还有效呢！” —— 于是直接用旧文件。**后果：用户看到的 Bug 还在，死活更新不了。**
*   **新用户访问**：下载了新的 `bundle.js`，没问题。

#### 3. 以前的笨办法
为了解决这个问题，以前我们会在 HTML 里手动加版本号（查询参数）：
```html
<script src="bundle.js?v=1.0"></script>
<!-- 发布新版改成 -->
<script src="bundle.js?v=1.1"></script>
```
**缺点**：有些 CDN 或代理服务器会忽略 `?` 后面的参数，依然强制缓存旧文件，这种方法不可靠。

---

### 二、 解决方案：文件名指纹 (Contenthash)

`[name].[contenthash].js` 的出现，就是为了完美解决这个问题。

*   **`[name]`**：入口文件的名字（比如 `main`、`vendor`）。
*   **`[contenthash]`**：**根据文件内容生成的哈希值（指纹）**。

#### 1. 它的原理
构建工具（Webpack）在打包时，会去计算这个文件的**内容**。
*   如果内容变了（哪怕改了一个空格），计算出来的 hash 值（一串乱码）就会变。
*   如果内容没变，hash 值就绝对不会变。

#### 2. 它是怎么解决问题的？
**利用“覆盖式发布”转变为“非覆盖式发布”。**

*   **版本 1.0**：打包出 `main.a1b2c3.js`。用户浏览器缓存了它。
*   **版本 1.1**（你改了代码）：打包出 `main.x9y8z7.js`。
*   **结果**：
    *   HTML 里的引用自动变成了 `<script src="main.x9y8z7.js"></script>`。
    *   浏览器一看：文件名变了！这肯定是个新文件，缓存里没有。
    *   **浏览器强制去服务器下载新文件。**

---

### 三、 带来了什么优化？(面试核心)

面试官问：“为什么不用 `[hash]` 而要用 `[contenthash]`？”
这里涉及到 **缓存利用率** 的极致优化。

#### 1. 颗粒度控制 (Granularity)
Webpack 有三种 hash：
*   **`[hash]`**：项目级。只要项目里有一个文件改了，**所有**打包出来的文件名都会变。（太浪费了）
*   **`[chunkhash]`**：入口级。同一个入口的文件变了才变。
*   **`[contenthash]`**：**文件级（最优）**。谁变了谁才变。

#### 2. 场景演示 (带来的优化)
假设你的项目代码分离成了两部分：
1.  **`main.js`**：你的业务代码（经常改）。
2.  **`vendor.js`**：第三方库，如 Vue、Axios（几乎不改）。

**如果你用 `[hash]` (差)：**
你只改了一行业务代码，结果 `main.js` 和 `vendor.js` 的名字都变了。用户不得不把巨大的 `vendor.js` 也重新下载一遍。

**如果你用 `[contenthash]` (优)：**
你改了业务代码：
*   `main.js` 的内容变了 -> 文件名变成 `main.new123.js` -> 用户**重新下载**。
*   `vendor.js` 的内容没变 -> 文件名依然是 `vendor.old456.js` -> 用户**命中缓存**（直接用本地的），秒开。

---

### 四、 代码实战

#### Webpack 配置
```javascript
const path = require('path');

module.exports = {
  entry: {
    main: './src/index.js',
    vendor: ['vue', 'axios'] // 第三方库单独打包
  },
  output: {
    // 🔴 核心配置在这里
    // [name] 会被替换为 main 或 vendor
    // [contenthash:8] 取哈希值的前8位
    filename: '[name].[contenthash:8].js',
    path: path.resolve(__dirname, 'dist'),
    // 每次打包前清除 dist 目录，防止旧的 hash 文件堆积
    clean: true 
  }
};
```

#### 打包结果对比

**第一次打包：**
```text
dist/
  index.html      (引用 main.ab12.js 和 vendor.cd34.js)
  main.ab12.js    (业务代码)
  vendor.cd34.js  (Vue源码)
```

**你修改了 `index.js` 里的业务逻辑，第二次打包：**
```text
dist/
  index.html      (引用 main.ef56.js 和 vendor.cd34.js)
  main.ef56.js    (变了！用户重新下载)
  vendor.cd34.js  (没变！用户直接用缓存)
```

### 💡 总结

> "`filename: '[name].[contenthash].js'` 是前端工程化中做 **长效缓存 (Long Term Caching)** 的标准手段。
>
> 在没有它之前，我们需要手动管理版本号来对抗浏览器的强缓存，容易出错且效率低。
>
> 它的核心作用是**根据文件内容生成唯一指纹**。文件内容变了，文件名就变；内容没变，文件名就不变。
>
> 带来的最大优化是**最大化利用浏览器缓存**。特别是配合代码分割（Code Splitting）时，我们可以把业务代码和第三方库（Vendor）分开打包。当我们修改业务代码时，第三方库的文件名保持不变，用户浏览器可以直接读取本地缓存，无需重新下载，从而极大地提升了页面加载速度。"



## 二。content-type 

---

### 第一部分：`Content-Type` 常见值与作用

`Content-Type`（内容类型），也叫 **MIME 类型**。
**作用：** 它告诉接收方（浏览器或服务器）：**“我发给你的是个什么东西，请你用对应的方式去解析它。”**

如果没有它，服务器收到一堆 `010101` 的二进制流，根本不知道该把它当成图片看，还是当成文字看。

#### 必背的 3 个核心值（Request Header）

在前端给后端发请求（POST/PUT）时，这三个最常用：

| 值 (Value)   | **application/json**      | **application/x-www-form-urlencoded** | **multipart/form-data**         |
| :----------- | :------------------------ | :------------------------------------ | :------------------------------ |
| **简称**     | **JSON 格式**             | **表单格式 (Query String)**           | **文件上传格式**                |
| **数据样子** | `{"name":"Tom","age":18}` | `name=Tom&age=18`                     | (二进制流，包含分界符 boundary) |
| **对应 JS**  | `JSON.stringify(data)`    | `qs.stringify(data)`                  | `new FormData()`                |
| **场景**     | **90% 的现代接口**        | 老旧后端、简单数据提交                | **上传图片/视频/文件**          |
| **Axios**    | 默认就是这个              | 需配合 `qs` 库转换                    | 传 FormData 对象时自动设置      |

#### 其他常见值（Response Header）

浏览器收到服务器响应时，根据这些值决定怎么渲染：

1.  **`text/html`**：告诉浏览器这是网页，请渲染成页面。
2.  **`text/css`**：告诉浏览器这是样式表。
3.  **`application/javascript`**：告诉浏览器这是 JS 脚本。
4.  **`image/png`、`image/jpeg`**：告诉浏览器这是图片。
5.  **`application/octet-stream`**：**二进制流**（不知道具体是啥）。通常浏览器收到这个就会**触发下载**。

---

## 三。MD5 和 `filename: '[name].[contenthash].js'` 的区别

这个问题其实是在问：**“算法”和“应用场景”的区别**。

**结论先行：**

*   **MD5** 是一个**算法**（工具）。
*   **`[contenthash]`** 是 Webpack 的一个**配置策略**（应用）。
*   **关系：** Webpack 的 `[contenthash]` **底层就是用 MD5**（或其他类似的哈希算法）算出来的。

我们可以把它们比作：**“尺子”** 和 **“衣服的尺码”**。

*   MD5 是那把“尺子”。
*   `[contenthash]` 是用尺子量出来的“衣服尺码”。

#### 1. MD5 (Message-Digest Algorithm 5)

*   **本质**：一种**哈希算法**。
*   **作用**：把任意长度的数据，压缩成一个固定的 32 位字符串（指纹）。
*   **特点**：
    *   内容变了，MD5 必变。
    *   不可逆（算不出原文）。
*   **场景**：
    *   密码加密存储。
    *   接口签名（Sign）。
    *   文件完整性校验（秒传）。

#### 2. `[name].[contenthash].js` (Webpack 配置)

*   **本质**：前端工程化的**缓存策略**。
*   **作用**：给打包出来的 JS/CSS 文件名加一个“指纹后缀”。
*   **生成逻辑**：
    1.  Webpack 把你的 `index.js` 文件内容拿出来。
    2.  **调用 MD5 算法**（或者 xxhash）对内容进行计算。
    3.  算出一个哈希值，比如 `a1b2c3d4`。
    4.  把这个值拼接到文件名里：`index.a1b2c3d4.js`。

#### 3. 为什么要区分？(面试考点)

面试官问区别，其实想看你是否理解 **“Hash 带来的缓存优化”**。

*   **MD5** 只是一个数学工具，哪里都能用。
*   **`[contenthash]`** 是专门为了解决**浏览器强缓存更新问题**而设计的。

**对比总结：**

| 维度         | MD5                            | `[contenthash]`                        |
| :----------- | :----------------------------- | :------------------------------------- |
| **是什么**   | **算法** (数学公式)            | **Webpack 占位符** (配置项)            |
| **底层关系** | 它是底层实现                   | 它**使用**了 MD5 来计算结果            |
| **关注点**   | 数据的**唯一性**和**不可篡改** | 文件的**版本控制**和**浏览器缓存**     |
| **变化规律** | 只要内容变，结果就变           | 只有当**该模块的内容**变了，文件名才变 |

### 💡 举个代码例子说明关系

Webpack 内部大概是这样工作的（伪代码）：

```javascript
const crypto = require('crypto'); // Node 自带加密库

// 假设这是你的源码
const fileContent = "console.log('Hello World')";

// 1. 使用 MD5 算法
const hash = crypto.createHash('md5')
    .update(fileContent)
    .digest('hex')
    .slice(0, 8); // 取前8位，比如 '7d77e206'

// 2. 生成最终文件名 (这就是 [contenthash] 的由来)
const filename = `main.${hash}.js`; 

console.log(filename); // main.7d77e206.js
```

**一句话总结：**
**MD5 是计算指纹的工具，`[contenthash]` 是利用这个工具给前端资源文件打上指纹，从而实现“文件内容不变，缓存永久有效”的性能优化。**





## 四。MD5

这是关于**数据完整性**和**早期安全**的一个里程碑式算法。

虽然现在 **MD5** 在加密领域已经被视为“不再安全”（容易被碰撞破解），但在**文件校验**、**数据标识**等非安全领域，它依然是业界的霸主。

我们按照 **“历史痛点” -> “解决方案” -> “带来的优化”** 来拆解。

---

### 一、 历史痛点 (没它之前)

在 MD5（1992年）普及之前，主要面临三个大问题：

#### 1. 弱不禁风的数据校验 (CRC32)

*   **问题**：以前下载文件或传输数据，常用 **CRC32** 做校验。CRC32 主要防的是“网络传输噪音”（比如比特翻转），但防不住“人为篡改”。
*   **痛点**：黑客可以修改文件内容，同时轻易地伪造出一个相同的 CRC32 值。你以为你下载的是正版软件，其实里面被植入了病毒，但校验码却是一样的。

#### 2. 裸奔的密码存储

*   **问题**：早期网站数据库里，用户的密码是**明文存储**的（直接存 "123456"）。
*   **痛点**：一旦数据库被脱库（被黑客下载），所有用户的密码直接泄露，没有任何防线。

#### 3. 不定长数据的索引困难

*   **问题**：如果你要给一篇 1万字的文章生成一个 ID，或者给一个 1GB 的视频生成一个唯一标识，怎么做？用文件名？文件名容易重名。用内容？内容太长了，数据库存不下。

---

### 二、 它出现解决了什么？

MD5 (Message-Digest Algorithm 5) 的出现，提供了一个**“数字指纹”**的概念。

#### 1. 确定的“数字指纹” (不可篡改性)

*   **解决**：无论输入是 "hello" 还是 "100GB 的电影"，MD5 都会把它压缩成一个 **128位（32个字符）** 的固定长度字符串。
*   **雪崩效应**：只要改动文件里的**一个标点符号**，算出来的 MD5 值就会天差地别。这完美解决了**完整性校验**的问题。

#### 2. 单向散列 (不可逆性)

*   **解决**：MD5 是单向的。你给我原文，我能算出 Hash；但给你 Hash，你（在理论上）算不出原文。
*   **密码保护**：数据库存 `e10adc3949ba59abbe56e057f20f883e` (123456 的 MD5)。黑客拿到了也看不懂原文是什么（虽然现在可以通过彩虹表撞库，但在当年是非常安全的）。

---

### 三、 带来了什么优化？

#### 1. 秒传功能的实现 (存储优化)

*   **场景**：网盘上传。
*   **优化**：当你上传一部电影时，前端先算一下文件的 MD5 发给服务器。服务器一查数据库：“诶，这个 MD5 我库里已经有了（别人传过）”。
*   **结果**：服务器直接告诉你“上传成功”，根本不需要真的传输这 1GB 的数据。极大地节省了带宽和存储空间。

#### 2. 前端静态资源缓存 (HTTP优化)

*   **场景**：Webpack 打包文件名 `main.a1b2c3.js`。
*   **优化**：这个 `a1b2c3` 就是文件内容的 MD5（或类似的 Hash）。内容不变，Hash 不变，浏览器就一直用缓存，不用重复下载。

#### 3. 接口签名 (Sign)

*   **优化**：如之前所述，将参数拼接后进行 MD5，防止了请求参数在网络传输过程中被中间人篡改。

---

### 四、 代码实战

在前端和 Node.js 中使用 MD5 非常简单。

#### 1. Node.js 原生写法 (自带 crypto 模块)

```javascript
const crypto = require('crypto');

function getMd5(content) {
  // 1. 创建 hash 对象
  const hash = crypto.createHash('md5');
  
  // 2. 输入数据
  hash.update(content);
  
  // 3. 输出 hex (十六进制) 格式
  return hash.digest('hex');
}

console.log(getMd5('123456')); 
// 输出: e10adc3949ba59abbe56e057f20f883e
```

#### 2. 前端写法 (使用 crypto-js)

前端没有原生 MD5 API，通常用库。

```javascript
// npm install crypto-js
import CryptoJS from 'crypto-js';

const password = '123456';
const hash = CryptoJS.MD5(password).toString();

console.log(hash); 
// 输出: e10adc3949ba59abbe56e057f20f883e
```

#### 3. 大文件计算 MD5

如果是上传 1GB 的文件，直接读入内存算 MD5 浏览器会崩。要用 **切片读取** 的方式。

```javascript
// npm install spark-md5
import SparkMD5 from 'spark-md5';

function calculateFileMD5(file) {
  return new Promise((resolve) => {
    const blobSlice = File.prototype.slice || File.prototype.mozSlice || File.prototype.webkitSlice;
    const chunkSize = 2097152; // 每片 2MB
    const chunks = Math.ceil(file.size / chunkSize);
    let currentChunk = 0;
    
    const spark = new SparkMD5.ArrayBuffer();
    const fileReader = new FileReader();

    fileReader.onload = function (e) {
      console.log('读取分片:', currentChunk + 1, '/', chunks);
      // 1. 把这 2MB 数据喂给 spark
      spark.append(e.target.result); 
      currentChunk++;

      if (currentChunk < chunks) {
        // 2. 没读完，继续读下一片
        loadNext();
      } else {
        // 3. 读完了，生成最终 MD5
        const md5 = spark.end();
        resolve(md5);
      }
    };

    function loadNext() {
      const start = currentChunk * chunkSize;
      const end = ((start + chunkSize) >= file.size) ? file.size : start + chunkSize;
      // 读取文件切片
      fileReader.readAsArrayBuffer(blobSlice.call(file, start, end));
    }

    loadNext();
  });
}
```

### ⚠️ 安全警示 

 MD5 时，最后一定要补一句：

> “不过，由于计算机算力的提升，MD5 现在已经**不再安全**了（容易被碰撞攻击和彩虹表破解）。
> 所以在现在的业务中：
>
> 1. **密码加密**：我们不再单用 MD5，而是用 **`bcrypt`** 或者 **`MD5 + 随机盐 (Salt)`**。
> 2. **文件校验**：MD5 依然是主流，因为它速度快，兼容性好。”





## 五。http

### HTTP/1.1 vs HTTP/2 

1.  **多路复用 (Multiplexing)**：
    *   **HTTP/1.1**：就像单车道。你要加载 10 张图，得排队，或者开 6 个连接（浏览器限制）。
    *   **HTTP/2**：就像**高速公路**。1 个连接里可以同时跑无数个请求，不用排队，速度极快。
2.  **头部压缩 (Header Compression)**：
    *   HTTP/1.1 每次都要传一大堆重复的 Header（User-Agent, Cookie...）。
    *   HTTP/2 会把这些压缩，省流量。
3.  **服务器推送 (Server Push)**：
    *   你要 `index.html`，服务器猜到你肯定还要 `style.css`，就主动顺手推给你，不用你再发请求要。



###  HTTP/2

#### 🛑 历史痛点 (HTTP/1.1)

*   **队头阻塞 (Head-of-line blocking)**：就像在单车道开车，前面的请求卡住了，后面的都得等。
*   **并发限制**：浏览器限制同一个域名最多同时发 6-8 个请求。所以以前前端要做“雪碧图”（把小图标拼成大图）来减少请求数。
*   **头部冗余**：每次请求都带一堆 Cookie、User-Agent，浪费流量。

#### ✅ 解决方案

**HTTP/2 的三大法宝：**

1.  **多路复用 (Multiplexing)**：在一条 TCP 连接上，同时跑无数个请求（就像多车道高速公路）。
2.  **二进制分帧**：传输更高效，机器解析更快。
3.  **头部压缩 (HPACK)**：重复的 Header 就不传了。

#### 💻 代码实战

**前端不需要写代码！** 这是一个协议层的升级。
只要你的网站上了 **HTTPS**，并且服务器（Nginx）开启了 HTTP/2，浏览器会自动使用。

**优化策略的改变：**

*   **以前**：要把所有 JS 合并成一个 `main.js`，减少请求次数。
*   **现在 (HTTP/2)**：可以把 JS 拆成很多小模块，因为请求多一点也不怕了，反而更有利于缓存。

#### 🚀 带来的优化

解决了请求堵塞问题，大幅提升了**多资源加载**（图片多、脚本多）页面的速度。



### 疑问

1.  **为什么 HTTP/2 敢拆分很多小模块？**
2.  **TCP 确实比直播用的协议（通常是 UDP）慢，那为什么网页还要用 TCP？**

---

### 第一部分：为什么 HTTP/2 敢把 JS 拆成很多小模块？

在 HTTP/1.1 时代，我们不敢拆，必须合并成一个大 `bundle.js`。但在 HTTP/2 时代，拆分反而更好。

#### 1. 核心原因：多路复用 (Multiplexing)
*   **HTTP/1.1 (排队模式)**：浏览器限制同一个域名同时只能有 6 个 TCP 连接。如果你有 100 个小 JS 文件，它们得排队下载。前 6 个下完了，后 6 个才能开始。**建立连接和排队的时间**远大于下载文件的时间。
*   **HTTP/2 (并发模式)**：**一个 TCP 连接**就可以传输无数个文件。这 100 个小 JS 文件，可以通过这**唯一的一条连接**，同时并行传输，根本不需要排队，也不需要重复建立连接。

#### 2. 为什么有利于缓存？(Granularity 颗粒度)
*   **合并打包 (Old)**：你有一个 1MB 的 `main.js`。你只改了一行代码，整个 1MB 的文件 Hash 变了，用户必须重新下载这 1MB。
*   **拆分打包 (New)**：你把代码拆成了 `header.js` (5KB), `footer.js` (5KB), `utils.js` (10KB)...
    *   当你改了 `header.js` 里的代码。
    *   用户只需要重新下载这个 5KB 的 `header.js`。
    *   其他的 `footer.js` 和 `utils.js` **直接用浏览器缓存**。
    *   **结论：** 拆得越细，缓存命中率越高，更新越快。

---

### 第二部分：TCP 确实比直播协议慢，为什么网页还用它？

你的直觉是对的：**TCP 确实比 UDP（直播常用）慢/延迟高。**

但是，网页（加载 HTML/CSS/JS）和直播（视频流）的**核心需求**完全不同。

#### 1. TCP vs UDP (经典面试比喻)

*   **TCP (网页用)**：**“挂号信” / “强迫症”**
    *   **机制**：三次握手（先打招呼）、确认重传（丢包了必须补发）、顺序控制（乱了必须排好）。
    *   **优点**：**绝对可靠**。发了 100 个字，对面一定能收到 100 个字，一个标点符号都不差。
    *   **缺点**：慢，有延迟。因为要确认、要重传。
    *   **场景**：**网页资源 (JS/CSS)**。
        *   *为什么？* 如果 JS 文件丢了一个字符，代码就报错崩了！网页哪怕慢 0.1 秒加载出来，也不能加载出一堆乱码。**准确性 > 速度**。

*   **UDP (直播/游戏用)**：**“大喇叭广播” / “摆烂”**
    *   **机制**：只管发，不管你收没收到。没有握手，没有重传。
    *   **优点**：**极快**，低延迟。
    *   **缺点**：不可靠，会丢包。
    *   **场景**：**直播 (WebRTC)、FPS 游戏、语音通话**。
        *   *为什么？* 视频通话时，画面偶尔花一下（丢包了）或者声音卡一下，无所谓，只要不延迟就行。如果为了找回那丢失的一帧画面，导致画面卡顿 3 秒，用户体验反而更差。**实时性 > 准确性**。

#### 2. 为什么说 TCP 慢？(TCP 的原罪)
TCP 有一个著名的问题叫 **“队头阻塞” (Head-of-Line Blocking)**。
*   假设 HTTP/2 在一个 TCP 连接里传 3 个包：`包1`、`包2`、`包3`。
*   如果 `包1` 在路上丢了。
*   虽然 `包2` 和 `包3` 已经到了，但 TCP 协议规定：**必须按顺序给浏览器**。
*   所以浏览器只能傻等 `包1` 重传过来，才能拿到 `包2` 和 `包3`。
*   这就是 TCP 在网络不稳定时变慢的根本原因。

---

### 💡 总结

**你的回答：**

> “1. **关于 HTTP/2 拆包：**
> HTTP/2 引入了**多路复用**，解决了 HTTP/1.1 的连接数限制问题，一个连接可以并发传输多个文件。所以我们将 JS 拆成小模块，不仅不会因为请求多而变慢，反而因为**颗粒度变小**，能更好地利用**浏览器强缓存**，修改业务代码时不影响第三方库的缓存。
>
> 2. **关于 TCP 和直播协议：**
>   直播协议（如 WebRTC）通常基于 **UDP**，注重的是**实时性**，允许丢包（画面花一点没关系，只要快就行）。
>   而网页资源（JS/CSS）必须使用 **TCP**，因为注重的是**准确性**。JS 代码哪怕丢一个字节都会报错，必须依赖 TCP 的**重传机制**来保证 100% 可靠。
>
> 3. **展望 HTTP/3：**
>   不过 TCP 确实有**队头阻塞**导致的延迟问题。所以最新的 **HTTP/3 (QUIC)** 协议底层已经换成了 **UDP**，在 UDP 上实现了可靠传输，结合了两者的优点，这是未来的趋势。”







## 六。上传的数据格式

- **` sendBeacon` 限制了数据格式，而后端接口（比如 Java/PHP）也有格式限制。**

- ** 如果后端不支持 JSON，你完全可以（甚至必须）把数据转为 **字符串（Query String）** 或 **FormData** 发送。

---

### 一、 字符串形式 (`application/x-www-form-urlencoded`)

这就是你说的 **Query 数据格式**，也就是 `qs` 库转出来的那个东西。

#### 1. 历史痛点 (没它之前)
在 Web 最早期（1995年左右），HTML 只有 `<form>` 表单。那时候没有 JSON，浏览器和服务器需要一种**通用标准**来传递数据。
**问题：** 怎么把用户填在输入框里的名字、年龄打包发给服务器？

#### 2. 它出现解决了什么？
它定义了一种简单的**键值对拼接规则**：`key=value&key2=value2`。
*   **解决了**：HTML 表单数据的标准化提交。
*   **浏览器原生支持**：浏览器会自动把 `<form>` 里的内容拼成这个字符串发出去。

#### 3. 现在的应用与优化 (Axios/sendBeacon)
虽然 JSON 现在更流行，但很多老后端（SpringMVC 默认配置、PHP）依然首选这个。

**💻 代码实战：**

**场景 A：Axios 配合 qs**
```javascript
import qs from 'qs';

const data = { name: 'Tom', age: 18 };
// 转成字符串: "name=Tom&age=18"
const payload = qs.stringify(data); 

axios.post('/api/user', payload, {
  headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
});
```

**场景 B：sendBeacon 发送**
```javascript
const data = new URLSearchParams({
  event: 'click',
  id: '123'
});
// data.toString() 就是 "event=click&id=123"
// sendBeacon 会自动识别这是表单格式
navigator.sendBeacon('/api/log', data);
```

---

### 二、 FormData (`multipart/form-data`)

#### 1. 历史痛点 (没它之前)
上面的“字符串格式”有一个致命弱点：**传不了文件**。
**问题：** 字符串只能传文本。如果你想上传一张图片，把它转成字符串编码（Base64）会让体积增大 33%，且解析非常慢。

#### 2. 它出现解决了什么？
**FormData** 是专门为了**混合传输**设计的。
它把 HTTP 请求体切成好几块（Multipart），一块放文本，一块放二进制文件。
*   **解决了**：文件上传的问题，以及“文件+文本”同时发送的问题。

#### 3. 现在的应用与优化
它是目前前端上传文件的**唯一标准**。

**💻 代码实战：**

**场景：上传头像和昵称**
```javascript
// 1. 创建对象
const formData = new FormData();

// 2. 追加数据 (看起来像 JSON，但其实内部结构完全不同)
formData.append('nickname', 'Tom'); // 文本
formData.append('avatar', fileInput.files[0]); // 二进制文件

// 3. 发送
// 优化点：浏览器会自动设置 Content-Type 并加上 boundary 分隔符，非常智能
axios.post('/upload', formData); 

// sendBeacon 也可以直接发 FormData
navigator.sendBeacon('/api/upload_log', formData);
```

---

### 三、 Blob (Binary Large Object)

这是最底层、最强大的格式。JSON、字符串、文件，**本质上都是 Blob**。

#### 1. 历史痛点 (没它之前)
在 JS 里，处理**二进制数据**非常痛苦。
**问题：** 比如你想在前端生成一个 `.txt` 文件让用户下载，或者你想把一个 JSON 伪装成文件发给后端。以前 JS 只能操作字符串，操作不了“字节”。

#### 2. 它出现解决了什么？
**Blob** 就是一个**“只读的二进制大对象”**。
*   **解决了**：前端生成文件、处理音视频流、或者**手动指定 Content-Type** 的问题。

#### 3. 核心优化
**这是 `sendBeacon` 的必杀技。**
`sendBeacon` 默认不支持直接传 JSON 对象。如果你直接传 `{a:1}`，它会变成 `[object Object]` 字符串。
但是！你可以**把 JSON 包装成 Blob**，强行告诉浏览器：“这是 JSON”。

**💻 代码实战：**

**场景：用 sendBeacon 强行发 JSON 给后端**
```javascript
const data = { name: 'Tom', action: 'close_page' };

// 1. 把 JSON 转成字符串
const jsonString = JSON.stringify(data);

// 2. 核心：封装成 Blob，并指定 MIME 类型
// 这一步做了什么？
// 把字符串变成了二进制流，并贴了个标签说 "我是 application/json"
const blob = new Blob([jsonString], { 
  type: 'application/json' 
});

// 3. 发送
// 后端收到后，会认为这就是一个标准的 JSON 请求
navigator.sendBeacon('/api/log', blob);
```

---

### 💡 总结对比表 (面试速记)

| 数据格式         | **字符串 (String/Query)**        | **FormData**                   | **Blob**                                      |
| :--------------- | :------------------------------- | :----------------------------- | :-------------------------------------------- |
| **样子**         | `a=1&b=2`                        | 内部也是键值对，但包含文件     | 二进制流 (黑盒)                               |
| **Content-Type** | `x-www-form-urlencoded`          | `multipart/form-data`          | **你说了算** (可自定义)                       |
| **解决痛点**     | 最早的数据提交标准，兼容性最强。 | **文件上传**，文本与文件混传。 | **万能载体**。能把 JSON 伪装成文件发送。      |
| **实战场景**     | 老旧后端接口、简单的 GET 参数。  | 上传图片、视频。               | **sendBeacon 发 JSON**、前端生成 Excel 下载。 |

所以回答你最开始的问题：
**是的，如果后端不支持 JSON，你可以用 `qs` 转成字符串；如果 `sendBeacon` 想发 JSON，你可以转成 `Blob`。它们都是“数据”在网络传输中的不同形态。**





## 七，案例，后端不支持json格式

如果后端不支持 JSON 格式（即不支持 `application/json`），那通常意味着后端期望接收的数据格式是 **表单格式**。

最常见的两种替代格式是：
1.  **`application/x-www-form-urlencoded`**（最常见，类似 GET 请求参数拼接 `a=1&b=2`）。
2.  **`multipart/form-data`**（通常用于文件上传，也可以传普通数据）。

以下是针对不同场景的解决方案（包含 **Axios** 和 **UniApp** 的写法）：

---

### 方案一：使用 `application/x-www-form-urlencoded` (最常见)

后端希望收到的数据长这样：`name=Tom&age=18&gender=male`。

#### 方法 A：使用 `qs` 库（推荐，Axios 社区标准）
如果你用的是 Axios，最稳妥的方法是使用 `qs` 库把对象“序列化”成字符串。

1.  **安装**：`npm install qs`
2.  **代码**：

```javascript
import axios from 'axios';
import qs from 'qs'; // 引入 qs

const data = {
  name: 'Tom',
  age: 18,
  hobbies: ['eat', 'sleep'] // qs 能很好地处理数组
};

axios.post('/api/user', qs.stringify(data), {
  headers: {
    // 显式告诉后端：我发的是表单格式
    'Content-Type': 'application/x-www-form-urlencoded'
  }
});
```

#### 方法 B：使用原生 `URLSearchParams` (无需安装库)
如果不想引入额外的包，可以用浏览器原生的 API。

```javascript
const params = new URLSearchParams();
params.append('name', 'Tom');
params.append('age', '18');

axios.post('/api/user', params, {
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded'
  }
});
```

---

### 方案二：使用 `multipart/form-data` (FormData)

如果后端是用 Java SpringMVC 的 `@RequestParam` 接收，或者是 PHP 的 `$_POST`，或者是需要**上传文件**，用这个最稳。

**核心 API：** `new FormData()`

```javascript
// 1. 创建 FormData 对象
const formData = new FormData();
formData.append('username', 'Tom');
formData.append('password', '123456');

// 如果有文件
// formData.append('file', fileObject);

// 2. 发送请求
axios.post('/api/login', formData, {
  headers: {
    // 注意：传 FormData 时，通常不需要手动写 Content-Type
    // 浏览器会自动识别并加上 boundary 分隔符
    // 但为了保险，有些旧项目需要指定 multipart/form-data
    'Content-Type': 'multipart/form-data' 
  }
});
```

---

### 方案三：UniApp 中的特殊处理

你既然在做 UniApp，UniApp 的 `uni.request` 有一点不一样。

#### 1. 默认情况
UniApp 默认就是 `application/json`。

#### 2. 改为表单格式
你需要修改 `header` 中的 `content-type`。

```javascript
uni.request({
  url: 'https://api.example.com/login',
  method: 'POST',
  header: {
    // 🔴 核心：改为表单格式
    'content-type': 'application/x-www-form-urlencoded' 
  },
  data: {
    name: 'Tom',
    age: 18
  },
  success: (res) => {
    console.log(res.data);
  }
});
```
*   **注意：** UniApp 在检测到 `content-type` 为 `application/x-www-form-urlencoded` 时，会**自动**帮你把 `data` 对象转换成 `key=val&key2=val2` 的格式，**不需要**你自己引入 `qs` 或 `URLSearchParams`。

---

### 如果后端不支持 JSON，怎么封装axios

这是一个体现你工程化能力的考题。你可以这样回答：

> “如果后端不支持 JSON，说明它是老旧系统或者特定的企业级接口（如 XML 或 Form）。我会利用 Axios 的 **拦截器 (Interceptors)** 来统一处理。
>
> 1.  **请求拦截器：** 判断接口需要的 `Content-Type`。
> 2.  **数据转换：** 如果是 `x-www-form-urlencoded`，我会引入 **`qs`** 库，在请求发出去之前，使用 `qs.stringify(config.data)` 将 JSON 对象序列化成键值对字符串。
> 3.  **统一配置：** 在创建 Axios 实例时，设置默认的 Header。
>
> 这样业务层的代码依然写 JSON 对象，底层的转换由封装层自动完成。”

**封装代码示例（request.js）：**

```javascript
import axios from 'axios';
import qs from 'qs';

const service = axios.create({
  baseURL: '/api',
  // 可以在这里设置默认头，或者在拦截器里动态改
  // headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
});

service.interceptors.request.use((config) => {
  // 如果请求头是 form-urlencoded，自动序列化 data
  if (config.headers['Content-Type'] === 'application/x-www-form-urlencoded') {
    config.data = qs.stringify(config.data);
  }
  return config;
});

export default service;
```