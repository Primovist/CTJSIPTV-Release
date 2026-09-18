# CTJSIPTV Release

江苏电信 IPTV 服务端公开发行仓库。本仓库只面向最终用户，提供编译后的发行版、部署方法、配置说明和客户端接入方法。服务端源码、上游协议分析和开发实现文档保存在私有开发仓库。

> 使用前提：运行 CTJSIPTV 的设备需要能够访问已经融合到本地网络的江苏电信 IPTV 专网。本项目不负责光猫、路由器或运营商侧的 IPTV 接入与业务订购。

## 下载

请从 Releases 下载最新正式版：

| 文件 | 平台 |
|---|---|
| `ctjsiptv-macos-arm64` | macOS Apple Silicon |
| `ctjsiptv-linux-arm64` | Linux arm64 |
| `ctjsiptv-linux-amd64` | Linux x86_64 / amd64 |

macOS Apple Silicon 已在实际 IPTV 网络中验证。Linux 构建能否正常使用还取决于系统运行库、网卡和 IPTV 路由配置。

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

# 可选：不填写时自动检测
# IPTV_MAC=AA:BB:CC:DD:EE:FF
# IPTV_IP=192.168.1.100
# NIC=en0

# 服务监听
# LISTEN=[::]:8765

# 可选：直播转发
# RTP2HTTPD=http://192.168.1.2:5140
```

启动：

```sh
./ctjsiptv -c ./ctjsiptv.conf
```

默认监听 `[::]:8765`。可验证：

```sh
curl http://127.0.0.1:8765/api/health
curl http://127.0.0.1:8765/api/status
```

## 配置

配置文件使用 `KEY=VALUE` 格式。主要配置：

| 配置项 | 必填 | 说明 |
|---|---:|---|
| `IPTV_USER_ID` | 是 | IPTV 用户 ID |
| `IPTV_PASSWORD` | 是 | IPTV 密码 |
| `IPTV_STB_ID` | 是 | 机顶盒 STB ID |
| `IPTV_MAC` | 否 | 未配置时从选定网卡读取 |
| `IPTV_IP` | 否 | 未配置时从选定网卡读取 IPv4 |
| `NIC` | 否 | IPTV 网络接口；未配置时自动检测 |
| `LISTEN` | 否 | 默认 `[::]:8765` |
| `RTP2HTTPD` | 否 | RTP/HTTP 转发服务地址 |
| `LIVE_OVERRIDES` | 否 | 直播源管理数据文件 |
| `XTREAM_ENABLED` | 否 | 是否启用 Xtream |
| `XTREAM_USERNAME` | 否 | Xtream 用户名 |
| `XTREAM_PASSWORD` | 否 | Xtream 密码 |
| `XTREAM_PUBLIC_URL` | 否 | Xtream 对外访问基址 |
| `TVBOX_PUBLIC_URL` | 否 | TVBox 对外访问基址 |

`RTP2HTTPD` 示例：

```ini
RTP2HTTPD=http://192.168.1.10:5140
```

直播源启用、禁用、自定义源和排序默认保存在配置文件同目录的 `live-overrides.json`。如需指定位置：

```ini
LIVE_OVERRIDES=/path/to/live-overrides.json
```

## 使用方式

CTJSIPTV 使用同一套 IPTV 数据提供四种访问方式：

```text
江苏电信 IPTV
      │
      ▼
   CTJSIPTV
      │
      ├── 原生 API ── 内置网页
      ├── Xtream Codes
      └── TVBox / MacCMS
```

原生 API 是基础接口；内置网页、Xtream 和 TVBox 是针对不同客户端提供的访问方式。

## 内置网页

浏览器访问：

```text
http://设备IP:8765/
```

或：

```text
http://设备IP:8765/index.html
```

网页提供影视分类、搜索、详情、分集、播放、直播、EPG、运行状态以及直播源管理。

直播源管理支持查看上游频道、启用/禁用频道、添加/编辑/删除自定义直播源以及调整全局频道顺序。

## 原生 API

成功的 JSON API 使用 `code=0`。EPG 直接输出 XMLTV，直播列表输出扩展 M3U。

### 状态与分类

| Method | Path | 说明 |
|---|---|---|
| GET | `/api/health` | 服务健康状态 |
| GET | `/api/status` | 服务、IPTV 会话、网卡等状态 |
| GET | `/api/categories` | 业务类型 |
| GET | `/api/categories/{type}` | 动态分类 |
| GET | `/api/categories/variety` | 综艺分类 |
| GET | `/api/categories/esports` | 电竞分类 |
| GET | `/api/filters/{type}` | 筛选项 |

支持的业务类型：

| ID | 内容 |
|---|---|
| `movie` | 电影 |
| `drama` | 剧集 |
| `short_drama` | 短剧 |
| `anime` | 动漫 |
| `kids` | 少儿 |
| `variety` | 综艺 |
| `esports` | 电竞 |

### 点播

```text
GET /api/vod?type=movie&page=1
GET /api/vod?type=drama&page=1
GET /api/vod?type=short_drama&page=1
GET /api/vod?type=anime&page=1
GET /api/vod?type=kids&page=1
GET /api/vod?type=variety&category=<分类ID>&page=1
GET /api/vod?type=esports&category=<分类ID>&page=1
```

详情：

```text
GET /api/vod/{id}?type=drama
```

播放：

```text
GET /api/play/{id}?type=movie
GET /api/play/{id}?type=drama&episode_id={episode_id}
```

### 搜索

```text
GET /api/search?q=庆余年
GET /api/search?q=庆余年&type=drama&page=1&page_size=20
GET /api/search/suggest?q=QYN
```

### 直播

```text
GET /api/live/rtp
GET /api/live/http
GET /api/live/rtp/rtp2httpd
GET /api/live/http/rtp2httpd
```

前两个接口提供直连直播列表；后两个在配置 `RTP2HTTPD` 后输出经过转发的地址。

### EPG

```text
GET /api/epg
GET /api/epg?days=7
GET /api/epg/CCTV-1
GET /api/epg/CCTV-1?days=7
```

EPG 使用 XMLTV 格式。服务会缓存频道目录和节目数据，并在每天北京时间零点更新缓存。

## Xtream Codes

适用于 Lume 等支持 Xtream Codes 的客户端。

配置：

```ini
XTREAM_ENABLED=true
XTREAM_USERNAME=iptv
XTREAM_PASSWORD=change-me

# 使用域名或反向代理时建议配置
XTREAM_PUBLIC_URL=https://iptv.example.com
```

客户端填写：

```text
Server:   http://设备IP:8765
Username: iptv
Password: change-me
```

使用反向代理时将 Server 改为 `XTREAM_PUBLIC_URL`。

兼容的主要能力包括：

```text
/player_api.php
/xmltv.php

get_live_categories
get_live_streams

get_vod_categories
get_vod_streams
get_vod_info

get_series_categories
get_series
get_series_info

get_short_epg
get_simple_data_table
```

播放路径包括：

```text
/live/{username}/{password}/{stream_id}.ts
/live/{username}/{password}/{stream_id}.m3u8

/movie/{username}/{password}/{stream_id}.*
/series/{username}/{password}/{stream_id}.*

/timeshift/{username}/{password}/{duration}/{start}/{stream_id}.ts
```

XMLTV：

```text
/xmltv.php?username=iptv&password=change-me
```

直播的 `epg_channel_id` 与 XMLTV channel ID 对应。支持该能力的客户端可以显示节目单和历史节目回放。

配置 `RTP2HTTPD` 后，Xtream 的直播、点播和回放会使用相应的转发输出。

## TVBox

TVBox 支持采用 `type: 1` 的 MacCMS JSON 接口接入。

### 推荐：直接使用配置入口

```text
http://设备IP:8765/tvbox.json
```

如果经过反向代理：

```ini
TVBOX_PUBLIC_URL=https://iptv.example.com
```

则使用：

```text
https://iptv.example.com/tvbox.json
```

`TVBOX_PUBLIC_URL` 未设置时会依次使用 `XTREAM_PUBLIC_URL` 和监听地址作为生成地址的依据。

生成的 TVBox 站源核心结构为：

```json
{
  "key": "ctjsiptv",
  "name": "江苏电信 IPTV",
  "type": 1,
  "api": "http://设备IP:8765/api.php/provide/vod/",
  "searchable": 1,
  "quickSearch": 1,
  "filterable": 0
}
```

配置同时提供 CTJSIPTV 的 HTTP 直播列表。

### TVBox / MacCMS API

入口：

```text
/api.php/provide/vod/
```

分类分页：

```text
GET /api.php/provide/vod/?ac=detail&t=movie&pg=1
```

搜索：

```text
GET /api.php/provide/vod/?wd=庆余年&ac=detail
```

详情：

```text
GET /api.php/provide/vod/?ac=detail&ids={vod_id}
```

详情返回标准字段，例如：

```text
vod_id
vod_name
vod_pic
vod_remarks
vod_year
vod_area
vod_actor
vod_director
vod_content
vod_play_from
vod_play_url
```

多集内容使用 TVBox/MacCMS 常见的：

```text
第1集$URL#第2集$URL#第3集$URL
```

格式。

播放通过：

```text
/tvbox/play.m3u8
```

按需解析真正的上游播放地址，因此读取剧集详情时不会预先解析所有分集的临时播放 URL。

> TVBox 支持目前位于开发临时分支，在正式 Release 合并并发布对应版本前，以正式 Release 中的实际功能为准。

## 更新

停止服务后，用最新 Release 中对应平台的二进制替换旧文件即可。配置文件和 `live-overrides.json` 独立保存，不需要随程序覆盖。

升级后建议检查：

```sh
curl http://127.0.0.1:8765/api/status
```

## 安全

不要将真实 IPTV 用户 ID、密码、Cookie、Token、STB 信息或临时鉴权参数提交到公开仓库。

如果服务暴露到公网，建议通过反向代理、VPN 或可信网关增加访问控制。CTJSIPTV 本身不应被视为面向公网的多用户认证网关。
