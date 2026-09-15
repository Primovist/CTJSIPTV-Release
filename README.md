# CTJSIPTV

CTJSIPTV 是面向 **江苏电信 IPTV** 的家庭自托管服务，提供直播、回看、点播、搜索、节目单（EPG）以及可选的网页前端。

本仓库作为 **CTJSIPTV 发行仓库** 使用：后续构建的 macOS / Linux 可执行文件及版本 Release 均统一发布在这里。源码仓库保持私有，本仓库只保留发行版、使用说明和面向用户的部署文档。

> 本项目不负责光猫、路由器上的 IPTV 专网融合，也不提供 IPTV 业务订购。使用前必须已经具备可正常使用的江苏电信 IPTV 业务和网络环境。

## 功能

- 江苏电信 IPTV 直播频道列表
- HTTP HLS 与 RTP/UDP 直播源
- 支持可用频道的回看信息
- 电影、剧集、短剧、动漫、少儿、综艺、电竞
- 分类、筛选、关键词搜索与拼音首字母搜索
- 详情、分集与实时播放地址获取
- XMLTV EPG 节目单
- 可选 RTP2HTTPD 直播代理地址输出
- 单文件 Web 前端，无需 Node.js / npm
- macOS Apple Silicon、Linux arm64、Linux amd64 发行版

## 使用环境

服务端所在网络需要已经融合 IPTV 专网。

目前已实际验证：

- **macOS Apple Silicon（arm64）**：已在真实江苏电信 IPTV 网络中验证。

同时提供以下由 GitHub Actions 编译并执行离线自测的版本：

- **Linux arm64**
- **Linux amd64 / x86_64**

Linux 版本尚需根据实际设备、系统运行库和 IPTV 路由环境验证。Linux arm64 可能适用于部分 OpenWrt / ARM Linux 设备。

## 下载

请从本仓库的 **Releases** 页面下载对应平台的最新版本：

| 平台 | 文件 |
|---|---|
| macOS Apple Silicon | `ctjsiptv-macos-arm64` |
| Linux ARM64 | `ctjsiptv-linux-arm64` |
| Linux x86_64 / AMD64 | `ctjsiptv-linux-amd64` |

下载后赋予执行权限：

```bash
chmod +x ctjsiptv-*
```

## 配置

创建 `ctjsiptv.conf`：

```ini
IPTV_USER_ID=你的IPTV用户ID
IPTV_PASSWORD=你的IPTV密码
IPTV_STB_ID=你的机顶盒ID
```

以上三项为必填。

可选配置：

```ini
# IPTV_MAC=00:11:22:33:44:55
# IPTV_IP=192.168.1.100
# NIC=en0
# LISTEN=[::]:8765
# RTP2HTTPD=http://192.168.1.100:5140
```

| 参数 | 是否必填 | 说明 |
|---|---:|---|
| `IPTV_USER_ID` | 是 | IPTV 用户 ID |
| `IPTV_PASSWORD` | 是 | IPTV 密码 |
| `IPTV_STB_ID` | 是 | IPTV STB ID |
| `IPTV_MAC` | 否 | 不填写时自动读取所选网卡 MAC |
| `IPTV_IP` | 否 | 不填写时自动读取所选网卡 IPv4 |
| `NIC` | 否 | 不填写时自动选择 IPTV 所使用的网络接口 |
| `LISTEN` | 否 | 默认 `[::]:8765`，同时支持 IPv4 / IPv6 |
| `RTP2HTTPD` | 否 | RTP/UDP 转 HTTP 服务地址 |

环境变量可以覆盖配置文件中的同名项目。

> **不要**把真实 `ctjsiptv.conf`、IPTV 密码、Cookie、Token 或临时播放鉴权地址上传到 GitHub 或放进 Web 根目录。

## 启动

```bash
./ctjsiptv-macos-arm64 -c ./ctjsiptv.conf
```

Linux ARM64：

```bash
./ctjsiptv-linux-arm64 -c ./ctjsiptv.conf
```

Linux AMD64：

```bash
./ctjsiptv-linux-amd64 -c ./ctjsiptv.conf
```

默认服务地址：

```text
http://<服务器IP>:8765
```

先检查：

```bash
curl http://127.0.0.1:8765/api/health
curl http://127.0.0.1:8765/api/status
```

`/api/status` 正常时应返回 JSON；IPTV 鉴权成功时 `data.iptv_authenticated` 为 `true`。

## 常用地址

| 地址 | 用途 |
|---|---|
| `/api/status` | 服务版本、网络和 IPTV 鉴权状态 |
| `/api/live/http` | HTTP HLS 直播列表 |
| `/api/live/rtp` | RTP/UDP 直播列表 |
| `/api/live/http/rtp2httpd` | 经 RTP2HTTPD 处理的 HTTP 直播/回看列表 |
| `/api/live/rtp/rtp2httpd` | 经 RTP2HTTPD 处理的 RTP/UDP 直播/回看列表 |
| `/api/epg` | XMLTV EPG |
| `/api/search?q=关键词` | 搜索点播内容 |

直播列表为扩展 M3U8，可直接提供给兼容的 IPTV 播放器使用。

## 网页前端

CTJSIPTV 提供可选的单文件 `index.html` 前端，不需要 Node.js、npm 或单独构建。

前端可用于浏览影视内容、搜索、查看详情与分集、获取播放地址、查看 XMLTV EPG、直播列表以及后端状态。

前端依赖同源：

```text
/api/
/iptv-image/
```

因此不要直接使用 `file://` 双击打开 `index.html`，应通过 Apache、Nginx 等 Web 服务器提供。

## HTTPS 前端部署

推荐结构：

```text
浏览器
  ↓ HTTPS
Apache / Nginx
  ├── /                 → index.html
  ├── /api/             → CTJSIPTV:8765
  └── /iptv-image/      → IPTV 图片服务器
```

以下示例假设：

```text
域名：iptv.example.com
CTJSIPTV：127.0.0.1:8765
前端目录：/path/to/ctjsiptv-web
```

### Apache

```apache
<VirtualHost *:443>
    ServerName iptv.example.com

    SSLEngine on
    SSLCertificateFile "/path/to/cert/fullchain.pem"
    SSLCertificateKeyFile "/path/to/cert/privkey.pem"

    DocumentRoot "/path/to/ctjsiptv-web"

    <Directory "/path/to/ctjsiptv-web">
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.html
    </Directory>

    ProxyRequests Off
    ProxyPreserveHost Off
    AllowEncodedSlashes NoDecode

    ProxyPass        /api/ http://127.0.0.1:8765/api/ nocanon
    ProxyPassReverse /api/ http://127.0.0.1:8765/api/

    ProxyPass        /iptv-image/ http://imagecache.itv.jsinfo.net:8080/
    ProxyPassReverse /iptv-image/ http://imagecache.itv.jsinfo.net:8080/

    <Location "/iptv-image/">
        Header set Cache-Control "public, max-age=86400"
    </Location>

    RequestHeader set X-Forwarded-Proto "https"
</VirtualHost>
```

检查并重启：

```bash
sudo apachectl -t
sudo apachectl restart
```

### Nginx

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name iptv.example.com;

    ssl_certificate     /path/to/cert/fullchain.pem;
    ssl_certificate_key /path/to/cert/privkey.pem;

    root /path/to/ctjsiptv-web;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 10s;
        proxy_read_timeout 120s;
        proxy_send_timeout 120s;
    }

    location /iptv-image/ {
        proxy_pass http://imagecache.itv.jsinfo.net:8080/;
        proxy_http_version 1.1;
        proxy_set_header Host imagecache.itv.jsinfo.net:8080;
        proxy_connect_timeout 10s;
        proxy_read_timeout 30s;
        add_header Cache-Control "public, max-age=86400";
    }
}
```

检查并重载：

```bash
nginx -t
nginx -s reload
```

> `/api/` 代理应尽量保留原始 URI。部分影视内容 ID 可能包含 `%2F` 等编码字符，错误的 URI 重写可能导致详情或播放接口失败。

## 家庭局域网无证书部署

如果只在可信家庭局域网使用，不需要域名和 HTTPS，可以让 Apache / Nginx 监听 `8080`，通过：

```text
http://<服务器局域网IP>:8080/
```

访问。

### Apache 示例

```apache
Listen 8080

<VirtualHost *:8080>
    DocumentRoot "/path/to/ctjsiptv-web"

    <Directory "/path/to/ctjsiptv-web">
        Options FollowSymLinks
        AllowOverride None
        Require all granted
        DirectoryIndex index.html
    </Directory>

    ProxyRequests Off
    ProxyPreserveHost Off
    AllowEncodedSlashes NoDecode

    ProxyPass        /api/ http://127.0.0.1:8765/api/ nocanon
    ProxyPassReverse /api/ http://127.0.0.1:8765/api/

    ProxyPass        /iptv-image/ http://imagecache.itv.jsinfo.net:8080/
    ProxyPassReverse /iptv-image/ http://imagecache.itv.jsinfo.net:8080/
</VirtualHost>
```

### Nginx 示例

```nginx
server {
    listen 8080;
    listen [::]:8080;
    server_name _;

    root /path/to/ctjsiptv-web;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8765;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /iptv-image/ {
        proxy_pass http://imagecache.itv.jsinfo.net:8080/;
        proxy_set_header Host imagecache.itv.jsinfo.net:8080;
    }
}
```

局域网部署建议为服务器设置 DHCP 静态租约，避免 IP 改变。HTTP 无传输加密，只应在可信内网使用，并且不要把 `8080` 或 `8765` 直接映射到公网。

## 部署验证

建议依次测试：

```bash
curl http://127.0.0.1:8765/api/status
curl https://iptv.example.com/api/status
curl https://iptv.example.com/api/live/http
curl -o epg.xml https://iptv.example.com/api/epg
```

然后打开网页前端，检查影视列表、海报、搜索、详情、分集、EPG 和直播列表。

如果首页正常但 API 全部失败，优先检查 `/api/` 反向代理；如果影视内容正常但海报不显示，检查 `/iptv-image/`；如果详情或播放接口异常，检查代理是否错误处理了编码后的内容 ID。

## 播放说明

前端获取到的播放地址会交给浏览器或外部播放器处理，CTJSIPTV 不执行视频转码。

如果同一地址在 IINA / VLC 中能够播放，但 Safari 黑屏，通常属于流格式、音视频编码与浏览器播放器兼容性问题，并不代表 IPTV 鉴权失败。

播放地址可能包含临时鉴权信息，应实时获取，不要长期保存或公开分享。

## 安全建议

- 不公开 IPTV 用户 ID、密码、STB ID、Cookie、Token 或临时播放地址；
- 不把真实配置文件放入 Web 根目录或提交到 GitHub；
- 不建议直接将 CTJSIPTV 的 `8765` 端口暴露到公网；
- 公网 Web 部署应使用 HTTPS，并在 Web 服务器、VPN 或其他可信网关增加访问控制；
- 局域网 HTTP 部署只应运行于可信家庭网络。

## 更新

新版本统一在本仓库 **Releases** 发布。升级时下载对应平台的新二进制并替换旧文件即可，建议保留原有 `ctjsiptv.conf`。

如果同时更新网页前端，用新版 `index.html` 替换 Web 目录中的旧文件，并清理 Web/CDN 缓存或在浏览器中强制刷新。

## 说明

CTJSIPTV 仅用于用户自己的合法 IPTV 业务、家庭网络和学习研究场景。请遵守当地法律法规、电信运营商服务条款及内容版权要求。
