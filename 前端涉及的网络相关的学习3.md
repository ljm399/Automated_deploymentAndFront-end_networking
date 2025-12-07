## 一。sendBeacon (信标)

### 一、 历史痛点 (没它之前)

在 `sendBeacon` 出现之前，如果你想在用户**关闭页面**（`unload`）或**跳转路由**时，向后端发送一条统计数据（比如：“用户在页面停留了 5 秒”），你会面临进退两难的局面。

#### 1. 尝试用异步请求 (Async AJAX/Axios) —— **数据丢失**
浏览器在页面卸载时，会杀掉所有正在进行的异步请求。
*   **结果**：请求还没发出去，页面就关了。后端收不到数据，埋点失效。

```javascript
// ❌ 错误做法：异步请求
window.addEventListener('unload', function() {
  // 页面关闭那一刻，这个请求会被浏览器无情 cancel 掉
  axios.post('/api/log', { event: 'close' });
});
```

#### 2. 尝试用同步请求 (Sync AJAX) —— **用户体验灾难**
为了保证数据能发出去，以前的开发者会强制把 AJAX 设为**同步**（`async: false`）。浏览器必须等请求发完，才允许页面关闭或跳转。
*   **结果**：用户点了“下一页”，页面**卡住**了（假死），等了几百毫秒甚至 1 秒才跳转。这种体验极差。

```javascript
// ❌ 以前的笨办法：同步请求
window.addEventListener('unload', function() {
  const xhr = new XMLHttpRequest();
  // 第三个参数 false 表示“同步”，会卡住主线程
  xhr.open('POST', '/api/log', false); 
  xhr.setRequestHeader('Content-Type', 'application/json');
  xhr.send(JSON.stringify({ event: 'close' }));
  
  // 代码执行到这儿，请求没结束，用户就跳不走
});
```

---

### 二、 解决方案：`sendBeacon` 的出现

`navigator.sendBeacon` (信标) 就是为了完美解决这个问题而生的。它的设计哲学是：**“只管发，不等待，不断连”。**

#### 1. 它是怎么做的？
它告诉浏览器：“我有几个数据要发给服务器，但我马上要走了（页面关闭）。麻烦你（浏览器后台进程）帮我把这个请求发完，别因为我页面关了就掐断它。”

#### 2. 代码实战

```javascript
// ✅ 现代做法：sendBeacon
// 推荐监听 visibilitychange 而不是 unload（兼容性更好）
document.addEventListener('visibilitychange', function() {
  if (document.visibilityState === 'hidden') {
    // 准备数据
    const data = {
      event: 'page_close',
      timestamp: Date.now(),
      userId: 123
    };

    // sendBeacon 默认不能直接发 JSON 对象，需要转成字符串或 Blob
    // 这里用 Blob 封装，指定 type 为 json
    const blob = new Blob([JSON.stringify(data)], {
      type: 'application/json'
    });

    // 🚀 发射！
    // 参数1：URL
    // 参数2：数据 (ArrayBufferView, Blob, DOMString, or FormData)
    const result = navigator.sendBeacon('/api/log', blob);

    if (result) {
      console.log('埋点任务已移交浏览器后台');
    }
  }
});
```

---

### 三、 带来了什么优化？

#### 1. 可靠性提升 (Reliability)
*   **解决了“数据丢失”问题**：即使页面已经销毁，浏览器依然会在后台默默把数据发送完毕。这对于数据分析、错误监控（Sentry）至关重要。

#### 2. 用户体验飞跃 (Performance)
*   **解决了“页面卡顿”问题**：`sendBeacon` 是**完全异步**的，而且它的优先级比较低。它不会阻塞当前页面的卸载，也不会阻塞下一个页面的加载。用户点击跳转是**毫秒级**响应的，完全感觉不到背后发了请求。

#### 3. 浏览器的智能调度
*   普通 AJAX 请求会争抢网络资源。
*   `sendBeacon` 允许浏览器进行调度，浏览器可以在网络空闲时发送，减少对关键资源加载的影响。

---

### 四、 细节 

**Q: `sendBeacon` 有什么限制吗？**

1.  **只能发 POST**：它不支持 GET、PUT、DELETE。
2.  **不能接收响应**：因为请求发完的时候，页面早就不在了，没地方接收 `response`。它是“火后即焚”的。
3.  **数据大小限制**：虽然标准没写死，但各浏览器有限制（通常在 64KB 左右），不适合传大文件，只适合传短小的埋点数据。
4.  **Header 限制**：它无法像 Axios 那样随意修改 Header。如果你想指定 `Content-Type: application/json`，必须使用 **Blob** 包装（如上面的代码所示）。

### 💡 总结

> "`navigator.sendBeacon` 主要是为了解决**页面卸载时数据发送**的问题。
>
> 1.  **历史问题**：以前用异步 AJAX 会导致请求被浏览器截断（丢数据），用同步 AJAX 会导致页面跳转卡顿（用户体验差）。
> 2.  **解决**：`sendBeacon` 将数据发送任务移交给浏览器后台进程，不受页面生命周期限制。
> 3.  **优化**：它实现了**数据必达**的同时，做到了**零阻塞**，保证了用户切换页面时的流畅体验。"





## 二。API接口签名，敏感数据加密，Token无感刷新

这三个是**中高级前端**面试中含金量非常高的知识点。它们分别解决了数据传输中的**完整性（防篡改）**、**机密性（防偷窥）**和**用户体验（防掉线）**的问题。

我将按照你的要求，从**历史痛点**、**解决方案**到**代码实战**一一拆解。

---

### 一、 API 接口签名 (Sign)

#### 1. 历史痛点 (没它之前)
**“中间人篡改”与“重放攻击”。**
*   **场景**：你要给张三转账 100 元。请求参数是 `{ to: 'zhangsan', money: 100 }`。
*   **问题 A (篡改)**：黑客拦截了请求，把 `money` 改成 `10000`，后端收到后照单全收。
*   **问题 B (重放)**：黑客虽然改不了内容，但他把这个请求录下来，**重复发送** 100 次，你的钱就被扣了 100 次。

#### 2. 它出现解决了什么？
**保证了请求的“原装正版”和“时效性”。**
*   **防篡改**：通过参数排序+密钥哈希，黑客只要改一个标点符号，生成的签名（Sign）就对不上，后端直接拒绝。
*   **防重放**：引入时间戳 (`timestamp`) 和随机数 (`nonce`)，过期的请求或重复的请求直接无效。

#### 3. 带来了什么优化？
*   **安全性**：即使跑在 HTTP（非加密）上，参数逻辑也是安全的。
*   **信任机制**：后端只认签名，不认人，建立了严格的信任链路。

#### 💻 代码实战 (CryptoJS)

```javascript
import axios from 'axios';
import CryptoJS from 'crypto-js';

// 前后端约定的秘密钥匙，绝对不能在网络中传输
const SECRET_KEY = 'my_secret_888';

function getSign(params) {
  // 1. 字典序排序 (防止顺序不同导致 hash 不同)
  const sortedKeys = Object.keys(params).sort();
  
  // 2. 拼接字符串 "age=18&name=tom"
  let str = '';
  sortedKeys.forEach(key => {
    str += `${key}=${params[key]}&`;
  });
  
  // 3. 加盐 (关键！黑客不知道 SECRET_KEY，所以算不出正确的 sign)
  str += `key=${SECRET_KEY}`;
  
  // 4. 生成签名 (MD5 转大写)
  return CryptoJS.MD5(str).toString().toUpperCase();
}

// 发送请求
const params = {
  name: 'Tom',
  money: 100,
  // 关键：加上时间戳，后端会校验时间差（比如超过60秒的请求视为无效，防重放）
  timestamp: Date.now() 
};

// 计算签名
const sign = getSign(params);

axios.post('/api/transfer', params, {
  headers: {
    'X-Sign': sign // 签名放在 Header 里传给后端
  }
});
```

---

### 二、 敏感数据加密 (RSA / AES)

#### 1. 历史痛点 (没它之前)
**“裸奔的数据”。**
*   **问题**：虽然有了 HTTPS，它只保证传输链路安全。但是数据到了后端服务器，或者在浏览器的 Network 面板里，依然是**明文**的。
*   **隐患**：
    *   公司内部人员（运维/后端）看日志就能看到用户的明文密码。
    *   如果是 HTTP 代理抓包（如 Charles），HTTPS 解密后直接看到身份证号。

#### 2. 它出现解决了什么？
**端到端的机密性。**
*   **RSA (非对称)**：前端只管加密（公钥），黑客截获了也没用，因为只有后端手里的私钥能解开。主要用于**密码、银行卡号**。
*   **AES (对称)**：加密解密快，用于加密**大的响应体**（比如整个用户列表）。

#### 3. 带来了什么优化？
*   **纵深防御**：即使 HTTPS 证书泄露，或者数据库被拖库，黑客拿到的也全是乱码，数据价值为 0。

#### 💻 代码实战 (RSA 加密密码)

```javascript
// npm install jsencrypt
import JSEncrypt from 'jsencrypt';

// 后端给的公钥
const PUBLIC_KEY = `-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC...
-----END PUBLIC KEY-----`;

// 加密函数
function encryptData(data) {
  const encryptor = new JSEncrypt();
  encryptor.setPublicKey(PUBLIC_KEY);
  // 返回的是一串 Base64 乱码
  return encryptor.encrypt(data); 
}

// 登录场景
const password = '123456';
const encryptedPassword = encryptData(password);

// 实际发给后端的 password 是 "a8s7d87a8sd78..."
axios.post('/api/login', { 
  username: 'admin', 
  password: encryptedPassword 
});
```

---

### 三、 Token 无感刷新 (Silent Refresh)

#### 1. 历史痛点 (没它之前)
**“突然掉线”与“安全两难”。**
*   **场景**：为了安全，Token 有效期通常设得很短（比如 30 分钟）。
*   **问题**：用户正填表单填到一半，30 分钟到了，Token 过期。接口报 401 错误，前端强制跳转登录页，用户填的数据全丢了。体验极差。
*   **两难**：设长了不安全，设短了体验差。

#### 2. 它出现解决了什么？
**安全与体验的平衡。**
*   **双 Token 机制**：
    *   `Access Token`（短命，30分钟）：用来请求数据。
    *   `Refresh Token`（长命，7天）：专门用来换新的 Access Token。
*   **无感刷新**：当 Access Token 过期时，前端自动用 Refresh Token 去换一个新的，然后**重发**刚才失败的请求。用户完全感觉不到 Token 变了。

#### 3. 带来了什么优化？
*   **用户体验**：理论上只要用户经常用，就永远不需要重新登录。
*   **安全性**：Access Token 即使被偷，也只有 30 分钟有效期。Refresh Token 只能换 Token 不能请求业务数据，且可以随时在后端吊销。

#### 💻 代码实战 (Axios 响应拦截器)

这是面试中**最能体现技术水平**的代码段：

```javascript
import axios from 'axios';

let isRefreshing = false; // 锁：防止并发请求导致多次刷新
let requestsQueue = [];   // 队列：存储刷新期间积压的请求

const service = axios.create({ baseURL: '/api' });

service.interceptors.response.use(
  res => res.data,
  async error => {
    const { config, response } = error;
    
    // 1. 发现 Token 过期 (401)
    if (response && response.status === 401) {
      
      // 2. 如果当前没在刷新，就发起刷新
      if (!isRefreshing) {
        isRefreshing = true;
        
        try {
          const refreshToken = localStorage.getItem('refresh_token');
          // 调用刷新接口
          const res = await axios.post('/auth/refresh', { token: refreshToken });
          
          const newToken = res.data.accessToken;
          localStorage.setItem('token', newToken); // 更新本地
          
          // 3. 刷新成功！把队列里的请求都重发一遍
          requestsQueue.forEach(cb => cb(newToken));
          requestsQueue = []; // 清空队列
          
          // 4. 重发当前这个失败的请求
          config.headers.Authorization = `Bearer ${newToken}`;
          return service(config);
          
        } catch (e) {
          // 刷新失败（Refresh Token 也过期了），只能去登录页
          window.location.href = '/login';
        } finally {
          isRefreshing = false; // 解锁
        }
      } else {
        // 5. 如果正在刷新中，把当前请求挂起，放入队列
        // 返回一个 Promise，让它等着
        return new Promise(resolve => {
          requestsQueue.push((token) => {
            config.headers.Authorization = `Bearer ${token}`;
            resolve(service(config)); // 刷新完成后，这里会被执行
          });
        });
      }
    }
    return Promise.reject(error);
  }
);
```

### 💡 总结给

1.  **接口签名**：解决了**请求篡改**和**重放攻击**的问题，通过排序加盐 Hash，保证了接口的权威性。
2.  **数据加密**：解决了**明文泄露**的问题，利用 RSA 非对称加密，即使网络被监听，黑客也无法还原用户的敏感密码。
3.  **Token 无感刷新**：解决了**Token过期导致用户体验中断**的问题，通过拦截 401 和双 Token 机制，实现了既安全又流畅的会话管理。





## 三。数据校验

这是一个非常核心的工程化问题。**数据校验（Data Validation）** 是保证系统健壮性的第一道防线。

面试官问这个问题，其实是在问：**“如果不做校验会死得多惨？”** 以及 **“我们现在是怎么优雅地做校验的？”**

我们把它拆分为 **前端校验** 和 **后端校验** 两个维度来讲（因为两者缺一不可），按照你的要求：历史痛点 -> 解决方案 -> 优化 -> 代码。

---

### 一、 历史痛点 (没它之前)

在没有完善的校验机制之前，那是“蛮荒时代”。

#### 1. 后端裸奔 (最严重)
*   **问题**：后端假设前端传来的数据永远是对的。
*   **后果**：
    *   **脏数据入库**：用户年龄填了 `-1` 或 `2000`，数据库照存不误。报表统计全是错的。
    *   **代码报错崩盘**：后端代码写了 `data.name.substring(1)`，结果前端传了 `null`，后端直接抛出空指针异常，服务挂了。
    *   **安全漏洞**：最著名的 **SQL 注入**。黑客在输入框填 `admin' --`，因为没校验格式，直接拼接 SQL 导致数据库被拖库。

#### 2. 前端体验差
*   **问题**：所有校验都依赖后端。
*   **后果**：用户辛辛苦苦填了 50 项表单，点击提交，转圈转了 2 秒，后端报错说“手机号格式不对”。用户得改完再提交，再转圈……**反馈链路太长**，用户想砸电脑。

---

### 二、 它出现解决了什么？

**数据校验（Validation）** 的核心就是：**“先安检，再进站”**。

#### 1. 前端校验解决：用户体验 + 节省流量
*   **即时反馈**：输入框失去焦点（Blur）的一瞬间，或者输入时（Change），立马告诉用户“格式错了”。
*   **拦截无效请求**：格式不对的数据根本发不出去，节省了服务器的带宽和计算资源。

#### 2. 后端校验解决：数据安全 + 兜底
*   **数据完整性**：保证存入数据库的数据绝对是符合业务逻辑的（比如年龄必须 0-120）。
*   **防御式编程**：即使前端被黑客绕过了（直接调 API），后端依然能识别出非法数据并拒绝。

---

### 三、 带来了什么优化？

1.  **Schema 驱动 (模式验证)**：
    *   以前写校验是堆砌 `if...else`（面条代码）。
    *   现在定义一套 **Schema 规则**（如 `async-validator`、`Joi`、`Zod`），代码清晰度极高，维护性极好。
2.  **解耦业务逻辑**：
    *   校验逻辑从业务代码中剥离出来。业务代码只处理“干净”的数据，不需要再担心 `null` 或 `undefined`。

---

### 四、 代码实战 (历史对比)

#### ❌ 以前的写法 (刀耕火种)

这就叫“面条代码”，很难维护，漏写一个 `if` 就出 Bug。

```javascript
// 后端接收注册请求
function register(req, res) {
  const { username, age, email } = req.body;

  // 1. 手写一堆 if 判断
  if (!username) {
    return res.send('用户名不能为空');
  }
  if (username.length < 3) {
    return res.send('用户名太短');
  }
  if (!age || isNaN(age)) {
    return res.send('年龄必须是数字');
  }
  if (age < 0 || age > 120) {
    return res.send('年龄不合法');
  }
  // ... 还有邮箱正则，写几十行 ...

  // 2. 终于开始业务逻辑
  database.save(user);
}
```

#### ✅ 现在的写法 (Schema 校验)

现在我们使用 **配置化** 的方式。在 Vue/ElementUI 中用的是 `async-validator`，在 Node.js 中常用 `Joi` 或 `Zod`。

**前端 (Vue + ElementUI 风格):**

```javascript
// 1. 定义规则 (Schema)
const rules = {
  username: [
    { required: true, message: '必填', trigger: 'blur' },
    { min: 3, max: 10, message: '长度3-10位', trigger: 'blur' }
  ],
  phone: [
    { required: true, message: '必填', trigger: 'blur' },
    // 正则校验
    { pattern: /^1[3-9]\d{9}$/, message: '手机号格式不对', trigger: 'blur' }
  ]
};

// 2. 使用 (ElementUI 底层调用 async-validator)
this.$refs.form.validate((valid) => {
  if (valid) {
    // 校验通过，才发请求
    axios.post('/api/register', this.form);
  } else {
    console.log('格式不对，不许发请求！');
  }
});
```

**后端 (Node.js + Joi/Zod 风格):**

```javascript
import Joi from 'joi'; // 引入校验库

// 1. 定义规则 (Schema)
// 就像写文档一样清晰，这本身就是一种文档
const schema = Joi.object({
  username: Joi.string().min(3).max(10).required(),
  age: Joi.number().integer().min(0).max(120).required(),
  email: Joi.string().email().required()
});

// 2. 接口中使用
function register(req, res) {
  // 一行代码完成所有校验
  const { error, value } = schema.validate(req.body);

  if (error) {
    // 如果有错，直接返回错误信息，阻断流程
    return res.status(400).send(error.details[0].message);
  }

  // 到这里，value 里的数据绝对是干净、安全的
  database.save(value);
}
```

---

### 💡 总结

> “关于数据校验，在没有完善的校验机制之前，我们面临的主要问题是**脏数据入库**导致的系统异常，以及**前端反馈链路过长**导致的用户体验差。代码里也充斥着大量的 `if-else` 判断，维护困难。
>
> 引入 Schema 校验（如 async-validator 或 Joi）后：
> 1.  **在前端**：我们实现了**即时反馈**，避免无效请求发送到服务器，提升了用户体验。
> 2.  **在后端**：我们建立了**安全防线**，防止 SQL 注入和脏数据破坏业务逻辑。
> 3.  **在代码层面**：我们将校验逻辑与业务逻辑**解耦**，使用声明式的规则配置替代了冗余的判断代码，大大提高了代码的可读性和可维护性。”





## 四。浏览器缓存策略 (Browser Caching)

**场景：** 你修复了一个 Bug，发布上线了，但老板发火说：“怎么还是坏的？”——这通常是缓存背的锅。

面试官会问：**“强缓存和协商缓存有什么区别？”**

#### 1. 浏览器缓存 (强缓存 & 协商缓存)

#### 🛑 历史痛点 (没它之前)

每次用户刷新页面，都要重新去服务器下载 `logo.png`、`jquery.js` 这些万年不变的文件。
**后果：** 浪费用户流量，页面加载慢，服务器带宽压力巨大。

#### ✅ 解决方案

浏览器和服务器商量好一套**缓存策略**。

1.  **强缓存 (Cache-Control)**：服务器说“这文件一年别来烦我”。
2.  **协商缓存 (304)**：服务器说“你下次来问问我改没改，没改你就用旧的”。

#### 💻 代码实战 (前端怎么配合)

前端不能写 Response Header（那是后端/Nginx的事），但前端可以通过**文件名 Hash** 来利用强缓存。

**Webpack/Vite 配置：**

```javascript
// 打包后的文件名：main.a1b2c3.js
// 只要内容变了，Hash 就变了 -> 浏览器认为是新文件 -> 下载
// 内容没变，Hash 不变 -> 浏览器命中强缓存 -> 速度飞快
output: {
  filename: '[name].[contenthash].js',
}
```

**后端/Nginx 配置逻辑 (了解即可)：**

```nginx
# Nginx 配置强缓存
location ~ .*\.(js|css|png)$ {
  # max-age=31536000 秒 (一年)
  # 告诉浏览器：一年内直接读硬盘，不用请求我
  add_header Cache-Control "public,max-age=31536000";
}
```

#### 🚀 带来的优化

*   **首屏加载速度**提升几倍（因为不用下载大文件）。
*   **节省服务器成本**（带宽费很贵的）。



#### 1. 强缓存 (Strong Cache)

* **含义：** 浏览器**根本不问服务器**，直接从本地硬盘读文件。状态码通常是 `200 (from disk cache)`。

* **控制字段 (Response Header)：**

  *   **`Cache-Control: max-age=31536000`** (推荐)：告诉浏览器，这文件一年内都别来烦我，直接用旧的。
  *   `Expires` (老)：绝对时间，有 bug（本地时间不准就挂了）。

* **前端怎么做？**

  *   我们不能直接写 Header，但我们可以控制**文件名**。
  *   Webpack/Vite 打包时，会生成 `main.a1b2c3.js`。
  *   **原理：** 只要文件内容变了，Hash 值（文件名）就变了，浏览器就会把它当成新文件去下载；文件名没变，就一直用缓存。

  

#### 2. 协商缓存 (Negotiated Cache)  ---了解

*   **含义：** 浏览器发现强缓存过期了，就去**问一下服务器**：“这文件改了吗？”
    *   服务器说“没改” -> 返回 `304` -> 浏览器继续用旧的。
    *   服务器说“改了” -> 返回 `200` 和新内容。
*   **控制字段 (成对出现)：**
    *   **`ETag` / `If-None-Match`** (最常用)：文件的“指纹”（Hash）。前端下次请求带上指纹，服务器比对指纹。
    *   `Last-Modified` / `If-Modified-Since`：文件的修改时间（精确到秒，有误差）。



## 协商缓存 和 前端能对其做什么

这是一个非常关键的**工程化**问题。

直接给你结论：**在写 JavaScript 代码（如 Axios）时，你什么都不用做。但在配置服务器（如 Nginx）或打包策略时，你需要关注。**

协商缓存是**浏览器**和**服务器**之间的“自动对话”，前端 JS 代码无法干预，也无需干预。

但作为前端工程师，你必须懂**“部署策略”**，因为这决定了你的网站能不能**既秒开又实时更新**。

---

### 一、 为什么 JS 代码不用管？

**全自动机制：**

1.  **浏览器 (自动)**：
    *   第一次请求，收到服务器发来的 `ETag: "abc"`。浏览器自动把它存到硬盘的小本本上。
    *   第二次请求，浏览器自动在 Header 里带上 `If-None-Match: "abc"` 发给服务器。
    *   收到 `304`，浏览器自动去硬盘读取旧文件；收到 `200`，自动读取新内容。
2.  **Axios/Fetch (无感)**：
    *   它们拿到的已经是浏览器处理后的结果了。如果是 `304`，Axios 拿到的 `status` 依然可能是 200（或者浏览器直接让它看起来像成功），数据也是拿到手的。前端业务逻辑不需要关心这个过程。

---

### 二、 那前端工程师要做什么？（重点：部署策略）

虽然不用写 JS，但面试官问这个是考察你的**架构能力**。你需要配置**“入口文件”**和**“资源文件”**采用不同的缓存策略。

#### 1. 核心策略（必背）

*   **对于 `index.html`（入口文件）：**
    *   **策略：** **必须使用协商缓存**（或者不缓存）。
    *   **原因：** `index.html` 的名字永远叫 `index.html`，不能加 Hash。如果它被强缓存了（比如缓存 1 年），用户永远读的是旧的 HTML，里面引用的也是旧的 JS（如 `main.old.js`），你的新版发布根本不生效！
    *   **做法：** 告诉服务器：“每次请求 `index.html` 都要验证一下 ETag！”

*   **对于 `.js / .css / .png`（带 Hash 的资源）：**
    *   **策略：** **使用强缓存**（Cache-Control: max-age=1年）。
    *   **原因：** 因为我们用了 Webpack 的 `[contenthash]`。文件内容变了，文件名就变了。
    *   **做法：** 告诉服务器：“只要文件名没变，直接用，别问我！”

#### 2. Nginx 配置实战（前端通常要负责这个）

在 `nginx.conf` 里，我们通常这样写：

```nginx
server {
    listen 80;
    server_name example.com;
    root /usr/share/nginx/html;

    # 1. 针对 index.html：协商缓存
    location = /index.html {
        # no-cache 的意思是：可以使用缓存，但必须先问服务器(协商)
        # 千万别写成 no-store (那是禁止任何缓存)
        add_header Cache-Control "no-cache";
    }

    # 2. 针对静态资源 (js, css, img)：强缓存
    location ~ .*\.(js|css|png|jpg|gif)$ {
        # 31536000秒 = 1年
        # 只要文件名没变，浏览器一年内直接读硬盘
        add_header Cache-Control "public,max-age=31536000";
    }
}
```

---

### 三、 总结

> “在 **JavaScript 业务代码层面**，我们不需要做任何事，因为协商缓存是浏览器和服务器基于 HTTP Header 自动完成的。
>
> 但是在 **前端工程化和部署层面**，我需要负责制定缓存策略，通常我会在 Nginx 配置中做区分：
>
> 1.  对于 **`index.html`** 入口文件：配置 **`Cache-Control: no-cache`**，也就是强制开启**协商缓存**。因为入口文件名不能变，必须保证用户每次拿到的都是最新的 HTML，这样才能引用到最新的 JS 文件。
>
> 2.  对于 **JS/CSS/图片** 等资源：配置 **强缓存（max-age=1年）**。因为我在 Webpack 中配置了 **`[contenthash]`**，文件名本身就带有版本号。只要文件名没变，就说明内容没变，直接读取本地缓存能极大提升加载速度。”

**一句话总结：**
**“HTML 走协商，JS 走强缓（配合 Hash）”** —— 这是一个中高级前端必须具备的部署常识。



## 五。静态页面 和动态页面

这是一个非常基础但也非常重要的概念，尤其是在涉及**架构、部署（Nginx）和 SEO** 的时候。

简单来说：**静态页面是“写死”的，动态页面是“算出来”的。**

---

### 一、 核心区别（一句话总结）

*   **静态页面 (Static)**：服务器直接把**现成的 HTML 文件**扔给浏览器。所有人看到的内容都一样，内容不随时间/用户变化（除非改代码）。
*   **动态页面 (Dynamic)**：服务器接到请求后，**先跑一段代码（如 Node.js/Java/PHP），去查数据库**，拼装成 HTML（或返回 JSON），再发给浏览器。不同人、不同时间看到的内容可能不一样。

---

### 二、 详细对比表（面试必背）

| 特性               | 静态页面 (Static)                                     | 动态页面 (Dynamic)                                   |
| :----------------- | :---------------------------------------------------- | :--------------------------------------------------- |
| **文件后缀**       | 通常是 `.html`, `.htm`                                | `.jsp`, `.php`, `.asp`, 或者无后缀路由               |
| **服务器行为**     | **只读**。直接读取硬盘上的文件返回。                  | **计算**。执行脚本、连接数据库、逻辑判断。           |
| **内容差异**       | **千人一面**。所有人看到的都一样。                    | **千人千面**。根据用户身份、时间显示不同内容。       |
| **数据库**         | **无**。不需要连接数据库。                            | **强依赖**。数据通常存储在 MySQL/MongoDB 中。        |
| **速度/性能**      | **极快**。Nginx 处理静态文件无敌，且容易被 CDN 缓存。 | **较慢**。涉及逻辑计算和数据库查询。                 |
| **SEO (搜索排名)** | **极好**。爬虫最喜欢纯 HTML。                         | **一般**。如果是 CSR（客户端渲染），爬虫可能抓不到。 |
| **维护成本**       | **高**。改一个字可能要改 100 个 HTML 文件。           | **低**。改一条数据库记录，所有页面自动更新。         |

---

### 三、 深度解析

#### 1. 静态页面 (Static Web Page)

*   **生活例子**：**报纸**。印刷出来什么样，你拿到就是什么样，无法交互，大家看的都一样。
*   **技术栈**：纯 HTML + CSS + JS (这里的 JS 仅做特效，不请求数据)。
*   **部署**：只需要一个 Nginx 或者 Apache 服务器，把文件扔进去就能访问。
*   **场景**：
    *   公司官网介绍页。
    *   技术文档（如 VuePress, Hexo 生成的博客）。
    *   活动落地页（Landing Page）。

#### 2. 动态页面 (Dynamic Web Page)

*   **生活例子**：**淘宝 App**。你打开看到的是推荐给你的商品，我打开看到的是我关注的商品。显示库存、价格都在实时变化。
*   **技术栈**：
    *   **传统后端渲染 (SSR)**：JSP, PHP, ASP.NET。
    *   **现代前后端分离**：前端 (Vue/React) + 后端 API (Node/Java/Go)。
*   **场景**：
    *   用户登录系统、购物车、后台管理系统、社交媒体。

---

### 四、 现代前端的“模糊地带” (SPA 是静态还是动态？)

这是面试中最容易混淆的点。

**问：Vue/React 写出来的单页应用 (SPA)，打包出来是 `index.html` 和 `js`，它是静态还是动态？**

**答：**
从**部署**的角度看，它是**静态的**。
*   因为你只需要把 `dist` 目录扔到 Nginx 里，不需要后端服务器（如 Tomcat/Node）去“运行”它。

但从**内容**的角度看，它是**动态的**。
*   因为 `index.html` 里是空的（只有一个 `<div id="app"></div>`）。
*   内容是靠 JS 运行后，通过 AJAX/Axios 去后端抓取 **JSON 数据**，然后动态渲染到页面上的。

**这带来了两个问题（面试考点）：**

1.  **SEO 差**：百度爬虫抓取时，JS 还没运行，只看到空壳子。
    *   *解决*：使用 **SSR (Next.js / Nuxt.js)** 或 **SSG (静态站点生成)**。
2.  **首屏慢**：要等 JS 下载并执行完才能看到内容。

---

### 五、 结合网络知识的实战（缓存策略）

面试官可能会结合之前的缓存知识问你：

**Q：静态资源和动态接口，缓存策略有什么不同？**

1.  **静态资源 (JS/CSS/图片)**：
    *   **策略**：**强缓存**（Cache-Control: max-age=1年）。
    *   **理由**：配合 Webpack 的 Hash 文件名（`main.a1b2c.js`），文件没变就永久用缓存，变了就下载新的。

2.  **静态入口 (index.html)**：
    *   **策略**：**协商缓存**（no-cache）或者不缓存。
    *   **理由**：必须保证用户每次都能拿到最新的 html，才能引用到最新的 js/css。

3.  **动态接口 (API JSON 数据)**：
    *   **策略**：**不缓存**（no-store）。
    *   **理由**：你的购物车数据、订单状态是实时变化的，不能缓存旧数据。

---

### 💡 总结

> "从原理上讲，**静态页面**是服务器直接返回预先构建好的 HTML，不需要数据库参与，**性能高、SEO 好，利于 CDN 缓存**，但内容更新麻烦。
>
> **动态页面**是服务器根据请求**实时生成**内容（通常涉及数据库查询），**交互性强、易于维护内容**，但性能开销大。
>
> 在现在的**Vue/React 开发**中，我们通常采用**'静态 Shell + 动态数据'**的模式：
> 将前端打包成静态资源（HTML/JS/CSS）部署在 Nginx/CDN 上，通过 Axios 动态请求后端的 API 接口来渲染数据。为了解决这种模式的 SEO 问题，我们还会引入 **SSR (服务端渲染)** 技术。"





