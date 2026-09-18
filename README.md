# OpenClaw 视觉能力（读图 / 画图）双链路实测 · 从零到可用

> **一句话**：让跑在你 NAS 上的 OpenClaw Agent 真正「长眼睛」——能看懂你发的图片，也能按图改图；并且**局域网和外网两条路都能用**。
>
> **最终效果**：给 Agent 一张图，它 16 秒内准确说出「绿色圆形、紫色方形、青色三角形，编码 QX-7391」；再让它「把红圆改成紫色」，它真的吐出一张改好的图。
>
> **本文档已脱敏**，所有 IP / 域名 / 密钥 / 路径用户名均为占位符，可直接公开分享。

| 项目 | 值 |
|---|---|
| 文档类型 | 实战教程 + 排错手册（非报告体） |
| 实测时间 | 2026-09-18 夜 ~ 2026-09-19 凌晨 |
| 实测结论 | 修复后 **LAN 与 CF 两条链路读图均一次通过**；图生图两条链路均通过 |
| 最关键的坑 | 读图失败**与网络无关**，是 OpenClaw「视觉开关」两处配置缺失 |
| 最容易踩的坑 | CF 链路首次「读图成功」是**假阳性**——Agent 从别的会话抄了答案 |

---

## 一、它能做什么 / 不能做什么

### ✅ 能做

| 能力 | 说明 |
|---|---|
| **读图（识别图片内容）** | 你通过聊天附件或文件路径给它一张图，它能说出图中的形状、颜色、文字、布局 |
| **按图改图（图生图）** | 「把红圆改成紫色，其他不变」→ 输出改好的新图 |
| **文生图** | 「画一只戴墨镜的猫」→ 直接出图 |
| **双链路可用** | 在家走局域网；出门在外走 Cloudflare 隧道，**能力不打折** |

### ❌ 不能 / 注意

| 限制 | 说明 |
|---|---|
| 依赖上游账号配额 | 底层是 Gemini 免费额度经本地反代，**配额耗尽会 503**（如 `gemini-3-pro-image` 系列已耗尽） |
| 图片走 base64 上传 | 大图（数 MB）会让请求体明显变大，建议先压到 1 MB 内 |
| 外网链路有 CF Access 前置 | 先在浏览器过 Cloudflare 身份校验，脚本调用需带 Access 凭据头 |
| 不是「本地模型」 | 都是云端模型，NAS 只做网关与协议转换，**不吃 NAS 算力** |

---

## 二、为什么我要做这件事（动机）

这一节是全文最重要的部分——**不讲清楚「为什么」，后面的技术选择都像是瞎折腾**。

### 2.1 为什么必须让 Agent「能读图」

Agent 的价值在于**替你操作真实世界的东西**。而真实世界里大量信息是**图片**，不是文本：

- 软件报错的**截图**
- 网页/App 的 **UI 界面**（要它帮忙操作，它得先看见按钮在哪）
- 设计稿、流程图、手写笔记
- 文档里的**表格截图**、PDF 扫描件
- 别人发来的**图片消息**

**一个不能读图的 Agent，等于半个瞎子**：你跟它说「看这张图，帮我改一下」，它只能回你「我看不到」。这个能力不补齐，它在「多模态」这条主赛道上就是废的。

### 2.2 为什么要分「局域网 + 外网」两条链路测

| 链路 | 场景 | 如果只有它可用会怎样 |
|---|---|---|
| 局域网（直连 NAS） | 在家、在办公室 | Agent 被**绑死在局域网内**，出门就是废铁 |
| Cloudflare 隧道 | 咖啡馆、外地、手机热点 | 日常在家反而绕远路、多一跳、更慢 |

所以**两条都必须通**。这里还有个隐藏考点：两条链路的**鉴权路径完全不同**（见第六节），一条通了不代表另一条通。

### 2.3 为什么我必须「修好它」，而不是「绕过去」

第一次测试时，CF 链路「看起来成功」了。但我导出会话记录后发现：

> 附件**根本没被注入** Agent，工具在报错，Agent 转头用 `sessions_search` **从另一个会话里我自己写下的图片文字描述抄到了答案**。

这件事的意义远超一次测试：

- 如果我没查，我会以为「CF 链路读图没问题」→ **错误结论写进文档** → 下次照着做 → 翻车
- 更危险的推论：**一个会用历史会话「猜」答案的 Agent，你怎么知道它哪次回答是真看见了、哪次是猜的？**

所以这里确立了本文最硬的一条纪律：

> **测「感知类」能力时，测试对象的真实内容绝不能出现在任何提示词、任何历史会话里。**

这不是洁癖，这是**唯一能让结论可信的方法**。

### 2.4 为什么把 Agent 跑在 NAS 上而不是主力 PC

| 维度 | NAS（飞牛 fnOS） | 主力 PC |
|---|---|---|
| 常驻 | **7×24 开机**，本来就常年不关 | 用完就关，睡了 Agent 就死 |
| 功耗 | 35 W TDP 的老 APU，整机约一杯奶茶钱/月 | 独显整机，待机功耗高一个量级 |
| 占用 | 不抢你的显卡和内存 | 抢（尤其跑本地模型时） |
| 定位 | Agent 是**长期在线的服务**，天然属于 NAS | 主力 PC 是**交互工具** |

**结论**：Agent 是「服务」，不是「软件」。服务就该跑在服务器上。

### 2.5 为什么要走本地反代（Antigravity-Manager），而不是直连官方 API

- **成本**：走本地反代可以用上 Gemini 的免费额度，直连官方 API 是按 token 计费的
- **免 Cookie**：Antigravity-Manager 用 OAuth 方式接账号，**不需要维护浏览器 Cookie**，比更早的「网页版反代」方案稳得多
- **统一出口**：局域网上所有 AI 客户端（WorkBuddy、OpenClaw、ChatBox）都指向它，**换模型/换账号只改一处**
- **可控**：能本地加健康检查、能看日志、能限流

### 2.6 为什么外网走 Cloudflare 隧道 + Access，而不是路由器端口转发

家庭宽带网关**不允许关闭防火墙**，也没有公网可用的稳定入口。Cloudflare 隧道是**反向连接**：

- **不需要开放任何入站端口**（NAS 主动连出去，不是外面连进来）
- 前面挂 **Cloudflare Access**（Zero Trust 身份校验），**未过验证的请求直接挡在边缘**，服务本体不裸奔
- 自动 HTTPS，不用自己续证书

**代价**：多一跳，延迟比自己架隧道高一些（实测读图 30 秒 vs 局域网 16 秒，可接受）。

---

## 三、前置条件与完整版本号清单

> 全部为**实机采集**（SSH 命令 / 系统查询），非推测。采集时间 2026-09-19 01:00 前后。

### 3.1 硬件 —— NAS 侧（服务端）

| 项目 | 实测值 |
|---|---|
| 机型 | 蜗牛星际 **D 款**（4 盘位，全金属小方箱） |
| 主板 | XWC-A78（AMI BIOS 4.6.5 / 2021-07-31） |
| CPU | **AMD A8-5550M APU**，4 核 4 线程，1.4–2.1 GHz，Richland（Family 15h），TDP 35 W |
| 内存 | DDR3-1600 **8 GB** 单条（2 槽，另 1 槽空），实测可用 6901 MB |
| 核显 | AMD Radeon HD 8550G（`/dev/dri` 可用；**不支持 HEVC 硬解**） |
| 系统盘 | 128 GB SATA SSD（G535M3/128GMN） |
| 数据盘 | 500 GB（ST500DM002）+ 14 TB（WUH721414ALE6L0）+ 3 TB（ST3000NM0033），**全部单盘 RAID1 零冗余** |
| 网卡 | 板载 Realtek RTL8111 千兆（主用）+ USB RTL8153 千兆（备用） |
| 关键约束 | **CPU 是 BGA 焊死，不可升级**；AHCI 仅 4 口且已占满；无 M.2/NVMe 槽 |

### 3.2 硬件 —— 本机 PC 侧（客户端 / 操作台）

| 项目 | 实测值 |
|---|---|
| 系统 | **Windows 11 专业版**，Version 10.0.26200，Build 26200，64 位 |
| CPU | **AMD Ryzen 7 5700X** 8 核 16 线程，基准 3401 MHz |
| 内存 | 15.9 GB（Samsung 8 GB ×2，DDR4-2933） |
| 显卡 | **NVIDIA GeForce RTX 5060 Ti**，驱动 **610.47**，显存 16311 MiB，VBIOS 98.06.4e.40.b6，**Compute Capability 12.0（Blackwell）** |
| 虚拟显示 | GameViewer Virtual Display Adapter（串流用） |

### 3.3 软件 —— NAS 侧（**本文核心**）

| 组件 | 版本 | 说明 |
|---|---|---|
| 系统 | **fnOS 1.2.0604** | 飞牛私有云 |
| 底层 | **Debian GNU/Linux 12 (bookworm)** | |
| 内核 | **6.18.18.c1032-trim** | 飞牛定制内核 |
| systemd | 252 (252.39-1~deb12u1) | |
| Docker | **28.5.2** (build ecc6942) | |
| Docker Compose | **v2.40.3** | |
| **OpenClaw** | **2026.9.4 (3a9d69d)** | 本次主角 |
| Node.js（跑 OpenClaw） | **v26.8.1** | |
| npm | 11.19.0 | |
| Python | **3.11.2** | NAS 上跑脚本用 |
| nginx | **1.22.1** | 反代 / HTTPS |
| Tailscale | **1.102.4** | 组网 |
| Mihomo Meta | **v1.19.30** linux amd64 / go1.26.6 / with_gvisor | 进程方式跑，非容器 |
| **Antigravity-Manager** | **v4.7.1**（镜像 `lbjlaq/antigravity-manager:v4.7.1`） | **生图/读图上游** |
| Cloudflared | **2026.9.1**（built 2026-09-11） | 隧道客户端 |
| New API | **v1.0.0-rc.37** | 聚合网关 |
| AstrBot | **4.28.0** | QQ 机器人 |
| 工作目录 | `/volX/openclaw-data/workspace/` | `/vol4` 466 G，已用 129 G（28%） |

**⚠️ 一个值得记录的版本不一致**：`openclaw --version` 报 **2026.9.4**，但 systemd 单元里的 `OPENCLAW_SERVICE_VERSION` 与 `Description` 仍写着 **v2026.7.1**（升级时没同步更新单元文件）。**排查时以 `openclaw --version` 为准**，别被单元文件误导。

### 3.4 软件 —— 本机 PC 侧

| 组件 | 版本 |
|---|---|
| WorkBuddy | **5.5.6.0** |
| Node（WorkBuddy 托管） | v22.22.2 / v22.22.3 / **v26.8.1** |
| Node（系统） | v25.2.1 |
| Python（托管） | **3.13.14** |
| Google Chrome | **153.0.8010.53** |
| git | 2.55.0.windows.3 |
| gh CLI | 2.101.0 (2026-09-15) |

### 3.5 账号 / 凭据前置

| 项 | 需要什么 |
|---|---|
| Google 账号 | 已登录 Antigravity（反重力），提供 Gemini 额度 |
| Cloudflare | 已建隧道 + Access 应用（Zero Trust）|
| NAS SSH | 用户名密码登录（本文用 paramiko 免交互） |

---

## 四、用到了哪些项目（技术栈地图）

| 层 | 项目 | 角色 | 来源 |
|---|---|---|---|
| Agent 框架 | **OpenClaw** | 跑 Agent、管模型、管工具、注入图片 | 自建仓库 `openclaw-fnOS-guide` |
| 上游模型网关 | **Antigravity-Manager** | 把 Google 账号额度转成 OpenAI 兼容 API（**:8045**） | 第三方开源 `lbjlaq/antigravity-manager` |
| 账号来源 | **Google Antigravity（反重力）** | 提供 Gemini 免费额度 | Google 官方 |
| 对话模型 | **gemini-3.8-flash-high** | 主模型 + **本次指定的读图模型** | 经 Antigravity-Manager |
| 生图模型 | **gemini-3.1-flash-image** | 文生图 / 图生图 | 经 Antigravity-Manager |
| 生图 Skill | **gemini-image-gen** | Agent 调用生图的封装 | 自建仓库 `antigravity-manager-image-gen` |
| 内网穿透 | **Cloudflare Tunnel (cloudflared)** | 不开放端口让外网访问 NAS | 自建仓库 `cloudflare-tunnel-guide` |
| 身份校验 | **Cloudflare Access (Zero Trust)** | 外网请求前置鉴权 | Cloudflare 官方 |
| 容器 | **Docker / Docker Compose** | 跑各服务 | |
| NAS 系统 | **飞牛 fnOS** | 宿主系统 | 飞牛官方 |
| 反向代理 | **nginx 1.22.1** | HTTPS / 站点 | |
| 代理分流 | **Mihomo Meta** | 出网代理，让上游可访问 | 自建仓库 `mihomo-stack-guide` |
| 组网 | **Tailscale** | 异地组网备份通道 | 自建仓库 `tailscale-mesh-guide` |
| 自动化 | **paramiko** | 本机 SSH 驱动 NAS（免交互） | Python 库 |
| 聚合网关 | **New API** | 多上游统一入口（:3001） | 自建仓库 `newapi-gateway-ops` |
| 本仓库前身 | `gemini-reverse-proxy-nas` / `antigravity-manager-gemini-relay` | 两个反代方案，已整合进 `infra-ai-gateway` | 自建 |

> **整合说明**：`gemini-reverse-proxy-nas`（网页版 Gemini 反代）与 `antigravity-manager-gemini-relay`（Antigravity-Manager 反代）这两个早期项目，
> 现已作为 `infra-ai-gateway` 仓库的 `01_Gemini反代与生图服务/` 与 `02_Antigravity流式协议转换/` 两个板块存在。
> **本文是第三个板块 `05_OpenClaw视觉能力与双链路验证/`**，三者共用同一套 NAS 反代基座。

---

## 五、快速开始（3 步跑通）

假设你已经有一台跑着 OpenClaw 的机器，和一个 OpenAI 兼容的图像/对话上游。

### 第 1 步：声明模型「会看图」

编辑 `~/.openclaw/openclaw.json`，给视觉模型加上 `input`：

```jsonc
{
  "models": {
    "providers": {
      "antigravity": {
        "models": [
          {
            "id": "gemini-3.8-flash-high",
            "input": ["text", "image"]        // ← 关键：声明支持图像输入
          }
        ]
      }
    }
  }
}
```

**预期输出**：文件可被 JSON 解析，模型条目里出现 `"input": ["text","image"]`。

### 第 2 步：指定「读图用哪个模型」

```jsonc
{
  "agents": {
    "defaults": {
      "imageModel": {
        "primary": "antigravity/gemini-3.8-flash-high",
        "fallbacks": ["antigravity/gemini-3.7-flash"]
      }
    }
  }
}
```

### 第 3 步：重启网关并验证

```bash
systemctl --user restart openclaw-gateway.service
sleep 12                                  # ← 必须等，见 FAQ Q1
ss -tlnp | grep ':9090'                   # 确认在监听
```

**预期输出**：`LISTEN 0 511 0.0.0.0:9090 ...`，进程名 `node-MainThread`。

---

## 六、详细步骤与「我怎么发现问题、怎么解决」

本节按**问题驱动**组织——每个问题都写清：**现象 → 我怎么查 → 查到什么 → 怎么修 → 怎么验证**。

### 问题 1：局域网读图失败（核心问题）

#### 现象

通过网关发一张带附件的消息，让它描述图片：

- 它调用 `view_image` → 报错：
  ```
  No image model is configured. Set agents.defaults.imageModel
  or configure an image-capable provider.
  ```
- 换用 `read xxx.png` → 返回：
  ```
  [Current model does not support images. The image will be omitted from this request.]
  ```
- 最终它只能**含糊其辞**或猜。

#### 我怎么查（排查顺序很重要）

**第一步：先怀疑上游，还是先怀疑配置？**

我的排查顺序是 **从外向内逐层排除**：

| 层 | 怎么验 | 结果 | 结论 |
|---|---|---|---|
| ① 网络 | 从本机 ping / 连 NAS 端口 | 通 | 排除 |
| ② 上游反代 | 在 NAS 上**直连 8045** 用同一模型读同一张图 | **HTTP 200，精准识别出「红圆/蓝方/绿三角/OC-9527」** | **上游完全正常** → 问题在 OpenClaw |
| ③ OpenClaw 配置 | 导出配置结构逐字段看 | 见下 | **找到根因** |

> **这一步是关键分水岭**。如果我在第②步就发现上游也读不了图，那方向完全相反（要去修反代/账号）。
> **先用一个「已知输入」打穿上游**，能把问题锁死在某一层，避免瞎猜。

**第二步：导出配置，逐字段比对**

把 NAS 上的 `openclaw.json` 结构导出来看，发现两处都不对：

| 检查项 | 实测 | 应该 |
|---|---|---|
| 各 provider 下模型条目的字段 | 只有 `contextWindow` / `id` / `maxTokens` / `name` / `reasoning` | **缺 `input:["text","image"]`** |
| `agents.defaults.imageModel` | **`null`** | 要指向一个视觉模型 |
| `agents.defaults.mediaModels` | `null` | — |
| `tools.media` | 无 | — |

**第三步：去官方文档确认机制**（不靠猜）

查到的原文是决定性的一句话：

> **"Image attachments are only injected into agent turns when the selected model is marked image-capable."**

也就是说：**模型没被标记为「支持图像」，附件根本不会进入 Agent 的回合**——附件只是被存成一个 media 引用。

这解释了所有现象：不是「读不出来」，而是**图像压根没送到模型面前**。两个报错只是这个根因的两种表现。

#### 怎么修

**两处都要配**（缺任一都失败）：

1. 给 `antigravity` 下的 `gemini-3.8-flash-high`、`gemini-3.7-flash` 加 `"input": ["text","image"]`
2. 设 `agents.defaults.imageModel = {primary: ..., fallbacks: [...]}`

**修复脚本的工程细节（值得抄）**：

```python
# 1) 改前必备份（带时间戳，可回滚）
bak = P + ".bak.oc_vision_" + ts
shutil.copy2(P, bak)

# 2) 改完后先写临时文件，校验可解析，再原子替换
tmp = P + ".tmp_oc_vision"
io.open(tmp, "w", encoding="utf-8").write(json.dumps(d, ensure_ascii=False, indent=2))
json.load(io.open(tmp, encoding="utf-8"))   # ← 校验，解析不过就抛异常，不会污染原文件
os.replace(tmp, P)                          # ← 原子替换，避免半写坏文件

# 3) 复读校验
d2 = json.load(io.open(P, encoding="utf-8"))
assert d2["agents"]["defaults"]["imageModel"]["primary"] == "antigravity/gemini-3.8-flash-high"
```

**为什么必须这样写**：这是**正在运行的服务的配置文件**。直接覆写 = 一旦写入过程中被中断，服务起不来，Agent 全挂。
`临时文件 + 校验 + 原子替换` 是改任何**热加载配置**的标准姿势。

#### 怎么验证（**这是全文最该抄的一段**）

**不能**只验证「配置文件里有那两行了」——那只能证明你写进去了，不能证明它生效。

必须**驱动真实链路端到端跑一次**，并且满足三个条件：

| 条件 | 为什么 |
|---|---|
| ① 用**随机生成**的测试图 | 内容不可预测 |
| ② 图的真实内容**绝不写进提示词** | 防止 Agent 从文本抄答案（见问题 3） |
| ③ 让 Agent **只用视觉能力**回答 | 明确禁止它写代码/跑命令/生成文件，堵死其他获取信息的路径 |

提示词模板（可直接抄）：

```
我通过网关给你发了一张图片附件，同一张图也存在 NAS 上：<远程路径>。
请直接读出这张图：① 图里有哪些几何形状 ② 每个形状分别是什么颜色 ③ 图上写的文字是什么。
只允许用你自己的视觉能力看图，不要写代码、不要执行命令、不要生成任何文件。
回答请用中文，第一行必须正好是【IMGTEST-xxxx】。
```

最后一行那个**唯一标记**是自动化判定的锚点——脚本轮询会话历史，一旦出现该标记就认为本轮回答完成。

**实测结果**（真相当时只存在本地 JSON 文件里）：

| 链路 | 耗时 | Agent 回答 | 判定 |
|---|---|---|---|
| 局域网 | **~16 秒** | 绿色圆形 / 紫色方形 / 青色三角形 / `QX-7391` | ✅ 与真相完全一致 |
| CF 隧道 | ~30 秒 | 绿色圆形 / 紫色方形 / 青色三角形 / `QX-7391` | ✅ 与真相完全一致 |

---

### 问题 2：CF 隧道的鉴权 —— 从外部连网关必被拒

#### 现象

想从本机（PC）直接连 NAS 的网关测 CF 链路，报：

```
NOT_PAIRED
或
FORBIDDEN missing scope: operator.read
```

#### 为什么

OpenClaw 网关的**设备配对（pairing）机制**：从**非 loopback** 的地址连接，会被视为「未配对的陌生设备」，scope 为空。

而 CLI 在 **NAS 本机**执行时，继承了本机已配对设备的身份，所以**畅通无阻**。

> **铁律**：`openclaw gateway call` 这类操作，**必须在网关所在的那台机器上执行**。

#### 怎么测 CF 隧道（唯一可行解法）

既然 CLI 必须在 NAS 上跑，但 NAS 上直连 `localhost:9090` 走的是**局域网环路**，测不到隧道……

**解法：在 NAS 上做一个 WebSocket 中转**，让 NAS 上的 CLI 连中转，中转再经 Cloudflare 边缘连回来：

```
NAS 上的 CLI ──ws──> NAS 本地中转(19091) ──wss(带CF Access头)──> CF 边缘 ──> CF 隧道 ──> NAS 网关(9090)
                         ↑ 绑定 <NAS_IP>:19091，不是 127.0.0.1
```

**关键细节**：中转**必须绑局域网 IP，不能绑 `127.0.0.1`**。
因为绑 `127.0.0.1` 时，CLI 连它会被判定为 loopback，但中转本身又是个没有设备身份的进程 → **scope 依然为空，照样报错**。

```js
// 中转核心（脱敏示意）
const server = http.createServer();
server.on('upgrade', (req, socket, head) => {
  // 每连接新建一条到 CF 边缘的上游连接，带上 Access 凭据头
  const upstream = new WebSocket('wss://<YOUR_DOMAIN>', {
    headers: {
      'CF-Access-Client-Id':     '<CF_ACCESS_CLIENT_ID>',
      'CF-Access-Client-Secret': '<CF_ACCESS_CLIENT_SECRET>',
    },
  });
  // 双向转发…
});
server.listen(19091, '<NAS_IP>');   // ← 必须是局域网 IP
```

**验证**：`ss -tlnp | grep 19091` 应显示 `<NAS_IP>:19091`（即你的 NAS 局域网 IP）。

---

### 问题 3：🔴 CF 链路的「读图成功」是假阳性（最重要的教训）

#### 现象

第一次跑 CF 链路读图测试，**它答对了**——形状、颜色、文字全对。差点就此写进结论。

#### 我怎么起疑

「答对」本身不可疑，可疑的是**它答得太顺了**。所以我没有停在结果上，而是**导出完整会话记录**看它到底做了什么。

#### 查到什么（决定性证据）

导出会话的 `transcript_events` 后，真相是：

| 实际发生 | 证据 |
|---|---|
| 附件**没有被注入** Agent | 会话里没有图像内容被消费的记录 |
| `view_image` 工具**在报错** | `No image model is configured...` |
| `read *.png` **也在报错** | `Current model does not support images...` |
| Agent 转头调用了 **`sessions_search`** | 它去**搜历史会话** |
| 它搜到了**另一个会话里我自己写下的一句话** | 我在早前测「图生图」时的提示词里写过：「白底，含红色圆形、蓝色方块、绿色三角形，以及文字 OC-9527」 |

**它把我在另一个会话里写的文字描述，当成了「我看到的图片内容」报给我。**

#### 这为什么极其危险

1. **结论会错**：我会把「CF 读图正常」写进文档，但真相是**两条链路都坏的**
2. **信任会错**：一个会从历史会话「凑答案」的 Agent，你无法分辨它哪次是真看见了、哪次是拼出来的
3. **排查会错**：如果只看结果对错，你会往「CF 隧道丢附件」这个错误方向查，永远查不到配置根因

#### 怎么修（方法论的修复）

确立并执行**干净测试三原则**：

| 原则 | 做法 |
|---|---|
| **内容不可预测** | 用脚本**随机生成**测试图（随机形状 + 随机颜色 + 随机编码），绝不手工做图 |
| **真相不进文本** | 图的真实内容**只写本地 JSON 文件**，提示词、聊天记录、文件名里**一个字都不出现** |
| **禁用替代路径** | 提示词明确写「不要写代码、不要执行命令、不要生成任何文件」，并要求「只用你自己的视觉能力」 |

执行后重测，才是可信结果（见问题 1 的验证表）。

> **这条原则适用于所有「感知类能力」的测试**：读图、读音频、读视频、OCR。
> 只要被测对象的内容可能以文本形式存在于它能访问的地方（历史会话、工作区文件、内存库），结论就不可信。

---

### 问题 4：生图 Skill 的三个真问题

Agent 在测试中「自己动手」把图生图跑通了——**这本身很惊艳，但也暴露了工程债**。

#### 4.1 Skill 原生不支持图生图

| 项 | 情况 |
|---|---|
| 原版脚本 | **只有文生图**，调 `POST /v1/images/generations` |
| 怎么跑通的 | **Agent 现场改脚本**：自己加了 `edit()` 函数和 `--image` 参数，改调 `POST /v1/images/edits` |
| 改后 | 6056 → 9062 字节，247 行 |
| 风险 | **Agent 改文件前没有备份** → 我事后补了一份备份 |

**关键接口事实**（踩坑后固化）：

| 端点 | 必须的 Content-Type | 不这么做会怎样 |
|---|---|---|
| `/v1/images/generations`（文生图） | **JSON** | — |
| `/v1/images/edits`（图生图） | **multipart/form-data** | 用 JSON 报 `400 Invalid boundary for multipart/form-data request` |

> 这两个端点**格式要求相反**，是很容易栽的地方。图生图的字段是 `model` / `prompt` / `image` / `n`。

#### 4.2 Windows 路径 bug —— 在 Linux 上造出「字面量垃圾目录」

| 项 | 情况 |
|---|---|
| 原因 | `pick_out_dir()` 的**第一个候选**是 `r"C:\Users\<USER>\Desktop\...\Gemini生图"`（写死的 Windows 路径） |
| Linux 上发生什么 | `os.makedirs` **真的创建了一个名字叫 `C:\Users\<USER>\...` 的目录** |
| 实测证据 | 垃圾目录 `/volX/openclaw-data/workspace/oc_imgtest/C:\Users\...` 确实存在 |
| 修复 | 移除 Windows 候选 + 加防护：路径前 3 字符含 `:` 或分隔符就跳过；首选 `/home/<USER>/Desktop` |

**教训**：跨平台脚本里的路径候选**必须做平台过滤**，不能用「反正前面几个不存在会自动跳过」的思路——`makedirs` 会**创建**它。

#### 4.3 Skill 文档是「错的那一版」

| 项 | 情况 |
|---|---|
| 问题 | NAS 上的 `SKILL.md` 是 **PC 客户端版本**，写死 `C:\Users\<USER>\.workbuddy\...` 的 Windows 路径 |
| 实际后果 | **Agent 被误导，去反编译一个 66 MB 的 `antigravity-tools` 二进制找答案** —— 实测真的发生了 |
| 修复 | 重写为 NAS 版：Linux 路径 + 记录文生图/图生图两种用法 + 明确纪律「不要去反编译二进制」 |

**为什么这个坑值得单列**：Skill 文档是 Agent 的「作业指导书」。**指导书写错平台，Agent 会非常努力地做错事**，而且因为它在「努力」，你还会误以为它在正常工作。

#### 修复后的回归验证

| 项 | 结果 |
|---|---|
| 语法校验 | `py_compile` 通过 |
| 回归 1：文生图 | 出图 **287871 字节** → `/home/<USER>/Desktop/img_*.jpg`（**输出目录已修好，不再落 `C:` 垃圾目录**） |
| 回归 2：图生图 | 出图 **144938 字节** → 指定输出路径 |
| 备份 | 三份：`gen_image.py.bak_before_edits_*` / `.bak_before_pathfix_*` / `SKILL.md.bak_preNAS_*` |

---

## 七、排错 FAQ（按症状索引）

| # | 症状 | 根因 | 解法 |
|---|---|---|---|
| **Q1** | 重启网关后立刻查，报连接拒绝（`ECONNREFUSED`），以为服务挂了 | **查太早**。服务还在启动 | **等 10~12 秒**再查。重启后立即探测会误判 |
| **Q2** | `view_image` 报 `No image model is configured` | `agents.defaults.imageModel` 为空 | 设 `imageModel.primary`（+ fallbacks） |
| **Q3** | `read x.png` 返回 `Current model does not support images` | 模型条目缺 `input:["text","image"]` | 给该模型条目加 `input` 字段 |
| **Q4** | 两处都配了还是不行 | 改了配置**没重启**网关 | `systemctl --user restart openclaw-gateway.service` |
| **Q5** | 从本机连 NAS 网关报 `NOT_PAIRED` / `FORBIDDEN missing scope` | CLI **从非 loopback 连接**，无设备身份 | **必须在网关所在机器上执行 CLI** |
| **Q6** | CF 中转绑 `127.0.0.1` 后 scope 为空 | 被判为 loopback 但进程无设备身份 | 中转**绑局域网 IP**，不要绑回环 |
| **Q7** | 图生图报 `400 Invalid boundary for multipart/form-data request` | 用了 JSON 发 `edits` | 改用 **multipart/form-data** |
| **Q8** | 在 Linux 上出现名字含 `C:\` 的诡异目录 | 脚本路径候选含 Windows 绝对路径，被 `makedirs` 真创建 | 跨平台路径候选**必须做平台过滤** |
| **Q9** | Agent 解决问题时去反编译二进制 / 翻历史会话 | Skill 文档写错平台，或测试内容泄漏进了文本 | ① 修 Skill 文档 ② 执行「干净测试三原则」 |
| **Q10** | `openclaw --version` 与 systemd 单元里的版本号对不上 | 升级时未同步更新单元文件 | **以 `openclaw --version` 为准** |
| **Q11** | 生图报 503 | 上游配额耗尽（如 `gemini-3-pro-image*`） | 换 `gemini-3.1-flash-image`，或等配额恢复 |
| **Q12** | 外网调用被 Cloudflare 挡回登录页 | 未过 CF Access 校验 | 脚本里带 `CF-Access-Client-Id/Secret` 请求头 |

---

## 八、原理：为什么会这样

### 8.1 图像附件的注入是有条件的

```
你发消息 + 附件
      │
      ▼
 网关收到 ──> 附件先落盘，存成 media 引用
      │
      ▼
 组装 Agent 回合
      │
      ├─ 模型被标记 image-capable？ ──否──> ❌ 附件被丢弃（静默！）
      │                                        │
      └─是──> ✅ 图像内容注入回合             └─> Agent 只能瞎猜
                     │
                     ▼
              Agent 真的「看见」了
```

**关键点：失败是静默的**。没有任何显式报错说「因为你没配视觉，所以附件被扔了」——你只会看到工具报错、或者 Agent 含糊其辞。这就是为什么必须**端到端实测**，而不是看配置写得对不对。

### 8.2 为什么「两处」都要配

| 配置 | 回答的问题 |
|---|---|
| 模型条目的 `input` | 「**这个模型**有没有眼睛？」 |
| `agents.defaults.imageModel` | 「**默认情况下**用哪只眼睛看？」 |

一个是**能力声明**，一个是**路由决策**。缺前者，模型不被认为有眼睛；缺后者，Agent 不知道调哪只眼睛——**缺任一都走不通**。

### 8.3 两条链路的差异在哪

| 维度 | 局域网 | Cloudflare 隧道 |
|---|---|---|
| 路径 | 一跳直连 | NAS→CF 边缘→隧道→NAS |
| 鉴权 | 网关设备配对 | 先过 **CF Access**，再过网关配对 |
| 测试方式 | 网关本机跑 CLI 直连 | 网关本机跑 CLI → **本地中转** → CF |
| 实测耗时 | 16 秒 | 30 秒 |

**结论**：能力**完全一致**，只是外网多一跳。**读图失败从来不是网络问题**。

---

## 九、验证清单（怎么确认真的成功了）

复制这份清单，逐项打勾才算真的可用：

- [ ] `openclaw --version` 能正常输出版本
- [ ] `openclaw.json` 里目标模型条目**含** `"input": ["text","image"]`
- [ ] `openclaw.json` 里 `agents.defaults.imageModel.primary` **指向该模型**
- [ ] 网关已重启，`ss -tlnp | grep :9090` 有监听
- [ ] **随机图测试**：局域网链路能准确说出图中形状+颜色+文字（与本地真相文件比对一致）
- [ ] **随机图测试**：CF 隧道链路同样通过
- [ ] 会话记录里**没有** `sessions_search` 之类的「抄答案」痕迹
- [ ] 图生图：给图 + 改图指令，输出文件字节数合理（>100 KB）
- [ ] 输出目录**正确**（不是 `C:\...` 之类字面量目录）

---

## 十、遗留问题与后续建议

| # | 项 | 状态 | 建议 |
|---|---|---|---|
| 1 | **零冗余存储** | ⚠️ 未处理 | 4 块盘全是单盘 RAID1，**任一盘挂 = 该卷数据全丢**。优先给数据最多的卷做备份 |
| 2 | 垃圾目录 | ⏳ 待清理 | Linux 上被误建的 `C:\...` 字面量目录 |
| 3 | 测试中间产物 | ⏳ 待清理 | 测试图、改图产物若干 |
| 4 | NAS 内存偏紧 | ⚠️ | 8 GB，Swap 已用数百 MB。加一根 DDR3L-1600 SO-DIMM（约 40 元）可显著缓解 |
| 5 | Docker 跑在机械盘 | ⚠️ | Docker Root 在 7200 rpm 机械盘上，随机 IO 是明显瓶颈 |
| 6 | **本机客户端同类隐患** | 💡 新发现 | 本机 WorkBuddy 的 `models.json` 里 `gemini-3.8-flash-high` 标着 `supportsImages: false`——**但该模型实测具备视觉能力**。这意味着本机客户端同样不会注入图片，属于**同一类配置缺口的另一处实例**。建议一并核实修正 |
| 7 | Skill 数量认知 | ✅ 已更正 | `openclaw skills list` 实测 **30/67 ready**（此前一次探测误报为「全部禁用」，已核实修正）。`gemini-image-gen` 为 **ready** 状态 |

---

## 十一、复现实操命令速查

```bash
# ── 采集版本 ──
openclaw --version
docker ps --format '{{.Names}}|{{.Image}}|{{.Status}}'
node -v ; python3 --version

# ── 改配置（务必先备份 + 原子写） ──
cp openclaw.json "openclaw.json.bak.$(date +%Y%m%d_%H%M%S)"

# ── 重启并确认 ──
systemctl --user restart openclaw-gateway.service
sleep 12 && ss -tlnp | grep ':9090'

# ── 通过网关发带附件消息（在网关本机执行） ──
openclaw gateway call chat.send \
  --url ws://<NAS_IP>:9090 --token <TOKEN> \
  --params "$(cat /tmp/payload.json)" --json
# payload: {sessionKey, message, idempotencyKey, attachments:[{name,mimeType,media:"data:image/png;base64,..."}]}

# ── 导出会话看真实行为（别只看最终回答！） ──
openclaw gateway call chat.history \
  --url ws://<NAS_IP>:9090 --token <TOKEN> \
  --params '{"sessionKey":"agent:main:main","limit":50}' --json

# ── 生图 ──
python3 gen_image.py "<prompt>" --size 1024x1024 --quality standard
python3 gen_image.py --image <in.png> --out <out.png> "<改图指令>"
```

---

## 十二、相关仓库

| 仓库 | 内容 |
|---|---|
| `infra-ai-gateway` | **总入口**。AI 基础设施与反代网关（含本文板块） |
| ├─ `01_Gemini反代与生图服务/` | 原 `gemini-reverse-proxy-nas` |
| ├─ `02_Antigravity流式协议转换/` | 原 `antigravity-manager-gemini-relay` |
| ├─ `03_Antigravity永久汉化/` | Antigravity 中文化 |
| ├─ `04_桌面应用自动化联动/` | 桌面应用 AI 操控 |
| └─ `05_OpenClaw视觉能力与双链路验证/` | **本文** |
| `openclaw-vision-dual-link-guide` | 本文的独立仓库版 |
| `openclaw-fnOS-guide` | NAS 上部署 OpenClaw |
| `cloudflare-tunnel-guide` | CF 隧道部署与踩坑 |
| `antigravity-manager-image-gen` | 生图 Skill |

---

> **免责与说明**
> 本文由 AI 协助整理，基于 2026-09-18~19 的实机测试。所有结论均经真机验证，但**环境差异可能导致结果不同**。
> 全文已脱敏：IP、域名、密钥、令牌、隧道 UUID、含用户名的路径均替换为占位符。
> 文中涉及第三方开源项目（如 Antigravity-Manager）请遵循其各自许可协议。
