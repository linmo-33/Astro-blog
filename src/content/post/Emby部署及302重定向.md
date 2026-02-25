---
title: 使用 Rclone + Emby + OpenList 实现 OneDrive 302 直链播放
description: Emby 刮削正常，播放时通过 302 重定向直接从微软 CDN 拉流，VPS 几乎不消耗带宽。
publishDate: 2026-02-25
tags:
  - emby
ogImage: /social-card.avif
---
## 1. Rclone 挂载 OneDrive

### 1.1 安装 Rclone

```bash
sudo -v ; curl https://rclone.org/install.sh | sudo bash
```

### 1.2创建Rclone配置

运行`rclone config`命令

```bash
# 进入rclone设置
rclone config
```

按以下步骤操作：

* n → New remote
* name → 随便起，例如 onedrive
* Storage → 输入onedrive对应的标号
* 输入之前保存的应用程序 `Client ID` 和 `Client secret`，选择对应地区。
* advanced config 和 auto config 选no
* config token输入之前在windows rclone获取的access_token
* 确认对应配置

### 1.3 fuse安装

```bash
apt-get install fuse3
```

### 1.4 创建挂载点并启动服务

```bash
sudo mkdir -p /home/onedrive
sudo chown $USER:$USER /home/onedrive
```

创建 systemd 服务文件：

```bash
sudo nano /etc/systemd/system/rclone-onedrive.service
```

保存以下内容

```bash
[Unit]
Description=Rclone OneDrive Mount via OpenList
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
ExecStart=/usr/bin/rclone mount onedrive: /home/onedrive \
  --config /home/$USER/.config/rclone/rclone.conf \
  --allow-other \
  --vfs-cache-mode full \
  --vfs-cache-max-size 50G \
  --vfs-cache-max-age 720h \
  --vfs-read-chunk-size 128M \
  --vfs-read-chunk-size-limit 1G \
  --buffer-size 128M \
  --dir-cache-time 168h \
  --poll-interval 1m \
  --onedrive-no-versions \
  --transfers 8 \
  --checkers 16 \
  --low-level-retries 10 \
  --retries 5 \
  --log-level INFO \
  --log-file /var/log/rclone-onedrive.log

ExecStop=/bin/fusermount -u -z /home/onedrive
Restart=on-failure
RestartSec=10
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

应用并启动：

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now rclone-onedrive.service
sudo systemctl status rclone-onedrive.service
```

检查挂载：

```bash
df -h /home/onedrive
ls /home/onedrive/emby/动漫   # 应该能看到文件
```

### 1.5 常见坑点

* 卡在 “vfs cache miss”：说明缓存没生效 → 确认 --vfs-cache-mode full 和缓存目录有写权限
* 报错 “transport endpoint is not connected”：重启服务或 fusermount -u /home/onedrive

## 2.Emby 部署（Docker）

### 2.1创建 Emby 数据目录

```bash
mkdir /home/emby/config
```

### 2.2新建docker-compose文件

```bash
version: "3.8"
services:
  emby:
    image: emby/embyserver:latest           # 官方最新版
    container_name: emby
    restart: unless-stopped
    network_mode: host                      # 简化端口映射
    volumes:
      - /home/emby/config:/config
      - /home/onedrive:/mnt/share:ro        # 只读挂载，防止 Emby 误写
    environment:
      - TZ=Asia/Shanghai
      - UID=0
      - GID=0
    # ports:                                # network_mode: host 后可省略
    #   - 8096:8096
    #   - 8920:8920
```

启动：

```bash
docker compose up -d
```

### 2.3 Emby 媒体库设置

* 不要添加根目录 /mnt/share
* 按分类添加子目录：

  * 动漫：/mnt/share/emby/动漫
  * 电影：/mnt/share/emby/电影
* 关闭“实时监控文件夹变化”
* 关闭“下载图像/字幕到媒体文件夹”
* 手动扫描库

这样 Emby 看到的 Path 就是 /mnt/share/emby/动漫/xxx.mkv

## 3.配置302重定向

### 3.1 使用 MediaLinker

```bash
services:
  medialinker:
    image: thsrite/medialinker:latest
    container_name: medialinker
    restart: unless-stopped
    volumes:
      - ./data:/opt/
    environment:
      - AUTO_UPDATE=true
      - SERVER=emby
      - NGINX_PORT=8091
    network_mode: host
```

启动后访问 http://你的IP:8091（不再用 8096）

### 3.2修改 MediaLinker 配置

1. emby API密钥

   访问ip:8096 进入emby完成基础配置后 点击设置-(高级)API密钥-新API密钥 随意填写应用程序名称 从而生成一个API密钥，并记录密钥
2. alist API密钥

   访问ip:5244登录，之后点击管理-设置-其他 复制令牌
3. 修改nginx配置文件

    编辑 `./data/config/constant.js`

   ```apex
   // Emby 看到的路径前缀（根据上面库设置）
   const mediaMountPath = [
      "/mnt/share/emby/",
      "/mnt/share/emby"
   ];
   //其余配置依次填入
   ```

保存后重载容器

### 3.3 测试

1. 用 Yamby / 浏览器访问 http://你的IP:8091
2. 播放一部片子
3. 查看 MediaLinker 日志

    `docker logs -f medialinker | grep -i "redirect\|alist\|path\|original"`

    看到类似：

    `redirect to: https://pan.zerovv.top/onedrive/emby/动漫/..`
   且没有 use original link → 成功！
4. 确认流量：播放时 VPS 上传带宽接近 0（iftop / vnstat）

## 4 qBittorrent 自动上传到 OneDrive

### 4.1 qBittorrent部署
```bash
version: "3.8"
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Asia/Shanghai
      - WEBUI_PORT=8080
      - TORRENTING_PORT=45481
    volumes:
      - /home/qbittorrent:/config
      - /home/downloads:/downloads
      - /usr/bin/rclone:/usr/bin/rclone:ro
      - /home/qbittorrent/rclone_config:/config/.config/rclone

    ports:
      - 8080:8080
      - 45481:45481
      - 45481:45481/udp
    restart: unless-stopped
```
启动：

`docker compose up -d`

### 4.2 上传脚本

创建脚本：

```bash
sudo mkdir -p /home/scripts
sudo nano /home/scripts/qb-finish.sh
```

```bash
#!/bin/bash

# --- 1. 参数配置 ---
torrent_name=$1
content_dir=$2
file_hash=$7
torrent_category=$8  

qb_username="admin"               # qbit用户名
qb_password="123"          # qbit密码
qb_web_url="http://localhost:8080"  # qbit webui地址 (docker映射后用localhost)
log_dir="/config/log"        # **修改:** 统一的日志目录

rclone_dest="onedrive"           # **修改:** 你的 rclone 远程名称
rclone_path="qbit"             # **修改:** 你想上传到的网盘目录
rclone_parallel="8"               # **修改:** 32太高了，8是更合理的值

auto_del_flag="rclone"
qb_version="5.1.2" 

# --- 2. 你的选择 (做种 还是 自动清理?) ---
# true  = 自动清理 (不能做种)
# false = 可以做种 (不能自动清理，未来需手动删除)
leeching_mode="true"
# 是否做种智能判断逻辑：
# 如果分类名称包含 "keep" 或 "seed" (不区分大小写)，则切换为做种模式
# ${torrent_category,,} 的意思是把变量转为全小写，方便匹配
if [[ "${torrent_category,,}" == *"keep"* ]] || [[ "${torrent_category,,}" == *"seed"* ]]; then
    leeching_mode="false"
    echo "[$(date)] MODE: Category matches 'keep/seed'. Switched to SEEDING mode." >> ${log_dir}/qb_upload.log
else
    echo "[$(date)] MODE: Standard mode. Will upload and DELETE." >> ${log_dir}/qb_upload.log
fi

# --- 3. 脚本正文  ---

if [ ! -d ${log_dir} ]
then
        mkdir -p ${log_dir}
fi

# 统一的日志文件
LOG_FILE="${log_dir}/qb_upload.log"

# 检查 qb 版本 
version=$(echo $qb_version | grep -P -o "([0-9]\.){2}[0-9]" | sed s/\\.//g)
if [ ${version} -gt 404 ]; then qb_v="1"; else qb_v="2"; fi


# 函数：登录 
function qb_login(){
    echo "[$(date)] SCRIPT: Logging in..." >> ${LOG_FILE}
    if [ ${qb_v} == "1" ]; then
        cookie=$(curl -i --silent --header "Referer: ${qb_web_url}" --data "username=${qb_username}&password=${qb_password}" "${qb_web_url}/api/v2/auth/login" | grep -P -o 'SID=\S{32}')
    else
        cookie=$(curl -i --silent --header "Referer: ${qb_web_url}" --data "username=${qb_username}&password=${qb_password}" "${qb_web_url}/login" | grep -P -o 'SID=\S{32}')
    fi

    if [ -n "${cookie}" ]; then
        echo "[$(date)] SCRIPT: Login SUCCESS." >> ${LOG_FILE}
    else
        echo "[$(date)] SCRIPT: Login FAILED." >> ${LOG_FILE}
    fi
}

# 函数：API操作 
function qb_api_actions(){
    # 1. 添加标签
    echo "[$(date)] SCRIPT: Adding tag '${auto_del_flag}'" >> ${LOG_FILE}
    if [ ${qb_v} == "1" ]; then
        curl --silent -X POST -d "hashes=${file_hash}&tags=${auto_del_flag}" "${qb_web_url}/api/v2/torrents/addTags" --cookie "${cookie}"
    else
        curl --silent -X POST -d "hashes=${file_hash}&category=${auto_del_flag}" "${qb_web_url}/command/setCategory" --cookie ${cookie}
    fi

    # 2. 根据 leeching_mode 决定是否删除
    if [ ${leeching_mode} == "true" ]; then
        echo "[$(date)] SCRIPT: leeching_mode=true. Deleting local files & torrent..." >> ${LOG_FILE}
        curl --silent -X POST -d "hashes=${file_hash}&deleteFiles=true" "${qb_web_url}/api/v2/torrents/delete" --cookie ${cookie}
    else
        echo "[$(date)] SCRIPT: leeching_mode=false. Keeping local files for seeding." >> ${LOG_FILE}
    fi
}

# 函数：Rclone Copy 
function rclone_copy(){
    echo "[$(date)] RCLONE: Starting 'rclone copy' for ${type}: ${content_dir}" >> ${LOG_FILE}
    if [ ${type} == "file" ]; then
        # 运行 rclone，函数将返回 rclone 的退出代码
        /usr/bin/rclone copy --transfers ${rclone_parallel} --log-file ${LOG_FILE} "${content_dir}" ${rclone_dest}:/${rclone_path}/
    elif [ ${type} == "dir" ]; then
        # 运行 rclone，函数将返回 rclone 的退出代码
        /usr/bin/rclone copy --transfers ${rclone_parallel} --log-file ${LOG_FILE} "${content_dir}"/ ${rclone_dest}:/${rclone_path}/"${torrent_name}"/
    fi
}

# --- 4. 主逻辑  ---

echo "-----------------------------------" >> ${LOG_FILE}
echo "[$(date)] SCRIPT: Triggered for ${torrent_name} (Hash: ${file_hash})" >> ${LOG_FILE}

# 判断类型
if [ -f "${content_dir}" ]; then
   type="file"
elif [ -d "${content_dir}" ]; then 
   type="dir"
else
   echo "[$(date)] SCRIPT: ERROR! Path not found: ${content_dir}" >> ${LOG_FILE}
   exit 1
fi

# 步骤 1: 尝试 Rclone Copy
rclone_copy

# 步骤 2: 检查 Rclone 的退出代码 ($?)
if [ $? -eq 0 ]; then
    # Rclone 成功 (退出代码 0)
    echo "[$(date)] RCLONE: SUCCESS. Proceeding to QB API actions." >> ${LOG_FILE}

    # 步骤 3: 只有在 Rclone 成功后，才执行登录和 API 操作
    qb_login
    if [ -n "${cookie}" ]; then
        qb_api_actions
    fi
else
    # Rclone 失败 (退出代码非 0)
    echo "[$(date)] RCLONE: FAILED! (Exit code: $?). Aborting API actions." >> ${LOG_FILE}
    echo "[$(date)] SCRIPT: Local files are KEPT. Rclone will retry on next run (if configured)." >> ${LOG_FILE}
    # 脚本退出，不执行任何 qB API 操作，保留本地文件
fi

echo "[$(date)] SCRIPT: Finished." >> ${LOG_FILE}
echo "-----------------------------------" >> ${LOG_FILE}

```

赋予权限：

`chmod +x /home/scripts/qb-finish.sh`

### 4.3 QB 中设置

QB → 工具 → 选项 → 下载 → “Torrent 完成时运行外部程序”：

`/bin/bash /home/scripts/qb-finish.sh "%N" "%F" "%R" "%D" "%C" "%Z" "%I" "%L"`

至此搭建完成，现在拥有了自己的emby影院！
