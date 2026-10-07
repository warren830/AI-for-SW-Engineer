# KiroCrew App 开发指南（实战版）

> 内容来自我们自己做过的四个 App，以及一台开发机上装着的 30 个 App（包括官方 builtin）：
>
> | App | 仓库 | 形态 | 一句话 |
> |---|---|---|---|
> | **aidlc-studio** | [warren830/kirocrew-app-aidlc](https://github.com/warren830/kirocrew-app-aidlc) | 后端跑在 gateway 进程内 + React UI + 一个只读 agent | AI-DLC 的审批中心，已发布到 v1.1.5 |
> | **devteam** | `warren830/dev-team`（私有） | 后端跑在 gateway 进程内 + React UI + 4 个角色 agent | 由代码编排多个 chat session 组成开发团队 |
> | **kirocrew-team** | `warren830/kirocrew-team`（私有） | 后端跑在 gateway 进程内 + React UI | 8 个角色协作做网站，配一个实时看板 |
> | **workshop-customizer** | [warren830/kirocrew-app-adlc](https://github.com/warren830/kirocrew-app-adlc)（`app/`） | 独立进程后端 + gateway 内路由 + 手写 ESM UI | 把客户需求整理成 Workshop 场景包 |
>
> **适用版本**：KiroCrew desktop **0.8.0-insider.12**。运行时以 App 包里打包的那份为准：
> `/Applications/KiroCrew.app/Contents/Resources/backend-dist/kirocrew-backend-arm64/lib/python3.12/site-packages/kiro_crew/apps/`
> 下文用 `$K` 代指这个目录。本地 checkout 的 KiroCrew 开源源码停在 0.3.0，manifest、module_loader、hooks 都和现在的运行时对不上，**只能当参考，不能当依据**。
> 运行时一升级，先重新核对 §3 和 §4 里引用的行号。

---

## 0. 三十秒速览

1. **默认选“后端跑在 gateway 进程内”**：`backend.hooks.{routes,on_startup,on_shutdown}`。UI 用 Vite library 模式打成一个 `ui/dist/index.mjs`，React 和 SDK 都标成 external。我们三个主力 App 都是这么做的。
2. **后端文件是按路径加载的，`sys.path` 不会被改动**。`routes.py` 和 `hooks.py` 各放一份同样的 namespace bootstrap 代码（§3.2）。这段代码直接从 devteam 抄过来。
3. **不要阻塞事件循环**。所有磁盘 IO 和子进程调用都放进 `asyncio.to_thread`。gateway 发现事件循环卡住大约 25 秒就会杀掉进程。
4. **`ctx.storage` 不提供 CAS 和 fsync**，状态重要就用 `ctx.data_dir` 下的 SQLite，或者用“写临时文件 → fsync → rename”的原子写。
5. **调 AI 有三条路**：
   - `ctx.spawn.run`：只能跑本 App 自己声明的 agent，并且是 auto-approve，所以 agent 的 tools 必须收窄；
   - 用 gateway 的 `/api/chat` 驱动真实的 chat session；
   - 用 `ctx.cron` 做定时任务。
6. **开发循环用 `dev-install.sh`**：依次 disable → update → trust → enable，**永远不要 uninstall**，否则 trust 授权会跟着丢。
7. **发布**：打 tag，把 `release` 分支 fast-forward 到这个 tag，在团队 registry 里登记。仓库要小，因为安装时 `git clone --depth 1` 的超时只有 60 秒。

---

## 1. 先选架构

### 1.1 三种后端形态

| 形态 | manifest | 路由挂在哪 | 适用 | 代价 |
|---|---|---|---|---|
| **无后端** | 不写 `backend` | – | 纯前端工具，只调 host API（agent-worlds、projects 等） | 没有服务端状态 |
| **gateway 进程内 hooks**（推荐） | `backend.hooks.routes/on_startup/on_shutdown` | `/api/apps/<name>/<path>` | 绝大多数场景。可以直接用 `ctx.*` SDK 和 aiohttp `request`，可以自己做 SSE | 和 gateway 同进程、无沙箱；一阻塞就拖垮整个 gateway；第三方 pip 依赖不好用（§3.7） |
| **独立进程** | `backend.entryPoint` + `port:"auto"` + `healthCheck:"/health"` | `/apps/<name>/api/<path>`，由 gateway 反向代理 | 需要自带依赖或 venv、跑重活，或者想与 gateway 隔离（md-notebook、workflows、workshop-customizer） | 每个请求都要校验 HMAC；代理有 **30 秒总超时**；**不支持 WebSocket**；拿不到 `ctx`（spawn、job 都用不了） |

workshop-customizer 两种形态混着用。独立进程负责业务和 console。凡是需要 `ctx.spawn` 的活，都交给 gateway 内的路由来做。进程内的那份 `routes.py` 由独立进程按路径加载，让两边共用一套代码（`kirocrew-app-adlc/app/backend/server.py:496-509`）。

### 1.2 选择规则

- 要调 agent，或者要用 `ctx.storage`、`ctx.audit`、`ctx.job` → 选 **gateway 进程内**。
- 有请求会超过 30 秒、需要 WebSocket，或者需要 numpy 这类原生依赖 → 选**独立进程**。也可以混用：独立进程扛重活，进程内路由负责调 spawn。
- 两种都不用写也行。builtin 的 command-bar 只用 `ui.overlays` 替换了 host 的 quick-search。

---

## 2. 目录骨架与 manifest

### 2.1 推荐布局（aidlc-studio 和 devteam 一致）

```
<app>/
├── app.json                 # manifest（放在仓库根目录；如果放子目录，安装时要加 subdirectory，见 §8.4）
├── backend/
│   ├── routes.py            # 给 host 的薄入口：bootstrap + register_routes
│   ├── hooks.py             # 给 host 的薄入口：bootstrap + on_startup/on_shutdown
│   └── <pkg>/               # 真正的逻辑：service、handlers、store、orchestrator……（只用相对 import）
├── agents/*.json            # App 自带的 agent
├── ui/
│   ├── package.json  vite.config.ts  vitest.config.ts  tsconfig.json
│   ├── src/                 # main.tsx 的 default export 就是页面组件
│   ├── types/kirocrew-app-sdk.d.ts   # 只声明 host 真正 export 的名字
│   └── dist/index.mjs, style.css     # ★ 构建产物要提交进 git
├── assets/                  # icon-512.png、hero*.png、screenshots/*.png（只接受 PNG/WebP/JPEG）
├── scripts/  check.sh  dev-install.sh  kcapi.sh
├── tests/                   # pytest；UI 测试放在 ui/src/**/*.test.tsx
└── docs/
```

`kirocrew app scaffold` 生成的骨架默认是**独立进程**后端，用 `http.server`（`$K/scaffold.py:170-221`）。想做 gateway 进程内 App，最好直接照抄 devteam 的骨架，而不是在 scaffold 生成的结果上改。

### 2.2 app.json 速查

必填字段：`name`（kebab-case，满足 `^[a-z0-9]+(?:-[a-z0-9]+)*$`，**不能有点号**）、`version`（严格 semver，`1.0` 不合法）、`displayName`、`description`。

aidlc-studio 的写法可以直接当模板用（删减后）：

```json
{
  "name": "aidlc-studio",
  "version": "1.1.5",
  "displayName": "AI-DLC Studio",
  "description": "…",
  "author": "ychchen",
  "license": "Apache-2.0",
  "minKiroCrewVersion": "0.3.0",
  "tags": ["developer-tools"],
  "iconPath": "assets/icon-512.png",
  "heroImage": "assets/hero-light.png", "heroImageDark": "assets/hero-dark.png",
  "screenshots": ["assets/screenshots/a.png"],
  "agents": ["agents/advisor.json"],
  "backend": { "hooks": {
    "routes": "backend.routes:register_routes",
    "on_startup": "backend.hooks:on_startup",
    "on_shutdown": "backend.hooks:on_shutdown"
  }},
  "ui": {
    "entry": "dist/index.mjs",
    "pages": [{ "route": "/apps/aidlc-studio", "label": "AI-DLC Studio", "icon": "GitBranch", "iconUrl": "icon.svg" }],
    "sidebar": { "section": "Apps", "order": 10 }
  },
  "permissions": {
    "api": ["/api/apps/aidlc-studio", "/api/chat"],
    "events": ["aidlc-studio:action"],
    "storage": true, "spawn": true, "cron": false, "network": false, "memory": ""
  },
  "dependencies": { "commands": [], "optionalCommands": ["bun", "git"] },
  "platform": { "os": ["macos", "linux"], "installMode": "server" }
}
```

**manifest 里的坑**：

| 坑 | 说明 |
|---|---|
| `ui.entry` 的相对基准是 `ui/` | 要写 `dist/index.mjs`。写成 `ui/dist/...` 会解析成 `ui/ui/...`（`$K/manifest.py:616-620`） |
| hook 路径里只能用标识符 | 格式必须是 `module.path:callable`，所以模块名和目录名都不能带连字符，比如 `backend/my-routes.py` 无法引用 |
| 权限布尔值必须是字面量 `true` | 写成字符串 `"true"` 等于拒绝；列表写成字符串会变成 `[]`。这类错误**都不报**，结果只是对应的 `ctx.spawn` 之类变成 `None` |
| 未知的顶层 key 不报错 | 会被存进 `extra`，hook 里通过 `ctx.config` 读到。拼错的 key 因此不会被发现 |
| `minKiroCrewVersion` 只比较数字部分 | `0.8.0-insider.12` 会按 `0.8.0` 处理 |
| `permissions.api` 只按前缀匹配 | 前端 SDK 的 allowlist **不认 `*`**，要声明普通前缀。builtin mochi 写了 `/api/apps/mochi/*`，但它编译在 host 包里，不走这条校验 |
| `sessionApproval: true` | App 能替**自己的** session 回答工具审批，但这个能力也够得着用户自己的 session，必须自己加归属校验（§4.3） |
| `platform.os` | 默认是 macos+linux。要支持 Windows 必须显式写上，而且 setup 脚本依赖 bash |
| 保留路径段 | `_jobs, config, dev, disable, enable, manifest, migrate-cleanup, open, token, uninstall, update` 归 host 所有，App 路由用了这些名字会被遮蔽（`$K/manifest.py:1353-1365`） |

其他可选字段：`crons[]`、`setup.{onInstall,onUpdate,onEnable,onDisable,onUninstall}`（bash 脚本，默认 30 秒超时）、`contributes.{commands,panelTabs,fileMenuItems,sessionControls}`、`notifications.channels`（最多 8 个）、`mcpServers`、`skills`、`jobFamilies`。完整的表在 `$K/manifest.py:2320-2348`。

---

## 3. 后端（gateway 进程内 hooks）

### 3.1 加载机制（必须先理解）

- host 把 `"backend.routes:register_routes"` 换算成文件路径 `<app>/backend/routes.py`，用 `spec_from_file_location` 加载，模块名是 `_kirocrew_app_<name>.backend.routes`（`$K/module_loader.py:285-346`）。
- **`sys.path` 不会被改动**。写 `import backend.util` 或者 `import mypkg` 这种绝对 import 都会失败。
- `routes.py` 和 `hooks.py` 是**两个互不相干的模块**，模块级变量不共享。
- 代码是否重新加载，取决于 enable：只有 enable 时才会重新 import。reconciler 每 15 秒比对一次磁盘上的 hook 签名，发现变化会自动 disable 再 enable（`$K/hook_reconcile.py:92`）。但关键步骤别依赖它，还是用 `dev-install.sh` 显式操作。
- 第三方代码在 gateway 进程里跑，**没有沙箱**。还必须拿到执行授权，也就是 App 名在 `agent.apps_trusted` 里，否则 App 显示 enabled，但 hooks、routes、crons 都不会运行。

### 3.2 Namespace bootstrap（直接复制）

真正的代码放在 `backend/<pkg>/` 里，通过一个私有 namespace 包导入。这样 gateway 里和 pytest 里的相对 import 行为一致。**`routes.py` 和 `hooks.py` 各放一份一模一样的代码**：

```python
# backend/routes.py（hooks.py 顶部放完全相同的 bootstrap 段）
from __future__ import annotations
import importlib, importlib.machinery, importlib.util, sys
from pathlib import Path

_BACKEND_DIR = Path(__file__).resolve().parent
_NS = "_myapp_backend"            # 每个 App 用不同的名字
_REQUIRED = ("service", "handlers")

def teardown_namespace() -> None:
    prefix = f"{_NS}."
    for key in [k for k in sys.modules if k == _NS or k.startswith(prefix)]:
        sys.modules.pop(key, None)

def bootstrap():
    for final in (False, True):
        root = sys.modules.get(_NS)
        if root is not None and list(getattr(root, "__path__", ())) != [str(_BACKEND_DIR)]:
            teardown_namespace(); root = None          # 安装目录变了：丢掉旧的 namespace
        if root is None:
            spec = importlib.machinery.ModuleSpec(_NS, None, is_package=True)
            root = importlib.util.module_from_spec(spec)
            root.__path__ = [str(_BACKEND_DIR)]
            sys.modules[_NS] = root
        try:
            mods = {n: importlib.import_module(f"{_NS}.myapp.{n}") for n in _REQUIRED}
        except BaseException:
            teardown_namespace(); raise                # ★ 失败时必须清掉，否则下次 enable 拿到的还是坏模块
        if all(mods.values()):
            return mods
        teardown_namespace()
        if final:
            raise ImportError(f"{_NS}.myapp imported incompletely")

_mods = bootstrap()
service, handlers = _mods["service"], _mods["handlers"]

def register_routes(ctx):
    from kiro_crew.apps.route_registry import AppRoute   # host 模块，在 gateway 里一定存在
    svc = service.get_or_build(ctx)
    return handlers.build_routes(svc, AppRoute)
```

```python
# backend/hooks.py（bootstrap 段之后）
async def on_startup(ctx) -> None:
    await service.get_or_build(ctx).start()

async def on_shutdown(ctx) -> None:
    try:
        svc = service.discard(getattr(ctx, "name", "myapp"))
        if svc is not None:
            await svc.stop()
    finally:
        teardown_namespace()      # 下次 enable 从磁盘重新读代码
```

出处：`dev-team/backend/routes.py:19-72`、`dev-team/backend/hooks.py:66-76`。aidlc-studio 和 kirocrew-team 用的也是同一套。

### 3.3 Service 按 App 名注册

`register_routes` 和 `on_startup` 拿到的是同一个 ctx，**`on_shutdown` 拿到的却是新的 ctx**。所以 service 要按 App 名放进一个注册表，关闭时按名字取出来：

```python
_REGISTRY: dict[str, Services] = {}
def get_or_build(ctx):
    name = getattr(ctx, "name", APP_NAME) or APP_NAME
    return _REGISTRY.setdefault(name, Services(ctx))
def discard(name): return _REGISTRY.pop(name, None)
```

### 3.4 路由与鉴权

- `register_routes(ctx)` 必须返回 `list[AppRoute(method, path, handler)]`，handler 的签名是 `async def handler(request, ctx)`。返回的不是 list，或者元素不是 AppRoute，都会被跳过，App 被标成 degraded。
- path 用相对路径，例如 `/health`、`/runs/{id}`，参数在 `request.match_info` 里。精确匹配优先于带参数的模式。
- **`register_routes` 里不要做重活**：它一抛异常，所有路由都没了，包括 `/health`。
- **把 `web` 和 `is_owner` 当参数注入**：写成 `build_routes(svc, AppRoute, web=aiohttp.web, is_owner=...)`，测试时换成 `FakeWeb`，不需要启动 gateway（`dev-team/backend/devteam/handlers.py:229-253`）。
- **统一用一个 `guarded()` 包装器**（`dev-team/backend/devteam/httpkit.py:60-97`）：
  - 没有 `request["user"]` 返回 401；
  - 有 `request["app"]`，也就是 app token 发来的请求，返回 403；
  - 不是 owner 返回 403，owner 判断用 host 的 `is_owner_dashboard_request`，import 失败按“不是 owner”处理；
  - 业务异常统一转成 `{error, code}` JSON。
  - `/health` 不加 guard。
- 在测试里把路由表钉住：路由缺失、重复，或者用了保留路径段，都让测试失败（`aidlc-studio backend/studio/handlers/common.py:580-603`）。

### 3.5 生命周期与后台循环

- 同步 hook 直接在事件循环上执行，会阻塞循环。**hook 一律写成 async**。
- async 的 `on_startup` 只有 **30 秒**。超时后任务**不会被取消**，而是转到后台继续跑，这期间再次 startup 会被拒绝。所以 `on_startup` 里只做“打开存储 + `create_task` 启动循环”，然后立刻返回。
- **gateway 还没准备好**：startup hook 比 chat slot 恢复更早执行。要调 `/api/chat` 的循环，第一次 tick 前先轮询 `GET /api/ready`（`dev-team/backend/devteam/service.py:52-61`）。
- `on_startup` 时 `request.app` 还不存在，host 的状态要等第一个请求进来才拿得到。别在启动时依赖它做决定，aidlc-studio 在这里踩过真 bug（`docs/open-defects.md:573-584`）。
- 循环骨架可以参考 devteam：
  - 繁忙时每 1 秒一次 tick，空闲时 20 秒一次；
  - 有写请求时调 `poke()` 立刻唤醒；
  - 出错时指数退避，最长 15 秒；
  - 一次 tick 内 slot 列表只拉一次，因为 `GET /api/chat/slots` 每次要 30-60ms。
- **`on_shutdown` 必须停掉所有后台任务**：先 cancel 加 `wait_for(…, 5)`，再 flush，最后关闭存储和 HTTP session。任务留着不停，reconciler 会一直持有旧记录，不再重启这个 App。
- 启动阶段出问题，用 `ctx.health.mark_degraded(...)` 标记，不要直接抛异常。

### 3.6 存储选型

| 需求 | 用法 | 出处 |
|---|---|---|
| 少量配置、偏好 | `ctx.storage.get/set/delete/list_keys`，每个 key 存成一个 JSON 文件（`data/kv/<key>.json`） | `$K/app_storage.py` |
| 要事务、CAS、并发安全 | `ctx.data_dir/xxx.sqlite3`：单连接加 `RLock`，`BEGIN IMMEDIATE`，WAL，`synchronous=FULL`，设 `busy_timeout`，开 `foreign_keys=ON` | `aidlc-studio backend/studio/storage.py:560-585` |
| 运行记录、事件流 | 普通文件，原子写（tmp → fsync → `os.replace`）；JSONL 按 `\n` 切行，**不要用 `splitlines`**（它会在 U+2028 处断开记录）；写到一半断掉的末行要修复；读到坏 JSON 就改名成 `.corrupt` 再返回默认值 | `dev-team/backend/devteam/store.py` |

规则：所有 IO 都走 `asyncio.to_thread`。JSON 先在事件循环上序列化好再交给线程，免得线程读到一个正在被修改的 dict。

### 3.7 依赖

- gateway 进程内的后端**最好只用标准库**，aiohttp 由 host 提供。`requirements.txt` 会被 `pip install --target` 装进 `data/.kirocrew-deps`，但只有 **spawn 出来的进程**能通过 PYTHONPATH 看到。进程内加载的 hook 模块不会改 `sys.path`，**多半看不到这些包**（这是读代码得出的推断，没有实测）。确实需要的话，就像 workshop-customizer 那样把依赖 vendor 进来，或者改用独立进程后端。
- `dependencies.commands` 只做 `shutil.which` 检查，缺了会报告，但不会阻止安装或运行。

### 3.8 推送给前端：自己写 SSE

host 目前**不会把 `app_event` 转发给 App 前端**，SDK 的 `useAppEvents` 也收不到事件（aidlc-studio 实测）。做法是：

- 后端自己提供 `GET /events` 的 SSE（aiohttp `StreamResponse`），再加一个 `/events/poll` 作为降级；
- 前端给每种事件类型分别 `addEventListener`，因为带 `event:` 字段的 SSE 帧不会进 `onmessage`；
- 要求不高的话，直接轮询也可以，比如 devteam 每 2 秒拉一次 `/state`。

---

## 4. 和 Agent 打交道

### 4.1 `ctx.spawn`：调用 App 自带 agent 做一次性分析

```python
spawn = ctx.spawn            # manifest 里没写 permissions.spawn: true 时是 None，一定要判断
if spawn is None: raise Unavailable()
spawn_id = await spawn.run(task=prompt, agent="aidlc-studio-advisor", silent=True)
# 之后用 spawn.is_done(spawn_id) 轮询；未知的 id 也会被当成 done
```

- `agent` 参数传的是 agent JSON 里的 `name`。只有本 App 安装出来的 agent 才能用，也就是文件名为 `~/.kiro/agents/<app>--*.json` 的那些。传 host 默认 agent 或其他 App 的 agent，会抛 `SpawnError` 并写一条审计（`$K/spawn_sdk.py:160-195`）。
- **spawn 以 `approval_mode="auto"` 运行，agent 的 `tools` 列表就是唯一的防线**。
  - aidlc-studio 的 advisor 只给了 `["thinking"]`，所有证据都在 prompt 里一次给全，输出要求是一个固定 key 的 JSON 代码块。
  - workshop-customizer 的 agent 没有任何工具，只返回数据，等用户点了 Apply 才写盘。
- 完成事件里的结果会被截断到 3000 字符，完整内容要去读 `result_path`。
- 返回空 id 表示 host 拒绝了（策略或配额原因），`run` 会抛异常。

### 4.2 `/api/chat`：驱动真实的 chat session（多 agent 编排）

需要用户能看到、能接管的会话（devteam 每个角色一个 session），或者需要触发 AI-DLC 这类 `userPromptSubmit` 流程（aidlc-studio）时，走 gateway 的 chat API。有两种做法：

| 做法 | 例子 | 关键点 |
|---|---|---|
| **UI 用用户的 cookie 发请求** | aidlc-studio | 后端只负责“先落一条 Delivering 记录，再返回请求描述”，由浏览器 `POST /api/chat?ws=1` 并回传 receipt。创建 slot 的顺序固定为 create → title → project → agent（`agent_kind:"template"`），**agent 必须在 project 之后设置**，否则 host 拒绝 |
| **后端用 app token 发请求** | devteam | 读取 `<app_dir>/.app_secret`，带 `X-App-Secret` 头调 `POST /api/apps/<app>/token` 换 token，放在 `?token=` 里。token 有效期 5 分钟，每 240 秒主动续一次。只有响应带 `X-Auth-Required` 时才重试，handler 自己返回的 403 就是最终结果。只连 loopback 地址（`dev-team/backend/devteam/gateway.py`） |

devteam 踩过的 chat API 坑，写编排时一定要防：

1. **`POST /api/chat` 会把不存在的 key 默默建成一个空白会话**，刚关掉的 key 也会。发送前先查 slot 列表；每次都带上 `agent`，并记下 slot 的 `created` 时间戳，时间戳变了就当作另一个会话。
2. App 发的消息只会排队，不能插到当前回合里。而且排队后只保留 `sendId`，所以要靠 `sendId` 加正文里的标记来识别自己发出的消息。
3. **防重复派发**：每次派发前先把状态记成 `dispatching` 写盘，重启后到 transcript 里找 `sendId` 和标记，找到了就不再重发。kirocrew-team 的做法是在任务首行打上 tag，找到 tag 就接管已存在的会话。
4. `reset-conversation` 不传 `replay` 时默认是 `true`，要**显式传 `replay: false`**。
5. 第一次 `/stop` 是软停止，要先取消自己排队的消息再 stop。被打断的回合用 `/continue` 续一次。
6. 回合完成的判断：连续两次轮询都是 idle 才算完。超时只计算“看到它在工作”的时间。忙了 20 分钟 transcript 却没有新行，就判为卡住。
7. session 在连续三次列表里都不见了才算丢失。**不要自动重建**，交给人来决定。

### 4.3 替 session 审批工具调用（`sessionApproval`）

- 必须做**归属校验**：只对 `app=<自己>`、key 已登记、`created` 时间戳也对得上的 slot 回答审批。devteam 把这个约束封装成一个只能在满足条件时构造出来的 `OwnedSlot` 类型（`dev-team/backend/devteam/guard.py`）。
- **YOLO 模式或者 trusted slot 会让审批根本到不了 App**：host 在路由给 App 之前就已经放行了。devteam 实测 19 次调用一次也没到。所以开跑前要读 `/api/admin/compliance/yolo-status`，开着就拒绝运行，运行中途打开就暂停。trusted 的 session 用 `mode=normal` 重置。
- 写文件的审批，`tool_input` 是渲染好的 unified diff，`tool_kind` 为空。要解析 `---`/`+++` 行，**不能当 JSON 读**。
- 审批记录一旦被回答就会从列表里消失，所以待审批时就要抓下 `tool_call_id` 和时间戳。180 秒没人回答会被自动拒绝。
- kiro-cli 重启后 request id 会从头编号，去重要用 request id 加 tool、kind、input 一起比较。
- 顺序：先把决定写盘，再回答，最后写一条 `ctx.audit.record(...)`。

### 4.4 `agents/*.json` 的规矩

- 安装时文件被复制成 `~/.kiro/agents/<app>--<file>.json`。`name, mcpServers, tools, allowedTools, prompt, managedToolPolicy, includeMcpJson, resources` 这几个字段每次注册都会重新生成，其余字段保留。
- **prompt 里只要出现 `${VAR}` 或者 `{UPPERCASE}` 这样的占位符，agent 就会被静默跳过，不注册**。devteam 用 `test_agent_specs.py` 专门拦这个问题。
- agent 的权限可以设计成：`tools` 宽、`allowedTools` 窄（只读），写入和 shell 用 `permissions.rules` 设为 `"ask"`，这样每次都会到 App 这里审批。注意：`permissions.rules` 对 kiro-cli 不生效（`dev-team/product-design.md`）。
- 有大量 agent 时用脚本生成，并且从后端常量 import 端口和路径，保证 prompt 和策略引擎一致（`dev-team/scripts/gen_agents.py`）。
- agent 在用户仓库里跑时，KiroCrew 会写 `.kiro/settings/cli.json`，要让用户的 `.gitignore` 包含 `.kiro/`。
- 跑的 agent 的 cwd 必须在 `~/workplace` 或 `~/workspace` 之下（kirocrew-team 的 `constants.py:9-11`）。

### 4.5 其他 SDK

| SDK | 用途 | 注意 |
|---|---|---|
| `ctx.cron` | 定时触发 agent 回合 | 在 handler 和 `on_startup` 里**必须用 `*_async` 版本**，同步版本会抛 `CronSyncOnLoopError`。cron 不能回调 App 代码 |
| `ctx.job` | 用户在前台发起的长任务，有 dedupe，可取消 | 在 `on_startup` 里 `register(kind, fn)`。`fn` 跑在线程里，要定期调 `handle.checkpoint()`。记录里没有结果，结果需要自己存 |
| `ctx.events` | 发布事件 | 发布没声明过的类型会抛 `PermissionError`。前端收到的类型是 `app_event`，数据会被脱敏。目前转发不到 App 前端（§3.8） |
| `ctx.audit` | 写安全事件日志（SEL） | 永远不会抛异常。SEL 只保留 1000 条，别刷屏：devteam 的 `kcapi.sh` 曾经每次请求都调一下 `/auth/me`，SEL 行数翻了一倍 |
| `ctx.scrub.outbound(text)` | 对外发布前去掉凭证和 URL | 抛 `PlatformCompositionError` 时就不要发布了 |

App **不能**挂 agent 生命周期 hook（`userPromptSubmit/preToolUse/...`），这些只能由 operator 在 `agent.kiro_hooks` 里配置。

---

## 5. 独立进程后端要点

- 环境变量：`PORT`、`KIROCREW_APP_NAME`、`KIROCREW_PROXY_SECRET`、`KIROCREW_HOME`。
- 每个请求都要校验 `X-KiroCrew-Proxy: <ts>:<hmac>`：`from kiro_crew.apps.proxy_auth import verify_proxy_request`，允许 ±60 秒的时间偏差，校验失败就拒绝。**`/health` 不签名**，因为 gateway 的探活请求本身不带签名。
- 代理会去掉 `cookie` 和 `authorization` 头。总超时 30 秒。SSE 能透传，WebSocket 不行（`upgrade` 头会被去掉）。
- 需要超过 30 秒，或者要给外部调用方开 API，可以另开一个 loopback 端口（workshop-customizer 的 `/v1` 在 8772 端口）。
- 解释器的选择顺序：gateway 的 Python 已经有 deps 树时直接用它；否则用 `<app>/.venv/bin/python3`，前提是能证明它可用且 Python 版本一致；再不行回到 gateway 的 Python。gateway 不会帮你创建 venv。缺依赖时，也可以像 workshop-customizer 那样自己 re-exec 到仓库的 `.venv`。

---

## 6. 前端

### 6.1 构建配置（照抄即可）

```ts
// ui/vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    lib: { entry: 'src/main.tsx', formats: ['es'], fileName: () => 'index.mjs' },
    outDir: 'dist', emptyOutDir: false, cssCodeSplit: false, sourcemap: false, target: 'es2022',
    rollupOptions: {
      external: ['react', 'react-dom', 'react-dom/client', 'react/jsx-runtime',
                 '@kirocrew/app-sdk', '@kirocrew/app-sdk/ui', 'lucide-react'],
      output: { assetFileNames: 'style[extname]' },
    },
  },
})
```

- React 放 devDependencies 或 peerDependencies。**bundle 里出现第二份 React，所有 hook 都会坏**。`dev-install.sh` 里要 grep bundle 中的 React 内部符号，发现了就失败。
- `main.tsx` 的 **default export 就是页面组件**，不接收 props。一个 App 想要多个视图，用 `?view=` 在组件里切换即可（devteam 就这么做），不必注册多个 page。
- `ui/dist/` **要提交进 git**。安装时会跳过 `node_modules`，只有 `package.json` 在 App 根目录时才会自动构建。

### 6.2 运行时注意事项（都是实测踩出来的）

1. **host 不会加载 App 的 CSS**。在入口处自己插入 `<link href="/apps/<app>/ui/dist/style.css">`（`aidlc-studio ui/src/lib/host.ts:154-165`）。样式只用 host 的 CSS 变量。CI 里 grep bundle，发现硬编码的十六进制颜色就失败。
2. **只能 import host 真正 export 的名字**。具名 import 一个不存在的名字，链接阶段直接 SyntaxError，整个页面白屏。
   - 在 `ui/types/kirocrew-app-sdk.d.ts` 里只声明真正存在的名字。
   - lucide 只有大约 40 个具名 export，其他图标统一走 default export 的 Proxy。
   - 加一个测试：把源码和 bundle 里的具名 import 跟**本机安装的 host vendor 包的 export 列表**比对（devteam `test_ui_imports.py`、workshop-customizer `test_ui_static.py`）。
3. **调 API**：`useAppApi()` 只接受绝对路径，并且必须落在 `permissions.api` 声明的前缀里。相对路径会被解析到 `http://localhost`，悄悄变成错误的 URL。SDK 抛出的错误是字符串 `API <status>: <body>`，要统一解析（`aidlc-studio ui/src/lib/api.ts`）。读请求可以带 `signal` 取消；写请求不应该取消，因为服务端可能已经提交了。
4. **主题和语言**：SDK 的 `useTheme().mode` 在非默认主题下不准，`useNotify` 和 `useAppEvents` 都不触发。用 MutationObserver 监听 `<html data-mode>` 和 `<html lang>`，再配合 `useSyncExternalStore`（`aidlc-studio ui/src/lib/host.ts:1-104`）。语言优先级依次是：App 内的设置、`localStorage['mc-lang']`、`<html lang>`、浏览器语言。
5. **i18n**：import map 里没有 i18next，只能自己写一个轻量的翻译器，再用脚本检查各语言的 key 是否对齐（`scripts/build_i18n.py --check`）。
6. 侧边栏的徽标用 `useNavBadge`（devteam 在用）。
7. 后端和 UI 共用的常量（错误码、枚举）用脚本生成到 UI 侧，并在 CI 里检查是否过期（`scripts/gen_ui_sources.py`）。手写两份迟早会对不上。

### 6.3 前端测试

vitest + jsdom。把 `@kirocrew/app-sdk`、`@kirocrew/app-sdk/ui`、`lucide-react` 用 alias 指向 `src/test/stubs/*`（`aidlc-studio ui/vitest.config.ts`）。kirocrew-team 另外配了一份 `vite.preview.config.ts`，把 React 打进包里并内联脚本，产物能直接用 `file://` 打开预览。

---

## 7. 开发循环

### 7.1 鉴权：`kcapi.sh`

gateway **不支持 `Authorization: Bearer`**，只认 cookie。流程是：用 gateway 自己的 Python 跑 `python -m kiro_crew token`，签出一个 5 分钟有效的 link token；用它访问 `/?token=` 换成 session cookie，存进 cookie jar 复用。

```bash
scripts/kcapi.sh GET  /api/apps
scripts/kcapi.sh POST /api/apps/<app>/enable '{}'
```

- 桌面版 Python 的路径：`/Applications/KiroCrew.app/Contents/Resources/backend-dist/kirocrew-backend-arm64/bin/python3.12`。可以用 `KC_PY` 覆盖。Docker 和 Linux 上直接用 `python3`。
- 先检查 cookie jar 是否还有效，再决定要不要重新签 token，不要每次都调一遍 `/api/auth/me`。

### 7.2 安装与更新：`dev-install.sh`

```
build UI → node --check dist/index.mjs → （grep bundle：不能带 React）
已安装 ? POST /api/apps/<app>/disable + POST /api/apps/<app>/update {"source": "<abs path>"}
       : POST /api/apps/install {"source": "<abs path>"}
POST /api/security/trusted-apps/<app>      # 只信任这一个 App
POST /api/apps/<app>/enable
[--dev] POST /api/apps/<app>/dev {"enabled": true}
GET /api/apps（看 hooks、backend_status） + GET /api/apps/<app>/health
```

- **永远不要 uninstall**，那会把 trust 授权一起删掉。一定要先 disable 再 update，因为 hook 模块只在 enable 时重新 import。
- 有首次 enable 时需要用户同意的权限（比如 `sessionApproval`），加 `--consent-session-approval` 之类的参数一并提交。
- 安装后的代码是一份**完整拷贝**，放在 `~/.kiro/crew/apps/<app>/`，`installed.json` 里的 `source` 记录源目录。改完源码不重新安装，gateway 里跑的还是旧代码。

### 7.3 UI 热更新：dev mode

- 打开后 `ui/` 下的文件以 `no-store` 方式返回。系统每秒轮询一次 `ui/` 目录，有变化就通过 WebSocket 推送 `app_reload`。
- 要实现“改完即生效”，只能**把整个 `ui/` 目录 symlink 到源码**。单个文件做 symlink 会 404，因为打开文件时用了 `O_NOFOLLOW`。指向安装目录之外时，CLI 需要加 `--confirm-out-of-install-root`，而且必须在宿主终端里执行，不能在 agent 沙箱里跑。
- **后端改动不会热更新**，要重新跑 `dev-install.sh`。

### 7.4 排查清单

| 现象 | 先看什么 |
|---|---|
| 页面白屏 | 浏览器控制台有没有 `SyntaxError: ... does not provide an export named`，属于 §6.2 第 2 条 |
| 页面没有样式 | 有没有插入 `<link>` 引入 style.css |
| 路由 404，或 App 显示 enabled 但不工作 | `GET /api/apps` 里的 hooks 状态；`agent.apps_trusted` 里有没有这个 App；`register_routes` 是否抛了异常 |
| 改了代码不生效 | 有没有 disable 再 update；有没有残留的 namespace（bootstrap 里有没有 teardown） |
| gateway 卡顿或被杀 | 事件循环上有没有同步 IO；hook 是不是写成了同步函数 |
| `ctx.spawn` 是 None | `permissions.spawn` 是不是字面量 `true` |
| agent 没注册上 | prompt 里有没有 `${}` 或 `{UPPER}` 占位符；JSON 是否合法 |
| 重启后 App 报 repo moved | macOS 重新挂载后设备号会变，`st_dev:st_ino` 跟着变了，需要重新绑定（aidlc-studio 的情况） |

---

## 8. 测试与质量门

### 8.1 后端测试

- `backend/<pkg>/` 尽量写成**不依赖 host 的纯 Python**。`conftest.py` 把 `backend/` 加进 `sys.path`，或者像 aidlc-studio 那样**按 host 的方式按路径加载入口文件**，并构造一个真实的 `AppContext`（`aidlc-studio tests/conftest.py`）。
- 运行时用 KiroCrew 的 venv：KiroCrew 源码 checkout 里的 `.venv/bin/python -m pytest tests -q -o addopts=`。此外再用桌面版自带的 Python 3.12 import 一遍后端，确认真实运行时能加载。
- **伪造 gateway 要贴近真实行为**。devteam 的 `tests/fakes.py` 用真实 transcript 截取的数据作为行格式，并模拟了几处真实行为：写入审批是 diff 文本；180 秒自动拒绝；队列 replay 后只保留 sendId。
- **契约测试**：读本机安装的 KiroCrew 源码，对 AST 做断言，并钉住 `CONTRACT_KIROCREW_VERSION`。host 版本一变，测试就失败，提醒你重新核对（`dev-team/tests/test_contract_kirocrew.py`）。
- **对真实 gateway 做冒烟**：`dev-team/scripts/contract_live.sh`。前置条件是 `/api/ready` 返回 200、版本匹配、当前没有运行中的任务、YOLO 关闭。结果写进 `docs/verification/evidence/`。
- 一些便宜的护栏测试：manifest 不变量（`test_manifest.py`）、AST 层面禁止某些调用（`test_static_policy.py`）、agent prompt 不能含占位符、UI import 必须在 host export 列表里、主题对比度检查（devteam `test_ui_contrast.py`）。

### 8.2 `scripts/check.sh`（aidlc-studio 的 11 道门）

1. 用 host 自己的 `AppManifest.validate` 校验 app.json，并检查发布用的素材都存在
2. 校验 payload 摘要（适用于 vendor 了第三方内容的 App）
3. 用 KiroCrew 的 venv 跑 pytest
4. 用桌面版自带的 Python 3.12 import 后端
5. 检查生成的 UI 源码有没有过期
6. `build_i18n.py --check`
7. `tsc --noEmit`
8. `vitest run`
9. `npm run build`
10. `node --check dist/index.mjs`
11. bundle 内容检查：不能有硬编码的十六进制颜色、不能有 `dangerouslySetInnerHTML`、不能带 React

新 App 至少保留 1、3、4、7、8、9、10、11 这几道。

### 8.3 证据化验收

“能用”要拿证据说话：`docs/verification/*.md` 里每一条结论都对应 `evidence/` 下的一个文件（截图、activity 记录、pytest 输出）。只有 owner 本人能做的步骤，比如真实点 gate、切换 YOLO、给出同意，单独列一份 `user-checklist.md`。devteam 第一次完整跑下来，发现指标和报告都偏乐观、成本是估算的 3.7 到 7.4 倍。这些都是靠证据才暴露出来的。

### 8.4 非标准布局

App 放在子目录时，比如 workshop-customizer 在 `app/` 下，有两种装法：registry 条目里写 `subdirectory: app`，或者 Install-from-Path 直接指向 `app/`。安装后要找同一仓库里的 engine，按这个顺序查：环境变量、`APP_DIR.parent`、`installed.json` 的 source（registry 安装时对应 `~/.kiro/crew/app-sources/<name>`）。查到的目录里有 `template-lock.json` 才算找对。

---

## 9. 发布

1. 修改 `app.json` 的 `version`（如果后端常量里也有版本号，一起改），合入 main，打 tag `vX.Y.Z`，在 GitHub 上建 Release。
2. 把 `release` 分支 fast-forward 到这个 tag。仓库根目录的 `app-registry.json` 指向 `release`，这样仓库本身就是一个自托管 registry：
   ```json
   [{ "name": "aidlc-studio", "gitUrl": "https://github.com/warren830/kirocrew-app-aidlc",
      "repo": "https://github.com/warren830/kirocrew-app-aidlc", "branch": "release" }]
   ```
3. **团队 registry**：就是本仓库（`main` 分支）的 `app-registry.json`，收录了我们所有 App。新 App 按 [README 的 Add an app](../README.md#add-an-app) 加一条。
4. **官方目录**：`kirodotdev/KiroCrewApps` 公开仓库，从 fork 发 PR。条目格式为 `{name, source{git,url,ref}, author, categories, note(≤300 字)}`，`ref` 固定到某个版本，每次更新都要重新改这个 ref。从 fork 发起的 CI 要等维护者批准才会跑。

发布时的坑：

- **仓库要小**。安装用 `git clone --depth 1`，超时只有 60 秒（`$K/registry.py:134`）。aidlc-studio 曾经因为 24 MB 截图导致装不上。大文件不要放在 App 根目录。
- **图片只接受 PNG/WebP/JPEG**，SVG 会被丢掉。尺寸：icon 512²、不透明；hero 1600×900；detail banner 1600×384；截图横版，亮色和暗色各一套。
- **PR 分支名不能以 `release/` 开头**，因为已经有一个叫 `release` 的分支，git ref 会冲突。改用 `cleanup/x`、`aidlc-2.10.0` 这样的名字。
- gateway 克隆第三方 App 时会**清掉代理环境变量，也不读 ~/.gitconfig**。代理要开 TUN / 全局模式才能连上 GitHub，见 [README 的网络要求](../README.md#network-requirement-kirocrew-must-reach-github-directly)。gitlab.aws.dev 不能作为 App 的来源。
- 用脚本调 `PUT /api/apps/registries` 会**整体替换**列表，先 GET 一份再修改。

---

## 10. 新 App 开工清单

- [ ] 选好后端形态（§1）。默认选 gateway 进程内 hooks
- [ ] 复制 devteam 的骨架：`backend/{routes,hooks}.py` 加 bootstrap，`backend/<pkg>/{service,handlers,httpkit,store}.py`
- [ ] app.json：name 用 kebab-case 且不带点号；权限布尔值写字面量 `true`；`api` 只写普通前缀；路由不用保留路径段
- [ ] 所有路由都套 `guarded()`（owner 校验、拒绝 app token、`{error, code}` 格式），`/health` 不加
- [ ] `on_startup` 只做“打开存储 + `create_task`”；`on_shutdown` 依次 cancel、flush、close，再 teardown namespace
- [ ] 所有 IO 走 `to_thread`；状态重要就用 SQLite 或原子写
- [ ] 调 AI：spawn 用的 agent 只给最少的 tools；驱动 chat 时要防重复派发、防凭空建出 session、显式传 `replay:false`
- [ ] UI：Vite lib 配置（external）、default export、插入 CSS、用 DOM 判断主题和语言、只 import host 真实 export 的名字
- [ ] 测试：伪造 gateway 照真实行为写；manifest、agent、UI import 的护栏测试；host 版本契约测试
- [ ] 写好 `scripts/{kcapi,dev-install,check}.sh`，机器相关路径（Python、Node、允许的 repo 根目录）都能用环境变量覆盖
- [ ] 提交 `ui/dist/`；素材用 PNG；仓库控制在几 MB 以内
- [ ] 发布：打 tag、更新 release 分支、在团队 registry 加条目

---

## 附：参考位置

| 主题 | 位置 |
|---|---|
| manifest 字段与校验 | `$K/manifest.py:2320-2348`（字段表）、`:661-891`（backend、permissions、setup） |
| 后端加载 | `$K/module_loader.py:256-362` |
| ctx 定义 | `$K/context.py:54-214` |
| 路由挂载 | `$K/route_registry.py:26-275` |
| spawn 的限制 | `$K/spawn_sdk.py:77-206` |
| 执行授权（trust） | `$K/execution.py:354-721` |
| hook 自动重载 | `$K/hook_reconcile.py:92,478-545` |
| dev mode | `$K/dev_mode.py` |
| 骨架模板 | devteam：`backend/routes.py`、`backend/hooks.py`、`backend/devteam/{service,httpkit,store,gateway,guard}.py` |
| 多 session 编排 | `dev-team/backend/devteam/orchestrator.py` |
| UI 的 host 适配 | `kirocrew-app-aidlc/ui/src/lib/{api,host,sse}.ts`、`ui/vite.config.ts`、`ui/vitest.config.ts` |
| 质量门与脚本 | `kirocrew-app-aidlc/scripts/{check,dev-install,kcapi}.sh` |
| 发布流程 | `kirocrew-app-aidlc/docs/publishing.md` |
| 独立进程 + HMAC | `kirocrew-app-adlc/app/backend/server.py:2639-2654`；`$K/proxy_auth.py` |
| 平台能力缺口（21 条） | `dev-team/product-design.md:605-626` |
