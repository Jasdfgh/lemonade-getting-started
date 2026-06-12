
# Lemonade 零基础部署：在 AMD Radeon GPU 上从零运行本地 AI

## 概览

你将在 30 分钟内，把一块 AMD Radeon GPU 变成你的私人 AI 服务器。从零开始安装，到最终和 AI 自由对话——所有数据留在本地，不需要任何云端账号，完全免费。

本教程会带你遇到真实世界的障碍（网络封锁、SSL 拦截），并一一解决它们。这不是理想化的演示，而是一个**从零到跑通**的实战手册。

### 你将学到什么

完成本教程后，你将能够：

1. 从零安装并运行 AMD 出品的 Lemonade Server，在 GPU 上加载 AI 大模型
2. 解决真实部署中的网络问题（SSL 拦截、HuggingFace 被墙），掌握国内环境下的实战技巧
3. 用 Python 代码调用本地 AI 的 API，构建自己的 AI 应用

---

## 准备工作 — 创建 GPU 工作环境

### 打开 AMD Radeon Cloud 平台

1. 浏览器访问 **https://radeon.anruicloud.com**
2. 登录你的账号（如果没有，先注册）
3. 点击 **创建 Workspace**
4. 镜像选择：**AMD OneClick Base**
5. GPU 配置：默认即可（RX 7900 单卡）
6. 等待 workspace 启动完成（通常 1-2 分钟）

[截图占位]

### 打开终端

Workspace 启动后，你会看到一个 JupyterLab 界面。点击左侧的 **Terminal** 打开一个命令行终端——接下来所有操作都在这里进行。

> 终端就是那个黑色背景、可以输入文字命令的窗口。输入命令后按回车，系统执行并显示结果。

[截图占位]

### 环境确认

你的环境应该具备以下配置（系统已预装，无需操心）：

| 项目 | 值 |
|------|-----|
| 操作系统 | Ubuntu 24.04 |
| 用户 | root |
| Python | 3.12 |
| GPU | AMD Radeon RX 7900（gfx1100，RDNA3 架构） |
| ROCm | 7.2.1 |
| 磁盘空间 | 3.5TB（绰绰有余） |

---

## 第一步 — 安装 Lemonade Server

Lemonade Server 是 AMD 官方出品的本地 AI 推理服务——你可以把它理解为一个"AI 引擎"，安装了它，GPU 就能运行 AI 模型了。

```bash
# 更新软件源列表
apt-get update -qq

# 添加 Lemonade 官方软件源
add-apt-repository -y ppa:lemonade-team/stable

# 再次更新（让系统识别新加的源）
apt-get update -qq

# 安装 Lemonade Server
apt-get install -y lemonade-server
```

### 验证安装

```bash
lemonade --version
```

你应该看到类似这样的输出：

```
lemonade version 10.7.0
```

看到版本号就说明安装成功了。

> **注意：** 如果看到 `Failed to fetch http://compute-artifactory...` 之类的警告信息，不用担心——这只是系统尝试更新一个不存在的内部源，不影响 Lemonade 的安装。

---

## 第二步 — 启动 AI 服务引擎

Lemonade 有两个组件：

| 组件 | 作用 |
|------|------|
| **lemond**（读作 lemon-d） | 后台服务程序，负责管理 GPU 和运行 AI 模型的"引擎" |
| **lemonade** | 命令行工具，你用来操作 lemond 的"遥控器" |

在云 7900 的容器环境中，lemond 不会自动启动（容器里没有 systemd），需要手动启动。

```bash
# 创建 lemond 需要的工作目录
mkdir -p /run/lemonade /var/lib/lemonade /tmp/lemonade-runtime

# 在后台启动 lemond，日志写入 /tmp/lemond.log
RUNTIME_DIRECTORY=/run/lemonade \
  XDG_RUNTIME_DIR=/tmp/lemonade-runtime \
  STATE_DIRECTORY=/var/lib/lemonade \
  nohup /usr/bin/lemond > /tmp/lemond.log 2>&1 &

# 等待服务完全启动
sleep 10
```

> 这条启动命令做了三件事：前面的环境变量告诉 lemond 把工作文件放在哪些目录；`nohup ... &` 让它在后台运行，关掉终端也不会停；`> /tmp/lemond.log 2>&1` 把运行日志保存到文件，方便排查问题。

### 验证服务状态

```bash
lemonade status
```

你应该看到类似这样的输出：

```
Server is running on port 13305

Property            Value
--------------------------------------------------
Version             10.7.0
WebSocket Port      9000
Max Models/Type     1
```

看到 `Server is running on port 13305` 就说明引擎启动成功了。

### 查看可用后端

```bash
lemonade backends
```

你应该看到类似这样的输出：

```
Recipe        Backend     Status          Message/Version
----------------------------------------------------------------------
llamacpp      rocm        installable     Backend is supported but not installed.
              vulkan      installable     Backend is supported but not installed.
              cpu         installable     Backend is supported but not installed.
```

> `installable` 表示"可以安装但还没装"。我们接下来要安装 Vulkan 后端——它对 AMD GPU 的兼容性最好。

---

## 第三步 — 安装 GPU 加速后端

"后端"（backend）是连接 AI 模型和 GPU 硬件的桥梁。我们选择 **Vulkan**——它对所有 AMD GPU 都有很好的兼容性。

### 为什么不能直接用 `lemonade backends install`？

正常情况下，一条命令就能安装后端：

```bash
lemonade backends install llamacpp:vulkan
```

但云 7900 的网络对 GitHub HTTPS 连接做了 SSL 检查，会导致下载失败，报 `SSL: CERTIFICATE_VERIFY_FAILED` 错误。

### 解决方案：用 Python 绕过 SSL 限制

```python
python3 << 'EOF'
import urllib.request, ssl, os, tarfile, shutil

# 创建不验证 SSL 证书的上下文（绕过 SSL 拦截）
ctx = ssl.create_default_context()
ctx.check_hostname = False
ctx.verify_mode = ssl.CERT_NONE

def download(url, dest):
    req = urllib.request.Request(url, headers={"User-Agent": "lemonade"})
    with urllib.request.urlopen(req, context=ctx, timeout=600) as resp:
        total = int(resp.headers.get('Content-Length', 0))
        got = 0
        with open(dest, 'wb') as f:
            while True:
                chunk = resp.read(1024*1024)
                if not chunk:
                    break
                f.write(chunk)
                got += len(chunk)
                if total:
                    print(f"\r  下载中: {got*100//total}%", end="", flush=True)
    print(" done")

# 下载 Vulkan 推理后端（约 36MB）
print("正在下载 Vulkan 推理后端...")
download(
    "https://github.com/ggml-org/llama.cpp/releases/download/b9585/llama-b9585-bin-ubuntu-vulkan-x64.tar.gz",
    "/tmp/vulkan.tar.gz"
)

# 解压并安装到 Lemonade 目录
print("解压中...")
with tarfile.open("/tmp/vulkan.tar.gz") as tf:
    tf.extractall("/tmp/vulkan-extract")

print("安装到 Lemonade 目录...")
for root, dirs, files in os.walk("/tmp/vulkan-extract"):
    if 'llama-server' in files:
        dest = "/root/.cache/lemonade/bin/llamacpp/vulkan"
        if os.path.exists(dest):
            shutil.rmtree(dest)
        shutil.copytree(root, dest)
        with open(f"{dest}/version.txt", "w") as f:
            f.write("b9585")
        print(f"后端已安装到: {dest}")
        break

# 清理临时文件
os.remove("/tmp/vulkan.tar.gz")
shutil.rmtree("/tmp/vulkan-extract", ignore_errors=True)
print("清理完成")
EOF
```

你应该看到类似这样的输出：

```
正在下载 Vulkan 推理后端...
  下载中: 100% done
解压中...
安装到 Lemonade 目录...
后端已安装到: /root/.cache/lemonade/bin/llamacpp/vulkan
清理完成
```

### 重启 lemond 使后端生效

安装新后端后，需要重启 lemond 才能识别。

```bash
# 杀掉旧进程
pkill -9 -f lemond
sleep 3

# 重新启动
mkdir -p /run/lemonade /var/lib/lemonade /tmp/lemonade-runtime
RUNTIME_DIRECTORY=/run/lemonade \
  XDG_RUNTIME_DIR=/tmp/lemonade-runtime \
  STATE_DIRECTORY=/var/lib/lemonade \
  nohup /usr/bin/lemond > /tmp/lemond.log 2>&1 &
sleep 10
```

### 验证后端安装

```bash
lemonade backends | grep vulkan
```

你应该看到：

```
              vulkan      installed       b9585
```

状态从 `installable` 变成了 `installed`，后端安装成功。

---

## 第四步 — 下载 AI 模型

引擎有了，驱动有了，该下载一个 AI 大脑了。我们选择 **Gemma-4-E2B-it-GGUF** 模型：

| 属性 | 说明 |
|------|------|
| 开发方 | Google |
| 类型 | 多模态（文本 + 视觉） |
| 体积 | 约 3.8GB |
| 语言 | 中文、英文 |
| 推理速度 | 160–270 tokens/秒（RX 7900） |

### 为什么不能直接 `lemonade pull`？

正常情况下，一条命令就能下载模型：

```bash
lemonade pull Gemma-4-E2B-it-GGUF
```

但 AI 模型存储在 HuggingFace（国外的模型仓库），国内网络无法直接访问。我们通过镜像服务器来下载。

### 解决方案：用 HuggingFace 镜像 + Python 下载

```python
python3 << 'EOF'
import os

# 使用镜像服务器，绕过 HuggingFace 网络封锁
os.environ['HF_ENDPOINT'] = ''
# 禁用不兼容镜像服务器的 xet 下载加速
os.environ['HF_HUB_DISABLE_XET'] = '1'

from huggingface_hub import snapshot_download

print("正在下载 Gemma-4-E2B-it 模型（约 3.8GB，请耐心等待）...")
print("通过镜像服务器下载，速度取决于网络状况...")

# 只下载需要的 3 个文件：模型权重 + 视觉处理器 + 配置
path = snapshot_download(
    "unsloth/gemma-4-E2B-it-GGUF",
    allow_patterns=["gemma-4-E2B-it-Q4_K_M.gguf", "mmproj-F16.gguf", "config.json"],
)
print(f"\n模型下载完成！")
print(f"   存储位置: {path}")

# 确保模型放在 Lemonade 能找到的路径
import shutil
hub_path = "/root/.cache/huggingface/hub/models--unsloth--gemma-4-E2B-it-GGUF"
src_path = "/root/.cache/huggingface/models--unsloth--gemma-4-E2B-it-GGUF"
os.makedirs("/root/.cache/huggingface/hub", exist_ok=True)
if os.path.exists(src_path) and not os.path.exists(hub_path):
    os.system(f"cp -al {src_path} {hub_path}")
    print("模型路径已配置，Lemonade 可以找到它了")
elif os.path.exists(hub_path):
    print("模型路径已就绪")
EOF
```

你应该看到类似这样的输出：

```
正在下载 Gemma-4-E2B-it 模型（约 3.8GB，请耐心等待）...
通过镜像服务器下载，速度取决于网络状况...
Fetching 3 files: 100%|##########| 3/3 [02:30<00:00]

模型下载完成！
   存储位置: /root/.cache/huggingface/hub/models--unsloth--gemma-4-E2B-it-GGUF/snapshots/...
模型路径已配置，Lemonade 可以找到它了
```

> **注意：** 下载 3.8GB 可能需要 2–5 分钟，取决于网络速度。如果中途断开，重新运行即可（已下载的部分会自动续传）。

---

## 第五步 — 运行你的第一个本地 AI

万事俱备。把 AI 模型加载到 GPU 上：

```bash
# 用 Vulkan 后端（GPU 加速）运行 Gemma-4-E2B-it-GGUF 模型
lemonade run Gemma-4-E2B-it-GGUF --llamacpp vulkan
```

你应该看到类似这样的输出：

```
Loading model: Gemma-4-E2B-it-GGUF
Model loaded successfully!
```

[截图占位]

"Model loaded successfully!" 表示 AI 模型已经加载到 GPU 上了。

### 验证模型状态

```bash
lemonade status
```

你应该看到：

```
Server is running on port 13305

Model                         Type      Device    Recipe        Checkpoint
----------------------------------------------------------------------------------------------------
Gemma-4-E2B-it-GGUF           llm       gpu       llamacpp      unsloth/gemma-4-E2B-it-GGUF:Q4_K_M
```

这个输出说明：

1. **Model: Gemma-4-E2B-it-GGUF** — 正在运行的模型名称
2. **Device: gpu** — 模型在 GPU 上运行（不是 CPU，所以速度很快）
3. **Recipe: llamacpp** — 使用的推理框架
4. **Q4_K_M** — 模型的量化格式（一种压缩技术，让大模型塞得进显存）

---

## 第六步 — 和 AI 对话

模型已经运行了，现在让我们和它说话。

### 第一次对话：简单数学

```python
python3 << 'EOF'
import json, urllib.request

url = "http://127.0.0.1:13305/api/v1/chat/completions"
data = json.dumps({
    "model": "Gemma-4-E2B-it-GGUF",
    "messages": [{"role": "user", "content": "What is 2+2? Answer directly."}],
    "max_tokens": 200,
    "temperature": 0
}).encode()

req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"})
with urllib.request.urlopen(req, timeout=120) as resp:
    result = json.loads(resp.read())
    answer = result['choices'][0]['message']['content']
    speed = result['timings']['predicted_per_second']
    # 输出结果和生成速度
    print(f"AI 回答: {answer}")
    print(f"生成速度: {speed:.1f} tokens/秒")
EOF
```

你应该看到类似这样的输出：

```
AI 回答: 2 + 2 equals **4**.
生成速度: 168.5 tokens/秒
```

[截图占位]

AI 正确回答了，而且速度超过 160 tokens/秒——这就是 GPU 加速的威力。

### 试试中文对话

```python
python3 << 'EOF'
import json, urllib.request

url = "http://127.0.0.1:13305/api/v1/chat/completions"
data = json.dumps({
    "model": "Gemma-4-E2B-it-GGUF",
    "messages": [{"role": "user", "content": "用三句话解释什么是人工智能，说给一个完全不懂技术的人听。"}],
    "max_tokens": 500,
    "temperature": 0.7
}).encode()

req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"})
with urllib.request.urlopen(req, timeout=120) as resp:
    result = json.loads(resp.read())
    answer = result['choices'][0]['message']['content']
    speed = result['timings']['predicted_per_second']
    print(f"AI 回答:\n{answer}")
    print(f"\n生成速度: {speed:.1f} tokens/秒")
EOF
```

### 多轮对话（AI 记住上文）

```python
python3 << 'EOF'
import json, urllib.request

url = "http://127.0.0.1:13305/api/v1/chat/completions"
messages = []

def chat(user_msg):
    """发送消息并获取 AI 回复，保留完整对话历史"""
    messages.append({"role": "user", "content": user_msg})
    data = json.dumps({
        "model": "Gemma-4-E2B-it-GGUF",
        "messages": messages,
        "max_tokens": 2000,
        "temperature": 0.7
    }).encode()
    req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=120) as resp:
        result = json.loads(resp.read())
        reply = result['choices'][0]['message']['content']
        messages.append({"role": "assistant", "content": reply})
        return reply

print("你: 我正在学习编程，推荐一门适合初学者的语言？")
print(f"AI: {chat('我正在学习编程，推荐一门适合初学者的语言？')}")
print()
print("你: 为什么推荐它？给我三个理由。")
print(f"AI: {chat('为什么推荐它？给我三个理由。')}")
print()
print("你: 我学会之后能做什么项目？")
print(f"AI: {chat('我学会之后能做什么项目？')}")
EOF
```

> AI 在第二轮和第三轮回答中会引用之前的对话内容——因为我们把完整对话历史都传给了它。这就是"多轮对话"（multi-turn conversation）。

---

## 第七步 — 构建一个中文翻译小工具

学会了和 AI 对话，让我们来做点实用的——一个中英互译工具。

```python
python3 << 'EOF'
import json, urllib.request

def translate(text, source="中文", target="英文"):
    """调用本地 AI 翻译文本"""
    url = "http://127.0.0.1:13305/api/v1/chat/completions"
    prompt = f"你是一个专业翻译。请将以下{source}翻译成{target}，只输出翻译结果，不要解释：\n\n{text}"
    data = json.dumps({
        "model": "Gemma-4-E2B-it-GGUF",
        "messages": [{"role": "user", "content": prompt}],
        "max_tokens": 500,
        "temperature": 0.3
    }).encode()
    req = urllib.request.Request(url, data=data, headers={"Content-Type": "application/json"})
    with urllib.request.urlopen(req, timeout=120) as resp:
        result = json.loads(resp.read())
        return result['choices'][0]['message']['content']

print("=" * 50)
print("  本地 AI 翻译工具（中英互译）")
print("=" * 50)
print()

examples = [
    ("今天天气真好，我们去公园散步吧。", "中文", "英文"),
    ("Machine learning is a subset of artificial intelligence.", "英文", "中文"),
    ("这个项目的截止日期是下周五。", "中文", "英文"),
]

for text, src, tgt in examples:
    print(f"原文（{src}）: {text}")
    result = translate(text, src, tgt)
    print(f"译文（{tgt}）: {result}")
    print()

print("=" * 50)
print("翻译完成！这个工具完全在本地运行，你的文本不会发送到任何外部服务器。")
EOF
```

你应该看到类似这样的输出：

```
==================================================
  本地 AI 翻译工具（中英互译）
==================================================

原文（中文）: 今天天气真好，我们去公园散步吧。
译文（英文）: The weather is really nice today, let's go for a walk in the park.

原文（英文）: Machine learning is a subset of artificial intelligence.
译文（中文）: 机器学习是人工智能的一个子集。

原文（中文）: 这个项目的截止日期是下周五。
译文（英文）: The deadline for this project is next Friday.

==================================================
翻译完成！这个工具完全在本地运行，你的文本不会发送到任何外部服务器。
```

[截图占位]

> 你刚刚构建了一个私密的翻译工具。所有文本处理都在 GPU 上完成，没有任何数据离开这台机器——这就是本地 AI 的核心价值。

---

## 理解发生了什么

回顾一下你刚才做了什么：

```
┌──────────────────────────────────────────────────────────────────┐
│                       你的 GPU 服务器                              │
│                                                                    │
│  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────────┐ │
│  │ 你的代码 │───→│ Lemonade │───→│  Vulkan  │───→│  AMD RX 7900 │ │
│  │(Python) │    │  Server  │    │  后端    │    │    GPU       │ │
│  └─────────┘    └──────────┘    └──────────┘    └──────────────┘ │
│                       ↑                                            │
│                 AI 模型 (Gemma-4)                                   │
│                 存储在本地磁盘                                      │
└──────────────────────────────────────────────────────────────────┘
                    ↕ 没有任何数据离开这台机器
```

### Lemonade Server 是什么？

Lemonade 是 AMD 官方开源的本地 AI 推理服务，主要做三件事：

1. **管理 AI 模型** — 下载、加载、卸载不同的 AI 模型
2. **提供 API 接口** — 让你的代码能和 AI 模型对话（兼容 OpenAI API 格式）
3. **优化 GPU 使用** — 让 AI 模型在 AMD GPU 上高效运行

### 本地 AI vs 云端 AI

| 对比项 | 本地 AI（你刚搭建的） | 云端 AI（如 ChatGPT） |
|--------|----------------------|---------------------|
| 数据隐私 | 数据完全不离开本机 | 数据发送到远程服务器 |
| 网络依赖 | 运行时不需要网络 | 必须联网 |
| 费用 | GPU 电费（或云平台积分） | 按 token 计费 |
| 模型能力 | 受限于本地 GPU 显存 | 可运行超大模型 |
| 响应速度 | 极快（无网络延迟） | 取决于网络和服务器负载 |
| 定制化 | 可以自由选择和切换模型 | 只能用提供的模型 |

### 为什么本地 AI 重要？

- **隐私** — 处理敏感信息（公司文档、个人数据）时，数据不会泄露
- **离线可用** — 模型加载后不需要网络也能运行
- **无审查** — 没有内容过滤，AI 按照模型本身的能力回答
- **成本可控** — 一次下载模型，无限次使用，不按 token 收费

---

## 下一步可以做什么

### 尝试更大的模型

当前的 Gemma-4-E2B（3.8GB）是轻量版。RX 7900 有 16GB 显存，可以运行更大的模型：

```bash
# 查看所有可用模型
lemonade list

# 例如：运行 27B 参数的 Qwen 模型（约 22GB，需要更长下载时间）
# HF_ENDPOINT= HF_HUB_DISABLE_XET=1 lemonade run Qwen3.6-35B-A3B-GGUF --llamacpp vulkan
```

### 接入更多工具

Lemonade 的 API 兼容 OpenAI 格式，你可以用它接入：

- **Open WebUI** — 一个漂亮的聊天界面
- **LangChain** — 构建复杂的 AI 应用管道
- **任何支持 OpenAI API 的客户端**

> 连接方式：Base URL = `http://127.0.0.1:13305/api/v1`

### 构建更复杂的应用

你已经掌握了 API 调用的基础。接下来可以尝试：

- 文档问答系统（让 AI 阅读你的文件并回答问题）
- 代码辅助工具（让 AI 帮你写代码、解释代码）
- 数据分析助手（让 AI 帮你理解数据、生成报告）

---

## 测验

检验一下你学到了什么。

### 问题 1

Lemonade Server 在 AMD Radeon GPU 上运行 AI 模型时，使用了哪种后端技术？

- A) CUDA（NVIDIA 专用）
- B) Vulkan（跨平台图形/计算 API）
- C) DirectX（Windows 专用）
- D) Metal（Apple 专用）

<details>
<summary>点击查看答案</summary>

**答案：B**

Vulkan 是一个跨平台的图形和计算 API，AMD Radeon GPU 通过 Vulkan 后端运行 AI 推理。CUDA 仅支持 NVIDIA GPU，不适用于 AMD 硬件。在本教程中，我们安装的就是 `llamacpp:vulkan` 后端。

</details>

### 问题 2

在 AMD Radeon RX 7900 (gfx1100) 上运行 Gemma-4-E2B-it-GGUF 模型，生成速度大约是多少？

- A) 10–20 tokens/秒
- B) 50–80 tokens/秒
- C) 160–270 tokens/秒
- D) 1000+ tokens/秒

<details>
<summary>点击查看答案</summary>

**答案：C**

实测数据显示，RX 7900 使用 Vulkan 后端运行 Gemma-4-E2B-it-GGUF (Q4_K_M) 模型，生成速度在 160–270 tokens/秒之间。这得益于 RDNA3 架构的 GPU 计算能力和 Q4_K_M 量化对显存带宽的高效利用。

</details>

### 问题 3

AMD Radeon RX 7900 使用的 GPU 架构代号是什么？

- A) gfx900（Vega 架构）
- B) gfx1030（RDNA2 架构）
- C) gfx1100（RDNA3 架构）
- D) gfx1201（RDNA4 架构）

<details>
<summary>点击查看答案</summary>

**答案：C**

AMD Radeon RX 7900 使用 gfx1100 代号，属于 RDNA3 架构。RDNA3 是 AMD 目前主流的桌面级 GPU 架构，在 AI 推理方面通过 Vulkan 和 ROCm 都有良好的支持。gfx1201 对应更新的 RDNA4 架构（如 R9700）。

</details>

---

## 附录 — 故障排查

### lemond 启动失败

```bash
# 查看日志
cat /tmp/lemond.log

# 检查端口是否被占用
ss -tlnp | grep 13305

# 如果端口被占用，先杀掉旧进程再重启
pkill -9 -f lemond
sleep 3
mkdir -p /run/lemonade /var/lib/lemonade /tmp/lemonade-runtime
RUNTIME_DIRECTORY=/run/lemonade \
  XDG_RUNTIME_DIR=/tmp/lemonade-runtime \
  STATE_DIRECTORY=/var/lib/lemonade \
  nohup /usr/bin/lemond > /tmp/lemond.log 2>&1 &
```

### 模型加载失败

```bash
# 检查模型是否已下载
ls /root/.cache/huggingface/hub/models--unsloth--gemma-4-E2B-it-GGUF/

# 检查 GPU 是否正常
rocm-smi

# 查看详细错误
cat /tmp/lemond.log | tail -30
```

### Python 调用 API 超时

```bash
# 确认服务在运行
lemonade status

# 如果没有模型信息，重新加载模型
lemonade run Gemma-4-E2B-it-GGUF --llamacpp vulkan
```

---

*本教程基于 AMD Radeon Cloud (radeon.anruicloud.com) 云 7900 环境实测验证。Lemonade Server v10.7.0, llama.cpp Vulkan 后端 b9585。*
