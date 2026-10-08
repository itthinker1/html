# JevK5 on MacBook Pro M5 (24 GB) — MLX 安装与使用指南

> **文档类型：** 安装与运维指南  
> **适用设备：** MacBook Pro，Apple M5，24 GB Unified Memory  
> **推荐技术栈：** MLX + Metal + `mlx-lm` + 第三方 JevK5 MLX 转换版  
> **文档日期：** 2026-10-08  
> **文档状态：** 实用部署指南

---

## 1. 文档概述

本文用于指导在 **MacBook Pro M5 / 24 GB Unified Memory** 上，通过 **Apple MLX** 在本机运行 JevK5，并使用 M5 GPU 完成推理加速。

本指南采用的模型为：

```text
SirSahOl/JevK5-chat-mlx-8bit
```

该模型是 `alibiserikbay/JevK5` 的**第三方 MLX 转换版**。模型卡当前描述其约 3.9B 参数、8-bit 量化、safetensors/MLX 格式，模型文件约 4.47 GB，并提供 `mlx_lm.chat` 与 `mlx_lm.generate` 的运行方式。

### 1.1 本指南适用范围

本方案适合以下目标：

- 在 Apple Silicon 上使用 MLX 原生运行模型；
- 使用 M5 GPU / Metal 进行推理；
- 进行交互式文本生成；
- 使用 `mlx_lm.server` 提供 OpenAI-compatible HTTP Chat API；
- 作为本地 LLM 后端供其他应用调用。

### 1.2 重要边界

JevK5 的核心能力并非普通聊天生成，而是 typed decision，例如：

- `noul`：是 / 否决策；
- `choice`：离散选项决策；
- `score`：等级/序数评分；
- 通过候选答案 logits 获得概率与 confidence。

**本指南的 `mlx_lm.server` 路线不等同于官方 JevK5 decision runtime。** 如果你的主要目标是获得官方 JevK5 的 calibrated decision 语义、`JevK5GGUF` readout 或 `/v1/systemone`，请使用配套的 **llama.cpp + Metal 安装与使用指南**。

---

## 2. 技术架构

### 2.1 推理链路

```text
JevK5 MLX 模型
      │
      ▼
    MLX-LM
      │
      ▼
    Metal
      │
      ▼
  Apple M5 GPU
```

### 2.2 HTTP 服务链路

```text
客户端 / Agent / 应用
           │
           │ HTTP
           ▼
  mlx_lm.server :8080
           │
           ▼
       MLX 模型
           │
         Metal
           │
           ▼
        M5 GPU
```

### 2.3 MLX 与 MPS 的关系

本方案**不使用 PyTorch MPS**。

二者的关系如下：

```text
PyTorch
   │
   ▼
  MPS
   │
   ▼
Apple GPU
```

而本指南采用：

```text
MLX
 │
 ▼
Metal
 │
 ▼
Apple GPU
```

因此：

- 不需要为了 GPU 加速安装 PyTorch；
- 不需要配置 `torch.device("mps")`；
- 不需要 CUDA；
- 由 MLX 直接使用 Apple GPU。

---

## 3. 推荐配置

| 项目 | 推荐值 |
|---|---|
| 设备 | MacBook Pro，Apple M5 |
| Unified Memory | 24 GB |
| 机器学习框架 | MLX |
| 推理/服务工具 | `mlx-lm` |
| GPU 后端 | Metal |
| JevK5 模型 | `SirSahOl/JevK5-chat-mlx-8bit` |
| 格式 | MLX / safetensors |
| 量化 | 8-bit |
| 模型大小 | 约 4.47 GB |
| 本地 HTTP 端口 | 8080 |
| HTTP API | OpenAI-compatible `/v1/chat/completions` |
| JevK5 原生 `/v1/systemone` | 不由 `mlx_lm.server` 提供 |

对于 24 GB Unified Memory，8-bit 版本属于合理的首选配置，可以为 macOS、模型运行时和其他应用保留较充足的内存空间。

---

## 4. 前置条件

### 4.1 检查 CPU 架构

执行：

```bash
uname -m
```

预期结果：

```text
arm64
```

再次检查 Python：

```bash
python3 -c 'import platform, sys; print(platform.machine()); print(sys.version)'
```

应使用原生 ARM64 Python。避免在 Rosetta/x86_64 Python 环境中部署。

### 4.2 检查硬件

```bash
system_profiler SPHardwareDataType
```

确认：

- 芯片为 Apple M5；
- 内存为 24 GB。

### 4.3 检查 macOS

```bash
sw_vers -productVersion
```

MLX 官方项目针对 Apple Silicon 提供原生支持；具体最低版本要求应以当前 MLX 官方仓库说明为准。

---

## 5. 创建独立 Python 环境

为避免与其他项目的 Python 依赖冲突，建议创建专用环境。

### 5.1 使用 `uv`（推荐）

```bash
brew install uv
```

创建环境：

```bash
mkdir -p ~/jevk5-mlx
cd ~/jevk5-mlx

uv venv
source .venv/bin/activate
```

### 5.2 使用标准 `venv`

```bash
mkdir -p ~/jevk5-mlx
cd ~/jevk5-mlx

python3 -m venv .venv
source .venv/bin/activate

python3 -m pip install -U pip
```

两种方式任选其一，不建议同时维护两个独立环境。

---

## 6. 安装 MLX-LM

在已激活的虚拟环境中执行：

```bash
pip install -U mlx-lm
```

验证：

```bash
mlx_lm --help
```

可进一步检查版本：

```bash
python -c "import mlx; import mlx_lm; print('MLX:', mlx.__version__); print('mlx-lm:', mlx_lm.__version__)"
```

---

## 7. 下载并运行 JevK5

### 7.1 交互式运行

```bash
mlx_lm.chat \
  --model SirSahOl/JevK5-chat-mlx-8bit
```

首次运行时，模型会从 Hugging Face 下载并缓存到本地。

### 7.2 单次生成

```bash
mlx_lm.generate \
  --model SirSahOl/JevK5-chat-mlx-8bit \
  --prompt "Explain the difference between confidence and probability."
```

### 7.3 模型来源说明

本指南的模型链路为：

```text
官方 JevK5 权重
alibiserikbay/JevK5
        │
        ▼
第三方 MLX 转换
SirSahOl/JevK5-chat-mlx-8bit
        │
        ▼
MLX-LM / Apple Silicon
```

如需复现实验结果，应固定记录 Hugging Face 仓库和 revision/commit。

---

## 8. 启动 HTTP API

`mlx-lm` 提供面向文本生成的 HTTP server，其接口风格与 OpenAI Chat API 类似。

### 8.1 启动服务器

```bash
mlx_lm.server \
  --model SirSahOl/JevK5-chat-mlx-8bit \
  --host 127.0.0.1 \
  --port 8080
```

服务地址：

```text
http://127.0.0.1:8080
```

### 8.2 查询模型列表

```bash
curl http://127.0.0.1:8080/v1/models
```

使用 `jq` 格式化：

```bash
brew install jq
curl http://127.0.0.1:8080/v1/models | jq
```

### 8.3 调用 Chat Completions

```bash
curl http://127.0.0.1:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "SirSahOl/JevK5-chat-mlx-8bit",
    "messages": [
      {
        "role": "user",
        "content": "What is the capital of France?"
      }
    ],
    "max_tokens": 128,
    "temperature": 0.7
  }' | jq
```

---

## 9. 使用 OpenAI-Compatible Client

### 9.1 Python / OpenAI SDK

安装：

```bash
pip install -U openai
```

示例：

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://127.0.0.1:8080/v1",
    api_key="local",
)

response = client.chat.completions.create(
    model="SirSahOl/JevK5-chat-mlx-8bit",
    messages=[
        {
            "role": "user",
            "content": (
                "Classify this support ticket: "
                "The customer was charged twice and wants a refund."
            ),
        }
    ],
    max_tokens=128,
    temperature=0.2,
)

print(response.choices[0].message.content)
```

### 9.2 适用客户端

任何支持 OpenAI-compatible `base_url` 的客户端，都可以尝试指向：

```text
http://127.0.0.1:8080/v1
```

普通文本生成调用：

```text
POST /v1/chat/completions
```

具体功能支持仍取决于客户端、server 以及模型本身，不能仅因“OpenAI-compatible”就假定所有高级特性都完全兼容。

---

## 10. JevK5 决策能力与本方案的边界

### 10.1 JevK5 的目标任务

JevK5 的核心任务可以抽象为：

```text
state
  +
question
  +
candidate options
  ↓
candidate-answer logits
  ↓
probability distribution
```

官方 runtime 支持：

```text
noul
choice
score
```

### 10.2 为什么 `mlx_lm.server` 不等价于官方 JevK5 decision runtime？

如果仅通过 prompt 要求模型：

```text
Choose A, B or C
```

普通 Chat API 的目标是生成文本，例如：

```text
A. billing
```

而 JevK5 decision runtime 关心的是候选答案的 logits 和经过 JevK5 calibration 的概率，例如：

```text
billing    0.xx
shipping   0.xx
technical  0.xx
```

因此，不应将以下两个接口视为等价：

```text
MLX-LM Chat API
```

和：

```text
JevK5 typed-decision API
```

---

## 11. 若目标是 `/v1/systemone`

如应用层真正需要：

- `noul`；
- `choice`；
- `score`；
- calibrated probabilities；
- `JevK5GGUF` 官方 readout；
- `/v1/systemone`。

则建议采用配套的 **JevK5 + llama.cpp + Metal** 方案。

其逻辑为：

```text
llama-server
      ↓
answer-letter log-probabilities
      ↓
JevK5GGUF
      ↓
JevK5 calibration / readout
      ↓
structured decision
```

如果坚持使用 MLX，则需要自行实现 decision adapter，包括：

1. JevK5 prompt 构造；
2. 候选答案 token 定位；
3. logits readout；
4. JevK5 calibration；
5. `noul` / `choice` / `score` schema；
6. `/v1/systemone` HTTP 层。

这属于额外的软件开发工作，而不是 `mlx_lm.server` 的现成配置项。

---

## 12. GPU 加速验证

MLX 在 Apple Silicon 上使用 Metal。

运行模型时，可在 macOS 中打开：

```text
活动监视器
→ 窗口
→ GPU 历史记录
```

同时再次确认 Python 为原生 ARM64：

```bash
uname -m
```

预期：

```text
arm64
```

---

## 13. 内存与性能建议

M5 使用 Unified Memory：

```text
24 GB Unified Memory
        │
   ┌────┴────┐
   ▼         ▼
  CPU       GPU
```

约 4.47 GB 的 8-bit 模型属于较轻量配置，但实际运行占用还包括：

- macOS；
- Python 运行时；
- 模型执行 buffer；
- KV cache；
- 输入与输出张量；
- 其他应用。

建议顺序：

```text
先验证正确性
      ↓
记录内存与延迟
      ↓
再调整 context / cache / concurrency
```

不建议第一次安装时就追求最大 context 或最大并发。

---

## 14. 网络访问与安全

### 14.1 本机访问（推荐）

保持：

```bash
--host 127.0.0.1
```

这样服务仅绑定本机回环地址。

### 14.2 局域网访问

如果确需局域网访问，可使用：

```bash
--host 0.0.0.0
```

但不建议将未认证的模型服务直接暴露到公网。生产或长期运行时，应至少增加认证、访问控制和防火墙策略。

---

## 15. 常见问题与故障排查

### 15.1 `mlx_lm: command not found`

先激活虚拟环境：

```bash
cd ~/jevk5-mlx
source .venv/bin/activate
```

再执行：

```bash
mlx_lm --help
```

### 15.2 Python 是 `x86_64`

检查：

```bash
python3 -c 'import platform; print(platform.machine())'
```

如果是：

```text
x86_64
```

应改用 Apple Silicon 原生 Python。

### 15.3 模型无法下载

确认 Hugging Face 可访问，并检查仓库名称：

```text
SirSahOl/JevK5-chat-mlx-8bit
```

必要时固定 revision 进行复现。

### 15.4 HTTP 可以访问，但 Agent 行为异常

首先确认客户端使用的是：

```text
base_url = http://127.0.0.1:8080/v1
```

并调用：

```text
/v1/chat/completions
```

其次，不要因为“OpenAI-compatible”就假定所有工具调用、结构化输出、流式行为都与 OpenAI 官方服务完全一致。

### 15.5 输出与官方 JevK5 概率不一致

如果仅使用 `mlx_lm.server`，这属于预期范围。

如需验证 JevK5 官方 decision 结果，应使用官方 JevK5 runtime / GGUF readout 路径。

---

## 16. 可选：macOS 登录后自动启动

确认手动运行稳定后，可使用 `launchd` 自动启动。

建议先创建启动脚本：

```bash
#!/bin/zsh

cd "$HOME/jevk5-mlx"
source .venv/bin/activate

exec mlx_lm.server \
  --model SirSahOl/JevK5-chat-mlx-8bit \
  --host 127.0.0.1 \
  --port 8080
```

保存为：

```text
~/jevk5-mlx/start.sh
```

并：

```bash
chmod +x ~/jevk5-mlx/start.sh
```

待基础部署确认后，再配置 `~/Library/LaunchAgents/` 下的 plist。

---

## 17. 日常操作命令

### 17.1 启动交互式模型

```bash
cd ~/jevk5-mlx
source .venv/bin/activate

mlx_lm.chat \
  --model SirSahOl/JevK5-chat-mlx-8bit
```

### 17.2 启动 HTTP 服务

```bash
cd ~/jevk5-mlx
source .venv/bin/activate

mlx_lm.server \
  --model SirSahOl/JevK5-chat-mlx-8bit \
  --host 127.0.0.1 \
  --port 8080
```

### 17.3 查看模型

```bash
curl http://127.0.0.1:8080/v1/models | jq
```

### 17.4 停止服务

在运行 server 的终端执行：

```text
Ctrl+C
```

---

## 18. 参考资料

- JevK5 官方项目：  
  https://github.com/allebee/jevk5
- JevK5 官方模型：  
  https://huggingface.co/alibiserikbay/JevK5
- 本指南使用的第三方 MLX 转换：  
  https://huggingface.co/SirSahOl/JevK5-chat-mlx-8bit
- MLX：  
  https://github.com/ml-explore/mlx
- MLX-LM：  
  https://github.com/ml-explore/mlx-lm

---

## 19. 部署结论

对于 **MacBook Pro M5 / 24 GB Unified Memory**，MLX 路线建议采用：

```text
Apple M5
  │
  ▼
MLX
  │
  ▼
Metal
  │
  ▼
M5 GPU
  │
  ▼
JevK5 MLX 8-bit
  │
  ▼
mlx-lm
  ├─ CLI
  └─ OpenAI-compatible HTTP API
       └─ /v1/chat/completions
```

该方案适合以 **Apple Silicon 原生 MLX 推理**和**普通文本生成/API 服务**为主要目标。

如以 **JevK5 typed decisions + calibrated probabilities + `/v1/systemone`** 为核心目标，则应采用配套的 llama.cpp + GGUF + `JevK5GGUF` 方案。
