---
date: 2026-04-13T06:25:34+08:00
lastmod: 2026-04-13T06:25:34+08:00
title: 用 OpenList + emby2Alist + tmdb-proxy 搭一套可用的 Jellyfin
categories:
  - 折腾笔记
tags:
  - openlist
  - Jellyfin
  - 影视
---

## 用 OpenList + emby2Alist + tmdb-proxy 搭一套可用的 Jellyfin

这套方案一次解决两件事：

- TMDb 元数据访问不稳定
- `.strm` 播放时希望走 OpenList 的 `302` 直链，而不是让 Jellyfin 自己探测和中转

最终链路分成两段：

```text
元数据:
Jellyfin -> mitmproxy -> 你的 tmdb-proxy 域名 -> TMDb

播放:
客户端 -> emby2Alist -> Jellyfin
                     \-> OpenList -> 302 -> 真实网盘地址
```

## 一、先部署 TMDb 代理

项目使用 [`imaliang/tmdb-proxy`](https://github.com/imaliang/tmdb-proxy)。

部署步骤很简单：

1. 打开仓库主页  
   <https://github.com/imaliang/tmdb-proxy>
2. 点击 README 里的 `Deploy with Vercel`
3. 在 Vercel 创建项目
4. 绑定你自己的域名，例如：

```text
tmdb.example.com
```

说明：

- 根路径 `/` 跳转到 GitHub 项目页是正常现象
- 真正代理的是 API 路径和 `/t/p/*` 图片路径

## 二、在 OpenList 里挂载网盘并生成本地 `.strm`

这一段的目标是：让 OpenList 负责出真实下载地址，Jellyfin 只吃本地生成的 `.strm` 文件。

### 1. 普通网盘挂载

先在 OpenList 里正常挂载你的网盘。  
对需要直链播放的挂载，代理模式选择：

```text
302
```

### 2. 额外添加一个 `Strm` 驱动

再新增一个 `Strm` 驱动挂载，用来批量生成本地 `.strm` 文件。

建议这样填写：

- 挂载路径：`/strm`
- 前缀：默认 ` /d ` 即可
- 勾选“生成本地文件”

“生成本地文件保存位置”这项需要注意：

- 如果 OpenList 是裸机运行，直接填宿主机目录，例如：

```text
/opt/jellyfin/strm
```

- 如果 OpenList 是 Docker 运行，先把宿主机目录挂进 OpenList 容器，再在 OpenList 后台填写容器内路径。  
  例如宿主机挂载：

```yaml
- /opt/jellyfin/strm:/strm-local
```

此时在 OpenList 后台填写：

```text
/strm-local
```

不要把一个“宿主机路径”直接填进容器里的 OpenList 后台，否则 OpenList 容器看不到这个目录。

### 3. 生成本地 `.strm`

配置好以后，到 OpenList 后台执行索引和本地 `.strm` 生成。  
生成完成后，宿主机目录里会出现一批本地 `.strm` 文件。

后面 Jellyfin 就直接读取这个目录。

## 三、准备宿主机目录

建议至少准备这些目录：

```bash
mkdir -p /opt/jellyfin/config
mkdir -p /opt/jellyfin/cache
mkdir -p /opt/jellyfin/media
mkdir -p /opt/jellyfin/strm
mkdir -p /opt/mitmproxy
mkdir -p /opt/emby2alist/embyCache
mkdir -p /opt/emby2alist/log
```

这些目录分别用于：

- Jellyfin 配置
- Jellyfin 缓存
- 本地媒体
- OpenList 生成的本地 `.strm`
- mitmproxy 证书和配置
- emby2Alist 缓存和日志

## 四、部署 Jellyfin + mitmproxy + emby2Alist

这一版直接把三部分放进同一个 `docker-compose.yml`：

- `mitmproxy`：只负责 Jellyfin 出站 TMDb 代理
- `jellyfin`：媒体服务本体
- `emby2Alist`：只负责入站播放请求的 `302` 直链

建议保留双入口：

- `8096`：原始 Jellyfin 入口，不走 `302`
- `8091`：`emby2Alist` 入口，播放时走 OpenList `302`

将下面内容保存为 `docker-compose.yml`：

```yaml
services:
  mitmproxy:
    image: mitmproxy/mitmproxy:latest
    container_name: mitmproxy
    restart: unless-stopped
    expose:
      - "8080"
    ports:
      - 8081:8081
    command:
      - mitmweb
      - --listen-host
      - 0.0.0.0
      - --listen-port
      - "8080"
      - --set
      - confdir=/home/mitmproxy/.mitmproxy
      - --set
      - web_host=0.0.0.0
      - --set
      - web_port=8081
      - --set
      - web_open_browser=false
      - --set
      - web_password=${MITMWEB_PASSWORD}
      - --set
      - connection_strategy=lazy
      - --set
      - allow_hosts=^(api\.themoviedb\.org|image\.tmdb\.org):443$
      - --set
      - map_remote=|^https://api\.themoviedb\.org/|https://${TMDB_PROXY_HOST}/
      - --set
      - map_remote=|^https://image\.tmdb\.org/|https://${TMDB_PROXY_HOST}/
    volumes:
      - /opt/mitmproxy:/home/mitmproxy/.mitmproxy

  jellyfin:
    image: jellyfin/jellyfin
    container_name: jellyfin
    user: 0:0
    depends_on:
      - mitmproxy
    ports:
      - 8096:8096/tcp
    environment:
      HTTP_PROXY: http://mitmproxy:8080
      HTTPS_PROXY: http://mitmproxy:8080
      NO_PROXY: localhost,127.0.0.1,::1,host.docker.internal,mitmproxy
      http_proxy: http://mitmproxy:8080
      https_proxy: http://mitmproxy:8080
      no_proxy: localhost,127.0.0.1,::1,host.docker.internal,mitmproxy
    volumes:
      - /opt/jellyfin/config:/config
      - /opt/jellyfin/cache:/cache
      - /opt/jellyfin/media:/media:ro
      - /opt/jellyfin/strm:/strm:ro
      - /opt/mitmproxy:/mitmproxy-ca:ro
    restart: unless-stopped
    extra_hosts:
      - host.docker.internal:host-gateway
    entrypoint:
      - /bin/sh
      - -lc
      - |
        until [ -f /mitmproxy-ca/mitmproxy-ca-cert.pem ]; do
          echo "waiting for mitmproxy CA..."
          sleep 2
        done
        cp /mitmproxy-ca/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy-ca-cert.crt
        update-ca-certificates
        exec /jellyfin/jellyfin

  emby2alist:
    image: nginx:1.27.1
    container_name: nginx-emby
    network_mode: host
    volumes:
      - /opt/emby2alist/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - /opt/emby2alist/nginx/conf.d:/etc/nginx/conf.d:ro
      - /opt/emby2alist/embyCache:/var/cache/nginx/emby
      - /opt/emby2alist/log:/var/log/nginx
    restart: always
```

同目录再创建 `.env`：

```dotenv
TMDB_PROXY_HOST=tmdb.example.com
MITMWEB_PASSWORD=替换成强密码
```

启动：

```bash
docker compose up -d --force-recreate
```

补充说明：

- 这里保留 Jellyfin 原始端口 `8096`
- 访问 `8096` 时是原始 Jellyfin，不走 `302`
- 访问 `8091` 时才会经过 `emby2Alist`
- `emby2Alist` 这里采用的是上游 `docker-compose.yml` 默认推荐的 `host` 网络方式
- 如果你坚持用桥接网络，也可以改成 `ports: - 8091:8091`，但配置会比 `host` 网络更绕

### Jellyfin 媒体库建议

此时在 Jellyfin 里至少加两个库来源：

- `/media`：你的本地媒体
- `/strm`：OpenList 生成的本地 `.strm` 目录

如果你的网盘内容主要依赖 `.strm`，就直接给 `/strm` 单独建库。

## 五、配置 emby2Alist

`emby2Alist` 用来拦截 Jellyfin 的播放请求，并把视频请求改写成 OpenList 的 `302` 直链。

项目地址：

- <https://github.com/bpking1/embyExternalUrl/tree/main/emby2Alist>

上面 compose 已经把 `emby2Alist` 服务写进去了。  
你现在要做的是把上游发布包里的 `nginx/` 配置目录放到宿主机，再修改其中几个关键文件。

### 1. 下载官方发布包

按上游 README 的方式下载并解压：

```bash
wget https://codeload.github.com/bpking1/embyExternalUrl/tar.gz/refs/tags/v0.4.8
mkdir -p /opt/emby2alist
tar -xzvf ./embyExternalUrl-0.4.8.tar.gz -C /opt/emby2alist --strip-components=2 embyExternalUrl-0.4.8/emby2Alist
```

> 注意版本号，这里采用当下最新的v0.4.8

解压后，重点是里面的 `nginx/` 目录。

### 2. 准备目录结构

建议整理成：

```text
/opt/emby2alist/
├── nginx/
│   ├── nginx.conf
│   └── conf.d/
├── embyCache/
└── log/
```

### 3. 修改 `constant.js`

文件：

```text
/opt/emby2alist/nginx/conf.d/constant.js
```

按上面这份 compose，`emby2Alist` 使用 `host` 网络，最稳妥的写法就是直接回源宿主机上的 Jellyfin：

```js
const embyHost = "http://127.0.0.1:8096";
const embyApiKey = "替换成你的Jellyfin_API_Key";
const mediaMountPath = [""];
```

说明：

- 这里虽然变量名叫 `embyHost`，但 Jellyfin 也是用这套
- `mediaMountPath = [""]` 适合主要使用 `.strm` 的场景

### 4. 修改 `constant-mount.js`

文件：

```text
/opt/emby2alist/nginx/conf.d/config/constant-mount.js
```

最少改这些：

```js
const alistAddr = "http://127.0.0.1:5244";
const alistToken = "替换成你的OpenList_Token";
const alistSignEnable = false;
const alistSignExpireTime = 12;
const alistPublicAddr = "https://openlist.example.com";
const clientSelfAlistRule = [];
const redirectCheckEnable = false;
const fallbackUseOriginal = true;
```

说明：

- `alistAddr` 是 `emby2Alist` 访问 OpenList API 的地址
- `alistPublicAddr` 是客户端真正去请求直链时使用的公网地址
- `fallbackUseOriginal = true` 表示取直链失败时回退到原始 Jellyfin 链路

### 5. 修改 `constant-pro.js`

文件：

```text
/opt/emby2alist/nginx/conf.d/config/constant-pro.js
```

`mediaPathMapping` 是这套方案里最容易配错的一项。  
它的作用不是“随便改路径”，而是把 `emby2Alist` 拿到的路径改写成 OpenList API 真正能识别的资源路径。

你要先分清楚当前 `.strm` 里写的到底是哪一种：

### 情况 1：`.strm` 里是相对路径

例如：

```text
/d/pan10086/影视资源/电影/xxx.mkv
```

这种情况下，用：

```js
const mediaPathMapping = [
  [0, 1, "/d/", "/"],
];
```

说明：

- `0`：普通字符串替换
- `1`：只处理 `.strm` 内部以 `/` 开头的相对路径
- 把 `/d/xxx` 改成 `/xxx`

### 情况 2：`.strm` 里是完整远程链接

例如：

```text
http://openlist.example.com:5244/d/pan10086/影视资源/电影/xxx.mkv
```

这种情况下，用：

```js
const mediaPathMapping = [
  [0, 2, "http://openlist.example.com:5244/d/", "/"],
];
```

说明：

- `2`：只处理 `.strm` 内部的远程链接
- 这条规则会把：

```text
http://openlist.example.com:5244/d/pan10086/影视资源/电影/xxx.mkv
```

改成：

```text
/pan10086/影视资源/电影/xxx.mkv
```

这通常才是 OpenList `fs/get` 之类接口更容易识别的库路径。

### 情况 3：Jellyfin 取到的是本地 `.strm` 文件路径

例如日志里看到的是：

```text
/media/移动云盘/影视资源/电影/B/某电影.strm
```

而你真正想让它去查的是：

```text
/strm/电影/B/某电影.strm
```

这种情况不是优先推荐方案，只适合作为临时补丁。可以这样写：

```js
const mediaPathMapping = [
  [0, 3, "/media/移动云盘/影视资源/", "/strm/"],
];
```

说明：

- `3`：处理当前媒体文件路径本身
- 这类规则适合排错时临时修正路径
- 更好的做法仍然是让 Jellyfin 从源头就直接扫描正确的 `.strm` 目录

### 当前这套方案的推荐写法

如果你的 `.strm` 文件内容本身就是这种远程地址：

```text
http://openlist.example.com:5244/d/pan10086/影视资源/电影/B/某电影.mkv
```

那么推荐直接这样配：

```js
const mediaPathMapping = [
  [0, 2, "http://openlist.example.com:5244/d/", "/"],
];
```

这也是最贴近当前验证通过的逻辑：

- Jellyfin 拿到远程 `.strm` 链接
- `emby2Alist` 把 `/d/...` 远程下载链接映射成 OpenList 库路径
- OpenList 返回真实下载地址
- `emby2Alist` 再把客户端 `302` 到最终网盘直链

## 六、启动

配置改完后，直接启动：

```bash
docker compose up -d --force-recreate
```

双入口模式下，对外会同时保留两个入口：

```text
客户端 -> emby2Alist:8091 -> Jellyfin:8096
```

以及：

```text
客户端 -> Jellyfin:8096
```

区别是：

- 访问 `8091`：播放请求会进入 `emby2Alist`，可以触发 `302` 直链
- 访问 `8096`：完全绕过 `emby2Alist`，就是原始 Jellyfin

## 七、验证

### 1. 验证 TMDb 代理

查看日志：

```bash
docker logs -f jellyfin
docker logs -f mitmproxy
```

如果 Jellyfin 启动时出现：

```text
waiting for mitmproxy CA...
Updating certificates in /etc/ssl/certs...
1 added, 0 removed; done.
```

说明 TMDb 代理证书已导入成功。

再去 Jellyfin 手动刷新一个媒体元数据，`mitmweb` 中应该能看到：

- `api.themoviedb.org`
- `image.tmdb.org`

### 2. 验证 `.strm` 直链

查看 `emby2Alist` 日志：

```bash
docker logs -f nginx-emby
```

然后从客户端播放一个来自 `/strm` 库的媒体。

正常情况是：

- 页面和库信息仍然由 Jellyfin 提供
- 视频播放请求被 `emby2Alist` 接管
- `emby2Alist` 调 OpenList API
- OpenList 返回 `302`
- 客户端最终直连真实网盘地址

## 八、关键点

- OpenList 网盘挂载的代理模式选 `302`
- 生成本地 `.strm` 的 `Strm` 驱动挂载路径用 `/strm`
- 前缀可以先用默认 `/d`
- 本地 `.strm` 保存目录要同时让 OpenList 和 Jellyfin 都能看到
- Jellyfin 的 `.strm` 库路径就是你挂进去的 `/strm`
- 外部播放入口最终应走 `emby2Alist`，不要直接打 Jellyfin `8096`

## 久、题外话

**网易爆米花安卓APP**经过我本人的实测应该是不支持纯粹ipv6地址jellyfin服务端的连接，最后我是通过手机和Nas同时安装Easytier解决了这个问题。不得不说，真够复杂的，配置完了我也能删除了，研究意义大于实用价值

## 参考链接

- `imaliang/tmdb-proxy`：<https://github.com/imaliang/tmdb-proxy>
- `bpking1/embyExternalUrl`：<https://github.com/bpking1/embyExternalUrl>
- `emby2Alist`：<https://github.com/bpking1/embyExternalUrl/tree/main/emby2Alist>
- Vercel 域名文档：<https://vercel.com/docs/domains/working-with-domains/add-a-domain>
- Jellyfin 元数据文档：<https://jellyfin.org/docs/general/server/metadata/>
- mitmproxy 文档：<https://docs.mitmproxy.org/stable/>
