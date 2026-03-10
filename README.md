# OpenCode / Codex 接管式安装：从 `nvm` 到 `OpenClaw + Feishu`

> [!IMPORTANT]
> 这份文档的核心不是“手工配置 OpenClaw”，这是一份用AGNET代理完成的openclaw部署，大幅降低配置时间。

> [!NOTE]
> 截至 **2026-03-10**，OpenClaw 官方与 OpenCode 官方都更推荐 Windows 用户优先使用 **WSL2**。本文先给出 **原生 Windows** 路线，并在文末说明何时应切换到 WSL2。

## 目录

- [总览](#总览)
- [阶段 0：准备项](#阶段-0准备项)
- [阶段 1：安装 nvm-windows](#阶段-1安装-nvm-windows)
- [阶段 2：用 nvm 安装 Node.js](#阶段-2用-nvm-安装-nodejs)
- [阶段 3：安装并登录 OpenCode](#阶段-3安装并登录-opencode)
- [阶段 4：让 OpenCode 或 Codex 接管电脑](#阶段-4让-opencode-或-codex-接管电脑)
- [阶段 5：由 agent 安装并配置 OpenClaw](#阶段-5由-agent-安装并配置-openclaw)
- [阶段 6：人工完成 Feishu 后台](#阶段-6人工完成-feishu-后台)
- [阶段 7：首轮联调](#阶段-7首轮联调)
- [常见问题](#常见问题)
- [何时切换到 WSL2](#何时切换到-wsl2)

---

## 总览

### 目标链路

```mermaid
flowchart LR
    A[nvm-windows] --> B[Node.js LTS]
    B --> C[OpenCode 或 Codex]
    C --> D[OpenClaw]
    D --> E[@openclaw/feishu]
    D --> F[@openclaw/acpx]
    F --> G[opencode ACP agent]
    E --> H[Feishu]
```

### 最终效果

完成后，整体会变成这样：

```text
Feishu -> OpenClaw Gateway -> ACP(acpx) -> opencode agent
```

这里要严格区分两件事：

| 名称 | 含义 | 本文是否使用 |
| --- | --- | --- |
| `opencode/...` | 模型提供商里的 provider/model | 否 |
| `opencode` | ACP 里的外部 agent | 是 |

本文配置的是第二种，也就是 **让 OpenCode 作为 agent 接管 OpenClaw**。

### 人工与 agent 的分工

| 角色 | 负责内容 |
| --- | --- |
| 人工 | 下载 `nvm-windows`、执行最基础命令、登录 OpenCode、登录飞书后台、批准敏感命令 |
| OpenCode / Codex | 检查环境、安装 OpenClaw、安装插件、改配置、启服务、看日志、排错 |

---

## 阶段 0：准备项

在开始前，先确认这几项：

- Windows 10 或 Windows 11
- 可正常打开 PowerShell
- 可以访问 GitHub、Node.js、OpenCode、OpenClaw 官网
- 拥有一个飞书企业或团队环境
- 电脑上暂时没有会干扰 `nvm-windows` 的旧版 Node.js

> [!WARNING]
> 如果这台电脑之前用官方 MSI 安装过 Node.js，建议先在“应用和功能”里卸载旧版 Node.js。  
> `nvm-windows` 与旧 Node 安装残留发生冲突，是后面最常见的问题之一。

---

## 阶段 1：安装 nvm-windows

### 1.1 下载

截至 **2026-03-10**，`nvm-windows` 的 GitHub Releases 最新版本为 **1.2.2**。

下载地址：

- https://github.com/coreybutler/nvm-windows/releases

建议下载：

- `nvm-setup.exe`

### 1.2 安装

运行安装程序后，按默认选项完成安装即可。  
没有特殊需求时，不建议改安装路径。

安装完成后：

1. 关闭当前 PowerShell
2. 重新打开一个新的 PowerShell

### 1.3 验证

```powershell
nvm version
```

正常情况下，会输出版本号。

<details>
<summary>如果 <code>nvm</code> 命令找不到</summary>

先做两件事：

1. 完全关闭 PowerShell
2. 重新打开后再执行 `nvm version`

如果仍然失败，优先怀疑：

- 安装过程未完成
- PATH 未刷新
- 安装器被安全软件拦截

</details>

---

## 阶段 2：用 nvm 安装 Node.js

### 2.1 推荐版本

截至 **2026-03-10**：

- Node.js 官方最新 LTS 为 `v24.13.1`
- OpenClaw 官方要求为 `Node.js 22+`

因此最简单的做法是直接装 **当前 LTS**。

### 2.2 安装与切换

```powershell
nvm install lts
nvm use lts
node -v
npm -v
```

### 2.3 正常结果

- `node -v` 输出 `v22.x.x`、`v24.x.x` 或更高
- `npm -v` 能正常输出版本号

<details>
<summary>如果 <code>nvm use lts</code> 失败</summary>

优先排查下面三项：

1. 当前 PowerShell 是否以管理员权限运行
2. 旧版 Node.js 是否仍残留在 `C:\Program Files\nodejs`
3. 当前窗口是否需要重开以刷新 PATH

推荐处理顺序：

```text
先重开 PowerShell -> 再管理员运行 -> 再检查旧版 Node 残留
```

</details>

---

## 阶段 3：安装并登录 OpenCode

### 3.1 安装 OpenCode

在 PowerShell 中执行：

```powershell
npm install -g opencode-ai
opencode --version
```

只要 `opencode --version` 能输出版本号，说明安装已经成功。

### 3.2 第一次启动

```powershell
opencode
```

### 3.3 连接模型提供商

OpenCode 是 agent，但仍然需要一个模型提供商才能真正工作。  
最稳妥的入门方式是直接连接 `OpenCode Zen`。

在 OpenCode 里执行：

```text
/connect
```

然后依次完成：

1. 选择 `OpenCode Zen`
2. 打开 `https://opencode.ai/auth`
3. 登录账号
4. 创建 API Key
5. 把 Key 粘贴回终端

如果偏好 CLI，也可以先尝试：

```powershell
opencode auth login
```

### 3.4 验证 OpenCode 已可用

再次执行：

```powershell
opencode
```

然后在界面中测试一条很短的指令，例如：

```text
列出当前目录下有哪些文件
```

如果 OpenCode 能正常响应，说明后面已经可以用它接管 OpenClaw。

> [!TIP]
> 如果机器上已经装好 `Codex`，从“接管 OpenClaw”这一步开始，可以直接把 OpenCode 换成 Codex。两者在本文的职责相同。

---

## 阶段 4：让 OpenCode 或 Codex 接管电脑

这一步先不安装 OpenClaw，只验证 agent 是否具备真正的本机接管能力。

将下面这段提示词发给 OpenCode 或 Codex：

```text
先不要安装 OpenClaw。先检查你自己是否已经具备接管这台电脑进行终端操作的能力。

检查项：
1. 是否能执行 PowerShell 命令
2. 是否能读取和编辑本地文件
3. 是否能在需要时向我申请权限
4. 是否能检查 Node.js、npm、Git 是否已安装

请先只做检查，并给出结论。不要开始安装任何与 OpenClaw 相关的内容。
```

### 这一步通过的标志

agent 能明确给出这几类反馈：

- 能否执行 PowerShell 命令
- 能否读写本地文件
- 是否可以申请权限
- 是否能检查本机环境

如果这一步都通不过，就不要继续推进 OpenClaw。

---

## 阶段 5：由 agent 安装并配置 OpenClaw

确认 agent 自身具备接管能力后，再把 OpenClaw 安装与配置整个交给它。

发送下面这段提示词：

```text
现在开始接管这台机器上的 OpenClaw 配置工作。

目标：
1. 安装并初始化 OpenClaw
2. 安装 @openclaw/feishu
3. 安装 @openclaw/acpx
4. 启用 acpx 插件
5. 配置 ACP，使 defaultAgent=opencode
6. allowedAgents 只保留 opencode
7. 等我提供 Feishu App ID 和 App Secret 后，再把 Feishu 配置写入
8. 最后启动 openclaw gateway，并持续观察日志

执行要求：
1. 你优先自己执行，不要把命令转述给我手工操作
2. 每当需要权限时，明确提示我批准
3. 每完成一个阶段，只告诉我结果和下一步需要我提供什么
4. 如果遇到错误，先自行排查和修复，再汇报结果
5. 修改配置时，以 ~/.openclaw/openclaw.json 为主
```

### agent 最终应写出的核心配置

ACP 部分至少应收敛到：

```json5
{
  acp: {
    enabled: true,
    dispatch: { enabled: true },
    backend: "acpx",
    defaultAgent: "opencode",
    allowedAgents: ["opencode"]
  }
}
```

这四项最关键：

- `acp.enabled = true`
- `acp.dispatch.enabled = true`
- `acp.backend = "acpx"`
- `acp.defaultAgent = "opencode"`

> [!IMPORTANT]
> 这里的 `opencode` 指的是 **ACP agent**，不是 `opencode/某个模型`。

---

## 阶段 6：人工完成 Feishu 后台

飞书后台必须人工登录，因此这一段不能完全交给 agent。

### 6.1 进入后台

国内飞书：

- https://open.feishu.cn/

国际版 Lark：

- https://open.larksuite.com/app

### 6.2 需要人工完成的动作

1. 创建企业自建应用
2. 开启机器人能力
3. 发布到企业内部可见范围
4. 记录 `App ID`
5. 记录 `App Secret`

### 6.3 把凭证交给 agent

将下面这段发给 OpenCode 或 Codex：

```text
继续完成 OpenClaw 的 Feishu 接入配置。

这是应用信息：
App ID: cli_xxx
App Secret: xxxxxxxxxx

要求：
1. 写入 Feishu 渠道配置
2. 保持 dmPolicy=pairing
3. 保持 acp.defaultAgent=opencode
4. 启动 gateway
5. 观察日志，直到可以进行飞书联调
```

如果是 `Lark`，改为：

```text
继续完成 OpenClaw 的 Lark 接入配置。

这是应用信息：
App ID: cli_xxx
App Secret: xxxxxxxxxx

要求：
1. Feishu 渠道配置和账户配置都加上 domain=lark
2. 保持 acp.defaultAgent=opencode
3. 启动 gateway
4. 观察日志，直到可以进行联调
```

### Feishu 目标配置

最终通常会接近下面这种状态：

```json5
{
  channels: {
    feishu: {
      enabled: true,
      dmPolicy: "pairing",
      groupPolicy: "open",
      streaming: true,
      blockStreaming: true,
      accounts: {
        main: {
          appId: "cli_xxx",
          appSecret: "replace_with_real_secret",
          botName: "OpenCode Bot"
        }
      }
    }
  }
}
```

---

## 阶段 7：首轮联调

当 agent 报告下面这些状态后，就可以开始测试：

- `gateway` 已启动
- `Feishu` 已连接
- `acpx` 已启用
- `ACP backend` 正常

### 7.1 第一次私聊机器人

在飞书里发一条最简单的消息：

```text
你好
```

如果 `dmPolicy` 仍是 `pairing`，机器人通常会先返回一个配对码。  
这是正常现象，不是错误。

### 7.2 把配对码交给 agent

```text
飞书里已经返回配对码。请替我完成批准，并继续验证链路是否正常。

配对码：在这里替换真实配对码

批准后请继续：
1. 检查日志
2. 确认 Feishu 私聊是否已放通
3. 告诉我何时可以执行 /acp 命令
```

### 7.3 启动真正的 OpenCode 会话

在 agent 确认配对完成后，再到飞书里执行：

```text
/acp spawn opencode --mode persistent --thread off
```

> [!TIP]
> Feishu 首轮联调建议固定使用 `--thread off`。  
> 先把链路跑通，再考虑更复杂的 thread binding。

---

## 常见问题

<details>
<summary><strong>1. 旧 Node.js 与 nvm-windows 冲突</strong></summary>

表现：

- `nvm use lts` 失败
- `node -v` 没切到新版本

处理方式：

- 卸载旧版 Node.js
- 重开 PowerShell
- 重新执行 `nvm use lts`

</details>

<details>
<summary><strong>2. PowerShell 权限不够</strong></summary>

表现：

- `nvm use lts` 提示权限相关错误
- 安装流程无法创建链接或修改路径

处理方式：

- 使用“以管理员身份运行”的 PowerShell

</details>

<details>
<summary><strong>3. OpenCode 已安装，但没有真正可用</strong></summary>

表现：

- `opencode` 命令可以启动
- 但 agent 无法真正执行任务

处理方式：

- 回到第三阶段
- 重新完成 `/connect` 或 `opencode auth login`

</details>

<details>
<summary><strong>4. <code>@openclaw/acpx</code> 装了，但 <code>/acp</code> 仍不可用</strong></summary>

优先检查：

- 插件是否真的安装成功
- 插件是否已启用
- `acp.backend` 是否为 `acpx`
- `acp.dispatch.enabled` 是否为 `true`

</details>

<details>
<summary><strong>5. <code>/acp spawn opencode</code> 提示 target agent not allowed</strong></summary>

原因通常是：

- `allowedAgents` 没有包含 `opencode`

处理方式：

- 将 `allowedAgents` 改为至少包含 `opencode`

</details>

<details>
<summary><strong>6. 飞书里搜不到机器人</strong></summary>

优先检查：

- 应用是否已发布
- 当前账号是否在可见范围里
- 机器人能力是否真的已开启

</details>

<details>
<summary><strong>7. 国际版 Lark 配置看起来没问题，但就是不通</strong></summary>

优先检查：

- `channels.feishu.domain` 是否设置为 `lark`
- 账户配置里是否也带了 `domain: "lark"`

</details>

<details>
<summary><strong>8. 配对码不是报错</strong></summary>

如果当前配置是：

```json5
dmPolicy: "pairing"
```

那么第一次私聊机器人时返回配对码是正常行为。

</details>

<details>
<summary><strong>9. Windows 原生环境反复不稳定</strong></summary>

如果 agent 连续排查后仍反复出现这些问题：

- 插件异常
- gateway 不稳定
- ACP 行为异常
- 工具链兼容性差

建议直接切换到 WSL2。

</details>

---

## 何时切换到 WSL2

如果出现下面任一情况，继续坚持原生 Windows 通常收益不高：

- OpenCode 在 Windows 原生终端下长期不稳定
- OpenClaw 插件在 Windows 下反复报兼容问题
- agent 可以执行命令，但整条链路持续不稳
- 后续还要接入更多依赖 Linux 工具链的能力

此时更稳妥的路线是：

1. 安装 WSL2
2. 在 WSL 中重新安装 OpenCode
3. 在 WSL 中重新安装 OpenClaw
4. 继续让 OpenCode 或 Codex 在 WSL 环境中接管

---

## 最短执行顺序

如果只看最核心步骤，顺序如下：

```powershell
# 1. 安装并验证 nvm-windows 后
nvm install lts
nvm use lts
node -v
npm -v

# 2. 安装 OpenCode
npm install -g opencode-ai
opencode --version

# 3. 启动 OpenCode，执行 /connect 完成登录
opencode
```

然后：

1. 让 OpenCode 或 Codex先做“接管能力自检”
2. 再让它接管 `OpenClaw + Feishu + acpx + opencode`
3. 人工去飞书后台创建应用并提供 `App ID / App Secret`
4. 在飞书里执行：

```text
/acp spawn opencode --mode persistent --thread off
```

---

## 参考资料

- nvm-windows Releases: https://github.com/coreybutler/nvm-windows/releases
- nvm-windows README: https://github.com/coreybutler/nvm-windows
- Node.js 下载页: https://nodejs.org/en/download
- OpenCode 文档: https://opencode.ai/docs/
- OpenCode CLI: https://opencode.ai/docs/cli/
- OpenCode Providers: https://opencode.ai/docs/providers/
- OpenClaw 安装: https://docs.openclaw.ai/install/index
- OpenClaw Windows: https://docs.openclaw.ai/platforms/windows
- OpenClaw Getting Started: https://docs.openclaw.ai/start/getting-started
- OpenClaw ACP Agents: https://docs.openclaw.ai/tools/acp-agents
- OpenClaw Feishu: https://docs.openclaw.ai/channels/feishu
- 飞书开放平台: https://open.feishu.cn/
- Lark 开放平台: https://open.larksuite.com/app

---


