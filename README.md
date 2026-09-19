# CTJSIPTV

## 1. 功能

- 江苏电信 IPTV 动态认证：`auth → uploadAuthInfo → getServiceList → UserGroupNMB → loadbalanced → Portal`
- 动态获取 Portal，不依赖固定 Portal IP、固定 33200 端口或 DNS fallback
- 直播、回看、EPG/XMLTV
- 电影、剧集、短剧、动漫、少儿、综艺、电竞等 VOD
- 内置单文件 Web UI，无需额外部署前端
- Xtream Codes、TVBox/MacCMS
- rtp2httpd 转发
- 直播源启停、排序、自定义源与本地频道 Logo
- VOD 标题清理

## 2. 下载与启动

正式版本统一从 `CTJSIPTV-Release` 的 **Latest Release** 下载：

- [macOS Apple Silicon — ctjsiptv-macos-arm64](https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64)
- [Linux arm64 — ctjsiptv-linux-arm64](https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-linux-arm64)
- [Linux x86_64 / amd64 — ctjsiptv-linux-amd64](https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-linux-amd64)

命令行可直接下载最新版本。以 macOS Apple Silicon 为例：

```sh
curl -fL https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64 -o ctjsiptv
chmod +x ctjsiptv
xattr -d com.apple.quarantine ./ctjsiptv 2>/dev/null || true
```

Linux arm64：

```sh
curl -fL https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-linux-arm64 -o ctjsiptv
chmod +x ctjsiptv
```

Linux x86_64 / amd64：

```sh
curl -fL https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-linux-amd64 -o ctjsiptv
chmod +x ctjsiptv
```

复制 `ctjsiptv.conf.example` 为 `ctjsiptv.conf`，至少填写：

```ini
IPTV_USER_ID=
IPTV_PASSWORD=
IPTV_STB_ID=
```

多网卡、VPN 或 Surge 环境建议显式指定 IPTV 网卡：

```ini
NIC=en0
IPTV_MAC=
IPTV_IP=
```

未填写 MAC/IP 时程序会从选定网卡读取。

启动：

```sh
./ctjsiptv -c ./ctjsiptv.conf
```

默认监听 `[::]:8765`。检查：

```sh
curl http://127.0.0.1:8765/api/health
curl http://127.0.0.1:8765/api/status
```

浏览器直接访问：

```text
http://设备IP:8765/
```

内置 Web UI 已由 Release 构建过程嵌入二进制，因此普通用户**不需要 Apache/Nginx，也不需要单独部署 index.html**。

## 3. 主要配置

完整配置和注释见 `ctjsiptv.conf.example`。

| 配置项 | 必填 | 说明 |
|---|---:|---|
| `IPTV_USER_ID` | 是 | IPTV 用户 ID |
| `IPTV_PASSWORD` | 是 | IPTV 密码 |
| `IPTV_STB_ID` | 是 | STB ID |
| `NIC` | 否 | IPTV 网络接口；未配置时自动检测 |
| `IPTV_MAC` | 否 | 未配置时从 NIC 读取 |
| `IPTV_IP` | 否 | 未配置时从 NIC 读取 IPv4 |
| `LISTEN` | 否 | 默认 `[::]:8765` |
| `PUBLIC_URL` | 否 | 统一公开根地址 |
| `RTP2HTTPD` | 否 | rtp2httpd HTTP/HTTPS 根地址 |
| `LIVE_LOGO_DIR` | 否 | 本地频道 PNG Logo 目录 |
| `LIVE_OVERRIDES` | 否 | 直播源管理文件 |
| `VOD_TITLE_CLEAN_RULES` | 否 | VOD 标题清理规则文件 |
| `XTREAM_ENABLED` | 否 | 启用 Xtream |
| `XTREAM_USERNAME` / `XTREAM_PASSWORD` | 否 | Xtream 独立凭据 |
| `XTREAM_PUBLIC_URL` | 否 | 单独覆盖 Xtream 公开地址 |
| `TVBOX_PUBLIC_URL` | 否 | 单独覆盖 TVBox 公开地址 |

推荐反向代理/公网场景统一配置：

```ini
PUBLIC_URL=https://iptv.example.com
```

## 4. rtp2httpd

客户端不能直接访问 IPTV 业务网，或需要将直播/回看统一经外部入口输出时：

```ini
RTP2HTTPD=https://rtp2httpd.example.com:444
```

rtp2httpd 只负责播放输出转发，不参与 IPTV 认证和 Portal 发现。未配置时保留上游原始播放地址。

## 5. 内置 Web UI

访问 `http://设备IP:8765/` 即可使用。网页提供影视分类、搜索、详情、分集、播放、直播、EPG、运行状态、直播源管理及 VOD 标题规则管理。

`live-overrides.json` 与 `vod-title-clean.json` 默认自动定位到 `ctjsiptv.conf` 所在目录，也可以通过配置项指定路径。

直播源结构：

```json
{
  "disabled": [],
  "order": [],
  "custom": []
}
```

VOD 标题规则结构：

```json
{
  "prefix": [],
  "suffix": [],
  "regex": []
}
```

## 6. Xtream Codes

```ini
XTREAM_ENABLED=true
XTREAM_USERNAME=iptv
XTREAM_PASSWORD=change-me
PUBLIC_URL=https://iptv.example.com
```

客户端填写：

```text
Server:   http://设备IP:8765
Username: iptv
Password: change-me
```

主要兼容 `/player_api.php`、`/xmltv.php`、直播/VOD/Series、EPG 与 timeshift。Xtream 凭据仅用于 CTJSIPTV 客户端鉴权，不要复用运营商 IPTV 密码。

## 7. TVBox / MacCMS

TVBox 推荐直接使用：

```text
http://设备IP:8765/tvbox.json
```

MacCMS type=1 API：

```text
/api.php/provide/vod/
```

使用域名或反向代理时优先配置 `PUBLIC_URL`；如需单独覆盖 TVBox 地址再使用 `TVBOX_PUBLIC_URL`。

## 8. 原生 API

主要入口：

```text
GET /api/health
GET /api/status
GET /api/categories
GET /api/categories/{type}
GET /api/filters/{type}
GET /api/vod?type=movie&page=1
GET /api/vod/{id}?type=drama
GET /api/play/{id}?type=movie
GET /api/search?q=关键词
GET /api/live/rtp
GET /api/live/http
GET /api/live/rtp/rtp2httpd
GET /api/live/http/rtp2httpd
GET /api/epg?days=7
```

EPG 输出 XMLTV，直播列表输出扩展 M3U。

## 9. 内网与公网访问

CTJSIPTV 自带 HTTP 服务和 Web UI，因此内网无需额外 Web Server：

```text
客户端 → http://CTJSIPTV主机:8765
```

需要域名、HTTPS 或公网访问时，可在 CTJSIPTV 前增加 Apache/Nginx/Caddy 等反向代理，将请求原样转发到 `127.0.0.1:8765`，并设置：

```ini
PUBLIC_URL=https://iptv.example.com
```

反向代理应保留原始 URI；部分 VOD ID 可能包含 `%2F` 等编码字符。公网入口应自行增加身份验证、VPN、IP 白名单或可信网关。不要直接裸露 IPTV 凭据或临时播放鉴权信息。

## 10. 更新

停止旧进程后替换 `ctjsiptv` 二进制并重新启动即可。保留：

```text
ctjsiptv.conf
live-overrides.json
vod-title-clean.json
频道 Logo 目录
```

升级后通过 `/api/status` 检查版本和 IPTV 会话状态。


## 项目说明

这是 CTJSIPTV 的公开发行与使用说明仓库。正式二进制由本仓库 Releases 提供；源码与开发实现维护在开发仓库。

## License

仅用于个人学习、协议研究和合法的自有 IPTV 服务接入。使用者应自行确保符合当地法律、运营商服务协议及内容授权要求。
