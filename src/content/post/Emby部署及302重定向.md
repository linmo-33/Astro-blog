---
title: Emby部署及302重定向  
date: 2025-01-10 06:43:27   
categories: 折腾日记
tags: 
keywords: Emby
description: 实现Emby播放走直链，不消耗本地流量
top_img: https://cdn.jsdelivr.net/gh/linmo-33/imgs/img/202501101015644.png
cover: https://cdn.jsdelivr.net/gh/linmo-33/imgs/img/202501101015644.png
---

# Emby+302重定向

## 1.Rclone挂载alist

### 1.1安装Rclone

```bash
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```

### 1.2创建Rclone配置

运行`rclone config`命令

```bash
# 进入rclone设置
rclone config
# 选择新远程
No remotes found, make a new one?
n) New remote
s) Set configuration password
q) Quit config
n/s/q> n #这里选择n
# 设置名字
name> 189Cloud
Type of storage to configure.
Choose a number from below, or type in your own value
[snip]
XX / WebDAV
   \ "webdav"
[snip]
Storage> webdav #这里输入webdav，也可以选择有个webdav的字段XX
# 设置远程地址url http://your_alist_ip:port/dav
URL of http host to connect to
Choose a number from below, or type in your own value
 1 / Connect to example.com
   \ "https://example.com"
url> http://127.0.0.1:8080/dav #这里设置alist的地址和端口，后面要带dav,http://#.#.#.#:5244/dav/，#号记得替换为ip
# 这里选6 选择带有other的字段
Name of the WebDAV site/service/software you are using
Choose a number from below, or type in your own value
 1 / Fastmail Files
   \ (fastmail)
 2 / Nextcloud
   \ (nextcloud)
 3 / Owncloud
   \ (owncloud)
 4 / Sharepoint Online, authenticated by Microsoft account
   \ (sharepoint)
 5 / Sharepoint with NTLM authentication, usually self-hosted or on-premises
   \ (sharepoint-ntlm)
 6 / Other site/service or software
   \ (other)
vendor> 6    
# 设置远程账号
User name
user> admin #alist的账号
# 设置远程密码
Password.
y) Yes type in my own password
g) Generate random password
n) No leave this optional password blank
y/g/n> y #这里输入y
Enter the password: #alist密码，密码是看不到的
password:
Confirm the password: #再次输入密码
password:
# 这里直接回车即可
Bearer token instead of user/pass (e.g. a Macaroon)
bearer_token>

# 这里可能会问你是默认还是高级
Edit advanced config?
y) Yes
n) No (default)
y/n> n  #选择n

#后面的回车即可
# 你的远程信息
--------------------
[remote]
type = webdav
url = http://#.#.#.#:5244/dav/
vendor = Other
user = admin
pass = *** ENCRYPTED ***
--------------------
# 确认
y) Yes this is OK
e) Edit this remote
d) Delete this remote
y/e/d> y #输入y即可，
# 最后按q退出设置
```

###  1.3 fuse安装

```bash
apt-get install fuse3
```

###  1.4挂载OneDrive到本地

创建服务配置文件：

```bash
sudo vim /etc/systemd/system/rcloned.service
```

保存以下内容

```bash
[Unit]
Description=Rclone Daemon
After=network-online.target
Wants=network-online.target

[Service]
Type=forking
ExecStart=/usr/bin/rclone mount onewebDav: /home/onedrive --network-mode --no-check-certificate --allow-other --allow-non-empty --header "Referer:" --vfs-cache-mode full --buffer-size 512M --vfs-read-chunk-size 64M --vfs-read-chunk-size-limit 1G --vfs-cache-max-size 10G --cache-dir /home/rclone/cahe --cache-info-age 4h  --timeout 2h --drive-chunk-size 64M --attr-timeout 72h --log-level ERROR --log-file /home/rclone/log/rclone-onedrive.log --umask 000 --use-mmap --daemon
ExecStop=/bin/fusermount -quz /home/onedrive
Restart=on-abort
RestartSec=10

[Install]
WantedBy=multi-user.targetsermount -quz /home/onedrive
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**注:此挂载文件夹的名字需要与 Alist 中的盘名相同**



重新加载 Systemd 配置：

```bash
sudo systemctl daemon-reload
```

重新启用服务：

```bash
sudo systemctl enable rcloned.service
```

启动服务

```bash
sudo systemctl start rcloned.service
```

检查服务状态：

```bash
sudo systemctl status rcloned.service
```

查看详细日志：

```bash
sudo journalctl -xe
```



## 2.安装Emby

### 2.1创建 Emby 数据目录

```bash
mkdir /home/emby/config
```

### 2. 2新建docker-compose文件

```bash
services:
  emby:
    image: lovechen/embyserver:latest
    container_name: emby  # 容器名称
    restart: unless-stopped
    security_opt:
      - apparmor:unconfined
    cap_add:
      - SYS_ADMIN
    volumes:
      - /home/emby/config:/config  # 挂载配置文件路径
      - /home/onedrive:/movies  # 挂载下载路径
    environment:
      - TZ=Asia/Shanghai  # 时区设置
      - UID=0  # 用户 ID
      - GID=0  # 组 ID
      - GIDLIST=0  # GID 列表
    ports:
      - 8096:8096
```

## 3.配置302重定向

emby2alist github项目地址：[https://github.com/chen3861229/embyExternalUrl/tree/main/emby2Alist](https://www.nodeseek.com/jump?to=https%3A%2F%2Fgithub.com%2Fchen3861229%2FembyExternalUrl%2Ftree%2Fmain%2Femby2Alist)

### 3.1下载项目

目录结构：

```bash
/root/emby2Alist
├── docker
│   ├── docker-compose.yml
│   ├── nginx-emby.syno.json
│   └── nginx-jellyfin.syno.json
├── nginx
│   ├── conf.d
│   │   ├── api
│   │   │   ├── alist-api.js
│   │   │   └── emby-api.js
│   │   ├── common
│   │   │   ├── events.js
│   │   │   ├── live-util.js
│   │   │   ├── periodics.js
│   │   │   ├── url-util.js
│   │   │   └── util.js
│   │   ├── config
│   │   │   ├── constant-common.js
│   │   │   ├── constant-ext.js
│   │   │   ├── constant-mount.js
│   │   │   ├── constant-nginx.js
│   │   │   ├── constant-pro.js
│   │   │   ├── constant-strm.js
│   │   │   ├── constant-symlink.js
│   │   │   └── constant-transcode.js
│   │   ├── constant.js
│   │   ├── docs
│   │   │   ├── test-data.md
│   │   │   └── UA.txt
│   │   ├── emby.conf
│   │   ├── emby.js
│   │   ├── exampleConfig
│   │   │   ├── constant-all.js
│   │   │   └── constant-main.js
│   │   ├── includes
│   │   │   ├── http.conf
│   │   │   ├── https.conf
│   │   │   ├── proxy-header.conf
│   │   │   └── server-group.conf
│   │   └── modules
│   │       ├── emby-example.js
│   │       ├── emby-items.js
│   │       ├── emby-live.js
│   │       ├── emby-search.js
│   │       ├── emby-system.js
│   │       ├── emby-transcode.js
│   │       ├── emby-v-media.js
│   │       └── ngx-ext.js
│   └── nginx.conf
└── README.md

```

### 3.2修改相关配置

1. emby API密钥

   访问ip:8096 进入emby完成基础配置后 点击设置-(高级)API密钥-新API密钥 随意填写应用程序名称 从而生成一个API密钥，并记录密钥

   ![image-20250110092041011](https://cdn.jsdelivr.net/gh/linmo-33/imgs/img/202501100920135.png)

2. alist API密钥

   访问ip:5244登录，之后点击管理-设置-其他 复制令牌

   ![image-20250110093525105](https://cdn.jsdelivr.net/gh/linmo-33/imgs/img/202501100935211.png)

3. 修改nginx配置文件

   修改`/emby2Alist/nginx/conf.d/constant.js`

   ```bash
   // 这里默认 emby/jellyfin 的地址是宿主机,要注意 iptables 给容器放行端口
   const embyHost = "http://容器ip地址:8096";
   
   // emby/jellyfin api key, 在 emby/jellyfin 后台设置
   const embyApiKey = "上边记录的API密钥";
   
   // 挂载工具 rclone/CD2 多出来的挂载目录, 例如将 od,gd 挂载到 /mnt 目录下: /mnt/onedrive /mnt/gd ,那么这里就填写 /mnt
   const mediaMountPath = ["/movies"];
   ```

   **注意：**

   这一项修改原则为：

   挂载路径为/home/onedrive， alist路径为/ondrive

   emby中文件存放目录为/movies/xxx.mp4
   alist中文件为/xxx.mp4
   emby相比alist中多出的部分就是填入的部分

   

   修改`/emby2Alist/nginx/conf.d/config下的constant-mount.js`

   ```bash
   // rclone/CD2 挂载的 alist 文件配置,根据实际情况修改下面的设置
   // 访问宿主机上 5244 端口的 alist 地址, 要注意 iptables 给容器放行端口
   const alistAddr = "http://容器ip地址:5244";
   
   // alist token, 在 alist 后台查看
   const alistToken = "记录的alist密钥";
   
   // alist 公网地址,用于需要 alist server 代理流量的情况,按需填写
   const alistPublicAddr = "https://xxx.xxx";
   ```

### 3.3 测试

完成以上所有步骤后重启nginx-emby，播放前请将emby的播放中互联网质量调至最高、关闭转码
**注意：现在需要访问的是服务器ip:8091，而不再是8096**

当播放时服务器未出现大流量或查看日志`/emby2Alist/nginx/log/error.log`中出现网盘相关的域名即可确认成功走直链

