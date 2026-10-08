# JevK5 本地部署指南：MacBook Pro M5 (24 GB) · llama.cpp + Metal

| 项目 | 内容 |
|---|---|
| 适用设备 | MacBook Pro，Apple M5，24 GB 统一内存，macOS |
| 推荐技术栈 | `jevk5-4b-v0.3-Q8_0.gguf` + llama.cpp (Metal) + `jevk5` 的 `JevK5GGUF` 客户端 |
| 目标 | 在 M5 GPU 上本地运行 JevK5，并保留其 typed-decision（yes/no、单选、评分）与校准概率输出 |
| 文档日期 | 2026-10-08（依赖的 llama.cpp、jevk5 迭代很快，命令以当日核对结果为准） |
| 读者 | 有基础命令行与 Python 经验的开发者 |

> **标记说明：** 文中 ⚠️ 表示该点我在编写时无法从公开资料中直接核实，请在本机按"附录 D"确认。

---

## 目录

- [快速开始](#快速开始)
- **第一部分　概念与选型**
  - [1. 方案概览](#1-方案概览)
  - [2. 模型与校准参数选型](#2-模型与校准参数选型)
- **第二部分　安装**
  - [3. 前置检查](#3-前置检查)
  - [4. 安装 llama.cpp](#4-安装-llamacpp)
  - [5. 启动 llama-server](#5-启动-llama-server)
  - [6. 验证 server 与 Metal](#6-验证-server-与-metal)
  - [7. 安装 JevK5 客户端](#7-安装-jevk5-客户端)
- **第三部分　使用**
  - [8. 第一次决策调用](#8-第一次决策调用)
  - [9. 三种问题类型](#9-三种问题类型)
  - [10. 读出机制与使用限制](#10-读出机制与使用限制)
  - [11. 对外提供 /v1/systemone](#11-对外提供-v1systemone)
- **第四部分　运维**
  - [12. 验收清单](#12-验收清单)
  - [13. 性能与资源](#13-性能与资源)
  - [14. 网络与安全](#14-网络与安全)
  - [15. 开机自启（launchd）](#15-开机自启launchd)
  - [16. 日常操作、升级与卸载](#16-日常操作升级与卸载)
  - [17. 故障排查](#17-故障排查)
- **附录**
  - [附录 A　改用 9B 模型](#附录-a改用-9b-模型)
  - [附录 B　零依赖参考客户端](#附录-b零依赖参考客户端)
  - [附录 C　参考资料](#附录-c参考资料)
  - [附录 D　核对状态与待确认项](#附录-d核对状态与待确认项)

---

## 快速开始

已有 Homebrew、Python ≥ 3.10、git 的话，三步即可跑通。每一步的含义见后文对应章节。

**终端 1：启动推理服务**（首次会下载约 4.5 GB）

```bash
brew install llama.cpp jq

llama-server \
  --hf-repo alibiserikbay/JevK5-GGUF \
  --hf-file jevk5-4b-v0.3-Q8_0.gguf \
  -c 8192 -ngl 99 -np 1 \
  --host 127.0.0.1 --port 8080
```

**终端 2：安装客户端并调用**

```bash
mkdir -p ~/jevk5-gguf && cd ~/jevk5-gguf
python3 -m venv .venv && source .venv/bin/activate
pip install -U pip

# 先查最新 v0.3.x 标签，再替换下面的 TAG（见 §7）
git ls-remote --tags --refs https://github.com/allebee/jevk5 | tail -5
TAG=v0.3.3
pip install --no-deps "jevk5 @ git+https://github.com/allebee/jevk5@${TAG}"
```

```bash
cat > test.py <<'EOF'
from jevk5 import JevK5GGUF

model = JevK5GGUF(url="http://127.0.0.1:8080",
                  temperature=1.22, knockout_temperature=0.93)

print(model.decide(
    "Order #7120 shows delivered to No. 17; the customer lives at No. 71.",
    {"type": "choice",
     "instructions": "What happened to the parcel?",
     "criteria": ["delivered", "misdelivered", "unknown"]},
))
EOF
python test.py
```

能打印出带各选项概率的结果，即部署成功。

---

# 第一部分　概念与选型

## 1. 方案概览

### 1.1 JevK5 是什么

JevK5 是社区开源（Apache-2.0）的"决策模型"，基于 Qwen3.5 微调，**不是聊天模型**。你给它一段材料（state）和一个带选项的问题，它只做一次前向计算，读取各选项字母的下一 token 对数概率，经温度校准后输出每个选项的概率，**不生成任何文本**。它与 TypeSafe 的 Jev 无隶属关系。

### 1.2 整体架构

```text
应用 / Agent
    │  （可选）POST /v1/systemone  :8090
    ▼
HTTP adapter（FastAPI，§11）
    │
    ▼
JevK5GGUF（jevk5 包：拼 prompt、取字母 logprobs、温度校准）
    │  /tokenize + /completion
    ▼
llama-server  :8080
    │
    ▼
llama.cpp → Metal → M5 GPU
```

各层职责：

| 层 | 职责 |
|---|---|
| llama.cpp / llama-server | 加载 GGUF，在 Metal 上推理，并返回字母 token 的 logprobs |
| `JevK5GGUF` | JevK5 专有逻辑：prompt 格式、读出、校准温度、超过 16 个选项时的淘汰赛 |
| HTTP adapter | 仅在需要对外提供 HTTP 接口时才加；保持很薄 |

### 1.3 为什么用 llama.cpp (Metal)，而不是 MLX 或 PyTorch (MPS)

- **官方只提供 CUDA 路径与 GGUF 路径。** 官方 `jevk5` 的 transformers 运行时面向 CUDA（性能数据也来自 H100 + CUDA graphs），而作者发布了 GGUF 供 llama.cpp 在 Apple GPU、CPU 等上运行。Mac 上走 GGUF 是有官方验证的路径。
- **没有 MLX 版本。** 若用 MLX，需要自行转换权重并重写读出逻辑，且无法保证与官方结果一致。
- **不需要 MPS。** MPS 是 PyTorch 的 Apple GPU 后端；本方案不安装 PyTorch，也不依赖 CUDA。llama.cpp 直接使用 Metal。

> 从硬件角度看，MLX 与 llama.cpp 都能用到 M5 GPU，差别在上层软件栈与模型格式，而不在 GPU 本身。

---

## 2. 模型与校准参数选型

### 2.1 推荐模型

24 GB 统一内存下，**默认选 `jevk5-4b-v0.3-Q8_0.gguf`**（文件约 4.48 GB）。原因：

1. 作者本人对 4B 的建议是默认首选；9B 在作者的内部验证集上略好，但在 JevBench 公开题的困难档上反而更差，校准也更差，且更大更慢。
2. Q8_0 对 bf16 的答案一致性很高，量化损失可忽略。
3. 占用小，留足内存给系统与其他应用。

### 2.2 可选文件一览

| 文件 | 约大小 | 与 bf16 答案一致（231 道公开题） | 说明 |
|---|---:|---:|---|
| `jevk5-4b-v0.3-Q8_0.gguf` | 4.48 GB | ⚠️ 约 229 / 231 | **推荐**。文件大小已核实 |
| `jevk5-4b-v0.3-Q5_K_M.gguf` | ⚠️ 约 3 GB | ⚠️ 未核实 | 文件存在，指标待查 |
| `jevk5-4b-v0.3-Q4_K_M.gguf` | ⚠️ 约 2.7 GB | ⚠️ 未核实（v0.2 版为 219 / 231） | 仅适合 8 GB 以下设备，24 GB 不必使用 |
| `jevk5-9b-v0.3-Q8_0.gguf` | ⚠️ 约 9.5 GB | 229 / 231 | 见[附录 A](#附录-a改用-9b-模型) |

24 GB 设备没有压缩模型的必要，**直接用 Q8_0**。作者明确指出，更低的量化（如 9B 的 Q4_K_M、2B 的 Q4_K_M）会改变明显更多的答案。

> **不要使用 HF 页面上的 `...GGUF:Q8_0` 简写。** 该仓库里有多个 Q8_0 文件（如 2B v0.2 与 4B v0.3），简写可能匹配到错误的文件。本文一律用 `--hf-file` 显式指定。

### 2.3 校准温度

校准温度属于 **JevK5 读出参数**，与文本生成里的 sampling temperature 无关，且**不存储在 GGUF 内**，必须由客户端传入，并与模型版本对应：

| 模型文件 | `temperature`（≤16 选项） | `knockout_temperature`（>16 选项） |
|---|---:|---:|
| `jevk5-4b-v0.3-Q8_0` / `Q5_K_M` | 1.22 | 0.93 |
| `jevk5-9b-v0.3-Q8_0` | 1.049 | 1.2 |
| `jevk5-2b-v0.2-Q8_0` | 1.42 | 0.77 |

用错温度不会报错，只会让概率"偏自信"或"偏保守"，需要特别留意。

---

# 第二部分　安装

## 3. 前置检查

```bash
uname -m                              # 预期：arm64
sw_vers -productVersion               # 记录 macOS 版本
system_profiler SPHardwareDataType    # 确认 Apple M5、24 GB
python3 --version                     # ⚠️ 建议 ≥ 3.10
git --version                         # pip 从 GitHub 安装需要
brew --version                        # 使用 Homebrew 安装 llama.cpp 时需要
df -h ~                               # 预留 ≥ 10 GB 空间
```

补充说明：

- 若 `python3` 低于 3.10（macOS 自带或 Xcode CLT 附带的版本可能较旧），先 `brew install python@3.12`，后文用 `python3.12` 代替 `python3`。⚠️ jevk5 的最低 Python 版本我未能核实，这是保守建议。
- 若 `git` 提示需安装命令行工具，执行 `xcode-select --install`。
- 首次部署需要访问 GitHub 与 Hugging Face。

---

## 4. 安装 llama.cpp

### 4.1 推荐：Homebrew

```bash
brew install llama.cpp jq
llama-server --version
```

`jq` 用于后文格式化 JSON 输出。

### 4.2 备选：官方 llama.app 安装器

模型卡给出的安装方式：

```bash
curl -LsSf https://llama.app/install.sh | sh
llama --version
```

该安装器提供统一的 `llama` 命令，启动服务的写法是 `llama serve -hf ...`（注意是 `serve`，不是 `server`）。⚠️ 它是否同时提供独立的 `llama-server` 命令、以及是否支持 `-ngl`、`-np` 等所有参数，我未能核实。**为保证本文命令一致，建议使用 4.1。**两种方式只保留一种，避免出现两套 llama.cpp。

若命令找不到，检查 PATH：

```bash
echo 'export PATH="$HOME/.llama-app:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

### 4.3 版本说明

llama.cpp 迭代很快，请以 `llama-server --version` 输出为准并记录下来，便于排查。截至本文日期，llama.cpp 已新增原生 `/v1/systemone` 端点，但它只支持特定的决策模型家族，**JevK5 不在其列**，详见 [§11](#11-对外提供-v1systemone)。

---

## 5. 启动 llama-server

### 5.1 启动命令

```bash
llama-server \
  --hf-repo alibiserikbay/JevK5-GGUF \
  --hf-file jevk5-4b-v0.3-Q8_0.gguf \
  -c 8192 \
  -ngl 99 \
  -np 1 \
  --host 127.0.0.1 \
  --port 8080
```

首次启动会下载模型并缓存，之后直接读取缓存。保持这个终端不关闭（后台常驻见 [§15](#15-开机自启launchd)）。

### 5.2 参数说明

| 参数 | 含义 |
|---|---|
| `--hf-repo` / `--hf-file` | 指定 Hugging Face 仓库与具体 GGUF 文件 |
| `-c 8192` | 上下文长度。JevK5 要把整份材料放进一次前向计算，材料过长会被截断或报错 |
| `-ngl 99` | 最多把 99 层放到 GPU。这是**层数上限**，不是"GPU 使用率 99%" |
| `-np 1` | 并行槽位数设为 1。⚠️ 这是我加的保守设置：`-c` 是总上下文，若 server 默认开多个槽位，单次请求能用的上下文可能变小；单用户场景下 1 个槽位最稳妥 |
| `--host 127.0.0.1` | 仅本机可访问 |
| `--port 8080` | HTTP 端口 |

> 作者只在 `llama-server` 上测过。Ollama、LM Studio 等基于 llama.cpp 的应用可以加载这些文件，但 JevK5 的校准读出需要"对已分词的 prompt 返回答案字母 logprobs"的接口，这些应用是否提供，作者未验证。**本文不推荐用它们替代 llama-server。**

---

## 6. 验证 server 与 Metal

在第二个终端执行。

```bash
# 健康检查（模型加载完成前可能返回 loading 状态）
curl http://127.0.0.1:8080/health

# 模型信息
curl -s http://127.0.0.1:8080/v1/models | jq
```

浏览器打开 `http://127.0.0.1:8080` 可看到内置 Web UI，用于确认 server 正常。

**确认 Metal 生效：**

```bash
llama-server --list-devices
```

同时查看启动日志，应能看到 Metal 后端被初始化、识别到 Apple GPU，以及模型层被卸载到 GPU 的信息。如果日志显示全部在 CPU 上运行，参见 [§17](#17-故障排查)。

---

## 7. 安装 JevK5 客户端

### 7.1 创建独立环境

```bash
mkdir -p ~/jevk5-gguf && cd ~/jevk5-gguf
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install -U pip
```

### 7.2 选择版本标签并安装

先查看仓库有哪些标签，选最新的 `v0.3.x`：

```bash
git ls-remote --tags --refs https://github.com/allebee/jevk5 | tail -5
```

然后安装（把 `TAG` 换成上面查到的值）：

```bash
TAG=v0.3.3
pip install --no-deps "jevk5 @ git+https://github.com/allebee/jevk5@${TAG}"
```

说明：

- 用 `--no-deps` 是为了避免拉取 PyTorch、CUDA 等 Mac 上用不到的重型依赖；GGUF 路径只需要 `JevK5GGUF` 与向 llama-server 发 HTTP 请求。
- ⚠️ 若导入时报 `ModuleNotFoundError`，按报错补装对应的轻量包即可（不要装 torch）。
- ⚠️ 旧文档写的是 `@v0.3.0`。官方 9B 模型卡写明运行时需要"0.3.0 或更高"，但我无法确认 `JevK5GGUF` 是否从 0.3.0 就已包含，所以建议使用最新 v0.3.x。
- 不要安装带 `[fast]` 的 extra：它面向 CUDA 的 flash-linear-attention，对本方案没有用。

### 7.3 确认类的签名

构造参数名以你安装的版本为准：

```bash
python -c "from jevk5 import JevK5GGUF; help(JevK5GGUF)"
```

后文示例使用 `url=`、`temperature=`、`knockout_temperature=`。⚠️ 其中 `temperature`、`knockout_temperature` 已见于第三方项目的用法，`url` 这个参数名我未能核实，若不同请按 `help()` 的输出调整。

---

# 第三部分　使用

## 8. 第一次决策调用

确保终端 1 的 llama-server 在运行，然后：

```bash
cd ~/jevk5-gguf
source .venv/bin/activate
```

创建 `test.py`：

```python
from jevk5 import JevK5GGUF

model = JevK5GGUF(
    url="http://127.0.0.1:8080",
    temperature=1.22,
    knockout_temperature=0.93,
)

result = model.decide(
    "Order #7120 shows delivered to No. 17; the customer lives at No. 71.",
    {
        "type": "choice",
        "instructions": "What happened to the parcel?",
        "criteria": ["delivered", "misdelivered", "unknown"],
    },
)
print(result)
```

运行 `python test.py`。预期输出里包含最终选项、各选项概率与 confidence。这一步同时验证了完整链路：`JevK5GGUF → llama-server → Metal → M5 GPU`。

---

## 9. 三种问题类型

`decide(state, question)` 的 `question` 是一个字典，`type` 决定问题类型。

| 类型 | 用途 | `criteria` 写法 | 返回要点 |
|---|---|---|---|
| `noul` | 是 / 否 | 可选，`{"true": "...", "false": "..."}` | `noul`：为"真"的概率 |
| `choice` | 单选 | 字典 `{键: 描述}` 或键的列表 | 最终选项、各选项概率、confidence |
| `score` | 等级评分 | 列表，**从低到高**排列 | 评分、各等级概率、confidence |

### 9.1 `noul`（是 / 否）

```python
model.decide(
    "The customer bought the item 12 days ago but has no receipt.",
    {
        "type": "noul",
        "instructions": ("Is the customer eligible for a refund under a policy "
                         "requiring a receipt and a purchase within 30 days?"),
    },
)
```

### 9.2 `choice`（单选）

```python
model.decide(
    "The customer was charged twice and wants the duplicate charge refunded.",
    {
        "type": "choice",
        "instructions": "Which team should handle this request?",
        "criteria": {
            "billing": "Payments and refunds",
            "shipping": "Delivery and parcel problems",
            "technical": "Software bugs and outages",
        },
    },
)
```

### 9.3 `score`（评分）

```python
model.decide(
    "The production server is down and the company is losing money.",
    {
        "type": "score",
        "instructions": "How urgent is this incident?",
        "criteria": ["can wait", "this week", "today", "right now"],
    },
)
```

### 9.4 写好问题的几点建议

- 材料与问题用**英文**写：JevK5 目前只面向英文。
- 给选项写清楚区分度高的描述，比只给一个名称更稳。
- 不要把多个判断塞进一个问题；拆成多个问题分别提问。
- 这类模型对日期与数字推理较弱（作者在已知弱点中明确列出），涉及精确计算的判断不要只依赖它。

---

## 10. 读出机制与使用限制

### 10.1 与普通 LLM 的区别

```text
普通 LLM：  prompt → 逐个生成 token → 文本回答

JevK5：     state + 问题 + 选项字母
              → 一次前向计算
              → 取答案字母的下一 token logprobs
              → softmax(logprob / 温度)
              → 各选项概率
```

因此**不要用 `/v1/chat/completions` 的普通聊天调用来代替 JevK5 的决策读出**：那样得到的是生成文本，没有经过校准的概率。

### 10.2 使用限制

| 项目 | 说明 |
|---|---|
| 语言 | 仅英文 |
| 选项数 | ≤ 16 个选项一次读出；更多时自动分组做淘汰赛（使用 `knockout_temperature`），最多 255 个，准确率与延迟都会受影响 |
| 输入长度 | 受 `-c` 限制。官方运行时对超过 16384 token 的输入会直接拒绝而不是截断；本方案的 `-c 8192` 更保守 |
| 弱项 | 日期与数字推理、长篇政策类材料、概率估计类问题（见作者模型卡"已知弱点"） |
| 真实业务效果 | 作者说明未在真实工作流中对比过 Jev；上线前请用自己的标注数据评估 |

### 10.3 读出细节（用于排查）

`JevK5GGUF` 向 llama-server 请求 `n_probs=40` 的 top logprobs。如果某个选项字母没有出现在 top 40 里，会被赋予一个很低的兜底值（比已出现的最小值再低 2.0）。因此选项很多、或模型"完全不倾向"某个选项时，该选项概率会趋近于 0，这是正常现象。

---

## 11. 对外提供 /v1/systemone

### 11.1 先判断是否需要

- **只在本机 Python 里用：** 直接用 `JevK5GGUF.decide()`，不需要这一节。
- **其他程序要通过 HTTP 调用，或要兼容 TypeSafe 风格客户端：** 需要一个 `/v1/systemone` 端点。

### 11.2 llama.cpp 原生端点 ≠ JevK5

llama.cpp 在近期版本中新增了原生的 `POST /v1/systemone`，但它**只支持特定的决策模型家族**，据版本说明为 laya、julia-1、lev、openjev、kev、nimble 等，**未包含 JevK5**；对不支持的模型会返回 501。⚠️ 此列表会随版本变化，请以你所装版本的说明为准。

因此，要让 JevK5 获得完整、校准过的 typed-decision 行为，仍然应走：

```text
llama-server → answer-letter logprobs → JevK5GGUF → JevK5 校准读出
```

### 11.3 官方 `jevk5-serve` 不适用于 Mac

JevK5 官方文档里 `jevk5-serve --model ...` 可以起一个 `/v1/systemone` 服务，但它基于 transformers + CUDA 运行时，**不是本方案的 GGUF 路径**，在这台 Mac 上不要用。

### 11.4 自建薄 adapter

adapter 只做 HTTP 协议转换，不重写任何概率逻辑。

**安装依赖**

```bash
cd ~/jevk5-gguf
source .venv/bin/activate
pip install -U fastapi uvicorn
```

**创建 `server.py`**

```python
import os
import urllib.request
from typing import Any

from fastapi import FastAPI, Header, HTTPException
from pydantic import BaseModel

from jevk5 import JevK5GGUF

LLAMA_URL = os.getenv("LLAMA_URL", "http://127.0.0.1:8080")
MODEL_NAME = os.getenv("MODEL_NAME", "jevk5-4b-v0.3-Q8_0")
TEMPERATURE = float(os.getenv("JEVK5_TEMPERATURE", "1.22"))
KNOCKOUT_TEMPERATURE = float(os.getenv("JEVK5_KNOCKOUT_TEMPERATURE", "0.93"))
API_KEY = os.getenv("API_KEY")  # 可选；设置后要求 Bearer 认证

app = FastAPI(title="Local JevK5 Decision API")

model = JevK5GGUF(
    url=LLAMA_URL,
    temperature=TEMPERATURE,
    knockout_temperature=KNOCKOUT_TEMPERATURE,
)


class SystemOneRequest(BaseModel):
    state: Any
    questions: dict


def check_auth(authorization: str | None) -> None:
    if API_KEY and authorization != f"Bearer {API_KEY}":
        raise HTTPException(status_code=401, detail="unauthorized")


@app.get("/health")
def health():
    try:
        urllib.request.urlopen(f"{LLAMA_URL}/health", timeout=2)
        backend_ok = True
    except Exception:
        backend_ok = False
    return {"status": "ok" if backend_ok else "backend_unavailable",
            "backend_llama_connected": backend_ok,
            "model": MODEL_NAME}


@app.post("/v1/systemone")
def systemone(req: SystemOneRequest, authorization: str | None = Header(default=None)):
    check_auth(authorization)
    try:
        answers = {qid: model.decide(req.state, q) for qid, q in req.questions.items()}
    except Exception as exc:
        raise HTTPException(status_code=502, detail=str(exc)) from exc
    return {"model": MODEL_NAME, "answers": answers}
```

> 这是一个最小实现。它返回的是 `JevK5GGUF.decide()` 的原始结构，**不保证与 TypeSafe 官方响应逐字段一致**（例如 `usage` 字段）。如果必须严格对齐，参见 11.6。

**启动**

```bash
uvicorn server:app --host 127.0.0.1 --port 8090
```

### 11.5 测试

```bash
curl http://127.0.0.1:8090/v1/systemone \
  -H "Content-Type: application/json" \
  -d '{
    "state": "The customer was charged twice for an order and wants the duplicate charge refunded.",
    "questions": {
      "team": {
        "type": "choice",
        "instructions": "Which team should handle this?",
        "criteria": {
          "billing": "Payments and refunds",
          "shipping": "Delivery problems",
          "technical": "Software bugs"
        }
      },
      "refund": {
        "type": "noul",
        "instructions": "Does the customer request a refund?"
      }
    }
  }' | jq
```

### 11.6 备选：现成的第三方封装

GitHub 上有第三方项目 `Code2qing/jevk5-typesafe-server`，同样基于 `JevK5GGUF` + llama-server，声称与 TypeSafe System One API 严格对齐，带 Bearer 鉴权、`/health`、`/v1/models`，以及 Docker Compose 部署。我没有审计过其代码与行为，使用前请自行评估；它的温度配置表与本文 §2.3 一致。

### 11.7 通用聊天接口

llama-server 还提供 OpenAI 风格的通用接口（如 `/v1/chat/completions`），较新版本可能还提供其他兼容接口，具体以 `llama-server --help` 与官方文档为准。这些接口适合普通文本生成，**不能代替 JevK5 的决策读出**。

---

# 第四部分　运维

## 12. 验收清单

| # | 检查项 | 命令 | 通过标准 |
|---|---|---|---|
| 1 | 架构 | `uname -m` | `arm64` |
| 2 | llama.cpp | `llama-server --version` | 正常输出版本 |
| 3 | GPU | `llama-server --list-devices` 与启动日志 | 出现 Metal / Apple GPU，层已卸载到 GPU |
| 4 | Server | `curl http://127.0.0.1:8080/health` | 返回健康状态 |
| 5 | 客户端导入 | `python -c "from jevk5 import JevK5GGUF"` | 无报错 |
| 6 | 决策调用 | `python test.py` | 返回含概率的结构化结果 |
| 7 | 三类问题 | 分别跑 `noul` / `choice` / `score` | 至少各成功一次 |
| 8 | 概率合理 | 各选项概率之和 | 约等于 1 |
| 9 | Adapter（如启用） | `curl http://127.0.0.1:8090/health` | `status` 为 `ok` |

---

## 13. 性能与资源

### 13.1 延迟预期

作者公布的 4B Q8_0 实测：M1 Pro (Metal) 上约 0.6 秒/次（短文档，约 170 token），文档越长越慢。M5 的实测数据没有公开，**不要拿 H100 上的 13 ms 作预期**——那是 CUDA graphs 路径，本方案用不上。

### 13.2 内存

4.48 GB 的模型文件对 24 GB 很宽裕。实际占用还包括 macOS、Metal 缓冲区、KV cache、Python 进程与其他应用。先用 `-c 8192` 建立基线，再调整。

### 13.3 自己测一次延迟

```python
import time
from jevk5 import JevK5GGUF

model = JevK5GGUF(url="http://127.0.0.1:8080", temperature=1.22, knockout_temperature=0.93)
q = {"type": "noul", "instructions": "Is this a refund request?"}
text = "I was billed twice for order #4411. Please refund the duplicate charge today."

model.decide(text, q)  # 预热，首次调用更慢
ts = []
for _ in range(20):
    t = time.perf_counter()
    model.decide(text, q)
    ts.append(time.perf_counter() - t)
ts.sort()
print(f"p50={ts[10]*1000:.0f} ms  p95={ts[18]*1000:.0f} ms")
```

用你真实业务的材料长度来测，结果才有意义。

### 13.4 优化顺序

1. 确认 Metal 与 GPU 卸载正常；
2. 缩短输入材料；
3. 一个材料有多个问题时，合并成一次 `/v1/systemone` 请求（同一份 state 共享）；
4. 最后才考虑上下文大小与并发。

---

## 14. 网络与安全

- **默认只监听本机：** 保持 `--host 127.0.0.1`（llama-server 与 adapter 都一样）。
- **需要局域网访问时：** 才改为 `0.0.0.0`，并配合防火墙、API 密钥（adapter 已支持 `API_KEY`）、TLS 反向代理、请求大小与并发限制。
- **不要把未认证的服务暴露到公网。**
- JevK5 会读入外部文本（工单、邮件等），材料里可能夹带"忽略前面指令"之类的注入内容。它只输出选项概率，攻击面比聊天模型小，但**不要把概率直接用于高风险自动决策**，需要有阈值与人工兜底。

---

## 15. 开机自启（launchd）

建议先**手动运行稳定**，再配置自启。拆成两个服务：llama-server 与 adapter。下面的路径需替换为你的实际用户名；plist 中不能使用 `~`。

### 15.1 llama-server 服务

文件：`~/Library/LaunchAgents/local.jevk5.llama.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>local.jevk5.llama</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/bin/llama-server</string>
    <string>--hf-repo</string><string>alibiserikbay/JevK5-GGUF</string>
    <string>--hf-file</string><string>jevk5-4b-v0.3-Q8_0.gguf</string>
    <string>-c</string><string>8192</string>
    <string>-ngl</string><string>99</string>
    <string>-np</string><string>1</string>
    <string>--host</string><string>127.0.0.1</string>
    <string>--port</string><string>8080</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/你的用户名/jevk5-gguf/llama.log</string>
  <key>StandardErrorPath</key><string>/Users/你的用户名/jevk5-gguf/llama.err.log</string>
</dict>
</plist>
```

### 15.2 adapter 服务（可选）

文件：`~/Library/LaunchAgents/local.jevk5.adapter.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key><string>local.jevk5.adapter</string>
  <key>WorkingDirectory</key><string>/Users/你的用户名/jevk5-gguf</string>
  <key>ProgramArguments</key>
  <array>
    <string>/Users/你的用户名/jevk5-gguf/.venv/bin/uvicorn</string>
    <string>server:app</string>
    <string>--host</string><string>127.0.0.1</string>
    <string>--port</string><string>8090</string>
  </array>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><true/>
  <key>StandardOutPath</key><string>/Users/你的用户名/jevk5-gguf/adapter.log</string>
  <key>StandardErrorPath</key><string>/Users/你的用户名/jevk5-gguf/adapter.err.log</string>
</dict>
</plist>
```

### 15.3 加载与管理

```bash
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.jevk5.llama.plist
launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/local.jevk5.adapter.plist

launchctl kickstart -k gui/$(id -u)/local.jevk5.llama    # 重启
launchctl bootout gui/$(id -u)/local.jevk5.llama         # 停止并卸载
```

说明：

- launchd 本身不保证启动顺序，adapter 靠 `KeepAlive` 重试；adapter 的 `/health` 会反映后端是否就绪。⚠️ `JevK5GGUF` 在后端未就绪时构造是否会失败，我没有验证，如失败则 `KeepAlive` 会在后端起来后自动重启它。
- 首次必须先手动运行过一次，让模型下载进缓存；否则自启动时要在后台下载 4.5 GB。
- 路径 `/opt/homebrew/bin/llama-server` 是 Apple Silicon 上 Homebrew 的默认位置，可用 `which llama-server` 确认。

---

## 16. 日常操作、升级与卸载

### 16.1 日常命令

| 操作 | 命令 |
|---|---|
| 启动 llama-server | 见 [§5.1](#51-启动命令) |
| 启动 adapter | `cd ~/jevk5-gguf && source .venv/bin/activate && uvicorn server:app --host 127.0.0.1 --port 8090` |
| 检查后端 | `curl http://127.0.0.1:8080/health` |
| 检查 adapter | `curl http://127.0.0.1:8090/health` |
| 前台停止 | 在对应终端按 `Ctrl+C` |

### 16.2 升级

```bash
brew upgrade llama.cpp                       # 升级 llama.cpp 后重新跑 §12 验收
cd ~/jevk5-gguf && source .venv/bin/activate
pip install --no-deps --upgrade "jevk5 @ git+https://github.com/allebee/jevk5@<新标签>"
```

更换模型版本（如 v0.3 → 更新版本）时，**同步更换校准温度**，并回到 §2.3 核对。

### 16.3 卸载

```bash
launchctl bootout gui/$(id -u)/local.jevk5.llama 2>/dev/null
launchctl bootout gui/$(id -u)/local.jevk5.adapter 2>/dev/null
rm -f ~/Library/LaunchAgents/local.jevk5.*.plist
rm -rf ~/jevk5-gguf
brew uninstall llama.cpp
```

模型缓存位置取决于 llama.cpp 版本，可用 `llama-server --help` 查看缓存目录相关说明后手动清理。

---

## 17. 故障排查

| 现象 | 可能原因 | 处理 |
|---|---|---|
| `llama-server: command not found` | 未安装或 PATH 未生效 | `brew install llama.cpp`；`which llama-server`；若用 llama.app 安装器，检查 `~/.llama-app` 是否在 PATH 中 |
| 用 `llama server` 启动报错 | 子命令写错 | llama.app 的子命令是 `llama serve` |
| 启动日志显示全部在 CPU | Metal 未启用或 `-ngl` 未生效 | 确认启动命令含 `-ngl 99`；用 `llama-server --list-devices` 检查设备；升级 llama.cpp |
| 下载了错误的模型文件 | 用了 `:Q8_0` 简写 | 改用 `--hf-repo` + `--hf-file` 显式指定 |
| `/health` 长时间非健康 | 模型仍在下载或加载 | 查看启动日志；首次下载约 4.5 GB |
| `pip install` 找不到标签 | 标签不存在 | 用 `git ls-remote --tags --refs` 查实际标签 |
| `ImportError` / `ModuleNotFoundError` | `--no-deps` 漏装轻量依赖 | 按报错补装缺失包，不要装 torch |
| `JevK5GGUF(...)` 参数报错 | 版本间参数名不同 | `help(JevK5GGUF)` 查看签名 |
| 概率异常自信或异常保守 | 温度与模型版本不匹配 | 对照 §2.3 修正 |
| 结果与官方报告差异大 | 绕过了 `JevK5GGUF`，或用了聊天接口 | 只通过 `JevK5GGUF` 调用 |
| 超长材料报错或结果变差 | 超出 `-c` | 缩短材料，或适当调大 `-c` 并复测 |
| `/v1/systemone`（8080）返回 501 | llama.cpp 原生端点不支持 JevK5 | 属预期，改走 §11.4 的 adapter（8090） |
| 延迟偏高 | GPU 未生效、上下文过大、内存压力、并发过高 | 逐项检查 §13.4，并用 §13.3 的脚本复测 |

---

# 附录

## 附录 A　改用 9B 模型

仅在需要处理**选项很多**的场景（>16 个，如数十至上百类的意图分类）时考虑。作者数据显示 9B 在这类任务上比 4B 高约 8 个百分点。其余场景作者建议默认用 4B。

要点：

- 文件：`jevk5-9b-v0.3-Q8_0.gguf`，⚠️ 约 9.5 GB，24 GB 内存可承受。
- 与 bf16 答案一致：Q8_0 为 229/231，Q5_K_M 为 225/231。
- 温度：`temperature=1.049`，`knockout_temperature=1.2`（务必同步修改，见 §2.3）。
- 速度：约比 4B 慢 2–3 倍。
- 缺点：在 JevBench 公开题困难档上准确率与校准都不如 4B。
- 不要使用 9B 的 bf16（约 19 GB），在 24 GB 机器上过于吃紧。

启动命令只需把文件名换掉：

```bash
llama-server \
  --hf-repo alibiserikbay/JevK5-GGUF \
  --hf-file jevk5-9b-v0.3-Q8_0.gguf \
  -c 8192 -ngl 99 -np 1 \
  --host 127.0.0.1 --port 8080
```

```python
model = JevK5GGUF(url="http://127.0.0.1:8080",
                  temperature=1.049, knockout_temperature=1.2)
```

---

## 附录 B　零依赖参考客户端

用作 `JevK5GGUF` 的交叉验证，或在不想安装 jevk5 包时使用。**只用 Python 标准库**，仅支持 ≤16 个选项。逻辑来自作者的 GGUF 模型卡，我把温度改成了 v0.3 4B 的 1.22（模型卡该示例的原始值对应的是更早的版本）。

```python
import json, math, urllib.request

URL = "http://127.0.0.1:8080"   # llama-server
T = 1.22                         # v0.3 4B；9B 用 1.049（见 §2.3）
LETTERS = "ABCDEFGHIJKLMNOP"
SYSTEM = ("Apply the supplied criterion to the supplied evidence. Choose exactly one listed option. "
          "Respond with only its uppercase letter, with no explanation or reasoning.")

def post(path, body):
    req = urllib.request.Request(URL + path, json.dumps(body).encode(),
                                 {"Content-Type": "application/json"})
    return json.load(urllib.request.urlopen(req))

def decide(evidence, criterion: str, options: dict) -> dict:
    """options: {id: description}；返回每个 id 的校准概率。"""
    ids = list(options)
    user = json.dumps({"evidence": evidence, "criterion": criterion,
                       "options": [{"letter": LETTERS[i], "description": f"{k}: {options[k]}"}
                                   for i, k in enumerate(ids)]}, ensure_ascii=False)
    prompt = (f"<|im_start|>system\n{SYSTEM}<|im_end|>\n<|im_start|>user\n{user}<|im_end|>\n"
              "<|im_start|>assistant\n<think>\n\n</think>\n\n")
    tokens = post("/tokenize", {"content": prompt, "add_special": False,
                                "parse_special": True})["tokens"]
    top = post("/completion", {"prompt": tokens, "n_predict": 1, "n_probs": 40,
                               "temperature": 0, "cache_prompt": False}
               )["completion_probabilities"][0]["top_logprobs"]
    seen = {t["token"]: t["logprob"] for t in top}
    z = [seen.get(LETTERS[i], min(seen.values()) - 2.0) for i in range(len(ids))]
    w = [math.exp((v - max(z)) / T) for v in z]
    return {k: x / sum(w) for k, x in zip(ids, w)}

print(decide("I was billed twice for order #4411. Please refund the duplicate charge today.",
             "Which team should handle this?",
             {"billing": "Payments and refunds", "tech": "Bugs", "sales": "New purchases"}))
```

约定（来自模型卡）：

- 是/否问题：选项按顺序传 `{"true": "...", "false": "..."}`。
- 评分问题：按等级顺序传 `{"0": "...", "1": "...", ...}`。
- prompt 必须以 `parse_special: true` 分词，聊天标记才能保持为单个 token。

用它与 `JevK5GGUF.decide()` 对同一输入各跑一次，两者概率应基本一致；明显不一致时，先怀疑温度或 jevk5 版本。⚠️ 该示例的 prompt 格式与第三方项目对 v0.3 的移植一致，但我没有在 v0.3 上亲自跑过。

---

## 附录 C　参考资料

- JevK5 运行时与说明：https://github.com/allebee/jevk5
- JevK5 4B 模型：https://huggingface.co/alibiserikbay/JevK5
- JevK5 9B 模型：https://huggingface.co/alibiserikbay/JevK5-9B
- JevK5 GGUF 文件：https://huggingface.co/alibiserikbay/JevK5-GGUF
- llama.cpp：https://github.com/ggml-org/llama.cpp
- llama.cpp 发布页：https://github.com/ggml-org/llama.cpp/releases
- 第三方 System One 封装（未审计）：https://github.com/Code2qing/jevk5-typesafe-server
- JevBench：https://github.com/fstandhartinger/jevbench

---

## 附录 D　核对状态与待确认项

### D.1 已从公开资料核实

| 内容 | 依据 |
|---|---|
| 模型是基于 Qwen3.5 的决策模型，一次前向输出各选项概率，无生成 | 官方模型卡 |
| 官方运行时面向 CUDA；GGUF 供 llama.cpp 在 Apple GPU 等运行 | 官方模型卡 |
| `jevk5-4b-v0.3-Q8_0.gguf`，4,482,402,720 字节 (≈4.48 GB) | 第三方打包项目记录的哈希与大小 |
| 4B v0.3 温度 1.22 / 0.93；9B 1.049 / 1.2 | 第三方封装配置表；官方 9B 卡 |
| 9B Q8_0 与 bf16 答案一致 229/231，Q5_K_M 为 225/231 | 官方 9B 卡 |
| `llama-server` 的 `--hf-repo`、`--hf-file`、`-c`、`-ngl`、`/health`、`/v1/models`、`/tokenize`、`/completion`、`n_probs` | llama.cpp 与官方 GGUF 卡的用法 |
| llama.cpp 原生 `/v1/systemone` 存在，但仅支持特定模型家族 | llama.cpp 版本说明及相关文章 |
| 作者只测过 `llama-server`，其它基于 llama.cpp 的应用未验证 | 官方 GGUF 卡 |
| 4B Q8_0 在 M1 Pro Metal 上约 0.6 秒/次 | 官方 GGUF 卡 |

### D.2 待你在本机确认

| # | 待确认项 | 如何确认 |
|---|---|---|
| 1 | 4B v0.3 Q8_0 / Q5_K_M / Q4_K_M 与 bf16 的答案一致数 | 看 HF 上 GGUF 仓库的最新模型卡 |
| 2 | 9B Q8_0 文件大小 | HF 文件页 |
| 3 | 可用的 jevk5 标签；`JevK5GGUF` 从哪个版本起提供 | `git ls-remote --tags --refs ...` |
| 4 | `JevK5GGUF` 构造参数名（尤其 `url`） | `help(JevK5GGUF)` |
| 5 | jevk5 的最低 Python 版本，以及 `--no-deps` 是否需要补装包 | 安装后 `import` 测试 |
| 6 | llama.app 是否提供独立 `llama-server` 及参数是否完整 | `llama --help` |
| 7 | `-np 1` 是否必要 | 看你所用版本 `--help` 对 `-np` 默认值的说明 |
| 8 | `JevK5GGUF` 在 llama-server 未就绪时构造是否失败（影响自启顺序） | 先停 server 再 `python test.py` |
| 9 | M5 上的真实延迟 | §13.3 脚本 |
| 10 | 新版 llama.cpp 是否已加入对 JevK5 的原生 `/v1/systemone` 支持 | 查看版本说明 |
