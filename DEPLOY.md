# 部署到服务器

本工具是一个**零依赖纯前端 + 一个 Node 小服务**的项目。把文件放到服务器上后，`index.html` 怎么用到 `data.json`，取决于你用的是哪种托管方式。下面两种都支持，按你手头的服务器选一种。

---

## 先说结论（最重要的三句话）

1. **浏览器永远不能直接写服务器上的文件**（这是浏览器安全限制，任何网站都一样）。所以"网页把数据存进服务器 data.json"这件事，必须有一个后端来中转——本项目自带的 `serve.js` 就是干这个的。
2. **最省事、功能最全的部署**：在服务器上装好 Node，把整个项目目录传上去，用 `node serve.js` 把服务跑起来（用进程守护如 pm2 / systemd）。页面会自动探到 `/api/data` 接口，**读和写都直接落在服务器 data.json 上**，跟你在本机用「启动服务.bat」完全一样。
3. **只传静态文件、不跑 Node**（nginx / Apache / 对象存储 / GitHub Pages 之类）：页面能**只读**打开 `data.json`（能看、能测算），但**改了保存不回去**。要保存必须改用方案 1，或本机编辑后把 `data.json` 重新上传覆盖。

---

## 方案 A：跑 serve.js（推荐，读写都能落盘）

### 1. 上传文件
把项目整个目录（含 `serve.js`、`index.html`、`data.json`、`js/`、`styles.css` 等）传到服务器任意目录，例如 `/opt/logi-quote`。

### 2. 启动（对外监听）
```bash
cd /opt/logi-quote
# HOST=0.0.0.0 让服务监听所有网卡（默认只监听 127.0.0.1，只有本机能访问）
# PORT 可改端口；DATA_FILE 可把数据文件指到别处（默认与本目录同级的 data.json）
HOST=0.0.0.0 PORT=5173 node serve.js
```
打开 `http://服务器IP:5173/` 即可。`index.html` 启动时会请求 `/api/health`，拿到 `{ok:true}` 就自动切到"直连 data.json"模式，之后所有增删改都通过 `POST /api/data` 整份写回服务器上的 `data.json`。

### 3. （强烈建议）加一道令牌，别让陌生人改写你的数据
默认情况下，任何能访问该端口的人都能读写 `data.json`。对外暴露时请加上：
```bash
HOST=0.0.0.0 PORT=5173 \
ALLOWED_HOSTS=报价.example.com,1.2.3.4:5173 \
DATA_TOKEN=$(openssl rand -hex 16) \
node serve.js
```
- `ALLOWED_HOSTS`：允许访问 `/api/*` 的 Host（用 `*` 表示不校验；多个用逗号分隔，可带 `:端口`）。这挡掉"别的网站借你浏览器的手打你服务器"的 DNS rebinding 攻击。
- `DATA_TOKEN`：设了之后，`/api/*` 必须带 `Authorization: Bearer <token>`。
  - 页面侧已配套支持：第一次打开时带上 `?token=xxx`（例如 `http://报价.example.com:5173/?token=abc...`），页面会把令牌记进浏览器 localStorage，之后免带。
  - 不带令牌打开时，接口返回 401，页面会自动降级为**只读模式**（能看不能改），不会白屏。

### 4. 用反向代理 + 进程守护（生产建议）
直接用 `node serve.js` 跑在前台不适合长期运行。推荐：
```bash
# 用 pm2 守护（自动重启、崩溃拉起）
npm i -g pm2
pm2 start serve.js --name logi-quote -- --HOST 0.0.0.0 --PORT 5173
pm2 save
```
或前面挡一层 nginx（顺手加 HTTPS 和 Basic Auth）：
```nginx
location / {
    proxy_pass http://127.0.0.1:5173;
    proxy_set_header Host $host;
}
```
> 注意：若经 nginx 反代，`ALLOWED_HOSTS` 要填你对外暴露的域名；`DATA_TOKEN` 仍生效。`serve.js` 还支持 `DATA_FILE=/var/lib/logi/data.json` 把数据挪到更合适的位置。

### 5. 数据备份 / 迁移
- `data.json` 就是数据库本体，直接复制即可备份。
- 换机器：把整个目录（尤其 `data.json`）整体拷过去，重启 `serve.js` 即可。
- 应用内「导出备份 / 从备份恢复」同样可用。

---

## 方案 B：纯静态托管（只读）

如果你只有静态文件空间（nginx 静态目录、对象存储、GitHub Pages 等），**不跑 Node**：

1. 把项目文件传上去，确保 `index.html` 和 `data.json` 在同一级目录可访问到 `http(s)://你的域名/data.json`。
2. 打开页面。`/api/health` 探不到（没有后端），页面自动改为 GET `data.json` 加载，进入**只读模式**：
   - 顶栏显示 `data.json · 只读（无写入接口）`，黄灯。
   - 所有数据能看、能测算；但任何"保存"都不会落到服务器。
3. 要保存修改，二选一：
   - 改用**方案 A** 在该服务器跑 `serve.js`；
   - 或在本机用「启动服务.bat」打开、改好，再把新 `data.json` 上传覆盖服务器那份。

> 纯静态方案下，建议不要在公开只读站点放敏感报价；任何能拿到 URL 的人都能读 `data.json`。如需隐藏读取，请用方案 A 并叠加 nginx 鉴权，或把 `data.json` 放到非公开路径。

---

## 三种模式的判定逻辑（页面启动时自动走）

`js/app.js` 的 `persistenceFactories` 按顺序尝试，命中第一个能读到的：

| 顺序 | 模式 | 触发条件 | 读 | 写 |
|---|---|---|---|---|
| 1 | `http` 直连 | `/api/health` 返回 `{ok:true}`（serve.js 在跑） | ✅ 服务器 data.json | ✅ 服务器 data.json |
| 2 | `static` 只读 | 没有 `/api/health`，但能 GET 到 `/data.json` | ✅ 服务器 data.json | ❌ 浏览器写不进服务器 |
| 3 | `fs` 选本地文件 | 浏览器支持 File System Access API（本机 Chrome/Edge） | ✅ 本地选的文件 | ✅ 本地选的文件 |
| 4 | `degrade` 浏览器存储 | 以上都不行 | ✅ IndexedDB | ✅ IndexedDB（清缓存会丢） |

部署到远程服务器时，第 1、2 档会优先命中（远程 https 页面下 `fs` 档去"选本地文件"反而是错的，所以排在只读之后）。

---

## 环境变量速查（serve.js）

| 变量 | 默认 | 说明 |
|---|---|---|
| `HOST` | `127.0.0.1` | 监听地址；对外用 `0.0.0.0` |
| `PORT` | `5173` | 监听端口 |
| `ALLOWED_HOSTS` | 空（只认本机） | 允许访问 `/api/*` 的 Host，`*` 表示不校验 |
| `DATA_TOKEN` | 空 | 设了之后 `/api/*` 需 `Authorization: Bearer` |
| `DATA_FILE` | 与本文件同目录的 `data.json` | 数据文件绝对路径，可挪到别处 |

---

## 常见问题

- **Q：放到服务器后页面空白 / 一直是"本地存储"？**
  A：多半是 `/api/health` 探不到。确认 `serve.js` 确实在跑、端口通、且 `HOST=0.0.0.0`（默认只监听本机，外部访问不到）。若你只传了静态文件没跑 Node，那会进入只读模式（黄灯、能看不能存），这是预期行为。

- **Q：设了 DATA_TOKEN 后页面打不开 / 一直只读？**
  A：用 `http://域名/?token=你的令牌` 打开一次，页面会记住它。之后正常访问即可。

- **Q：能部署在子路径（如 example.com/logi/）吗？**
  A：当前 `/api/*` 和 `/data.json` 按根路径探测，子路径部署需要反向代理把前缀剥掉再转给 serve.js，或自行改 `fsio.js` 里的探测基址。建议直接部署在域名根或子域名。

- **Q：多人在同一份 data.json 上改会冲突吗？**
  A：后端是"后写覆盖"（和本机文件语义一致），不做并发锁。个人/小团队顺序使用没问题；并发高频同改可能互相覆盖。
