# CTJSIPTV

江苏电信 IPTV 服务端，使用 Swift 实现。CTJSIPTV 完成运营商 IPTV 认证、动态 Portal/Session、直播、回看、EPG、点播与搜索，并通过内置 Web UI、原生 API、Xtream Codes 和 TVBox/MacCMS 提供统一访问。

# 一、使用说明

## 1. 功能

- 江苏电信 IPTV 动态认证，动态获取 Portal
- 直播、回看、EPG/XMLTV
- 电影、剧集、短剧、动漫、少儿、综艺、电竞等 VOD
- 内置单文件 Web UI，无需额外部署前端
- Xtream Codes、TVBox/MacCMS
- rtp2httpd 转发
- 直播源启停、排序、自定义源与本地频道 Logo
- VOD 标题清理

## 2. 运行环境

运行 CTJSIPTV 的设备必须能够访问江苏电信 IPTV 专网资源。适用环境包括：

- 江苏电信官方软终端能够正常运行、可直接访问 IPTV 专网的网络环境；
- 由路由器模拟 IPTV 认证并获取 IPTV 专网 IP，再将该网络提供给 CTJSIPTV 主机的环境。

如果主机存在多个网卡、VPN、虚拟接口，或者 IPTV 专网由路由器通过指定接口提供，建议在配置中明确设置 `NIC`，例如：

```ini
NIC=en0
```

CTJSIPTV 不负责建立 IPTV 专网接入本身；启动前应先确保所指定网卡能够访问运营商 IPTV 专网资源。

## 3. 下载与启动

正式版本统一从 `CTJSIPTV-Release` 的 **Latest Release** 下载：

- [macOS Apple Silicon — ctjsiptv-macos-arm64](https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64)
- [macOS Intel — ctjsiptv-macos-x86_64](https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-x86_64)
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

正常的江苏电信 IPTV 网络环境可直接零配置启动：

```sh
./ctjsiptv
```

程序会生成并持久化稳定软终端设备身份，通过运营商 Zero Config 获取 IPTV Account、Password、STBID 等启动身份，然后继续现有动态 Portal/Auth。启动身份默认保存在配置文件同目录的 `bootstrap.json`（权限 `0600`），后续 Zero Config 暂时不可用时可回退使用。CTJSIPTV 不启动官方软终端的管理心跳，避免持续上报终端在线状态或接收远程退出、重启等控制指令。

成功认证后，直播频道快照默认保存在配置文件同目录的 `channel-cache.json`（权限 `0600`）。Portal 暂时无法刷新时，服务可继续使用最后一次成功快照；快照不保存 JSESSIONID 或 UserToken，但频道播放地址本身属于敏感数据，不应对其他用户开放该文件。

EPG 最近一次成功获取的今天及过去 6 天节目单默认原子保存到 `epg-cache.json`（权限 `0600`）。服务启动时读取该快照，随后补全一次，并在每天凌晨刷新；刷新失败保留旧快照。EPG 定时任务不再触发直播频道拉取，直播频道由认证流程和 `channel-cache.json` 独立管理。

已有账号配置仍可复制 `ctjsiptv.conf.example` 为 `ctjsiptv.conf` 并显式覆盖：

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

内置 Web UI 直接保存在 Swift 源码并编译进二进制，因此普通用户**不需要 Apache/Nginx，也不需要单独部署 index.html**。

## 4. 后台服务

### macOS：launchctl

以下示例将可执行文件安装到 `/usr/local/bin/ctjsiptv`，配置文件保存在 `/usr/local/etc/ctjsiptv/ctjsiptv.conf`：

```sh
sudo install -m 755 ctjsiptv /usr/local/bin/ctjsiptv
sudo mkdir -p /usr/local/etc/ctjsiptv
sudo chown "$(id -un):$(id -gn)" /usr/local/etc/ctjsiptv
install -m 600 ctjsiptv.conf /usr/local/etc/ctjsiptv/ctjsiptv.conf
```

创建 `~/Library/LaunchAgents/com.ctjsiptv.server.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.ctjsiptv.server</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/ctjsiptv</string>
        <string>-c</string>
        <string>/usr/local/etc/ctjsiptv/ctjsiptv.conf</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/tmp/ctjsiptv.log</string>

    <key>StandardErrorPath</key>
    <string>/tmp/ctjsiptv-error.log</string>
</dict>
</plist>
```

加载：

```sh
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/com.ctjsiptv.server.plist
```

停止并卸载：

```sh
launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.ctjsiptv.server.plist
```

修改配置或替换二进制后可重新启动：

```sh
launchctl kickstart -k gui/$(id -u)/com.ctjsiptv.server
```

### Linux：systemd

以下示例将二进制放在 `/usr/local/bin`，配置与持久化数据放在 `/usr/local/etc/ctjsiptv`：

```sh
sudo install -m 755 ctjsiptv /usr/local/bin/ctjsiptv
sudo mkdir -p /usr/local/etc/ctjsiptv
sudo install -m 600 ctjsiptv.conf /usr/local/etc/ctjsiptv/ctjsiptv.conf
```

创建 `/etc/systemd/system/ctjsiptv.service`：

```ini
[Unit]
Description=CTJSIPTV
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/usr/local/etc/ctjsiptv
ExecStart=/usr/local/bin/ctjsiptv -c /usr/local/etc/ctjsiptv/ctjsiptv.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

启用并立即启动：

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now ctjsiptv
```

查看状态和日志：

```sh
systemctl status ctjsiptv
journalctl -u ctjsiptv -f
```

重启或停止：

```sh
sudo systemctl restart ctjsiptv
sudo systemctl stop ctjsiptv
```


## 5. 主要配置

完整配置和注释见 `ctjsiptv.conf.example`。

使用 `-c /path/to/ctjsiptv.conf` 时，设置 API 始终读写该文件；未传 `-c` 时使用进程当前工作目录下的 `ctjsiptv.conf`，文件可在首次从设置页保存时创建。环境变量优先级高于配置文件，因此被环境变量覆盖的项目需要同时修改服务环境才能在重启后改变实际值。

| 配置项 | 必填 | 说明 |
|---|---:|---|
| `IPTV_USER_ID` | 否 | IPTV 用户 ID；凭据未完整填写时仅作为 Zero Config 的历史账号提示 |
| `IPTV_PASSWORD` | 否 | IPTV 密码；凭据未完整填写时仅作为 Zero Config 的历史密码提示 |
| `IPTV_STB_ID` | 否 | STB ID；三项凭据完整填写时直接用于 Portal 认证 |
| `NIC` | 否 | IPTV 网络接口；未配置时自动检测 |
| `IPTV_MAC` | 否 | 优先复用缓存的软终端 MAC；无缓存时自动模式生成并持久化，完整显式凭据模式读取 NIC |
| `IPTV_IP` | 否 | 未配置时从 NIC 读取 IPv4 |
| `LISTEN` | 否 | 默认 `[::]:8765` |
| `PUBLIC_URL` | 否 | 统一公开根地址 |
| `DATA_DIR` | 否 | 持久化数据目录；默认是配置文件所在目录 |
| `IPTV_BOOTSTRAP_CACHE` | 否 | Zero Config 身份文件；默认 `DATA_DIR/bootstrap.json` |
| `LIVE_CHANNEL_CACHE` | 否 | 直播频道快照；默认 `DATA_DIR/channel-cache.json` |
| `EPG_CACHE` | 否 | 最近成功的七天 EPG 快照；默认 `DATA_DIR/epg-cache.json` |
| `IMAGE_CACHE_DIR` | 否 | 图片缓存目录；默认 `DATA_DIR/image-cache` |
| `RTP2HTTPD` | 点播必需 | rtp2httpd HTTP/HTTPS 根地址；未配置时点播路由返回 HTTP 503 |
| `LIVE_LOGO_DIR` | 否 | 本地频道 PNG Logo 目录 |
| `LIVE_OVERRIDES` | 否 | 直播源管理文件 |
| `VOD_TITLE_CLEAN_RULES` | 否 | VOD 标题清理规则文件 |
| `XTREAM_ENABLED` | 否 | 启用 Xtream |
| `XTREAM_USERNAME` / `XTREAM_PASSWORD` | 否 | Xtream 独立凭据 |
| `TVBOX_ENABLED` | 否 | 启用 TVBox / MacCMS；默认启用 |

推荐反向代理/公网场景统一配置：

```ini
PUBLIC_URL=https://iptv.example.com
```

推荐为服务单独准备数据目录。例如当前登录用户直接运行：

```sh
sudo mkdir -p /usr/local/etc/ctjsiptv/data
sudo chown -R "$(id -un):$(id -gn)" /usr/local/etc/ctjsiptv
chmod 700 /usr/local/etc/ctjsiptv/data
```

配置：

```ini
DATA_DIR=/usr/local/etc/ctjsiptv/data
```

如果通过 launchd 或 systemd 使用其他账号运行，应将目录所有者改为实际运行账号。该账号必须能够读取配置目录并读写 `DATA_DIR`；身份、频道快照、EPG 快照和规则文件建议保持 `0600`。不要把 `bootstrap.json`、`channel-cache.json` 或 `epg-cache.json` 提交到版本库。

## 6. rtp2httpd

点播播放地址必须经 rtp2httpd 输出；直播和回放仍可按需选择原始或转发接口：

```ini
RTP2HTTPD=https://rtp2httpd.example.com:444
```

rtp2httpd 只负责播放输出转发，不参与 IPTV 认证和 Portal 发现。未配置时，点播、TVBox 播放以及 Xtream VOD/Series 返回 HTTP 503；直播与回放行为不变。

## 7. 内置 Web UI

访问 `http://设备IP:8765/` 即可使用。网页提供影视分类、搜索、详情、分集、播放、直播、EPG、运行状态、网络诊断、重新认证、服务重启、完整配置编辑、直播源管理及 VOD 标题规则管理。

设置页只提交发生变化的配置。`IPTV_USER_ID`、`IPTV_PASSWORD`、`IPTV_STB_ID` 会立即替换当前运行态认证身份并使旧会话失效；其他项目会明确标记为“重启后生效”，保存后由用户确认是否调用重启接口。敏感字段不会从后端回传明文，留空表示保留，勾选“清除”才会写入空值。

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

## 8. Xtream Codes

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

## 9. TVBox / MacCMS

TVBox 推荐直接使用：

```text
http://设备IP:8765/tvbox.json
```

MacCMS type=1 API：

```text
/api.php/provide/vod/
```

使用域名或反向代理时配置唯一的 `PUBLIC_URL`；Web、Xtream 与 TVBox 会共同使用它。

## 10. 原生 API

主要入口：

```text
GET /api/health
GET /api/status
GET /api/diagnostics
POST /api/diagnostics/reauthenticate
POST /api/diagnostics/restart
GET /api/metrics
GET /api/settings
POST /api/settings
GET /api/settings/title-clean
POST /api/settings/title-clean/save
GET /api/categories
GET /api/categories/{type}
GET /api/filters/{type}
GET /api/vod?type=movie&page=1
GET /api/vod/{id}?type=drama
GET /api/play/{id}?type=movie
GET /play/v1/vod.m3u8?type=movie&id={id}
GET /api/play/resolve?media_type={type}&media_code={code}
GET /api/search?q=关键词
GET /api/live/rtp
GET /api/live/http
GET /api/live/rtp/rtp2httpd
GET /api/live/http/rtp2httpd
GET /api/epg?days=7
```

EPG 输出 XMLTV，直播列表输出扩展 M3U。上游实测支持今天及过去 6 天；明天的 `dateIndex=-1` 对 CCTV-1、CCTV-2 均返回空节目单，因此不伪造第 8 天。

`/api/play/{id}` 返回可长期保存的 CTJSIPTV 点播地址，而不是带时效的上游 URL。播放器请求 `/play/v1/vod.m3u8` 时，服务端才使用当前会话解析最新地址，转换为官方 HLS，并通过 `RTP2HTTPD` 包装后以 HTTP 302 返回；响应带 `Cache-Control: no-store`，避免客户端缓存临时重定向。TVBox 与 Xtream 的点播、剧集播放走同一策略，直播和回放不受影响。

`/api/diagnostics` 使用所选 `NIC` 检查本机 IPv4、IPTV DNS、认证服务器、Zero Config、图片 CDN、当前 EPG Portal 和播放能力，并按原软终端的错误类别返回可解释故障。它不上传诊断信息，也不调用官方管理心跳。

认证失败后可调用 `POST /api/diagnostics/reauthenticate` 清除旧会话并立即重新认证。`POST /api/diagnostics/restart` 会在响应发出后以相同可执行文件和命令行参数替换当前进程，PID 及 launchd/systemd 监管关系保持不变。内置诊断页提供对应按钮。如果修改了 `LISTEN`，重启后应改用新地址访问。

`GET /api/settings` 返回全部受支持配置项及分组、类型和生效方式；密码与 Secret 只返回是否已配置，不返回明文。`POST /api/settings` 接受 `{"values":{"KEY":"VALUE"}}`，保留原文件注释并原子更新启动时使用的配置文件，权限设置为 `0600`。敏感输入留空表示保留，确需清空时在 `clear_sensitive` 数组中列出键名。IPTV 用户 ID、密码和 STB ID 会立即进入当前运行态并使旧会话失效；其他设置在重启后生效。

示例：

```sh
# 修改认证身份；密码留空时不会覆盖已有密码
curl -X POST http://127.0.0.1:8765/api/settings \
  -H 'Content-Type: application/json' \
  --data '{"values":{"IPTV_USER_ID":"账号","IPTV_STB_ID":"STBID","IPTV_PASSWORD":"密码"}}'

# 清除 Xtream 密码
curl -X POST http://127.0.0.1:8765/api/settings \
  -H 'Content-Type: application/json' \
  --data '{"values":{},"clear_sensitive":["XTREAM_PASSWORD"]}'

curl -X POST http://127.0.0.1:8765/api/diagnostics/reauthenticate
curl -X POST http://127.0.0.1:8765/api/diagnostics/restart
```

设置读取/写入、标题规则写入、重新认证和重启属于管理接口，只接受本机或配置项 `NIC` 所在同一 IP 子网的客户端；其他来源返回 HTTP 403。若通过本机反向代理公开这些路径，请在代理侧额外配置身份验证、VPN 或 IP 白名单。

HTTP 服务与 Portal 认证相互独立：完成基础配置读取后先启动监听，再在后台认证。即使凭据错误，或者首次运行时无凭据、无缓存且 Zero Config 失败，首页、`/api/health`、`/api/status` 和 `/api/diagnostics` 仍保持可访问；`authentication_state` 与 `auth_message` 会显示当前阶段及失败原因。

`/api/metrics` 只返回当前进程内的匿名计数，例如认证、会话失效、图片缓存命中及按模板归类的请求数；不持久化，也不发送到运营商或第三方。

图片访问按原 APK 逆向结果处理：`imagecdn.jsitv.net:8080/<origin-host>:<port>/...` 优先通过服务端代理访问；CDN 不可用时按 APK 的行为回退到内嵌的 `ioss.jsitv.net:18080` 原站。`imagecache.itv.jsinfo.net:8080`、详情页 `/images/poster/...` 以及 frame326 的相对资源均由服务端通过 IPTV 网卡读取，并缓存到 `IMAGE_CACHE_DIR`，避免 HTTPS 页面混合内容或客户端无法访问 IPTV 专网导致海报空白。

原 APK 的直播频道记录包含 `ChannelLogoURL`。CTJSIPTV 会把该字段保存进频道快照，并用于 M3U、Xtream 和 Xtream XMLTV；远端台标统一经 `/api/image` 访问。安全白名单除固定图片节点外只额外接受本次认证得到的动态 Portal 主机。配置 `LIVE_LOGO_DIR` 时，本地 PNG 仍优先于上游台标。

`/api/play/resolve` 对应原软终端的通用 `GetSPMediaPlayUrl` 能力。逆向代码中的 `IptvOutWardService` 和 `IPlguinDataImpl` 从 `CTCSetConfig` 保存的同名配置读取地址，附加 `mediaType`、`mediaCode`，携带 `JSESSIONID` 发起 GET，并解析 `result`、`playUrl`、`message`。CTJSIPTV 按这个已验证契约实现，同时兼容 Portal 的 `CTCSetConfig` 与旧 `jsSetConfig` 写法。只有当前 Portal 动态发布该地址时才启用；不会猜测或硬编码未下发的上游地址。当前 Portal 不支持时，`/api/status` 的 `media_resolver_available` 为 `false`。

## 11. 内网与公网访问

CTJSIPTV 自带 HTTP 服务和 Web UI，因此内网无需额外 Web Server：

```text
客户端 → http://CTJSIPTV主机:8765
```

需要域名、HTTPS 或公网访问时，可在 CTJSIPTV 前增加 Apache/Nginx/Caddy 等反向代理，将请求原样转发到 `127.0.0.1:8765`，并设置：

```ini
PUBLIC_URL=https://iptv.example.com
```

反向代理应保留原始 URI；部分 VOD ID 可能包含 `%2F` 等编码字符。公网入口应自行增加身份验证、VPN、IP 白名单或可信网关。不要直接裸露 IPTV 凭据或临时播放鉴权信息。

## 12. 更新

停止旧进程后替换 `ctjsiptv` 二进制并重新启动即可。保留：

```text
ctjsiptv.conf
live-overrides.json
vod-title-clean.json
频道 Logo 目录
```

升级后通过 `/api/status` 检查版本和 IPTV 会话状态。

## License

仅用于个人学习、协议研究和合法的自有 IPTV 服务接入。使用者应自行确保符合当地法律、运营商服务协议及内容授权要求。
