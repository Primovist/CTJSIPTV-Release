# CTJSIPTV 发行版

Swift 实现的江苏电信 IPTV 服务端发行版，提供认证、直播、回看、VOD 分类/筛选、搜索、详情、分集、综艺、电竞、播放地址和 XMLTV EPG。服务端已内置网页前端，源码仓库保持私有，本仓库用于公开发布构建产物。

最新版本请见 [Releases](https://github.com/Primovist/CTJSIPTV-Release/releases)。

## 使用环境

服务端所在内网需要已经融合 IPTV 专网。已验证光猫拨号的软终端模式：IPTV 专网由光猫融合接入普通内网，服务端通过同一局域网接口访问互联网与江苏电信 IPTV 上游。也可能支持采用双 WAN 的 OpenWrt IPTV 专网融合方案，但需要在相应网络环境中实际验证。

macOS Apple Silicon 版本已经在实际 IPTV 网络中验证。Linux arm64 与 amd64 版本通过 GitHub Actions 编译和离线自测，但尚未进行真实 IPTV 网络验证；Linux arm64 版本可能可用于 OpenWrt，具体取决于设备架构、系统运行库和 IPTV 路由配置。

## 网页前端

服务端内置 `index.html` 单文件前端，访问服务根路径即可使用影视分类、搜索、详情与分集、播放地址、XMLTV EPG、直播列表、运行状态和 API 调试功能。前端默认通过同源 `/api/` 调用本服务，图片由服务端 `/api/image?url=...` 代理；外网只需反向代理服务根路径 `/`，不需要 `/iptv-image/`。

完整安装、Apache/Nginx HTTPS 配置、验证和故障排查见 [前端部署教程](./前端教程.md)。只在家庭局域网使用、没有域名和证书时，参见 [内网无证书部署教程](./内网部署教程.md)。前端不是运行服务端的必要条件，只使用 JSON/XMLTV/M3U8 API 的客户端可以忽略它。

## 构建和运行

```sh
swift build -c release
cp .build/release/ctjsiptv ./ctjsiptv
./ctjsiptv -c ./ctjsiptv.conf
```

程序为独立 Apple Silicon arm64 二进制，不依赖 Python 或 Node.js。配置文件使用 `KEY=VALUE`，环境变量会覆盖同名配置。

| 参数 | 必填 | 默认行为 |
|---|---:|---|
| `IPTV_USER_ID` | 是 | 无默认值 |
| `IPTV_PASSWORD` | 是 | 无默认值 |
| `IPTV_STB_ID` | 是 | 无默认值 |
| `IPTV_MAC` | 否 | 从选定网卡读取 |
| `IPTV_IP` | 否 | 从选定网卡读取 IPv4 |
| `NIC` | 否 | 根据到 IPTV Portal 的路由检测 |
| `LISTEN` | 否 | `[::]:8765`，双栈监听同一局域网的 IPv6 与 IPv4 |
| `RTP2HTTPD` | 否 | 未配置时两个 `/api/live/{protocol}/rtp2httpd` 接口返回配置错误 |

自动检测先查询到 `180.100.140.133` 的路由，再读取对应接口的 IPv4 和 MAC。显式配置 `NIC`、`IPTV_IP`、`IPTV_MAC` 时分别覆盖自动值。上游请求通过选定网卡发送。密码、Cookie 和 token 不应提交到仓库；真实配置文件仅保存在服务端。

默认 `LISTEN=[::]:8765` 使用 IPv6 wildcard：macOS 由 Network.framework 双栈监听，Linux 创建 IPv6 socket并设置 `IPV6_V6ONLY=0`，因此同一端口同时接受 IPv6 与 IPv4。若明确只需要 IPv4，可配置 `LISTEN=0.0.0.0:8765`；指定 IPv6 地址时必须使用方括号，例如 `LISTEN=[::1]:8765`。

## API

JSON API 成功响应使用 `code=0`，失败响应使用 `code=1001`。`/api/epg` 成功时直接输出 XMLTV；发生错误时仍返回统一 JSON 错误。所有对外接口统一使用 `/api` 前缀。

`/api/status` 会检查当前 IPTV 会话；没有有效会话时执行一次认证，并返回 `version`、`iptv_authenticated`、`session_active`、`nic`、`iptv_ip`、`rtp2httpd_enabled` 和 `rtp2httpd`。`rtp2httpd` 是供前端拼接或展示使用的规范化代理基址，未配置时为 `null`。认证失败时 `iptv_authenticated=false`，`auth_message` 提供不含密码和密钥的错误摘要。

```json
{
  "code": 0,
  "data": {
    "version": "0.5.1",
    "iptv_authenticated": true,
    "session_active": true,
    "nic": "en0",
    "iptv_ip": "192.168.1.100",
    "rtp2httpd_enabled": true,
    "rtp2httpd": "https://proxy.example.com"
  }
}
```

| Method | Path | 说明 |
|---|---|---|
| GET | `/api/health` | 服务状态 |
| GET | `/api/status` | 版本、网卡、IPTV IP 和实际鉴权状态 |
| GET | `/api/live/rtp` | RTP/UDP 直播源与可匹配的 HTTP 回看，输出 UTF-8 扩展 M3U8 |
| GET | `/api/live/http` | 去除鉴权查询串的官方 HTTP HLS 直播源及回看 |
| GET | `/api/live/rtp/rtp2httpd` | RTP/UDP 直播与回看均经 `RTP2HTTPD` 改写 |
| GET | `/api/live/http/rtp2httpd` | HTTP 直播与回看均经 `RTP2HTTPD` 改写 |
| GET | `/api/categories` | 全部已适配业务类型 |
| GET | `/api/categories/{type}` | 上游动态分类 |
| GET | `/api/filters/{type}` | 上游筛选标签 |
| GET | `/api/search?q=...&type=all&page=1&page_size=20&mode=auto` | 搜索点播内容 |
| GET | `/api/search/suggest?q=<拼音首字母>&page=1&page_size=20` | 根据调用方输入的拼音首字母搜索联想 |
| GET | `/api/vod?type=...&page=1&tags=...` | VOD 列表 |
| GET | `/api/vod/{id}?type=...` | 详情及分集 |
| GET | `/api/play/{id}?type=...&episode_id=...` | 播放地址 |
| GET | `/api/epg` | 输出内置频道清单的完整 XMLTV，默认过去 3 天至今天 |
| GET | `/api/epg/{channel}` | 按路径中的频道名输出单频道 XMLTV，默认过去 3 天至今天 |

业务类型：

| 后端 id | 前端名称 | 上游 type |
|---|---|---:|
| `movie` | 电影 | 1 |
| `drama` | 剧集 | 2 |
| `short_drama` | 短剧 | 2 |
| `anime` | 动漫 | 4 |
| `kids` | 少儿 | 7 |
| `variety` | 综艺 | 独立栏目协议 |
| `esports` | 电竞 | 独立栏目协议 |

综艺和电竞不使用 `getFilterData.jsp`：

```text
GET /api/categories/variety
GET /api/vod?type=variety&category=1003010B00
GET /api/vod?type=variety&category=album~P750214785438

GET /api/categories/esports
GET /api/vod?type=esports&category=1003010F030H&page=1&page_size=12
```

两类列表都返回统一的 `data.items[]`。其中 `id` 是服务端生成的自包含 ID，前端必须原样传递，不应拆分或自行构造：

```text
GET /api/vod/{id}
GET /api/play/{id}
GET /api/play/{id}?episode_id={详情返回的分集id}
```

综艺列表同时支持传统栏目和新版 album 聚合页。传统栏目来自 `ENT_ColumnList_P`，当前提供已验证的真人秀、情感、综合综艺、天天向上四个入口；`album~P750214785438` 使用抓包确认的动态签名规则请求综艺推荐页。详情会读取栏目资料并遍历上游分页，统一输出 `episodes`。电竞分类由上游实时返回；列表中的单集和系列通过 `program_type` 区分，详情阶段再解析为标准 frame805 内容编号。

`drama` 和 `short_drama` 使用独立的前端入口；短剧列表使用 `tags=duanju_dsj`。剧集和短剧详情会调用 `vod_detail.jsp`，从 `vodDetail.serieslist` 输出每集独立的 `program_code`、`content_code` 和 `episode_id`。

搜索支持 `type=all|movie|drama|short_drama|anime|kids`。`mode=auto` 时纯英文字母按拼音首字母搜索，其余输入按关键词搜索；也可显式指定 `mode=keyword` 或 `mode=pinyin`。`page_size` 范围为 1–50。搜索结果的 `id` 取上游 `matchID`，可直接用于 `/api/vod/{id}` 和 `/api/play/{id}`；`content_id` 单独保存上游 `programId`，两者不得互换。

```text
GET /api/search?q=<搜索内容>
GET /api/search?q=<搜索内容>&type=drama&page=1&page_size=20
GET /api/search?q=<拼音首字母>&mode=pinyin
GET /api/search/suggest?q=<拼音首字母>
```

上游搜索将动漫和少儿共同归入 `childList`，当前抓包没有证明二者存在独立搜索桶，因此 `type=anime` 与 `type=kids` 暂时查询同一上游结果集；每条结果仍依据真实 `typeDesc` 输出 `anime` 或 `kids`。`short_drama` 同样暂借电视剧桶，不能保证只返回短剧。

## 直播文本列表

四个直播接口都从认证阶段 `frameset_builder.jsp` 返回的 `jsSetConfig('Channel', ...)` 动态提取频道，不在运行时读取工作区 M3U。频道配置按上海日期缓存：启动或首次访问时没有当日缓存便立即获取，此后复用缓存；每天 0:00 主动刷新一次，失败时保留旧缓存并每 5 分钟重试。`tvg-name`、`tvg-id` 和显示名称继续使用项目提供的 `IPTV-IGMP-Home.m3u` 中 125 个频道名称的内置映射，上游名称仅用于匹配和选源；上游新增且无法映射的频道才保留整理后的上游名称。响应类型为 `application/vnd.apple.mpegurl; charset=utf-8`，并通过 `group-title` 按央视、江苏、卫视、广播、其他分组。

`/api/live/http` 只输出官方 HTTP HLS；直播 URL 删除 `AuthInfo`、账号、STB、session 等整段查询参数。`/api/live/rtp` 输出 RTP 及上游已有的 UDP 组播源。两份列表的频道名、`tvg-name` 和 `tvg-id` 完全一致：

```text
#EXTM3U
#EXTINF:-1 tvg-id="CCTV-1" tvg-name="CCTV-1" group-title="央视" source-protocol="http" catchup="default" catchup-source="http://gslb.itv.jsinfo.net:6060/00000002/.../index.m3u8?Playtype=1&Playseek=${(b)yyyyMMddHHmmss:utc}-${(e)yyyyMMddHHmmss:utc}",CCTV-1
http://gslb.itv.jsinfo.net:6060/00000002/.../index.m3u8
#EXTINF:-1 tvg-id="CCTV-1" tvg-name="CCTV-1" group-title="央视" source-protocol="rtp" catchup="default" catchup-source="http://gslb.itv.jsinfo.net:6060/00000002/.../index.m3u8?Playtype=1&Playseek=${(b)yyyyMMddHHmmss:utc}-${(e)yyyyMMddHHmmss:utc}",CCTV-1
rtp://239.x.x.x:8000?fcc=...
```

同一频道按 `8K > 4K/UHD > HD/1080P > 720P > 标清` 选择最高质量；HDR 与 SDR 视为相同分辨率等级。同一最高质量下保留各协议的一个来源，重复来源不输出。上游 `rtsp://` 按官方规则转换为无查询串的 `http://gslb.itv.jsinfo.net:6060/.../index.m3u8`；`udp://`、`rtp://` 保持原样；`igmp://` 规范化为 `rtp://` 并附加 `ChannelFCCServerAddr`。

回看只在同一标准频道、同一最高画质同时存在 HTTP 与 RTP/UDP 来源时写入。回看路径由 HTTP HLS 地址产生，`/00000001/` 规范化为 `/00000002/`，查询参数固定为 `Playtype=1&Playseek=${(b)yyyyMMddHHmmss:utc}-${(e)yyyyMMddHHmmss:utc}`。如果最高画质 RTP/UDP 找不到同画质 HTTP 来源，频道仍保留在 RTP 列表，但不写 `catchup` 与 `catchup-source`。

配置 `RTP2HTTPD=http://地址:端口` 或 `https://地址:端口` 后，可访问 `/api/live/rtp/rtp2httpd` 和 `/api/live/http/rtp2httpd`。两个接口会同时改写直播 URL 与 `catchup-source`，规则为 `<RTP2HTTPD>/<原协议>/<去掉 scheme:// 的原地址>`；不对剩余地址进行整体 URL Encode。例如：

```text
RTP2HTTPD=https://proxy.example.com

http://gslb.itv.jsinfo.net:6060/live/index.m3u8?x=1
→ https://proxy.example.com/http/gslb.itv.jsinfo.net:6060/live/index.m3u8?x=1

rtp://239.49.8.19:9614?fcc=180.100.72.185:15970
→ https://proxy.example.com/rtp/239.49.8.19:9614?fcc=180.100.72.185:15970
```

`RTP2HTTPD` 必须是无路径、无查询参数的 HTTP/HTTPS origin，末尾 `/` 会自动移除。此配置只影响两个 `/api/live/{protocol}/rtp2httpd` 接口，不会改变直连直播接口、VOD 播放或江苏电信上游认证。

播放示例：

```text
GET /api/play/SERI%2Fexample%40JSBC?type=drama&episode_id=11SPRO...
```

播放接口获取上游临时 `rtsp://` 后，按官方软终端规则转换为 `http://gslb.itv.jsinfo.net:6060/.../index.m3u8`，必须实时请求，不能长期缓存。

## 上游调用关系

认证：

```text
auth → CTCGetAuthInfo → DES Authenticator → uploadAuthInfo
→ getServiceList → UserGroupNMB/loadbalanced → funcportalauth.jsp
→ UserToken / JSESSIONID / X-Frame-SessionID
```

VOD：

```text
getFilterTags.jsp
→ getFilterData.jsp
→ getSearchResult_waterfall.jsp / getSuggestSearch.jsp
→ externalCode_Content.jsp
→ vod_detail.jsp（剧集、短剧分集）
→ getPlayURL.jsp
→ RTSP
```

综艺：

```text
ENT_column_list.jsp
→ ENT_ColumnList_P
→ getColumnInfo.jsp
→ get_vod_tri_list_yl.jsp
→ vod_url_play.jsp
→ RTSP
```

新版综艺 album 请求使用 `albumPageDataQuery`，签名字段顺序及签名键从上游 `interface.js` 动态解析并实时生成，不在代码中保存抓包值。签名时间、流水号、账号和 STB ID 均来自当前会话/配置。响应中的 `activityUrl` 会转换为本地自包含 ID；无法映射到标准详情或传统栏目的第三方活动项不会作为可播放节目输出。

电竞：

```text
get_column_list4game.jsp
→ get_vod_list4tiyu.jsp
→ vod_detail_4k.jsp
→ movieDetail.html / dramaDetail.html
→ externalCode_Content.jsp
→ vod_detail.jsp（系列内容）
→ getPlayURL.jsp
→ RTSP
```

已确认的 Portal 上游主机为 `180.100.140.133:33200`。

| Endpoint | 用途 | 主要参数 | 本地 API |
|---|---|---|---|
| `/iptvepg/frame805/epg30/iptv/jsp/getFilterTags.jsp` | 分类和标签 | `apikey,secretkey,sp,type,userId,softterminalflag` | categories/filters |
| `/iptvepg/frame805/epg30/iptv/jsp/getFilterData.jsp` | VOD 列表 | `type,tags,page,pageSize,date` | `/api/vod` |
| `/iptvepg/frame320/datas/getSearchResult_waterfall.jsp` | VOD 搜索 | `contentstr,curpage,pageSize,TYPE,searchCategory,serchTypeList,areaFlag` | `/api/search` |
| `/iptvepg/frame320/datas/getSuggestSearch.jsp` | 拼音搜索联想 | `searchPinYin,pageIndex,pageSize,searchCategory` | `/api/search/suggest` |
| `/iptvepg/frame805/epg30/iptv/jsp/externalCode_Content.jsp` | 内容详情映射 | `code=contentId` | `/api/vod/{id}` |
| `/iptvepg/frame805/epg30/iptv/jsp/vod_detail.jsp` | 剧集分集 | `programcode,columncode,ajax=1` | `/api/vod/{id}` |
| `/iptvepg/frame805/epg30/iptv/jsp/getPlayURL.jsp` | 播放地址 | `programcode,breakpoint,standardflag,playmediaservice` | `/api/play/{id}` |
| `/iptvepg/function/frameset_builder.jsp` | 认证初始化及直播频道配置 | `BUILD_ACTION,MAIN_WIN_SRC,NEED_UPDATE_STB` | `/api/live/{rtp,http}` |
| `/iptvepg/frame368/ENT_column_list.jsp` | 综艺栏目列表 | `columncode` | `/api/vod?type=variety` |
| `/iptvepg/frame224/4kDetail_datas/getColumnInfo.jsp` | 综艺栏目详情 | `cid,pid` | `/api/vod/{id}` |
| `/iptvepg/frame224/4kDetail_datas/get_vod_tri_list_yl.jsp` | 综艺分期列表 | `cid,programcode,pageindex,pagesize,vodCount,totalpage` | `/api/vod/{id}` |
| `/iptvepg/frame224/4kDetail_datas/vod_url_play.jsp` | 综艺播放地址 | `columncode,programcode,ajax,ptype` | `/api/play/{id}` |
| `/iptvepg/frame326/datas/get_column_list4game.jsp` | 电竞动态分类 | `columnId,colIdx` | `/api/categories/esports` |
| `/iptvepg/frame326/datas/get_vod_list4tiyu.jsp` | 电竞列表 | `cid,pidx,pnum,tri` | `/api/vod?type=esports` |
| `/iptvepg/frame224/vod_detail_4k.jsp` | 电竞 ID 转标准详情编号 | `columnid,programid,programtype,ContentID,CategoryID` | 电竞详情/播放内部调用 |
| `/iptvepg/frame1194/datas/zxin/channel_list.jsp` | 动态频道目录 | `columncode,framecode,stbtype,ajax` | `/api/epg` 内部频道匹配 |
| `/iptvepg/frame1194/datas/zxin/prevue_list.jsp` | 分频道 EPG | `channelcode,dateindex,framecode,stbtype,recommpara,ajax` | `/api/epg` |

字段关系：列表的 `contentId` 通过 `externalCode_Content.jsp` 映射到详情；搜索响应的 `matchID` 对应详情/播放使用的 `PROG/...` 或 `SERI/...` 标识，`programId` 是另一套内容编号；`programcode` 和 `contentcode` 是不同字段；播放使用 `programcode`，Referer 携带 `contentcode`。EPG 的 `channelcode` 与 VOD 内容 ID 无关。

## 当前状态

- 电影：列表、详情、播放已验证。
- 剧集：25 集分集解析和播放地址获取已验证。
- 短剧：30 集分集解析和播放地址获取已验证。
- 动漫、少儿：列表可用，独立分集协议尚未取得足够真实样本。
- 综艺：四个传统顶层入口及新版 album 推荐页均已适配；支持动态栏目、栏目详情、完整分期和播放。album 中无法映射为标准 VOD 的第三方活动项会被过滤。
- 电竞：动态分类、分页列表、标准单集详情与播放已适配；跳转独立游戏专区的部分系列可能返回“详情编号无法识别”。
- 搜索：已按 2026-09-12 抓包实现中文关键词、拼音首字母、分页、电影/剧集/少儿桶及联想词解析；本版本未在部署端进行在线验证。
- 直播：已从真实 `frameset_builder.jsp` 响应解析 211 条上游配置，按频道名和清晰度整理为 129 个频道；HTTP 与 RTP/UDP 分别通过独立 M3U8 接口输出，并按同频道同画质关系添加回看。其中 92 个频道采用参考 M3U 的标准名称，37 个参考文件中不存在的上游频道保留整理后的上游名称。
- EPG：抓包已验证 CCTV1-HD（`channelId=901`）和江苏综艺HD（`channelId=922`）分频道节目单；当前实现输出 UTF-8 XMLTV，其他频道按同一动态目录协议适配，尚待部署端全量验证。
- 播放接口：VOD 原始 RTSP 仍转换为官方 `gslb.itv.jsinfo.net:6060` HLS，不做服务端转码；`RTP2HTTPD` 仅供两个直播代理接口改写直播与回看地址。

## 播放兼容性验证

已用少儿和动漫内容验证官方 HLS 链路：官方 `index.m3u8`、媒体清单及 TS 分片均可正常返回，IINA 可以正常播放。

Safari 直接打开同一 HLS 地址时出现黑屏，但地址本身和上游播放链路正常，判断为 Safari 直接打开该类 HLS 的播放器/解码兼容性问题，不是认证失败。网页前端应使用 HTML5 `video` 播放，必要时配合 HLS 播放器库；调试时可使用 IINA 或 VLC 验证。

HLS 地址带有临时 `AuthInfo`、`usersessionid` 等鉴权参数，必须通过 `/api/play/{id}` 实时获取，不能长期缓存。复制地址时不要把查询参数中的 `&` 改成 `\\&`。

## XMLTV EPG

抓包确认的上游调用链为：

```text
GET /iptvepg/frame1194/datas/zxin/channel_list.jsp
  columncode=020A,0208,0204,0205,0207,0206,020B
  framecode=frame1194&stbtype=sdr&ajax=1
        ↓ channelAllList[]
channelname + tvodcode/channelcode + realmixno/globeid
        ↓
GET /iptvepg/frame1194/datas/zxin/prevue_list.jsp
  channelcode=<tvodcode 中可用的编号>
  dateindex=0...6
  framecode=frame1194&stbtype=sdr
  recommpara=userId=<账号>&channelId=<realmixno>&num=6
  ajax=1
        ↓ channelPrevue[]
prevuename + begintime + endtime + prevuecode
```

`channel_list.jsp` 的抓包响应包含 182 条频道。部分频道的 `tvodcode` 含多个以 `|` 分隔的候选编号，服务端会按上游 `channel.js` 的逻辑依次尝试。`dateindex=0` 是今天，`1...6` 是过去第 1 至第 6 天；上游脚本未提供未来日期索引。

服务端不读取也不修改 M3U 文件。播放器把 M3U 中的频道名放入请求路径，服务端直接用该名称匹配上游动态频道目录。XMLTV 的 `<channel id>` 使用请求中的频道名；每个 CCTV 频道会同时输出短横线、无短横线、`-HD`、上游原名和中文俗称等多个 `<display-name>`，以兼容不同播放器的自动关联规则。匹配会忽略央视频道编号后的栏目描述以及连字符、空格、`HD`、`高清`、`SDR`、`频道` 等差异，并处理 `中央一套` 等央视别名。因此 `CCTV1`、`CCTV-1`、`CCTV1-HD`、`CCTV-1 综合` 和 `中央一套` 均可匹配上游 `CCTV1综合-HD`；`CCTV-15`、`CCTV15`、`CCTV-音乐` 和 `央视音乐` 均匹配上游 CCTV-15/音乐频道。

单频道请求示例：

```text
GET /api/epg/CCTV-1
GET /api/epg/CCTV1
GET /api/epg/%E4%B8%AD%E5%A4%AE%E4%B8%80%E5%A5%97
GET /api/epg/CCTV-1?date=2026-09-12
GET /api/epg/CCTV-1?days=7
GET /api/epg
```

`/api/epg/{channel}` 每次只返回请求频道的 `<channel>` 和 `<programme>`；`/api/epg` 返回已确认匹配的电视频道，以及江苏新闻广播、江苏交通广播网、江苏经典流行音乐广播、江苏音乐广播、金陵之声、江苏新闻综合广播、江苏文艺广播、江苏故事广播、江苏财经广播、江苏健康广播。默认日期范围是过去第 3 天至今天（上游 `dateindex=3,2,1,0`）。`date` 可指定今天至过去 6 天中的某一天，`days=7` 可输出今天及过去 6 天。

完整 EPG 把“频道 × 日期”请求按 24 个一批限制并发。服务启动后如内存中没有当天的完整 EPG，会立即在后台生成；生成期间 HTTP 服务仍可接受请求，同一个 EPG 请求只会执行一份上游取数任务。此后程序按 `Asia/Shanghai` 时区每天 00:00 清除旧 EPG 缓存并重新生成 `/api/epg`，失败时每 5 分钟重试。完整 XMLTV 按自然日缓存，频道目录和分频道节目请求还保留 5 分钟内部缓存，并会在每日刷新前一并失效。EPG 接口统一返回标准 XMLTV。

启动抓包分析：APK 启动时先调用加密的 `apkStartupRequest` 初始化软终端；随后明文启动统计请求携带 `itv_no`、`stb_id`、`stb_mac` 和终端信息，心跳请求携带 `virtualAccount`、`itvAccount` 和 MAC。抓包可以确认 STBID 在启动阶段被使用，但由于初始化请求体和响应体加密，不能仅凭该抓包确定 STBID 是本地生成还是服务端下发。

## 验收

```sh
./ctjsiptv -h
./ctjsiptv --self-test
```

项目版本记录在 `VERSION`，并与程序内的 `appVersion` 保持一致。主分支推送仅在版本号提升，或同版本上一次构建失败时运行三平台构建；成功后自动创建对应的 `v<版本号>` GitHub Release。准备新提交前应先确定本次版本号。

Linux 使用 Foundation、FoundationNetworking 和 Glibc 提供轻量 POSIX HTTP/1.1 服务，不包含 SwiftNIO。服务端按连接读取请求头及 Content-Length 正文，限制单请求最大 64 KB，设置收发超时，并处理 TCP 分片、部分写入和 SIGPIPE。macOS 继续使用 Network.framework。
