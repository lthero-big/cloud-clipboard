# cloud-clipboard

因为不想为了手机和电脑互传文件这种小事就扫🐴登录某个辣鸡 APP，而自己折腾出来的一个在线剪贴板。

* 支持**传输纯文本**和**一键复制**
* 支持**传输文件**（选择文件 / 拖拽 / <kbd>Ctrl+V</kbd> 粘贴截图），对于图像可以显示缩略图
* 使用 WebSocket 实现实时通知
* 前端使用 [Vue 2](https://cn.vuejs.org) 和 [Vuetify](https://vuetifyjs.com/zh-Hans/) 构建
* 后端使用 [Node.js](https://nodejs.org) ([Koa](https://github.com/koajs/koa)) 构建 

*由于这个项目是针对个人在连接到同一局域网（比如家里的路由器）的设备之间使用而设计的，因此并没有额外考虑在公开的服务器上使用时可能面对的技术和安全问题。*

## 截图

<details>
<summary>桌面端</summary>

![](https://ae01.alicdn.com/kf/Hfce3a9b69b3d404c8e3073ab0fffa913v.png)

</details>

<details>
<summary>移动端</summary>

![](https://ae01.alicdn.com/kf/Hbf859dd0e42c4406bf94a6b6f2f4658cf.png)

</details>

## 使用方法

### Node.js 版服务端

#### 安装和运行


使用 [caxa](https://github.com/leafac/caxa) 和 GitHub Actions 打包成了可以在 Windows amd64 和 Linux amd64 使用的可执行文件，可以在[这里](https://nightly.link/TransparentLC/cloud-clipboard/workflows/ci/master)下载。

*caxa 的打包原理相当于将 Node.js 的可执行文件和所有代码一起做成了一个自解压压缩包，执行时会解压到临时文件夹，并且在退出时不会自动清空。*

配置文件是按照以下顺序尝试读取的：

* 和可执行文件放在同一目录的 `config.json`
* 在命令行中指定：`cloud-clipboard /path/to/config.json`

#### 从源代码运行

需要安装 [Node.js](https://nodejs.org)。

```bash
# 构建前端资源，只需要执行一次
# 也可以直接从 Actions 下载构建好的压缩包（static），解压到 server-node/static
cd client
npm install
npm run build

# 运行服务端
cd ../server-node
npm install
node main.js
```

配置文件是按照以下顺序尝试读取的：

* 和 `main.js` 放在同一目录的 `config.json`
* 在命令行中指定：`node main.js /path/to/config.json`

服务端默认会监听本机所有网卡的 IP 地址（也可以自己设定），并在终端中显示前端界面所在的网址，使用浏览器打开即可使用。

如果你使用的是 Node.js 17 或以上的版本，构建前端资源时可能会遇到 `Error: error:0308010C:digital envelope routines::unsupported` 的错误，在终端里设置环境变量 `NODE_OPTIONS=--openssl-legacy-provider` 可以解决这个问题。

#### 使用 Docker 运行

##### 自己打包

```bash
docker image build -t myclip .
docker container run -d -p 9501:9501 myclip
```

##### 从 Docker Hub 拉取

> [!TIP]
> Docker Hub 上的镜像是由他人打包的，仅为方便使用而在这里给出，版本可能会滞后于 repo 内的源代码。
>
> 如果你在使用时遇到了问题，请先确认这个问题在 repo 内的最新的源代码中是否仍然存在。

```bash
docker pull lthero1/lthero-onlineclip:latest
docker container run -d -p 9501:9501 lthero1/lthero-onlineclip
```

访问 [clipboard](http://127.0.0.1:9501)

### 配置文件说明

`//` 开头的部分是注释，**并不需要写入配置文件中**，否则会导致读取失败。

```json
{
    "server": {
        // 监听的 IP 地址，省略或设为null则会监听所有网卡的IP地址
        "host": [
            "127.0.0.1",
            "::1"
        ],
        "port": 9501, // 端口号
        "key": "localhost-key.pem", // HTTPS 私钥路径
        "cert": "localhost.pem", // HTTPS 证书路径
        "history": 10, // 消息历史记录的数量
        "auth": false // 是否在连接时要求使用密码认证，falsy 值表示不使用
    },
    "text": {
        "limit": 4096 // 文本的长度限制
    },
    "file": {
        "expire": 864000, // 上传文件的有效期，超过有效期后自动删除，单位为秒
        "chunk": 12582912, // 上传文件的分片大小，不能超过 5 MB，单位为 byte
        "limit": 1073741824 // 上传文件的大小限制，单位为 byte
    }
}
```
> HTTPS 的说明：
>
> 如果同时设定了私钥和证书路径，则会使用 HTTPS 协议访问前端界面，未设定则会使用 HTTP 协议。
> 自用的话，可以使用 [mkcert](https://mkcert.dev/) 自行生成证书，并将根证书添加到系统/浏览器的信任列表中。
> 如果使用了 Nginx 等软件的反向代理，且这些软件已经提供了 HTTPS 连接，则无需在这里设定。
>
> “密码认证”的说明：
>
> 如果启用“密码认证”，只有输入正确的密码才能连接到服务端并查看剪贴板内容。
> 可以将 `server.auth` 字段设为 `true`（随机生成六位密码）或字符串（自定义密码）来启用这个功能，启动服务端后终端会以 `Authorization code: ******` 的格式输出当前使用的密码。
>

---

# 使用HTTP访问

要将运行在Docker容器中的服务通过域名访问，并使用Nginx作为反向代理来转发到宿主机的9501端口，你需要完成几个步骤。这包括设置DNS记录、配置Nginx以及确保网络安全。下面是具体步骤：

## 步骤 1: 设置DNS记录

确保你的域名 `clip.lthero.top` 的DNS记录指向托管Nginx的服务器的IP地址。这通常在你的域名注册商处进行设置：

- **A记录**：将域名指向IPv4地址。
- **AAAA记录**：将域名指向IPv6地址（如果适用）。

## 步骤 2: 安装并启动Nginx

步骤 1: 更新软件包列表

打开终端，首先使用`apt`命令更新你的包列表，以确保你安装的是最新版本的Nginx。

```sh
sudo apt update
```

步骤 2: 安装Nginx

使用`apt`安装Nginx。

```sh
sudo apt install nginx
```

## 步骤 3: 配置Nginx

你需要在Nginx中创建一个新的服务器块（server block），或者在已有的默认配置中修改，以设置反向代理。以下是一个基本的Nginx配置示例，将会把所有到 `clip.lthero.top` 的请求转发到本地的9501端口：

1. 打开或创建一个新的Nginx配置文件：

   ```bash
   sudo nano /etc/nginx/sites-available/clip.lthero.top
   ```

2. 添加以下配置：

   ```nginx
   server {
       listen 80;
       server_name clip.lthero.top;
   
       location / {
           proxy_pass http://localhost:9501;
           proxy_http_version 1.1;
           proxy_set_header Upgrade $http_upgrade;
           proxy_set_header Connection 'upgrade';
           proxy_set_header Host $host;
           proxy_cache_bypass $http_upgrade;
       }
   }
   ```

   这个配置做了以下几点：

   - `listen 80;` 告诉Nginx监听80端口（HTTP标准端口）。
   - `server_name clip.lthero.top;` 设置这个块应当响应的域名。
   - `proxy_pass http://localhost:9501;` 指定所有传入的请求转发到本地的9501端口。
   - `proxy_set_header` 指令将重要的HTTP头信息转发给后端应用。

3. 启用配置文件通过创建一个符号链接：

   ```bash
   sudo ln -s /etc/nginx/sites-available/clip.lthero.top /etc/nginx/sites-enabled/
   ```

4. 检查Nginx配置文件是否有语法错误：

   ```bash
   sudo nginx -t
   ```

5. 如果没有错误，重启Nginx以应用配置：

   ```bash
   sudo systemctl restart nginx
   ```

## 步骤 4: 调整防火墙规则

确保你的服务器的防火墙规则允许HTTP（端口80）和HTTPS（端口443，如果你使用SSL）的流量。如果你正在使用`ufw`，可以使用以下命令：

```bash
sudo ufw allow 'Nginx Full'
sudo ufw reload
```

## 步骤 5: 测试配置

在浏览器中输入 `http://clip.lthero.top` 或使用命令行工具如 `curl` 来测试你的配置：

```bash
curl http://clip.lthero.top
```

你应该能看到从Docker容器中运行的服务响应的内容。

这样，你就配置好了Nginx作为反向代理，将域名 `clip.lthero.top` 的流量转发到宿主机的9501端口上的服务。如果你希望使用HTTPS，你还需要设置SSL证书，可以考虑使用Let's Encrypt免费证书并配置HTTPS。



# 使用HTTPS访问

要让你的域名 `clip.lthero.top` 使用 HTTPS，你需要获取 SSL/TLS 证书，并配置 Nginx 以使用这些证书来加密网页内容。以下是详细的步骤，包括如何使用 Let's Encrypt 提供的免费证书自动化这个过程。

## 步骤 1: 安装 Certbot

Certbot 是一个自动获取并安装 Let's Encrypt 证书的客户端。在 Ubuntu 上安装 Certbot 及其 Nginx 插件非常简单：

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx
```

## 步骤 2: 获取和安装证书

使用 Certbot 获取并为你的域名安装证书：

```bash
sudo certbot --nginx -d clip.lthero.top
```

此命令会自动为指定的域名 `clip.lthero.top` 配置 SSL 证书，并更新 Nginx 配置以使用这些证书。Certbot 会询问你一些问题，比如电子邮件地址（用于紧急联系和证书续订提醒），以及是否重定向所有 HTTP 请求到 HTTPS（强烈建议启用）。

## 步骤 3: 更新 Nginx 配置

如果你想手动编辑 Nginx 配置文件，可以按以下方式配置：

```nginx
server {
    listen 80;
    server_name clip.lthero.top;
    return 301 https://$server_name$request_uri;  # 强制重定向所有 HTTP 请求到 HTTPS
}


server {
    listen 443 ssl http2;
    server_name clip.lthero.top;

    ssl_certificate /etc/letsencrypt/live/clip.lthero.top/fullchain.pem;  # 证书文件路径
    ssl_certificate_key /etc/letsencrypt/live/clip.lthero.top/privkey.pem;  # 私钥文件路径

    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # 缓存 SSL 会话以提升性能
    ssl_session_tickets off;

    # 现代加密套件配置
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384';
    ssl_prefer_server_ciphers on;

    # 其他 SSL 优化设置
    ssl_stapling on;
    ssl_stapling_verify on;
	
    # 允许最大请求体大小为 1000MB
    client_max_body_size 1000m;  
    location / {
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;  # 指向你的密码文件

        proxy_pass http://localhost:9501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

这个配置不仅启用了 HTTPS，还包括了一些现代的安全实践，如启用 HTTP/2，配置加密套件和协议等。

> 允许最大请求体大小为 1000MB
>
> client_max_body_size 1000m;
>
> 这行最好与config.js中保持一致，如果**nginx设置小了，会出现“上传失败”的结果！**





## 步骤 4: 重新加载 Nginx

更改配置后，需要重新加载 Nginx 以应用新的配置：

```bash
sudo nginx -t  # 检查配置文件是否有语法错误
sudo systemctl reload nginx  # 重新加载配置
```

## 步骤 5: 验证 HTTPS

在浏览器中访问 `https://clip.lthero.top` 来检查是否配置成功。你应该能够看到一个安全锁标志，表明连接是通过 HTTPS 加密的。

## 步骤 6: 自动续订证书

Let's Encrypt 的证书有效期为90天，因此建议设置自动续订：

```bash
sudo certbot renew --dry-run
```

这个命令会测试证书续订过程。如果这个测试成功，Certbot 会在系统的定时任务中安排自动续订。

通过以上步骤，你的站点 `clip.lthero.top` 现在应该能够安全地使用 HTTPS 进行通信了。


---

# 密码认证

要求用户在访问 `clip.lthero.top` 域名时输入密码，可以通过在 Nginx 服务器上配置基本的 HTTP 认证来实现。这需要设置用户名和密码，以及修改 Nginx 的配置文件来要求认证。以下是详细步骤：

## 步骤 1: 创建密码文件

首先，你需要创建一个密码文件来存储用户名和经过加密的密码。这通常使用 `htpasswd` 工具完成，该工具随 Apache 提供的 `apache2-utils` 包一起安装。如果你的系统上还没有安装这个包，可以使用以下命令安装：

```bash
sudo apt update
sudo apt install apache2-utils
```

接着，创建密码文件并添加用户。例如，创建一个名为 `clipadmin` 的用户：

```bash
sudo htpasswd -c /etc/nginx/.htpasswd clipadmin
```

系统会提示你输入并确认密码。`-c` 选项用于创建新文件，如果已经有文件存在且仅需要添加或更新用户，不要使用 `-c` 选项。

## 步骤 2: 配置 Nginx 以要求认证

编辑你的 Nginx 配置文件，通常位于 `/etc/nginx/sites-available/clip.lthero.top`，在适当的 `location` 或 `server` 块中添加认证配置：

```nginx
server {
    listen 443 ssl http2;
    server_name clip.lthero.top;

    ssl_certificate /etc/letsencrypt/live/clip.lthero.top/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/clip.lthero.top/privkey.pem;

    location / {
        auth_basic "Restricted Content";
        auth_basic_user_file /etc/nginx/.htpasswd;  # 指向你的密码文件

        proxy_pass http://localhost:9501;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

这里的 `auth_basic` 指令启用基本认证，并设置提示信息为 "Restricted Content"。`auth_basic_user_file` 指定了包含用户名和密码的文件路径。

## 步骤 3: 重新加载 Nginx

更改配置后，确保测试 Nginx 配置文件没有错误，然后重新加载 Nginx 使配置生效：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

## 步骤 4: 测试认证功能

现在，当你访问 `https://clip.lthero.top` 时，浏览器应该会弹出一个登录对话框要求输入用户名和密码。只有正确输入认证信息后，用户才能访问站点内容。

这样你就完成了为 `clip.lthero.top` 配置基本 HTTP 认证的所有步骤，增加了一个访问该站点的安全层。



## 修改用户名

对于修改用户名，`htpasswd` 工具本身不提供直接修改用户名的选项。你需要先删除旧的用户名，然后添加新的用户名与密码。这里是如何操作的：

1. **删除旧用户名**:
   使用 `htpasswd` 命令删除旧的用户名 `clipadmin`：

   ```bash
   sudo htpasswd -D /etc/nginx/.htpasswd clipadmin
   ```

   这个命令会从指定的密码文件中删除 `clipadmin` 用户。

2. **添加新用户名**:
   现在，你可以添加新的用户名 `lthero`，并为其设置密码：

   ```bash
   sudo htpasswd -b /etc/nginx/.htpasswd lthero 新密码
   ```

   `-b` 参数允许你直接在命令行中提供密码，这可以简化脚本中的使用。但请注意，这种方式可能会导致密码暴露在历史记录中。如果安全是一个关注点，应该省略 `-b` 参数，命令将提示你输入密码。

### 确保 Nginx 使用更新的密码文件

完成用户名更新后，无需修改 Nginx 的配置，只要确保使用的是同一个 `.htpasswd` 文件。你可以通过重载 Nginx 来确保所有设置都是最新的：

```bash
sudo nginx -t  # 检查配置是否正确
sudo systemctl reload nginx  # 应用更改
```

这样就完成了用户名的更改，用户在访问 `clip.lthero.top` 时现在需要使用新的用户名 `lthero` 和其密码进行认证。


---


# 命令行上传文件|上传内容

## 【curl】上传文字内容

```sh
curl -X POST -H "Content-Type: text/plain" -d "这里是您要发送的文本" https://clip.lthero.top/text?room=test
```

* `?room`参数一定要有
* `?room=` 表示公共房间
* `?room=test`则是上传到test房间



## 【python】上传文字内容

简单版本

```python
import requests

# 发送文本到房间test，如果公共房间则用?room=即可
text_url = 'https://clip.lthero.top/text?room=test'
data = "testtest"
headers = {'Content-Type': 'text/plain'}
response = requests.post(text_url, data=data, headers=headers)
print(response.text)
```

复杂版本

`upload_text.py`

```python
import requests
import argparse

def send_text(text_url, data, room):
    # 如果提供了房间名称，将其添加到URL中
    text_url = text_url+"?room="+room
    headers = {'Content-Type': 'text/plain'}
    response = requests.post(text_url, data=data, headers=headers)
    return response

def main():
    parser = argparse.ArgumentParser(description='Send text to a specified room via the web API.')
    parser.add_argument('data', type=str, help='The text data to send.')
    parser.add_argument('--room', type=str, default="", help='The name of the room (optional).')

    args = parser.parse_args()

    text_url = 'https://clip.lthero.top/text'
    response = send_text(text_url, args.data, args.room)
    # 检查状态码来判断是否成功
    if response.status_code == 200:
        print("发送成功:")
    else:
        print(f"发送失败，状态码 {response.status_code}: {response.text}")

if __name__ == '__main__':
    main()
```

* 使用命令：`python upload_text.py 1234 ` 上传1234到公共房间





## 【python】上传文件

`upload_file.py`

> 支持同时上传多个文件

```python
import requests
import os
import argparse

def upload_file(url, file_paths, room):
    for file_path in file_paths:
        file_name = os.path.basename(file_path)
        file_size = os.path.getsize(file_path)

        # 初始化上传
        init_response = requests.post(f'{url}/upload', data=file_name, headers={'Content-Type': 'text/plain'})
        if init_response.status_code != 200:
            print(f"初始化失败: {file_name}, {init_response.text}")
            continue
        
        uuid = init_response.json()['result']['uuid']
        
        # 分块上传文件
        chunk_size = 12582912  # 12MB
        with open(file_path, 'rb') as f:
            uploaded_size = 0
            while uploaded_size < file_size:
                chunk = f.read(chunk_size)
                chunk_response = requests.post(f'{url}/upload/chunk/{uuid}', data=chunk, headers={'Content-Type': 'application/octet-stream'})
                if chunk_response.status_code != 200:
                    print(f"上传块失败: {file_name}, {chunk_response.text}")
                    break
                uploaded_size += len(chunk)
        
        # 完成上传
        finish_response = requests.post(f'{url}/upload/finish/{uuid}', params={'room': room})
        if finish_response.status_code != 200:
            print(f"完成上传失败: {file_name}, {finish_response.text}")
        else:
            print(f"文件上传成功: {file_name}")

def main():
    parser = argparse.ArgumentParser(description='File Upload Script')
    parser.add_argument('file_paths', type=str, nargs='+', help='Path(s) to the files to be uploaded')
    parser.add_argument('--room', type=str, default='', help='Room name for the upload context (optional)')

    args = parser.parse_args()

    upload_url = 'https://clip.lthero.top'
    upload_file(upload_url, args.file_paths, args.room)

if __name__ == '__main__':
    main()

```

* 使用命令：
* `python upload_file.py /media/Stuff/config.json` 上传config.json文件到公共房间
* `python upload_file.py /media/Stuff/config.json --room test ` 上传config.json文件到test房间
* `python upload_file.py file1.txt file2.jpg file3.pdf --room test ` 上传多个文件到test房间
* `python upload_file.py file* --room test ` 支持**符号写法**，上传多个文件（file1.txt file2.jpg file3.pdf）到test房间



## 全局使用

要在全局任意路径都可以使用 `upload_file.py` 和 `upload_text.py` 这两个脚本，涉及到以下几个步骤：

### 1. 将脚本移动到全局可访问的路径

将脚本放在一个所有用户都能访问的目录中，如 `/usr/local/bin`。这个目录通常已经在大多数 Linux 发行版的环境变量 `$PATH` 中，所以放在这里的程序可以被全局调用。

首先，确保脚本具有可执行权限：

```bash
chmod +x upload_file.py
chmod +x upload_text.py
```

然后，将它们移动到 `/usr/local/bin` 目录：

```bash
sudo cp upload_file.py /usr/local/bin/upFile2ClipBoard
sudo cp upload_text.py /usr/local/bin/upText2ClipBoard
```

### 2. 修改脚本的首行（Shebang）

修改`/usr/local/bin/upFile2ClipBoard`和`/usr/local/bin/upText2ClipBoard`

确保脚本的第一行指向 Python 解释器的正确路径。这被称为 "shebang" 行

```python
#!/usr/bin/env python3
```

如果不知道python路径，使用`which python`可以查询出来

```bash
$ which python
/opt/anaconda3/bin/python
```

则需要添加`#!/opt/anaconda3/bin/python`在`/usr/local/bin/upText2ClipBoard`最上面

### 3. 更新环境变量（如果需要）

如果将脚本放在除了 `/usr/local/bin` 之外的目录，可能需要将该目录添加到环境变量 `$PATH` 中。编辑 shell 配置文件（例如 `~/.bashrc` 或 `~/.zshrc`），添加以下行：

```bash
export PATH="$PATH:/path/to/your/script_directory"
```

之后，运行 `source ~/.bashrc` （或对应的配置文件），以使更改生效。

### 4. 直接调用脚本

完成上述步骤后，应该能够从任何目录直接调用 `upload_file` 和 `upload_text`，如下所示：

```bash
upFile2ClipBoard somefile.txt
upText2ClipBoard "Here is some text" --room "exampleRoom"
```

