---
name: railway-xray-node
description: >
  把 Railway 云主机变成一个 xray 代理节点：xray 提供 VMess（给 Surge）与 VLESS 两条
  WebSocket 通道，Node 前门负责发 Surge 订阅、发 VLESS 链接、根路径 400 伪装并把 WS
  流量反代给本机 xray。含生成钥匙、5 个部署文件全文、本地测试、两条部署路线
  （CLI 直传 / GitHub 集成）、验收清单、换区、客户端导入对照与费用生命周期说明。
  Use when the user wants to deploy / redeploy / move / troubleshoot an xray
  (VMess/VLESS) proxy node on Railway, import it into Surge/Clash/v2rayNG,
  change its region, or check its Railway usage and trial/free credit.
---

# Railway 变成 xray 代理节点

在 Railway 上部署一个自用代理节点的方法：生成两把钥匙、准备 5 个部署文件、用 Railway CLI 直传（或 GitHub 集成）部署、设环境变量、生成域名、验收后导入客户端使用。

## 架构（一句话版）

- Railway 服务里跑两个进程：xray（只监听本机 127.0.0.1）和 Node 前门（监听 Railway 分配的 PORT，对外）。

- xray 开两个 WS 入站：VMess 路径 `/tk-rail`（端口 20000）、VLESS 路径 `/tk-vless`（端口 20001），共用同一个 UUID。

- Node 前门做三件事：`GET /sub-<SUB_TOKEN>` 发 Surge 托管订阅；`GET /vless` 发 vless:// 链接；WS Upgrade 按路径反代到 xray 对应端口；其余一切请求返回纯文本 `400 Bad Request`（伪装成坏网站）。

- 对外只有一个域名、一个出口 IP。「VMess 节点」和「VLESS 节点」是同一个节点的两种进门方式，不是两个节点。

## 红线

- **密钥不入仓库、不进部署目录、不写日志明文**：UUID/SUB_TOKEN 只存部署目录之外的 600 权限文件，部署时通过环境变量注入。`railway up` 会打包整个目录，钥匙文件留在目录里等于把钥匙上传进镜像。

- **不绑卡、不购买、不升级付费档**，除非操作者明确决定。未绑卡时额度烧完只会停机，不会产生账单。

- **动代码先给方案、确认后实施**；部署、换区这类基础设施动作同样先说清再做。换区不改变域名和任何客户端链接，可随时换回。

## 步骤 1：生成两把钥匙

- UUID：`python3 -c "import uuid; print(uuid.uuid4())"` 或 `uuidgen`，形如 `xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx`。

- SUB_TOKEN：字母数字 24 位随机串，作为订阅链接的口令部分（太短可被猜）。

- 两者写入部署目录**之外**的 600 权限文件，值不抄进任何文档或仓库。

## 步骤 2：写 5 个部署文件

在项目目录根下放以下 5 个文件，不要改文件名大小写。

### Dockerfile

```dockerfile
FROM ghcr.io/xtls/xray-core:latest AS xray
FROM node:20-alpine
COPY --from=xray /usr/local/bin/xray /usr/local/bin/xray
WORKDIR /app
COPY . .
RUN chmod +x entrypoint.sh /usr/local/bin/xray
EXPOSE 8080
CMD ["sh", "entrypoint.sh"]
```

### config.json.template（`__UUID__` 是占位符，启动时替换）

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [
    {
      "listen": "127.0.0.1", "port": 20000, "protocol": "vmess",
      "settings": { "clients": [{ "id": "__UUID__", "alterId": 0 }] },
      "streamSettings": { "network": "ws", "wsSettings": { "path": "/tk-rail" } }
    },
    {
      "listen": "127.0.0.1", "port": 20001, "protocol": "vless",
      "settings": { "clients": [{ "id": "__UUID__" }], "decryption": "none" },
      "streamSettings": { "network": "ws", "wsSettings": { "path": "/tk-vless" } }
    }
  ],
  "outbounds": [{ "protocol": "freedom", "tag": "direct" }]
}
```

### entrypoint.sh

```sh
#!/bin/sh
set -e
if [ -z "$UUID" ]; then
  echo "UUID env var is not set" >&2
  exit 1
fi
sed "s/__UUID__/$UUID/g" config.json.template > config.json
echo "starting xray..."
/usr/local/bin/xray run -config config.json &
XRAY_PID=$!
trap "kill $XRAY_PID" EXIT INT TERM
echo "starting front door..."
exec node server.js
```

### server.js（Node 前门，零依赖，只用 http/net 内置模块）

```js
const http = require('http');
const net = require('net');

const PORT = parseInt(process.env.PORT || '8080', 10);
const UUID = process.env.UUID || '';
const SUB_TOKEN = process.env.SUB_TOKEN || '';

function badRequest(res) {
  res.writeHead(400, { 'Content-Type': 'text/plain' });
  res.end('400 Bad Request');
}

const server = http.createServer((req, res) => {
  const host = req.headers.host || 'localhost';
  const url = req.url || '/';

  if (SUB_TOKEN && url === `/sub-${SUB_TOKEN}`) {
    const subUrl = `https://${host}/sub-${SUB_TOKEN}`;
    const body = [
      `#!MANAGED-CONFIG ${subUrl} interval=43200 strict=false`,
      '',
      '[General]',
      'loglevel = notify',
      'dns-server = system, 1.1.1.1, 8.8.8.8',
      'skip-proxy = 127.0.0.1, 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12, localhost',
      'bypass-tun = 192.168.0.0/16, 10.0.0.0/8, 172.16.0.0/12',
      '',
      '[Proxy]',
      `Railway-US = vmess, ${host}, 443, username=${UUID}, vmess-aead=true, tls=true, ws=true, ws-path=/tk-rail, ws-headers=Host:${host}`,
      '',
      '[Proxy Group]',
      'PROXY = select, Railway-US, DIRECT',
      '',
      '[Rule]',
      'GEOIP,CN,DIRECT',
      'FINAL,PROXY',
      ''
    ].join('\n');
    res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
    res.end(body);
    return;
  }

  if (url === '/vless') {
    const link = `vless://${UUID}@${host}:443?encryption=none&security=tls&type=ws&host=${encodeURIComponent(host)}&path=%2Ftk-vless#Railway-US`;
    res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
    res.end(link + '\n');
    return;
  }

  badRequest(res);
});

server.on('upgrade', (req, socket, head) => {
  let targetPort = null;
  if (req.url === '/tk-rail') targetPort = 20000;
  else if (req.url === '/tk-vless') targetPort = 20001;
  else { socket.destroy(); return; }

  const backend = net.connect(targetPort, '127.0.0.1', () => {
    let raw = `${req.method} ${req.url} HTTP/${req.httpVersion}\r\n`;
    for (let i = 0; i < req.rawHeaders.length; i += 2) {
      raw += `${req.rawHeaders[i]}: ${req.rawHeaders[i + 1]}\r\n`;
    }
    raw += '\r\n';
    backend.write(raw);
    if (head && head.length) backend.write(head);
    backend.pipe(socket);
    socket.pipe(backend);
  });
  backend.on('error', () => socket.destroy());
  socket.on('error', () => backend.destroy());
});

server.listen(PORT, '0.0.0.0', () => {
  console.log(`front door listening on ${PORT}`);
  if (!UUID) console.log('UUID env var is not set');
  if (!SUB_TOKEN) console.log('SUB_TOKEN env var is not set');
});
```

### railway.json

```json
{
  "$schema": "https://railway.com/railway.schema.json",
  "build": { "builder": "DOCKERFILE", "dockerfilePath": "Dockerfile" },
  "deploy": { "restartPolicyType": "ON_FAILURE", "restartPolicyMaxRetries": 10 }
}
```

## 步骤 3：本地测试（部署前必做）

- 语法：`node --check server.js`、`python3 -m json.tool railway.json`、`python3 -m json.tool config.json.template`。

- 起前门：`PORT=18080 UUID=<uuid> SUB_TOKEN=<token> node server.js`，另开终端验三件：根路径返回 `400 Bad Request`；`/sub-<token>` 首行 `#!MANAGED-CONFIG` 且有 `Railway-US = vmess, ... ws-path=/tk-rail` 行；`/vless` 返回含 `path=%2Ftk-vless` 的链接。三件不过不要部署。

- 可选：把这 5 个文件存入一个私有 Git 仓库作代码备份（与部署无关，不构成自动部署）。

## 步骤 4：部署（二选一）

身份背景：Railway 没有自家用户名/密码体系，注册/登录只有 GitHub、Google、邮箱 magic link 三种方式，都是借第三方账号证明身份。

### 路线 A：CLI 直传

- 装 CLI：`npm i -g @railway/cli --prefix ~/.local`，得到 `~/.local/bin/railway`。

- 登录：`railway login --browserless` 输出一个 `https://railway.com/activate?user_code=XXXX-XXXX` 链接和短码；账号持有人在任意设备的浏览器打开链接，用 Google 或邮箱登录/建号并确认短码，CLI 随即在本地保存登录令牌。用 `railway whoami` 验证连接（应返回账号名和邮箱）。令牌保存在本地，后续操作不用重复登录。

- 建项目并绑定目录（在项目目录内）：`railway init --name <项目名> --json`。

- 直传部署：`railway up --detach -y`，把本地目录打包上传，Railway 按 Dockerfile 构建并运行。

- 设变量：`railway variable set UUID=<uuid>`、`railway variable set SUB_TOKEN=<token>`，每条都会触发一次重新部署，值从钥匙文件读取、不要回显。注意首次部署时还没有 UUID 变量，服务会以 "UUID env var is not set" 退出并按重启策略重试；设完变量即恢复，属于正常顺序，不是部署失败。

- 生成域名：`railway domain --service <服务名>`，得到 `https://<项目名>-production-xxxx.up.railway.app`。Target port 显示 `-` 属正常，Railway 会按服务实际监听的 PORT 路由。

- 等 `railway status` 显示服务 `Online`（构建约 1–2 分钟）。

### 路线 B：GitHub 集成

- 把 5 个文件推送到一个 GitHub 私有仓库（不勾选自动生成 README 等文件，根目录直接放 5 个文件）。

- 打开 railway.com，用 GitHub 登录（新账号即自动注册）。GitHub 若开启双因素认证，按页面提示完成。

- New Project → Deploy from GitHub repo，选择该仓库；GitHub App 授权时只授权这一个仓库。

- 等首次部署开始后，到服务的 Variables 添加 `UUID`、`SUB_TOKEN` 两条变量并保存，服务会自动重新部署。

- Settings → Networking → Generate Domain 生成公网域名（让选端口时选列表已有端口，没有就填 8080）；等服务状态变为 ACTIVE / Online。

## 步骤 5：验收清单（全过才算交付）

- **状态与日志**：服务为 Online / ACTIVE；部署日志中有 `front door listening on ...` 和 xray 的 started 字样，无 "UUID env var is not set"。

- **根路径**：打开 `https://<域名>/` 返回纯文本 `400 Bad Request`。

- **订阅**：打开 `https://<域名>/sub-<SUB_TOKEN>`，首行以 `#!MANAGED-CONFIG` 开头，且有一行以 `Railway-US = vmess, <域名>, 443, username=<UUID>` 开头、带 `ws-path=/tk-rail`。

- **VLESS + WS**：打开 `https://<域名>/vless` 返回以 `vless://` 开头、`path=%2Ftk-vless` 的链接；对 `/tk-rail` 与 `/tk-vless` 各发一次 WebSocket Upgrade 握手，应得 `101 Switching Protocols`。

## 步骤 6：换区（免费，链接不变）

- 命令：`railway scale --service <服务名> <目标区>=1`，再把旧区置 0。操作前先用 `railway status --json` 查看当前 `multiRegionConfig` 里的真实区域 key，并以该 key 为准置 0，确认最终只剩目标区 1 个副本。

- 区域对照（别名 / 区域 ID / 地点）：

  - `us-west` / `us-west2` / 美国西部（加州）
  - `us-east` / `us-east4-eqdc4a` / 美国东部（弗吉尼亚）
  - `eu-west` / `europe-west4-drams3a` / 欧洲（阿姆斯特丹）
  - `southeast-asia` / `asia-southeast1-eqsg3a` / 东南亚（新加坡）

- 换区不改域名、不改任何客户端链接，变的只有出口 IP 所在地；无数据卷的服务换区无停机。换完重跑「步骤 5 验收清单」。网页端改区入口：服务 Settings → Scale/Deploy → Regions。

## 步骤 7：客户端导入对照

- **Surge（iOS/Mac）**：用订阅链接 `https://<域名>/sub-<SUB_TOKEN>`，配置页「从 URL 下载配置」粘贴导入，策略组 PROXY 选 Railway-US。Surge 不支持 VLESS，所以它走 VMess 这一条。

- **Clash Meta / Mihomo**：导入 VLESS 链接（打开 `https://<域名>/vless` 复制全文，「从剪贴板导入」）。不支持 VLESS 的老版 Clash 改用 VMess 链接：`vmess://` + base64(JSON)，JSON 字段为 v=2、ps=Railway-US、add=<域名>、port=443、id=<UUID>、aid=0、net=ws、host=<域名>、path=/tk-rail、tls=tls、sni=<域名>。

- **v2rayNG（安卓）**：同样用 VLESS 链接，右上角 + → 从剪贴板导入 → 选中节点 → 连接。一句话记忆：除 Surge 外都用 VLESS。

- **测速**：App 内用「真连接延迟」类功能测延迟（经节点出站测得）；测带宽时保持连接后打开 fast.com。

## 步骤 8：费用与生命周期

- 新号 Trial：一次性 $5 额度，30 天有效（烧完或到期，先到为准）。未绑卡，烧完只停机、不扣费。

- Trial 结束后自动变 Free 档：每月 $1 额度、不累积；服务规格降到 1 vCPU / 0.5 GB 内存 / 1 副本。本架构内存占用在百 MB 量级、单副本，不受规格影响。

- 费率：出口流量约 $0.05/GB；内存约 $0.00000386/GB-秒（约合 $10/GB/月）。据此估：节点闲置光内存成本每月不到 $1；Free 的 $1 养常开节点偏紧，流量用得多时会在当月内烧完停机、下月额度刷新后恢复；要全月稳定可用 Hobby 档（$5/月，含 $5 用量）。

- 查用量：`railway usage`（工作区总额）、`railway usage projects --json`（按项目）、`railway metrics --service <服务名>`（CPU/内存/流量近况）。短期样本不要直接外推整月，向他人报数时注明口径。

- 满速下载最烧额度：速率（Mbps）÷ 8 × 3600 秒 ≈ 每小时 GB 数，再乘 $0.05 即每小时的出口成本。

## 故障排查速查

- 部署日志反复出现 "UUID env var is not set" → 变量没设或设错服务/环境；用 `railway variable list` 核对后重设，会自动重新部署。

- 构建失败 → 先查 5 个文件名大小写与后缀是否原样、Dockerfile 是否在根目录；重新部署一次再看构建日志定位。

- 域名打开 404 或连不上 → 先确认服务已 Online、域名生成后稍等同步；再查日志有无 `front door listening`。

- 客户端导入报错 → 核对协议是否匹配客户端（Surge 只认订阅/VMess、不认 VLESS；老版 Clash 不认 VLESS）、UUID 是否与服务变量一致、订阅链接是否填在「从 URL 下载配置」处、标点是否逐字一致。

- 连上但不通 → 核对 UUID 一致；确认客户端设备时间为自动（VMess 有约 90 秒的时间容差）；查服务日志与 Usage（额度是否已烧完导致停机）。

- CLI 命令报网络超时 → 检查当前网络与代理设置后重试。
