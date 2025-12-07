### 内容包括：自动化部署（server-jenkins-nginx）  和 [前端涉及的网络相关内容](#前端涉及的网络相关的学习)

前端设置网络知识：

- RENAME：websocket，JWT，CSRF，XXS
- 其他md

# 老师：codewhy
# 自动化部署

### 1. 先购买服务器和Nginx环境的搭建

**Nginx作用：高性能的 Web 服务器和反向代理服务器**

- **主要功能**

  1. **静态资源托管**：把前端 React/Vue 项目的 `build` 文件夹放进去，用户访问网站时，Nginx 就把 `index.html`、CSS、JS 静态文件返回给浏览器。
  2. **反向代理**：用户访问 `http://example.com/api`，Nginx 可以把请求转发到后端（比如 Node.js、Spring Boot、Go 服务）。
  3. **负载均衡**：如果有多个后端服务，Nginx 可以分发请求，提升并发性能。
  4. **HTTPS 配置**：通过 SSL 证书让网站支持 `https://`。

- 安装命令（CentOS）：

  ```bash
  # 安装 Nginx
  sudo yum install -y nginx
  
  # 启动 Nginx
  sudo systemctl start nginx
  
  # 设置开机自启
  sudo systemctl enable nginx
  
  # 查看运行状态
  sudo systemctl status nginx
  ```

  安装命令（Ubuntu）：

  ```bash
  # 更新软件源
  sudo apt update
  
  # 安装 Nginx
  sudo apt install -y nginx
  
  # 启动 Nginx
  sudo systemctl start nginx
  
  # 设置开机自启
  sudo systemctl enable nginx
  ```

#### 配置 Nginx

编辑 Nginx 配置文件：

```bash
sudo vim /etc/nginx/conf.d/vueplustypescript.conf
```

配置内容：

```nginx
server {
    listen 80;
    server_name your-domain.com;  # 替换为你的域名或服务器 IP

    # 前端静态资源目录
    root /var/www/vueplustypescript/dist;
    index index.html;

    # 支持 Vue Router 的 history 模式
    location / {
        try_files $uri $uri/ /index.html;
    }

    # 反向代理 API 请求到后端
    location /api/ {
        proxy_pass http://localhost:8000;  # 替换为你的后端服务地址
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 静态资源缓存配置
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }

    # Gzip 压缩
    gzip on;
    gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
}
```

重启 Nginx：

```bash
sudo nginx -t  # 测试配置文件
sudo systemctl reload nginx  # 重新加载配置
```



### 2. Jenkins环境的搭建 -- 需要java环境



- 安装java

  - ```bash
    # CentOS
    sudo yum install -y java-11-openjdk java-11-openjdk-devel
    
    # Ubuntu
    sudo apt install -y openjdk-11-jdk
    
    # 验证安装
    java -version
    ```

- 安装jenkins

  - linus清空： clear

  - 安装步骤：

    ```bash
    # 添加 Jenkins 仓库
    sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
    sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
    
    # 安装 Jenkins
    sudo yum install -y jenkins
    
    # 启动 Jenkins
    sudo systemctl start jenkins
    
    # 作用：自动启动
    sudo systemctl enable jenkins
    
    # 查看初始密码
    sudo cat /var/lib/jenkins/secrets/initialAdminPassword
    ```

- **作用：自动化构建和持续集成/持续部署（CI/CD）工具**

  - **主要功能**
    1. **拉取代码**：从 GitHub、GitLab 等仓库获取最新代码。
    2. **自动构建**：执行 `npm install && npm run build`（前端），或 `mvn package`（Java），或 `go build`（Go）等。
    3. **自动测试**：在构建时跑单元测试、集成测试。
    4. **自动部署**：构建成功后，把产物（如 `build` 文件夹、jar 包）推送到服务器指定目录。
       1. **持续集成/持续部署**：当你 `git push` 代码时，Jenkins 可以自动触发构建和部署，不用你手动一遍遍操作



### 3. jenkins的使用过程

- 配置jenkins的环境

1. 你在本地写代码 `git push` 到 GitHub

2. Jenkins 自动拉取代码 → 构建出 `build` 文件夹

3. Jenkins 把 `build` 放到 Nginx 的目录里

4. 用户访问网站时，Nginx 把页面返回

- #### 安装 Node.js 环境

  Jenkins 构建需要 Node.js 环境：

  - jenkins有可视化界面

  ```bash
  # 安装 nvm（Node 版本管理工具）
  curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.0/install.sh | bash
  
  # 加载 nvm
  source ~/.bashrc
  
  # 安装 Node.js
  nvm install 20
  nvm use 20
  
  # 验证安装
  node -v
  npm -v
  ```

  ---





### 4. 手动部署和自动化部署

***\*流程图：\****

本地开发 → git push → GitHub → Webhook → Jenkins

​                      ↓

​                   拉取代码 → 安装依赖

​                      ↓

​                    构建项目 → 上传服务器

​                      ↓

​                    Nginx 托管 → 用户访问

\```



### 5. 总结所有的操作顺序（ABC为顺序）

#### A.本地vscode里面 安装remote插件，用其连接远程服务器

#### B.选择linus，vscode终端就是远程服务器的终端，这个终端下载Nginx

#### C.然后继续在那个终端安装java环境和jenkins

#### D.jenkins可能安装不了，因为找不到对应包，下面要手动添加对应仓库

#### E. 然后去服务器配置安全策略（配置端口），之后访问服务器地址加端口就可以看见jenkins

#### F. 会看到要密码才能登录，前面有段路径，赋值，然后再刚刚那个终端执行： cat 那个路径，然后返回一个密钥就是密码

#### G. 进入后选择安装推荐插件

#### H. 先在那个终端执行pwd ，看当前路径，再 mkdir 文件夹名（你打包那个包名），ls 看文件包名创建了吗，

#### I.终端那个页面有个打开文件夹，点击打开刚刚创建的文件夹,把打包好那个文件夹里面的文件复制到刚刚打开的文件夹里面

#### J.配置ngnix，可以去问ai，概况就是在nginx.conf里面配置好：作用：即使打开服务器网站不是jenkins默认页面，而是你的页面

#### L. 执行Systemcl restart nginx 重启

#### M. 访问服务器，进入jenkins，配置node，需要安装；配置仓库github，要是仓库不是开源则配置协议

#### N.在jenkins服务器里面配置构建时间和 git push那些自动部署代码，当构建失败，可能要回到刚刚那个终端配置root权限





# 前端涉及的网络相关的学习

- 其他md里面有具体内容，这里是案例

## 一。实例场景：用户访问“在线银行个人中心”并进行转账

#### 第一阶段：页面加载 (初始化阶段)

**涉及技术：静态/动态页面、浏览器缓存策略**

1.  **用户输入 URL** (`bank.com/dashboard`)。
2.  **请求入口文件 (`index.html`)**：
    *   **技术点 (静态页面)**：这是一个**静态页面**（Shell），里面只有一个 `<div id="app"></div>` 和一些 JS 引用。
    *   **技术点 (协商缓存)**：浏览器发现缓存里有这个文件，但不敢直接用。于是发请求问服务器（带上 `If-None-Match: "abc"`）。
    *   **服务器响应**：服务器看了一眼，文件没变，返回 **`304 Not Modified`**。浏览器直接加载本地的 HTML。
3.  **请求资源文件 (`main.a1b2.js`, `logo.png`)**：
    *   **技术点 (强缓存)**：浏览器看到文件名带 Hash（`a1b2`），且 Cache-Control 是 1 年。
    *   **结果**：**完全不发网络请求**，直接从硬盘读取，页面秒开。

---

#### 第二阶段：建立连接与实时通信

**涉及技术：HTTPS、WebSocket**

4.  **建立安全通道**：
    *   **技术点 (HTTPS)**：所有请求在 TCP 握手后，立即进行 **SSL/TLS 握手**。客户端和服务器交换公钥，生成会话密钥。从此，所有数据（Token、密码）在网线上传输时都是乱码，黑客监听也没用。
5.  **建立长连接**：
    *   **技术点 (WebSocket)**：页面加载完后，JS 初始化 WebSocket 连接。
    *   **场景**：不需要用户刷新，银行的“实时金价”或“股票走势”通过 WS 通道，由服务器**主动推**给前端。

---

#### 第三阶段：用户填写表单 (转账前夕)

**涉及技术：数据校验**

6.  **用户输入转账金额**：用户输入了 `10000`，收款人 `张三`。
7.  **前端检查**：
    *   **技术点 (数据校验)**：在点击“提交”按钮的一瞬间，`async-validator` 或正则拦截：
        *   “金额必须是数字”
        *   “金额不能为负数”
    *   **作用**：如果校验失败，根本**不发网络请求**，节省流量，提升体验。

---

#### 第四阶段：发起转账请求 (请求准备)

**涉及技术：敏感数据加密、API 接口签名**

8.  **组装数据**：校验通过，准备发请求。
    *   原始数据：`{ money: 10000, pwd: '123456', to: 'zhangsan' }`
9.  **加密密码**：
    *   **技术点 (敏感数据加密)**：前端用**RSA 公钥**把 `'123456'` 加密成一串乱码 `x8z7c...`。防止运维人员看日志泄露密码。
10.  **防止篡改**：
     *   **技术点 (API 接口签名)**：
         1.  把参数按字典序排序：`money=10000&pwd=xxx&timestamp=169999&to=zhangsan`
         2.  加上前端藏好的 Secret Key。
         3.  进行 MD5 运算生成 `sign: A1B2C3D4`。
         4.  把 `sign` 放到 Header 里。
     *   **作用**：如果黑客半路把金额改成 `100000`，后端算出来的签名跟 Header 里的对不上，直接拒绝。

---

#### 第五阶段：发送请求 (传输过程)

**涉及技术：JWT、CSRF Token、Axios 拦截器**

11. **请求拦截器工作**：Axios 准备发出 HTTP 请求。
    *   **技术点 (JWT)**：拦截器从 LocalStorage 取出 `access_token`，塞入 Header：`Authorization: Bearer eyJhb...`。这是用户的身份证明。
    *   **技术点 (CSRF)**：拦截器从 meta 标签取出 `csrf_token`，塞入 Header：`X-CSRF-TOKEN: xyz...`。
    *   **作用**：证明这是“用户本人”在“银行官网”发出的请求，而不是黑客伪造的。

---

#### 第六阶段：处理响应 (后端反馈)

**涉及技术：Token 无感刷新、XSS 防御**

12. **意外发生 (Token 过期)**：
    *   请求发出去了，但后端返回 **`401 Unauthorized`**（Token 刚才过期了）。
13. **补救措施**：
    *   **技术点 (Token 无感刷新)**：
        1.  Axios 响应拦截器捕获到 401。
        2.  **暂停**当前的转账请求，把它挂起。
        3.  拿着长效的 `refresh_token` 去找后端换一个新的 `access_token`。
        4.  换成功后，**重发**刚才挂起的转账请求。
    *   **用户感知**：完全无感知，觉得转账一次就成功了。
14. **渲染结果**：
    *   后端返回：`{ status: 'success', note: '转账给 <b>张三</b> 成功' }`。
    *   **技术点 (XSS 防御)**：前端在渲染 `note` 时，使用转义函数把 `<b>` 变成 `&lt;b&gt;`，防止如果有恶意脚本被执行。

---

#### 第七阶段：页面关闭 (结束)

**涉及技术：sendBeacon**

15. **用户关闭网页**：用户点右上角叉号。
16. **发送埋点**：产品经理需要统计用户停留时长。
    *   **技术点 (sendBeacon)**：在 `visibilitychange` 事件中，调用 `navigator.sendBeacon('/api/log', blob)`。
    *   **作用**：即使页面进程被杀死了，浏览器也会在后台默默把这条数据发给服务器，确保统计数据不丢失。





## 二。第五阶段的详细介绍

这一步是前后端交互中**最关键的“安检环节”**。

你可以把**Axios 拦截器**想象成机场的**安检口**。所有要飞往后端服务器的请求（旅客），必须在这里停下来，接受两项最严格的检查：**身份检查 (JWT)** 和 **来源检查 (CSRF Token)**。

只有两个证件都齐全且有效，请求才会被放行。

---

### 1. 技术点一：JWT (JSON Web Token) —— “身份证”

**你的描述：** “拦截器从 LocalStorage 取出 access_token，塞入 Header。”

#### 它的核心作用：身份认证 (Authentication)

*   **解决的问题：** 服务器不知道你是谁。
*   **比喻：** 这就是你的 **“身份证”** 或 **“护照”**。

#### 详细过程：

1.  **取证**：当你在代码里写 `axios.post('/transfer')` 时，请求还没发出去。拦截器先运行，它去浏览器保险箱（LocalStorage）里翻找，找到了之前登录时存下的 `access_token`。
2.  **亮证**：拦截器把这个长长的字符串（eyJhb...）贴在请求的 **Header** 上，字段名叫 `Authorization`，格式通常是 `Bearer <token>`。
3.  **核验**：请求到达银行服务器。服务器解密这个 Token，读出里面的信息：“哦，你是用户 ID 10086，名叫曹迈，有效期还有 20 分钟。”
4.  **结果**：身份确认，服务器知道该扣谁的钱了。

**如果没有它？**
服务器会问：“你要转账？但我不知道你是谁啊？” -> 返回 **401 Unauthorized**。

---

### 2. 技术点二：CSRF Token —— “防伪印章”

**你的描述：** “拦截器从 meta 标签取出 csrf_token，塞入 Header。”

- meta这一步下面有

#### 它的核心作用：防伪/鉴权 (Authorization / Anti-Forgery)

*   **解决的问题：** 服务器不知道这个请求是你**自己点**的，还是**黑客骗你点**的。
*   **比喻：** 这就是机场安检时，在你登机牌上盖的那个 **“安检章”**，或者你们之间的 **“接头暗号”**。

#### 详细过程（为什么要这么做？）：

这里是难点，请仔细看：

1.  **暗号的埋藏**：
    *   当你打开“银行转账页”时，银行服务器在返回给你的 HTML 里，偷偷藏了一个暗号（随机字符串），通常放在 `<meta name="csrf-token" content="xyz...">` 里。
    *   **关键点**：由于浏览器的**同源策略**，**只有银行的网页**能读取这个 meta 标签。黑客的网站（evil.com）虽然能诱导你发请求，但他**读不到**你银行页面里的这个 meta 标签。

2.  **取暗号**：
    *   Axios 拦截器运行。它用 JS 代码 `document.querySelector` 去页面里把这个暗号拿出来。

3.  **对暗号**：
    *   拦截器把暗号贴在 Header 的 `X-CSRF-TOKEN` 字段里。

4.  **审问**：
    *   请求到达服务器。服务器一看：“哟，有身份证（JWT），确实是曹迈。但我得看看是不是本人操作。”
    *   服务器检查 `X-CSRF-TOKEN`：“暗号对上了！说明这个请求是从我们银行自家的页面发出来的，不是从黑客网站发出来的。”

**如果没有它？**
这就发生了 **CSRF 攻击**。黑客在他的网站放一个按钮，你一点，请求发到了银行。虽然你带了身份证（如果是 Cookie 模式），但因为没有这个“暗号”，银行分不清是你在操作还是黑客在利用你的身份操作。

---

### 3. 代码实战：Axios 拦截器里的“双剑合璧”

这就是传说中的“防御性编程”代码：

```javascript
// src/utils/request.js
import axios from 'axios';

const service = axios.create({ baseURL: '/api' });

// ✈️ 请求拦截器：起飞前的安检
service.interceptors.request.use((config) => {
  
  // --- 检查项 1：身份检查 (JWT) ---
  const token = localStorage.getItem('token');
  if (token) {
    // 告诉服务器：我是谁
    config.headers['Authorization'] = `Bearer ${token}`; 
  }

  // --- 检查项 2：来源检查 (CSRF Token) ---
  // 只有发 POST/PUT/DELETE 等修改数据的请求时才需要防伪
  // GET 请求一般不防，因为只读
  if (['post', 'put', 'delete'].includes(config.method)) {
    
    // 尝试从页面 meta 标签获取暗号
    const csrfMeta = document.querySelector('meta[name="csrf-token"]');
    const csrfToken = csrfMeta ? csrfMeta.getAttribute('content') : null;
    
    if (csrfToken) {
      // 告诉服务器：这是正版页面发出的请求
      config.headers['X-CSRF-TOKEN'] = csrfToken;
    }
  }

  return config; // 放行
}, (error) => {
  return Promise.reject(error);
});

export default service;
```

---

### 问题一：config.headers['Authorization'] = Bearer ${token}; 这个和      config.headers['X-CSRF-TOKEN'] = csrfToken;不会覆盖吗

这两个问题问得非常细致，说明你在认真思考代码的运行逻辑。

直接给你定心丸：

1.  **绝对不会覆盖**，它们和平共处。
2.  **`<meta>` 写法是业界惯例**，虽然不是 W3C 强制标准，但几乎所有主流后端框架（Spring Security, Laravel, Rails）都遵循这个套路。

我来详细拆解给你看。

**结论：它们是 HTTP Header 里的两个不同的“键值对”，互不干扰。**

你在代码里写的 `config.headers` 本质上是一个 **JavaScript 对象**。

#### 代码执行过程还原：

```javascript
// 1. Axios 刚开始准备请求头，可能有一些默认值
// config.headers = { 
//   'Accept': 'application/json',
//   'Content-Type': 'application/json' 
// }

// 2. 执行第一行代码：加入 Authorization
config.headers['Authorization'] = `Bearer eyJhbG...`;

// 3. 执行第二行代码：加入 X-CSRF-TOKEN
config.headers['X-CSRF-TOKEN'] = 'xyz-123...';
```

#### 最终发送给后端的 Header 长这样：

```javascript
{
  "Accept": "application/json",
  "Content-Type": "application/json",
  "Authorization": "Bearer eyJhbG...",  // 👈 身份证明在这
  "X-CSRF-TOKEN": "xyz-123..."          // 👈 防伪暗号在这
}
```

---

### 

### 问题二：const csrfMeta = document.querySelector('meta[name="csrf-token"]');这个标准是什么样的，是这样的吗<meta name=“csrf-token

你写的 `<meta name=“csrf-token”` 思路是对的，但**标点符号（中文引号）**是错的，且缺少了最重要的 **`content`** 属性。

#### 1. 标准的 HTML 写法（后端渲染出的 HTML）

这是**后端（如 Java JSP, PHP, Python 模板）**生成 HTML 时，约定俗成的标准写法：

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <!-- 👇 核心在这里：name 是键，content 是值 -->
  <!-- 注意：必须用英文引号 "" -->
  <meta name="csrf-token" content="这里是一串后端生成的随机乱码123456">
</head>
<body>
  ...
</body>
</html>
```

#### 2. 前端 JS 读取的标准写法

```javascript
// 1. 选中这个标签
const metaTag = document.querySelector('meta[name="csrf-token"]');

// 2. 如果标签存在，就读取它的 content 属性
if (metaTag) {
  const csrfToken = metaTag.getAttribute('content');
  // 结果：csrfToken = "这里是一串后端生成的随机乱码123456"
}
```

#### 3. 为什么是这个名字？（行业惯例）

虽然 HTML 标准里没有规定必须叫 `csrf-token`，但这是各大后端框架的**默认配置**：

*   **Laravel (PHP)**: 默认生成 `<meta name="csrf-token">`。
*   **Spring Security (Java)**: 默认叫 `_csrf`，但通常配置成 `X-CSRF-TOKEN`。
*   **Rails (Ruby)**: 默认叫 `csrf-token`。

#### 4.知识补充

> “这通常是后端 MVC 框架（如 Laravel 或 Spring）的**最佳实践**。后端在渲染 HTML 模板（如 JSP 或 Blade）时，会将 CSRF Token 放在 `<head>` 的 `<meta>` 标签中。前端通过 DOM API 读取这个标签的 `content` 属性，并在发送 AJAX 请求时将其放入 Header 中。这样做的好处是 Token 此时是静态存在于 DOM 中的，黑客跨域无法读取。”



#### 二的总结

“为什么请求头里既要有 Authorization 又要有 X-CSRF-TOKEN？”

> “这两个 Token 解决的是完全不同的安全问题：
>
> 1.  **`Authorization` (JWT)** 解决的是 **'Who are you' (你是谁)** 的问题。它用来证明用户的身份，让服务器知道当前操作者是哪个用户。
>
> 2.  **`X-CSRF-TOKEN`** 解决的是 **'Where are you from' (你从哪来)** 的问题。它用来证明这个请求是用户在**我们的网页**上主动发起的，而不是被黑客诱导在**第三方网页**上发起的。
>
> 黑客通过跨域也许能伪造请求（带上 Cookie），但他**拿不到**页面内隐藏的 CSRF Token（受同源策略限制）。所以，只有两个 Token 同时验证通过，服务器才会认为这是一个**安全、合法**的请求。”





### 

