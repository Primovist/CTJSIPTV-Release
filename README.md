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
- 直播源启停、排序、分组调整、自定义源与本地频道 Logo
- VOD 标题清理

## 2. 运行环境

macOS 最低支持版本为 14。Linux x86_64 与 arm64 发布版使用 Swift 6.1 构建，静态嵌入 Swift 运行库并执行 Strip，因此无需安装 Swift 工具链或 Swift 运行库。Linux 版以 Ubuntu 22.04 为构建基线，适用于 ABI 兼容的 glibc 系统。

Linux 二进制仍动态依赖系统运行库，包括 glibc、`libcurl.so.4`、`libssl.so.3`、`libcrypto.so.3`、`libstdc++.so.6`、`libgcc_s.so.1` 和 `libm.so.6`。此外，系统的 libcurl 可能依赖 HTTP/2、SSH、PSL、压缩及认证相关共享库；具体清单随发行版和 libcurl 构建选项而异。通过发行版包管理器安装 curl 和 OpenSSL 通常会一并安装这些传递依赖；若系统精简或使用自定义 libcurl，请确认相关共享库均已安装。还需要安装 `tzdata`，提供 EPG 和媒资时间处理使用的系统时区数据库。

Debian/Ubuntu 首次运行前安装依赖：

```sh
sudo apt update
sudo apt install -y curl openssl tzdata
```

可以运行 `ldd ./ctjsiptv` 检查动态依赖；输出中如有 `not found`，请安装对应的系统运行库。

当前 Linux 发布版面向 glibc 系统，不兼容标准 musl OpenWrt 固件；OpenWrt 原生版尚未提供。

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

程序会生成并持久化稳定软终端设备身份，通过运营商 Zero Config 获取 IPTV Account、Password、STBID 等启动身份，然后继续现有动态 Portal/Auth。启动身份默认保存在配置文件同目录的 `bootstrap.json`（权限 `0600`），后续 Zero Config 暂时不可用时可回退使用。官方软终端心跳默认关闭；设置 `IPTV_HEARTBEAT_ENABLED=true` 后每 30 秒发送一次，需要 Zero Config 返回的 `virtualAccount`。诊断页和 `/api/diagnostics` 只展示最近一次心跳的传输状态、业务码、说明及 HLS / 单播字段；不执行服务端指令或改变播放策略。原 APK 将 `hlsStatus=0` 映射为 HLS 偏好开启，将 `unicastNode` 用作 RTSP 单播节点选择；这两个字段都不是 HLS 服务器地址，CTJSIPTV 目前只记录、不应用它们。

程序启动时及之后每 24 小时检查一次官方 APK 更新，也可从配置网卡所在子网或本机调用 `POST /api/official-apk` 手动检查。`GET /api/official-apk` 查看最近的版本、下载地址和检查错误。检查使用已实测返回完整 APK 的 `/api/apk/query`；新版 `/api/apk/upgrade/query` 同时涉及插件更新，暂不将其版本写入启动请求。发现更高版本后自动更新 `DATA_DIR/official-client.json` 中的 `apkVersion` 和 `apkVersionName`；该 JSON 还保存 Zero Config 的固定客户端字段和 Portal 的 `userAgent`，首次启动自动创建。默认客户端身份采用 Android 17 / Apple TV；旧版默认 Android 12 / Apple Mac 档案会迁移，自定义终端描述保留。修改 `userAgent` 后须重启服务。检查记录保存在 `DATA_DIR/official-apk-status.json`。可用 `IPTV_OFFICIAL_CLIENT_PROFILE` 和 `IPTV_OFFICIAL_APK_STATUS` 指定路径。仅更新请求档案，不下载或安装 APK。

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
| `MANAGEMENT_CIDRS` | 否 | 管理接口允许访问的 IPv4/IPv6 CIDR，逗号分隔；设置后替代 IPTV NIC 子网规则，本机回环始终允许；留空时沿用 IPTV NIC 子网 |
| `PUBLIC_URL` | 否 | 统一公开根地址；未设置时，HTTP 响应链接沿用请求的 Host，后台和持久化链接使用 IPTV 网卡地址 |
| `DATA_DIR` | 否 | 持久化数据目录；默认是配置文件所在目录 |
| `IPTV_BOOTSTRAP_CACHE` | 否 | Zero Config 身份文件；默认 `DATA_DIR/bootstrap.json` |
| `IPTV_COOKIE_FILE` | 否 | 上游会话 Cookie 文件；默认 `DATA_DIR/session-cookies.txt` |
| `LIVE_CHANNEL_CACHE` | 否 | 直播频道快照；默认 `DATA_DIR/channel-cache.json` |
| `EPG_CACHE` | 否 | 最近成功的七天 EPG 快照；默认 `DATA_DIR/epg-cache.json` |
| `IMAGE_CACHE_DIR` | 否 | 图片缓存目录；默认 `DATA_DIR/image-cache` |
| `VOD_CATALOG` | 否 | 全局 VOD 身份与稳定播放映射文件；默认 `DATA_DIR/vod-catalog.json` |
| `RTP2HTTPD` | 否 | rtp2httpd HTTP/HTTPS 根地址；点播仍按现有策略使用转发，直播是否转发由 `LIVE_OUTPUT_MODE` 选择 |
| `LIVE_OUTPUT_MODE` | 否 | Xtream 与 TVBox 直播模式：`unicast`、`multicast`；配置 RTP2HTTPD 后还可选 `unicast-forwarded`、`multicast-forwarded`；默认 `unicast` |
| `LIVE_LOGO_DIR` | 否 | 本地频道 PNG Logo 目录；默认是配置文件同目录下的 `Logo/` 文件夹 |
| `LIVE_OVERRIDES` | 否 | 直播源管理文件 |
| `VOD_TITLE_CLEAN_RULES` | 否 | VOD 标题清理规则文件 |
| `STRM_ENABLED` | 否 | 启用 STRM 媒体库导出；默认关闭 |
| `STRM_OUTPUT_DIR` | 否 | STRM 输出目录；默认配置文件同目录的 `strm/` |
| `STRM_NAMING_MODE` | 否 | `standard` 或 `infuse-edition`；默认 `standard` |
| `XTREAM_ENABLED` | 否 | 启用 Xtream；默认关闭 |
| `XTREAM_USERNAME` / `XTREAM_PASSWORD` | 否 | Xtream 独立凭据 |
| `TVBOX_ENABLED` | 否 | 启用 TVBox / MacCMS；默认关闭 |

`XTREAM_REGISTRY` 和旧默认文件 `xtream-vod-registry.json` 不再读取或自动迁移。需要保留已有稳定 VOD ID 时，请在切换版本前自行把旧文件移到 `VOD_CATALOG` 指向的位置。

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

所有未单独配置的缓存、映射、规则文件和缓存目录都保存在配置文件同目录；显式设置 `DATA_DIR` 后，未单独配置的持久化内容改为保存到该目录。某个文件或目录配置项一旦单独设置，则优先于 `DATA_DIR`；相对路径始终以 `ctjsiptv.conf` 所在目录为基准。

HTTP 会话只使用一个 `IPTV_COOKIE_FILE`。每次请求会读取当前 Cookie，完成后覆盖为最新 Cookie，并删除其中已经到期的记录；旧版 `/tmp/jsiptv-cookies-<PID>` 文件会立即删除，更换配置后遗留且超过 24 小时未更新的同目录 Cookie 文件也会自动删除，当前活动文件不会被清理。

如果通过 launchd 或 systemd 使用其他账号运行，应将目录所有者改为实际运行账号。该账号必须能够读取配置目录并读写 `DATA_DIR`；身份、Cookie、频道快照、EPG 快照、Xtream 映射和规则文件保持 `0600`。不要把这些运行时文件提交到版本库。

## 6. rtp2httpd

rtp2httpd 是可选的播放转发层；直播和回放仍可按需选择原始或转发接口：

```ini
RTP2HTTPD=https://rtp2httpd.example.com:444
```

rtp2httpd 只负责播放输出转发，不参与 IPTV 认证和 Portal 发现。配置后，点播、TVBox 点播以及 Xtream VOD/Series 继续按原策略使用转发地址；Xtream/TVBox 直播是否转发由“Xtream / TVBox 直播输出”模式选择。直连模式返回上游原始地址，播放端需要具备 IPTV 专网访问能力。

## 7. 内置 Web UI

访问 `http://设备IP:8765/` 即可使用。网页提供影视分类、搜索、详情、分集、播放、直播、EPG、运行状态、网络诊断、重新认证、服务重启、完整配置编辑、直播源管理及 VOD 标题规则管理。

设置页只提交发生变化的配置。`IPTV_USER_ID`、`IPTV_PASSWORD`、`IPTV_STB_ID` 会立即替换当前运行态认证身份并使旧会话失效；其他项目会明确标记为“重启后生效”，保存后由用户确认是否调用重启接口。敏感字段不会从后端回传明文，留空表示保留，勾选“清除”才会写入空值。

设置页的「分类筛选」支持电影、剧集、短剧、动漫、少儿、综艺和电竞的拖动排序、上下移动及显示开关。点击「保存分类」后立即更新网页、TVBox 和 Xtream 的主分类筛选，隐藏项目保留原位置；「恢复默认」也需保存后生效。配置保存到 `DATA_DIR/vod-type-overrides.json`，使用 `order` 和 `disabled` 字段，重启后保留。允许隐藏全部分类，可随时在设置页重新显示。设置页各分区及配置子分区均可折叠，展开状态保存在当前浏览器。 TVBox 的分类、首页推荐和搜索结果排除隐藏类型；Xtream 的分类和列表按配置输出，电影与剧集分别排序（现有 Xtream 不提供电竞分类）。客户端缓存的分类需刷新后显示。隐藏不影响已有详情和播放地址，也不改变内部媒体索引及 STRM 导出。

直播频道编辑中的「分组」支持选择现有分组或输入新分组，留空保存可恢复自动分组。与排序、启停一样，修改保存在 `live-overrides.json`，重启后保留，并应用于 M3U 与 Xtream 直播输出。同一上游频道的不同清晰度源同步调整；JSON 的 `groups` 字段为频道 ID 到分组名称的映射。
「管理分组」支持新增分组、上下移动调整输出顺序，以及删除分组并将频道移入另一个分组。分组顺序不改变组内频道顺序。默认分组也可删除，未来自动匹配该分组的频道会转入指定分组；至少保留一个分组。空分组保留在管理界面和配置中，M3U 与 Xtream 仅输出包含当前协议可用且已启用频道的分组。`groupOrder` 保存分组列表及顺序，`groupRedirects` 保存已删除分组的迁移目标，旧 override 文件无需手动迁移。


`live-overrides.json` 与 `vod-title-clean.json` 默认自动定位到 `ctjsiptv.conf` 所在目录，也可以通过配置项指定路径。

直播源页面可设置原生 M3U 回放起止时间格式，保存后立即生效，并持久化到 `live-overrides.json` 的 `catchupFormats.direct`（含 `start`、`end`）。填写 `${}` 内播放器会展开的格式，例如 `(b)yyyyMMddHHmmss|Etc/GMT` 和 `(e)yyyyMMddHHmmss|Etc/GMT`。客户端展开时间后，请求 CTJSIPTV 生成的回看地址。配置了 `RTP2HTTPD` 时，频道组优先查找 HD 及以上画质且具备完整 `ch` 节目标识的 RTSP 回看源，使用 `/catchup.rtsp` 保留鉴权 Query 并跳转到 rtp2httpd；没有这类源的频道使用 `/catchup.m3u8` 输出去除鉴权 Query 的 HTTP HLS 地址。未配置 `RTP2HTTPD` 时，所有频道使用 `/catchup.m3u8`。自定义频道的 GSLB 地址暂不转换成鉴权 RTSP 回看。TVBox 将播放器时间改写为 Unix 时间戳，Xtream 使用其回放参数；两者与原生 M3U 使用同一套频道选择和回看解析逻辑。起止字段可分别留空恢复默认格式；修改后需刷新客户端列表。保存或读取到不支持的格式时会回退默认值并在页面提示及服务日志中记录。

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

此兼容接口默认关闭，使用前先配置 `TVBOX_ENABLED=true` 并重启服务。

TVBox 推荐直接使用：

```text
http://设备IP:8765/tvbox.json
```

MacCMS type=1 API：

```text
/api.php/provide/vod/
```

直接通过设备 IP 访问时无需设置 `PUBLIC_URL`：TVBox、直播清单及 Xtream 返回的本机链接会沿用请求的 Host；STRM 等后台生成的持久化链接使用 IPTV 网卡地址。使用域名、HTTPS 或反向代理时配置唯一的 `PUBLIC_URL`，它会覆盖上述推导并由 Web、Xtream 与 TVBox 共同使用。

设置页的“Xtream / TVBox 直播输出”选择统一影响这两个客户端：单播输出 HTTP/HTTPS，组播输出 RTP/UDP；配置 RTP2HTTPD 后可选择对应的转发模式。未配置转发地址时，转发选项会隐藏，遗留的转发模式也会自动回退到对应直连模式。原生 API 与 APTV 的直播列表不受此设置影响。

TVBox 配置中的直播入口 `/tvbox/live` 使用当前所选模式。单播回放模板以 Unix 秒级时间戳传递，避免设备时区造成 8 小时偏差；播放时调用对应配置下的 `/catchup.rtsp` 或 `/catchup.m3u8` 服务入口，与原生 M3U、Xtream 使用相同的回看解析器。切换输出模式后，刷新 TVBox 的直播列表。

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

`/api/play/{id}` 返回可长期保存的 CTJSIPTV 点播地址，而不是带时效的上游 URL。播放器请求 `/play/v1/vod.m3u8` 时，服务端才使用当前会话解析最新地址并转换为官方 HLS：配置 `RTP2HTTPD` 时包装为转发地址，未配置时直接返回上游原始地址，随后以 HTTP 302 跳转；响应带 `Cache-Control: no-store`，避免客户端缓存临时重定向。TVBox 与 Xtream 的点播、剧集播放走同一策略，直播和回放不受影响。

RTSP 转 HLS 使用官方 GSLB 入口 `http://gslb.itv.jsinfo.net:6060`，沿用 RTSP 的资源路径和查询参数生成 `index.m3u8` 地址；实际媒体节点由 GSLB 调度。心跳不下发 HLS 主机名：`hlsStatus` 是转换开关，`unicastNode` 选择 RTSP 单播节点，不能拿它替代 HLS 地址。诊断页展示 HLS 转换规则是否已实现，不代表已实际播放验证。

`/api/diagnostics` 使用所选 `NIC` 并行探测认证入口、Zero Config、图片 CDN 和当前 EPG Portal，分开报告 TCP 建连耗时与未携带认证信息的 HEAD 响应。HTTP 403／503 仍说明对端有响应，不等于网络不通；HTTP 200 不代表业务认证成功。首字节时间包含 DNS 与建连，总耗时是整次探测请求时间，均不是 ping 延迟。会话状态来自现有会话记录，诊断不会重新认证或触发播放。

域名通过系统解析器查询 IPv4，不代表已验证专网 DNS；直接 IP 不再列为 DNS 查询。当前 Portal 节点标明来自认证／负载均衡下发。播放部分展示配置与可选接口：`GetSPMediaPlayUrl` 仅供 `/api/play/resolve` 通用解析使用，未下发不影响使用独立链路的常规直播、回放和点播，也不能据此判定整体播放受限。

诊断还展示后台心跳的最近一次结果；读取诊断不会额外发送心跳，也不会执行返回的指令。管理页面的 APK 检查、心跳及带时区的媒资时间按浏览器本地时区显示，API 仍返回原始时间；EPG 节目表沿用现有的江苏业务时区。

认证失败后可调用 `POST /api/diagnostics/reauthenticate` 清除旧会话并立即重新认证。`POST /api/diagnostics/restart` 会在响应发出后以相同可执行文件和命令行参数替换当前进程，PID 及 launchd/systemd 监管关系保持不变。内置诊断页提供对应按钮。如果修改了 `LISTEN`，重启后应改用新地址访问。

网页重启（包括保存设置后的重启）不依赖 systemd 的 `Restart=` 策略；群晖任务计划通过命令直接启动、普通命令行或 `nohup` 启动也使用同一原地重启机制，保留命令行参数、环境变量和工作目录。Linux 监听及已接受的连接均设置 close-on-exec，替换进程时自动释放，避免旧连接占用端口导致新程序启动失败。群晖可使用 `nohup /path/to/ctjsiptv -c /path/to/ctjsiptv.conf >> /path/to/ctjsiptv.log 2>&1 &` 启动；路径应替换为实际位置，运行账号需有相应权限。

`GET /api/settings` 返回全部受支持配置项及分组、类型、生效方式、配置文件值和当前运行值；密码与 Secret 只返回是否已配置，不返回明文。设置页面的输入框只装载配置文件中实际存在的值，Zero Config、自动探测、环境变量和程序默认值仅显示为当前生效提示。点击保存只提交用户修改过的项目，不会把此前未配置的账号、MAC、IP、路径或布尔默认值写入配置文件。

`POST /api/settings` 接受 `{"values":{"KEY":"VALUE"}}`，保留原文件注释并仅原子更新提交的键，文件权限设置为 `0600`。敏感输入留空表示保留，确需清空时在 `clear_sensitive` 数组中列出键名。IPTV 用户 ID、密码和 STB ID 会立即进入当前运行态并使旧会话失效；其他设置在重启后生效。

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

设置读取/写入、标题规则写入、重新认证和重启属于管理接口。本机回环始终允许；默认只接受 `NIC` 所在子网的客户端。设置 `MANAGEMENT_CIDRS` 后，改为只接受列表中的 IPv4/IPv6 网段，适用于 IPTV 专网和管理 LAN 使用不同网卡、桥接或 VLAN 的路由器。例如 `MANAGEMENT_CIDRS=192.168.2.0/24,fd00:2::/64`。只填写可信网段，避免 `0.0.0.0/0` 或 `::/0` 这类全地址范围。其他来源返回 HTTP 403。若通过反向代理公开这些路径，请在代理侧额外配置身份验证、VPN 或 IP 白名单。

HTTP 服务与 Portal 认证相互独立：完成基础配置读取后先启动监听，再在后台认证。即使凭据错误，或者首次运行时无凭据、无缓存且 Zero Config 失败，首页、`/api/health`、`/api/status` 和 `/api/diagnostics` 仍保持可访问；`authentication_state` 与 `auth_message` 会显示当前阶段及失败原因。

`/api/metrics` 只返回当前进程内的匿名计数，例如认证、会话失效、图片缓存命中及按模板归类的请求数；不持久化，也不发送到运营商或第三方。

图片访问按原 APK 逆向结果处理：`imagecdn.jsitv.net:8080/<origin-host>:<port>/...` 优先通过服务端代理访问；CDN 不可用时按 APK 的行为回退到内嵌的 `ioss.jsitv.net:18080` 原站。`imagecache.itv.jsinfo.net:8080`、详情页 `/images/poster/...` 以及 frame326 的相对资源均由服务端通过 IPTV 网卡读取，并缓存到 `IMAGE_CACHE_DIR`，避免 HTTPS 页面混合内容或客户端无法访问 IPTV 专网导致海报空白。

Web、TVBox/MacCMS 和 Xtream 共用服务端海报选择与详情补全：电竞列表的 `default.png`、`defaultcolumn_n.jpg` 仅作为最后回退；优先获取详情中的真实海报，兼容备用海报字段和剧集海报。临时补全失败不会永久缓存为空图。

原 APK 的直播频道记录包含 `ChannelLogoURL`。CTJSIPTV 会把该字段保存进频道快照，并用于 M3U、Xtream 和 Xtream XMLTV；远端台标统一经 `/api/image` 访问。安全白名单除固定图片节点外只额外接受本次认证得到的动态 Portal 主机。本地 PNG 默认从配置文件同目录下的 `Logo/` 文件夹读取，也可用 `LIVE_LOGO_DIR` 改写；仅在对应文件存在且可读时优先，否则自动回退到上游台标。

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
