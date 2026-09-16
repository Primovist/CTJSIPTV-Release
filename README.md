# CTJSIPTV Release

江苏电信 IPTV 服务端公开发行仓库。这里仅提供编译后的发行版、部署方法和公开 API 说明；服务端实现细节与源码不在本仓库公开。

> 使用前提：运行 CTJSIPTV 的设备必须能够访问已经融合到本地网络的江苏电信 IPTV 专网。本项目不负责光猫、路由器或运营商侧的 IPTV 网络接入与业务订购。

## 下载

请从 [Releases](https://github.com/Primovist/CTJSIPTV-Release/releases/latest) 下载最新版本。

当前提供：

| 文件 | 平台 |
|---|---|
| `ctjsiptv-macos-arm64` | macOS Apple Silicon |
| `ctjsiptv-linux-arm64` | Linux arm64 |
| `ctjsiptv-linux-amd64` | Linux x86_64 / amd64 |
| `index.html` | 独立网页前端备份 |

macOS Apple Silicon 已在实际 IPTV 网络中验证。Linux arm64 与 amd64 版本由 GitHub Actions 构建并进行离线测试，实际可用性仍取决于系统运行环境、网卡与 IPTV 路由配置。

## 快速部署

### 1. 下载并安装二进制

以下以 macOS arm64 为例：

```sh
mkdir -p ~/CTJSIPTV
cd ~/CTJSIPTV

curl -L -o ctjsiptv \
  https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64
chmod +x ctjsiptv
```

Linux 根据架构将文件名替换为 `ctjsiptv-linux-arm64` 或 `ctjsiptv-linux-amd64` 即可。

macOS 如果因为下载文件的隔离属性阻止运行，可检查并移除该文件的 quarantine 属性：

```sh
xattr -d com.apple.quarantine ./ctjsiptv 2>/dev/null || true
```

### 2. 创建配置文件

在二进制同目录创建 `ctjsiptv.conf`：

```ini
IPTV_USER_ID=你的IPTV用户ID
IPTV_PASSWORD=你的IPTV密码
IPTV_STB_ID=你的STB_ID

# 以下均可选
# IPTV_MAC=AA:BB:CC:DD:EE:FF
# IPTV_IP=192.168.1.100
# NIC=en0
# LISTEN=[::]:8765
# RTP2HTTPD=https://iptv-proxy.example.com
```

配置项：

| 参数 | 必填 | 说明 |
|---|---:|---|
| `IPTV_USER_ID` | 是 | IPTV 用户 ID |
| `IPTV_PASSWORD` | 是 | IPTV 密码 |
| `IPTV_STB_ID` | 是 | STB ID |
| `IPTV_MAC` | 否 | 未指定时从选定网卡读取 |
| `IPTV_IP` | 否 | 未指定时从选定网卡读取 IPv4 |
| `NIC` | 否 | 未指定时自动选择 IPTV 路由对应接口 |
| `LISTEN` | 否 | 默认 `[::]:8765`，IPv4/IPv6 双栈监听 |
| `RTP2HTTPD` | 否 | RTP/UDP 转 HTTP 服务基址，例如 `http://192.168.1.2:5140` |

环境变量可以覆盖配置文件中的同名配置。不要把包含真实账号、密码、Cookie、Token 或临时鉴权信息的配置文件提交到公开仓库。

### 3. 启动

```sh
cd ~/CTJSIPTV
./ctjsiptv -c ./ctjsiptv.conf
```

默认监听：

```text
http://127.0.0.1:8765/
```

局域网其他设备可使用运行 CTJSIPTV 主机的实际 IP，例如：

```text
http://192.168.1.100:8765/
```

服务端已经内置网页前端，不需要单独部署 `index.html`。访问根路径 `/` 即可打开前端；`index.html` Release Asset 仅用于需要单独取得前端文件的场景。

### 4. 验证

```sh
curl http://127.0.0.1:8765/api/health
curl http://127.0.0.1:8765/api/status
```

随后浏览器访问：

```text
http://127.0.0.1:8765/
```

如果从其他设备访问，请将 `127.0.0.1` 换成服务器局域网 IP。

## 后台运行

### macOS LaunchDaemon

如果需要开机自动运行，可创建 `/Library/LaunchDaemons/com.ctjsiptv.server.plist`。以下假设程序安装在 `/Users/USERNAME/CTJSIPTV`，请先替换实际用户名和路径：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.ctjsiptv.server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/Users/USERNAME/CTJSIPTV/ctjsiptv</string>
        <string>-c</string>
        <string>/Users/USERNAME/CTJSIPTV/ctjsiptv.conf</string>
    </array>

    <key>WorkingDirectory</key>
    <string>/Users/USERNAME/CTJSIPTV</string>

    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/Users/USERNAME/CTJSIPTV/ctjsiptv.log</string>
    <key>StandardErrorPath</key>
    <string>/Users/USERNAME/CTJSIPTV/ctjsiptv-error.log</string>
</dict>
</plist>
```

加载：

```sh
sudo chown root:wheel /Library/LaunchDaemons/com.ctjsiptv.server.plist
sudo chmod 644 /Library/LaunchDaemons/com.ctjsiptv.server.plist
sudo launchctl bootstrap system /Library/LaunchDaemons/com.ctjsiptv.server.plist
sudo launchctl enable system/com.ctjsiptv.server
```

重启服务：

```sh
sudo launchctl kickstart -k system/com.ctjsiptv.server
```

### Linux systemd

建议将程序放到 `/opt/ctjsiptv/`：

```sh
sudo mkdir -p /opt/ctjsiptv
sudo cp ctjsiptv /opt/ctjsiptv/ctjsiptv
sudo cp ctjsiptv.conf /opt/ctjsiptv/ctjsiptv.conf
sudo chmod +x /opt/ctjsiptv/ctjsiptv
```

创建 `/etc/systemd/system/ctjsiptv.service`：

```ini
[Unit]
Description=CTJSIPTV Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/opt/ctjsiptv
ExecStart=/opt/ctjsiptv/ctjsiptv -c /opt/ctjsiptv/ctjsiptv.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用：

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now ctjsiptv
sudo systemctl status ctjsiptv
```

## HTTPS / 反向代理

CTJSIPTV 已将网页前端、API 和图片代理全部放在同一个服务中，因此反向代理时只需要代理整个根路径 `/` 到 `8765`，不再需要单独部署 Web 根目录，也不需要额外配置 `/iptv-image/`。

### Apache

```apache
<VirtualHost *:443>
    ServerName iptv.example.com

    SSLEngine on
    SSLCertificateFile "/path/to/fullchain.pem"
    SSLCertificateKeyFile "/path/to/privkey.pem"

    ProxyRequests Off
    ProxyPreserveHost On
    AllowEncodedSlashes NoDecode

    ProxyPass        / http://127.0.0.1:8765/ nocanon
    ProxyPassReverse / http://127.0.0.1:8765/

    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>
```

启用 `ssl`、`proxy`、`proxy_http`、`headers` 等必要模块后检查并重载：

```sh
sudo apachectl -t
sudo apachectl restart
```

### Nginx

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name iptv.example.com;

    ssl_certificate     /path/to/fullchain.pem;
    ssl_certificate_key /path/to/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 120s;
    }
}
```

检查并重载：

```sh
sudo nginx -t
sudo systemctl reload nginx
```

公网暴露时建议在反向代理、VPN 或其他可信网关层增加访问控制。CTJSIPTV 不应被视为面向公网的多用户认证网关。

## 更新

停止服务后，用最新 Release 中对应平台的二进制替换旧文件即可。`ctjsiptv.conf` 独立保存，无需随程序覆盖。

macOS 示例：

```sh
cd ~/CTJSIPTV
curl -L -o ctjsiptv.new \
  https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64
chmod +x ctjsiptv.new
xattr -d com.apple.quarantine ctjsiptv.new 2>/dev/null || true
mv ctjsiptv.new ctjsiptv
sudo launchctl kickstart -k system/com.ctjsiptv.server
```

更新后检查：

```sh
curl http://127.0.0.1:8765/api/status
```

## API 约定

所有公开接口统一使用 `/api` 前缀。

JSON 接口成功时：

```json
{
  "code": 0,
  "data": {}
}
```

失败时：

```json
{
  "code": 1001,
  "message": "error message"
}
```

`/api/epg` 成功时直接返回 XMLTV；直播列表接口返回 UTF-8 Extended M3U。

### 接口总览

| Method | Path | 说明 |
|---|---|---|
| GET | `/api/health` | 服务健康状态 |
| GET | `/api/status` | 版本、IPTV 会话、网卡、IPTV IP、RTP2HTTPD 状态 |
| GET | `/api/image?url=...` | 图片代理 |
| GET | `/api/live/rtp` | RTP/UDP 直播列表及可用回看信息 |
| GET | `/api/live/http` | HTTP HLS 直播列表及可用回看信息 |
| GET | `/api/live/rtp/rtp2httpd` | 经 RTP2HTTPD 改写的 RTP/UDP 直播列表 |
| GET | `/api/live/http/rtp2httpd` | 经 RTP2HTTPD 改写的 HTTP/回看列表 |
| GET | `/api/categories` | 支持的业务类型 |
| GET | `/api/categories/{type}` | 指定业务类型的分类 |
| GET | `/api/filters/{type}` | 指定业务类型的筛选项 |
| GET | `/api/search?q=...&type=all&page=1&page_size=20&mode=auto` | 搜索 |
| GET | `/api/search/suggest?q=...&page=1&page_size=20` | 拼音首字母联想 |
| GET | `/api/vod?type=...&page=1&tags=...` | VOD 列表 |
| GET | `/api/vod/{id}?type=...` | VOD 详情与分集 |
| GET | `/api/play/{id}?type=...&episode_id=...` | 获取播放地址 |
| GET | `/api/epg` | 完整 XMLTV EPG |
| GET | `/api/epg/{channel}` | 单频道 XMLTV EPG |

### `/api/status`

用于检查程序版本、网络参数以及 IPTV 会话状态。

示例：

```json
{
  "code": 0,
  "data": {
    "version": "0.5.3",
    "iptv_authenticated": true,
    "session_active": true,
    "nic": "en0",
    "iptv_ip": "192.168.1.100",
    "rtp2httpd_enabled": true,
    "rtp2httpd": "https://proxy.example.com"
  }
}
```

### 业务类型

可用 `type`：

| `type` | 名称 |
|---|---|
| `movie` | 电影 |
| `drama` | 剧集 |
| `short_drama` | 短剧 |
| `anime` | 动漫 |
| `kids` | 少儿 |
| `variety` | 综艺 |
| `esports` | 电竞 |

### 分类与筛选

```text
GET /api/categories
GET /api/categories/movie
GET /api/filters/movie
```

`/api/categories` 返回客户端可展示的业务类型；`/api/categories/{type}` 返回对应类型的动态分类；`/api/filters/{type}` 返回可用于列表筛选的标签。

### 搜索

```text
GET /api/search?q=<搜索内容>
GET /api/search?q=<搜索内容>&type=drama&page=1&page_size=20
GET /api/search?q=<拼音首字母>&mode=pinyin
GET /api/search/suggest?q=<拼音首字母>
```

支持：

```text
type=all|movie|drama|short_drama|anime|kids
mode=auto|keyword|pinyin
```

`page_size` 范围为 `1-50`。客户端应直接使用搜索结果返回的 `id` 请求详情，不应自行解析或重新构造 ID。

### VOD 列表

```text
GET /api/vod?type=movie&page=1
GET /api/vod?type=drama&page=1
GET /api/vod?type=short_drama&page=1&tags=duanju_dsj
GET /api/vod?type=variety&category=<category>
GET /api/vod?type=esports&category=<category>&page=1&page_size=12
```

列表统一通过 `data.items[]` 返回。客户端应把服务端返回的 `id` 视为不透明标识符并原样传递。

### 详情与分集

```text
GET /api/vod/{id}?type=movie
GET /api/vod/{id}?type=drama
GET /api/vod/{id}?type=variety
```

如果详情包含 `episodes`，播放某一集时使用详情返回的 `episode_id`。

### 播放地址

```text
GET /api/play/{id}?type=movie
GET /api/play/{id}?type=drama&episode_id=<episode_id>
```

播放 URL 可能包含时效性参数，应按需实时获取，不建议长期缓存或公开分享。

### 直播列表

```text
GET /api/live/http
GET /api/live/rtp
GET /api/live/http/rtp2httpd
GET /api/live/rtp/rtp2httpd
```

返回类型：

```text
application/vnd.apple.mpegurl; charset=utf-8
```

列表使用 Extended M3U，可包含 `tvg-id`、`tvg-name`、`group-title`、`catchup`、`catchup-source` 等字段。客户端应按标准 M3U 属性消费这些字段，不依赖服务端内部生成方式。

只有配置了 `RTP2HTTPD` 时，两个 `/rtp2httpd` 接口才可使用。

### EPG

完整 EPG：

```text
GET /api/epg
```

单频道：

```text
GET /api/epg/CCTV-1
```

成功时返回 XMLTV。频道名包含特殊字符时应进行 URL 编码。

### 图片代理

```text
GET /api/image?url=<URL编码后的图片地址>
```

网页前端通过该接口获取图片，因此反向代理部署时无需再单独代理 IPTV 图片服务器。

## 客户端接入建议

客户端只应依赖本 README 中公开的 `/api` 契约，不应依赖服务端内部页面、运营商上游 URL、Cookie、签名方式、临时 Token、内部内容编号转换规则或其他实现细节。

推荐流程：

1. 启动时请求 `/api/status` 检查服务状态。
2. 使用 `/api/categories` 获取业务入口。
3. 使用 `/api/categories/{type}`、`/api/filters/{type}` 和 `/api/vod` 构建列表。
4. 搜索使用 `/api/search`；将返回的 `id` 原样传递给 `/api/vod/{id}`。
5. 详情页使用 `episodes` 中的 `episode_id` 请求 `/api/play/{id}`。
6. 直播客户端直接订阅对应 `/api/live/...` 地址。
7. EPG 客户端订阅 `/api/epg`。

## 常见问题

### 网页能打开，但 IPTV 状态失败

先检查：

```sh
curl http://127.0.0.1:8765/api/status
```

如果 HTTP 服务正常但 `iptv_authenticated=false`，重点检查 IPTV 专网路由、配置文件中的账号参数，以及自动选择的 `nic` / `iptv_ip` 是否符合实际网络。

### 本机能访问，局域网设备不能访问

检查 `LISTEN`。默认 `[::]:8765` 应提供双栈监听；如系统环境不支持预期的双栈行为，可显式设置：

```ini
LISTEN=0.0.0.0:8765
```

同时检查主机防火墙是否允许 TCP 8765。

### RTP/UDP 地址无法直接播放

RTP/UDP 组播要求客户端本身能够访问 IPTV 组播网络。无法直接访问时，可以配置兼容的 RTP2HTTPD 服务，并使用 `/api/live/rtp/rtp2httpd`。

### 反向代理后部分详情接口异常

内容 ID 可能包含 URL 编码字符。Apache 建议保留：

```apache
AllowEncodedSlashes NoDecode
ProxyPass / http://127.0.0.1:8765/ nocanon
```

代理层应尽量保持原始 URI，避免对路径进行额外解码或重写。

## 说明

本仓库只维护公开发行版、部署文档与公开 API 契约。README 不再引用私有源码仓库中的 Markdown 文档，也不公开上游认证、页面解析、签名、频道提取、内容编号转换等内部实现原理。
