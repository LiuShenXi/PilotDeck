# PilotDeck 本地连不通排查记录

这份文档记录一次本地排查过程：Web UI 看起来已经进入
`Sending...` / `Thinking`，但模型中转后台完全看不到 PilotDeck 请求。

以后遇到「页面像是在跑，但中转没有请求」的问题，可以优先按这份清单排查。

## 典型症状

- Web UI 能提交 prompt，但一直显示 `Sending...` 或 `Thinking`。
- 模型中转 / 代理后台看不到 PilotDeck 发来的请求。
- 浏览器控制台可能出现 WebSocket 错误。
- 开发模式下，UI 可能先显示一个乐观状态，但实际 chat WebSocket 并没有把命令送到 UI server。
- `/api/config/provider` 仍然可能显示正确的 provider、base URL 和 model，所以这个问题容易被误判为模型配置问题。

## 本次确认的根因

### 1. Vite HMR 和 PilotDeck chat 共用了 `/ws`

本地 dev 模式里，Vite 的 HMR WebSocket 可能和 PilotDeck 的聊天 WebSocket 路径冲突，因为两者都用了 `/ws`。

服务端关键日志：

```text
[ERROR] Chat WebSocket error: Unexpected token 'p', "ping" is not valid JSON
```

这里的 `ping` 不是 PilotDeck 聊天帧，而是普通 WebSocket/HMR ping 被送进了 PilotDeck chat handler。之后 chat socket 可能被关闭，UI 就容易停在误导性的 `Sending...` / `Thinking`。

修复方式：

```js
// ui/vite.config.js
server: {
  hmr: {
    path: '/__vite_hmr'
  },
  proxy: {
    '/ws': {
      target: `ws://${proxyHost}:${serverPort}`,
      ws: true
    }
  }
}
```

改完 Vite 配置后，需要重启 dev server。

### 2. 生产 UI server 缺少 `PILOTDECK_GATEWAY_URL`

运行 built UI server 时，只设置 `PILOTDECK_GATEWAY_PORT` 不一定够。UI bridge 可能仍然用默认 gateway URL，导致它去连一个不存在的 gateway。

服务端关键日志：

```text
[pilotdeck-bridge] gateway connect failed after 30000ms: Failed to connect to gateway WebSocket.
```

推荐的本地生产模式启动方式：

```bash
SERVER_PORT=3100 \
PILOTDECK_GATEWAY_PORT=18791 \
PILOTDECK_GATEWAY_URL=ws://127.0.0.1:18791/ws \
pnpm --filter pilotdeck-ui start:built
```

## 快速隔离清单

### 1. 确认 PilotDeck 读到的 provider 配置

不要打印真实 secret：

```bash
curl -s http://localhost:3100/api/config/provider \
  | node -e "let s='';process.stdin.on('data',d=>s+=d);process.stdin.on('end',()=>{const j=JSON.parse(s); if(j.provider) j.provider.apiKey='[set]'; console.log(JSON.stringify(j,null,2));})"
```

期望类似：

```json
{
  "exists": true,
  "provider": {
    "type": "openai",
    "baseUrl": "https://example.com/v1",
    "apiKey": "[set]",
    "model": "your-model-id"
  }
}
```

如果这里不对，先修 `~/.pilotdeck/pilotdeck.yaml`。

### 2. 直接测试上游中转

从 `~/.pilotdeck/pilotdeck.yaml` 读取 key，但不打印 key：

```bash
node --input-type=module -e '
import fs from "node:fs";
import { parse } from "yaml";
const cfg = parse(fs.readFileSync(process.env.HOME + "/.pilotdeck/pilotdeck.yaml", "utf8"));
const providerId = cfg.agent.model.split("/")[0];
const model = cfg.agent.model.slice(providerId.length + 1);
const p = cfg.model.providers[providerId];
const url = `${String(p.url).replace(/\/+$/, "")}/chat/completions`;
const res = await fetch(url, {
  method: "POST",
  headers: { Authorization: `Bearer ${p.apiKey}`, "content-type": "application/json" },
  body: JSON.stringify({ model, messages: [{ role: "user", content: "ping" }], max_tokens: 1 })
});
console.log("HTTP", res.status, res.statusText);
const text = await res.text();
try {
  const j = JSON.parse(text);
  console.log(JSON.stringify({ id: j.id, model: j.model, choices: Array.isArray(j.choices) ? j.choices.length : undefined, error: j.error }, null, 2));
} catch {
  console.log(text.slice(0, 500));
}
'
```

如果这一步成功，但 UI 不通，通常不是 key、base URL 或 model ID 的问题。

### 3. 直接测试 PilotDeck chat WebSocket

```bash
node --input-type=module -e '
import WebSocket from "ws";
const ws = new WebSocket("ws://localhost:3100/ws");
const start = Date.now();
const timer = setTimeout(() => { console.log("TIMEOUT"); ws.close(); }, 30000);
ws.on("open", () => {
  console.log("OPEN");
  ws.send(JSON.stringify({
    type: "pilotdeck-command",
    command: "ping from direct websocket debug",
    options: {
      projectPath: process.env.HOME + "/.pilotdeck",
      cwd: process.env.HOME + "/.pilotdeck",
      permissionMode: "default"
    }
  }));
});
ws.on("message", (buf) => {
  const msg = JSON.parse(buf.toString());
  console.log(JSON.stringify({
    t: Date.now() - start,
    type: msg.type,
    kind: msg.kind,
    sessionId: msg.sessionId,
    content: msg.content?.slice?.(0, 120),
    error: msg.error,
    exitCode: msg.exitCode,
    success: msg.success
  }));
  if (msg.kind === "complete" || msg.kind === "error" || msg.type === "error") {
    clearTimeout(timer);
    ws.close();
  }
});
ws.on("error", e => console.log("WSERROR", e.message));
ws.on("close", () => { clearTimeout(timer); console.log("CLOSE"); });
'
```

如果直接 WebSocket 成功，但浏览器 UI 不成功，优先查：

- 浏览器 WebSocket 是否断开。
- Vite HMR 是否和 `/ws` 冲突。
- 是否有旧 tab、旧 service worker、旧 dev server 状态干扰。

### 4. 确认请求是否真正进入 PilotDeck

查看最近 session：

```bash
curl -s 'http://localhost:3100/api/projects/general/sessions?limit=5&offset=0' \
  | node -e "let s='';process.stdin.on('data',d=>s+=d);process.stdin.on('end',()=>{const j=JSON.parse(s); console.log(JSON.stringify({total:j.total, sessions:j.sessions?.map(x=>({id:x.id||x.sessionId,title:x.title||x.summary,lastActivity:x.lastActivity}))},null,2));})"
```

如果 UI 显示 `Sending...`，但这里没有新增 session，说明命令大概率没有进入 UI server 的 chat handler。

## 服务端日志判断

健康链路应该看到：

```text
[INFO] Chat WebSocket connected
[DEBUG] User message: <prompt>
[pilotdeck-bridge] submitTurn mode=...
[router] decision: tier=..., model=<provider>/<model>, ...
```

dev WebSocket/HMR 冲突时常见：

```text
[ERROR] Chat WebSocket error: Unexpected token 'p', "ping" is not valid JSON
```

生产模式 bridge URL 配错时常见：

```text
[pilotdeck-bridge] gateway connect failed after 30000ms: Failed to connect to gateway WebSocket.
```

## 本次排查结论

- 直接调用 OpenAI-compatible 中转是通的。
- 直接通过最小 WebSocket 客户端调用 PilotDeck gateway 是通的。
- 因此问题不在 provider key、base URL 或 model ID。
- 问题集中在浏览器 / dev server WebSocket 链路。
- 给 Vite HMR 设置专用路径 `/__vite_hmr` 后，可以避免 dev 模式继续把 HMR ping 打到 PilotDeck `/ws`。
- 使用 built UI 并显式设置 `PILOTDECK_GATEWAY_URL` 后，页面聊天请求可以成功进入后端并完成模型响应。
