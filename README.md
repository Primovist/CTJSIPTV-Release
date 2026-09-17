# CTJSIPTV Release

江苏电信 IPTV 服务端公开发行仓库。本仓库提供编译后的发行版、部署方法与公开接口说明；服务端源码保存在私有开发仓库。

> 使用前提：运行 CTJSIPTV 的设备需要能够访问已融合到本地网络的江苏电信 IPTV 专网。本项目不负责光猫、路由器或运营商侧的 IPTV 接入与业务订购。

## 0.7.0

0.7.0 在原有 `/api` 接口保持兼容的基础上，正式加入 Xtream Codes 兼容层，可用于 Lume 等支持 Xtream 的客户端，并已验证直播电视指南、历史节目回放、电影、剧集、动漫、少儿和综艺等主要路径。

本版本包含动态 IPTV Portal/会话发现，不再依赖单一固定 EPG Portal IP；Xtream XMLTV 的频道 ID 与直播 `epg_channel_id` 对齐；支持 `get_short_epg`、`get_simple_data_table` 和 `/timeshift/...` 回放；配置 RTP2HTTPD 后，Xtream 直播、点播和回放继续经 RTP2HTTPD 输出。对于被归类为 Series、但上游没有 `serieslist` 的单集节目，会自动合成为一个可播放 Episode，避免客户端显示“无可用剧集”。

图片代理和详情补全也已适配 Xtream 客户端。配置文件中当前版本不认识的配置项或格式异常的非关键行会给出警告并忽略；缺失必要配置或已知关键配置值非法仍会报错退出。

## 客户端兼容

### Xtream Codes

在 `ctjsiptv.conf` 中配置：

```ini
XTREAM_ENABLED=true
XTREAM_USERNAME=你的用户名
XTREAM_PASSWORD=你的密码

# 外网或反向代理访问时建议设置
XTREAM_PUBLIC_URL=https://iptv.example.com
```

客户端添加 Xtream 服务器时填写 CTJSIPTV 的访问地址、`XTREAM_USERNAME` 和 `XTREAM_PASSWORD`。兼容接口包括 `player_api.php`、`xmltv.php`、直播、电影、Series 与 timeshift 回放路径。

如果配置了 `RTP2HTTPD`，Xtream 返回的可播放内容会继续使用该代理路径，客户端不会直接依赖 IPTV 专网 RTP 地址。

### 原生 API / M3U / XMLTV

原有 `/api/*` 输出保持兼容，可继续使用：

| Method | Path | 说明 |
|---|---|---|
| GET | `/api/health` | 服务健康状态 |
| GET | `/api/status` | 版本、IPTV 会话、网卡、IPTV IP、RTP2HTTPD 状态 |
| GET | `/api/image?url=...` | 图片代理 |
| GET | `/api/live/rtp` | RTP/UDP 直播及回看信息 |
| GET | `/api/live/http` | HTTP HLS 直播及回看信息 |
| GET | `/api/live/rtp/rtp2httpd` | RTP/UDP 经 RTP2HTTPD 改写 |
| GET | `/api/live/http/rtp2httpd` | HTTP/回看经 RTP2HTTPD 改写 |
| GET | `/api/categories` | 业务类型 |
| GET | `/api/categories/{type}` | 动态分类 |
| GET | `/api/filters/{type}` | 筛选项 |
| GET | `/api/search?q=...` | 搜索 |
| GET | `/api/search/suggest?q=...` | 搜索联想 |
| GET | `/api/vod?type=...&page=1` | VOD 列表 |
| GET | `/api/vod/{id}?type=...` | 详情与分集 |
| GET | `/api/play/{id}?type=...&episode_id=...` | 播放地址 |
| GET | `/api/epg` | XMLTV EPG |
| GET | `/api/epg/{channel}` | 单频道 XMLTV EPG |

支持的 VOD 类型包括 `movie`、`drama`、`short_drama`、`anime`、`kids`、`variety` 和 `esports`。

## 后续计划

下一阶段计划增加 **TVBox 格式兼容**，目标是在不破坏现有 `/api` 与 Xtream 接口的前提下，为 TVBox 及其兼容客户端提供可直接添加的接口/配置输出。具体字段和兼容范围将在实现与实际客户端验证后确定。

## 下载

请从 Releases 下载最新正式版。当前提供：

| 文件 | 平台 |
|---|---|
| `ctjsiptv-macos-arm64` | macOS Apple Silicon |
| `ctjsiptv-linux-arm64` | Linux arm64 |
| `ctjsiptv-linux-amd64` | Linux x86_64 / amd64 |
| `index.html` | 网页前端备份 |

macOS Apple Silicon 已在实际 IPTV 网络中验证。Linux arm64 与 amd64 由 GitHub Actions 构建并进行离线自测，实际运行仍取决于系统运行库、网卡和 IPTV 路由配置。

## 快速部署

以 macOS arm64 为例：

```sh
mkdir -p ~/CTJSIPTV
cd ~/CTJSIPTV
curl -L -o ctjsiptv https://github.com/Primovist/CTJSIPTV-Release/releases/latest/download/ctjsiptv-macos-arm64
chmod +x ctjsiptv
xattr -d com.apple.quarantine ./ctjsiptv 2>/dev/null || true
```

Linux 将文件名替换为对应的 `ctjsiptv-linux-arm64` 或 `ctjsiptv-linux-amd64`。

创建 `ctjsiptv.conf`：

```ini
IPTV_USER_ID=你的IPTV用户ID
IPTV_PASSWORD=你的IPTV密码
IPTV_STB_ID=你的STB_ID

# 可选
# IPTV_MAC=AA:BB:CC:DD:EE:FF
# IPTV_IP=192.168.1.100
# NIC=en0
# LISTEN=[::]:8765
# RTP2HTTPD=http://192.168.1.2:5140
# XTREAM_ENABLED=true
# XTREAM_USERNAME=iptv
# XTREAM_PASSWORD=change-me
# XTREAM_PUBLIC_URL=https://iptv.example.com
```

启动：

```sh
./ctjsiptv -c ./ctjsiptv.conf
```

默认监听 `[::]:8765`，可通过以下命令验证：

```sh
curl http://127.0.0.1:8765/api/health
curl http://127.0.0.1:8765/api/status
```

访问服务根路径 `/` 可使用内置网页前端。

## 更新

停止服务后，用最新 Release 中对应平台的二进制替换旧文件即可。`ctjsiptv.conf` 独立保存，不需要随程序覆盖。升级后建议首先检查：

```sh
curl http://127.0.0.1:8765/api/status
```

## 安全提示

不要把真实 IPTV 账号、密码、Cookie、Token 或临时鉴权信息提交到公开仓库。若服务暴露到公网，建议在反向代理、VPN 或可信网关层增加访问控制。CTJSIPTV 不应被视为面向公网的多用户认证网关。
