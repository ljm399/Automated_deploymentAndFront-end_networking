# learning-server-jenkins-nginx
远程服务器，自动化部署的学习
# 老师：codewhy
### 自动化部署

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







