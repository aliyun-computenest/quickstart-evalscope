# EvalScope 社区版服务实例部署文档

## 概述

EvalScope 是 ModelScope 开源的大语言模型评测与压测框架，内置压力测试模块（`evalscope perf`），可对任意 OpenAI 兼容的模型 API 进行高并发压测，输出 TTFT、TPOT、吞吐、RPS 等关键指标，并支持将结果上报 SwanLab 进行可视化实验跟踪。

本服务在一台 ECS 上预装了 `evalscope` 与 `swanlab`（含压测所需的全部依赖），开箱即可对您自己部署的模型服务发起压测。本文介绍如何开通并部署该服务，以及部署后的访问方式与压测用法。

- 开源地址：[https://github.com/modelscope/evalscope](https://github.com/modelscope/evalscope)
- 官方文档：[https://evalscope.readthedocs.io/zh-cn/latest/](https://evalscope.readthedocs.io/zh-cn/latest/)

## 部署流程

### 部署步骤

1. 单击部署链接，进入服务实例部署界面：[部署链接](https://computenest.console.aliyun.com/service/instance/create?type=user&ServiceId=service-13c1c7d4aab245b5aa46)
2. 按界面提示填写参数。**如需私网压测，务必在「可用区配置」中选择模型服务所在的已有 VPC 与交换机。**
3. 单击「下一步：确认订单」，确认预估费用与参数后单击「立即创建」。
4. 等待部署完成（约 3-5 分钟），实例状态变为「已部署」。

### 验证结果

部署完成后，在服务实例详情页可看到 ECS 实例信息。登录实例（登录方式见下节）后执行以下命令验证环境就绪：

```bash
# 验证 evalscope 与压测依赖
evalscope --version
evalscope perf --help

# 验证 swanlab
swanlab --help
```

`evalscope perf --help` 能正常输出帮助信息，即表示压测模块及其依赖已完整安装。

---

## 访问方式：公网与私网

这是使用本服务最需要先想清楚的一件事，包含两个层面：**① 您如何登录 EvalScope 实例；② EvalScope 实例如何访问被压测的模型 API。**

### 一、登录 EvalScope 实例

本服务实例**默认不分配公网 IP**（公网带宽为 0），且安全组默认不放行任何入方向端口。因此有以下两种登录方式：

**方式一：免公网登录（推荐）**

无需任何网络改动，直接在 ECS 控制台使用 Workbench 远程连接：

1. 进入服务实例详情页，单击 ECS 实例 ID 跳转到 ECS 控制台。
2. 在实例列表中单击「远程连接」→「通过 Workbench 远程连接」。
3. 输入 root 与部署时设置的实例密码即可登录。

该方式走阿里云内部通道，不需要公网 IP、不需要改安全组，也不产生公网流量费用，适合绝大多数场景。

**方式二：公网 SSH 登录**

若您习惯用本地终端 SSH，需要手动开通公网访问（两步都要做，缺一不可）：

1. **分配公网 IP**：在 ECS 控制台为实例绑定一个弹性公网 IP（EIP），或修改实例的公网带宽为大于 0 的值。
2. **放行 SSH 端口**：在实例所属安全组的入方向添加规则，放行 TCP `22` 端口，授权对象填写您的办公网出口 IP（**不建议填 `0.0.0.0/0`**）。

完成后即可 `ssh root@<公网IP>`。

> 若 SSH 出现 `Operation timed out`，几乎都是上述第 2 步安全组未放行导致的，请优先检查安全组入方向规则。

### 二、EvalScope 访问被压测的模型 API

压测命令中 `--url` 填写的地址，决定了走公网还是私网。**强烈建议走私网。**

#### 私网访问（推荐）

**适用前提**：EvalScope 实例与模型服务在**同一 VPC**内。

```bash
# --url 使用模型服务的私网 IP
--url http://192.168.1.100:8000/v1/chat/completions
```

优势：延迟低、带宽大、不产生公网流量费用，压测得到的 TTFT/TPOT 更接近模型服务的真实性能（不被公网抖动干扰）。

**⚠️ 关键注意事项：同一 VPC 内的两台 ECS 默认并不能互相访问。** 私网连通还需要满足：

1. **模型服务侧安全组放行**：在**模型服务所在实例**的安全组入方向，添加规则放行模型端口（如 TCP `8000`），授权对象填写 EvalScope 实例的私网 IP（如 `192.168.1.175/32`）或其所在交换机网段（如 `192.168.1.0/24`）。
2. **模型服务监听地址正确**：模型服务需监听 `0.0.0.0`（而非 `127.0.0.1`），否则只能本机访问。以 vLLM 为例需带 `--host 0.0.0.0`。
3. **网段可达**：若两台实例在同一 VPC 的不同交换机（不同子网），VPC 内路由默认互通，仅需完成第 1 步；若在**不同 VPC**，则私网不通，需先打通（VPC 对等连接/云企业网），否则请改用公网方式。

#### 公网访问

**适用场景**：模型服务不在同一 VPC（例如在别的账号、别的地域，或使用第三方 API）。

```bash
# --url 使用模型服务的公网 IP 或域名
--url http://<公网IP>:8000/v1/chat/completions
```

需满足：

1. **模型服务侧**：已绑定公网 IP，且安全组入方向放行模型端口（如 TCP `8000`）给 EvalScope 实例的公网出口 IP。
2. **EvalScope 实例侧**：具备访问公网的能力——绑定 EIP，或所在 VPC 已配置 NAT 网关。**默认部署的实例没有公网出口，需要自行配置。**

注意：公网压测的时延包含了公网链路开销，测得的 TTFT 会明显高于私网，不适合作为模型性能基线，仅适合验证可用性或评估端到端用户体验。

#### 压测前的连通性自检

正式压测前，务必先用一条 `curl` 确认网络与鉴权都通，可以避免大量无效排查：

```bash
# 替换为您的地址与 API Key；若模型未开启鉴权可去掉 -H 参数
curl -s -m 10 http://<模型地址>:8000/v1/models \
  -H "Authorization: Bearer <YOUR_API_KEY>"
```

- 正常返回模型列表 JSON → 网络与鉴权均正常，**请记下返回中的 `id` 字段**，它就是压测时 `--model` 必须填写的模型名。
- 返回 `{"error":"Unauthorized"}` 或 401 → 网络通，但鉴权失败，需在压测命令中带上正确的 `--api-key`。
- 长时间无响应 / 超时 → 网络不通，请按上文检查安全组与监听地址。

---

## 压测使用：随机压测与自定义数据集

登录实例后即可执行 `evalscope perf`。以下两类用法覆盖绝大多数压测需求。

### 通用必填参数

| 参数 | 说明 |
| --- | --- |
| `--model` | 模型名，**必须与 `/v1/models` 返回的 `id` 完全一致**（例如是 `Qwen/Qwen3-32B` 而不是 `Qwen3-32B`，写错会报 model not found）。 |
| `--url` | 模型 API 地址，需带完整路径，如 `http://<IP>:8000/v1/chat/completions`。 |
| `--api` | 接口协议，OpenAI 兼容接口填 `openai`。 |
| `--api-key` | 模型服务的 API Key。**若模型开启了鉴权，此参数必填**，否则全部请求返回 401。 |
| `--parallel` | 并发数，可传多个值做并发梯度测试。 |
| `--number` | 总请求数，需与 `--parallel` 数量对应。 |

### 一、随机压测（`--dataset random`）

随机压测由 EvalScope 现场生成指定长度的 prompt，**输入/输出长度完全可控**，是衡量模型服务性能基线、做规格或参数对比的首选方式。

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai \
  --api-key sk-xxxxxxxx \
  --parallel 3 \
  --number 30 \
  --dataset random \
  --tokenizer-path Qwen/Qwen3-32B \
  --prefix-length 0 \
  --min-prompt-length 1024 \
  --max-prompt-length 1024 \
  --min-tokens 1024 \
  --max-tokens 1024 \
  --extra-args '{"ignore_eos": true}'
```

关键参数说明：

| 参数 | 说明 |
| --- | --- |
| `--tokenizer-path` | **`random` 模式必填**。用于生成并计数 token，填模型的 ModelScope ID（如 `Qwen/Qwen3-32B`，会自动下载分词器）或本地分词器目录。 |
| `--min-prompt-length` / `--max-prompt-length` | 输入长度范围。**两者设为相同值即可得到定长输入**（这是 `random` 独有的能力）。 |
| `--min-tokens` / `--max-tokens` | 输出长度范围，设为相同值可固定输出长度。 |
| `--prefix-length` | 公共前缀长度，设为 `0` 表示无公共前缀（可避免 prefix cache 干扰）；设为大于 0 可专门测试前缀缓存命中效果。 |
| `--extra-args '{"ignore_eos": true}'` | 让模型忽略结束符、必须生成到 `max-tokens`，从而保证每条输出长度严格一致，结果更可比。 |

> 提示：定长输入输出 + `ignore_eos` 是做性能基线的标准做法。若不加 `ignore_eos`，模型可能提前结束生成，导致各请求输出长度不一，吞吐数据失去可比性。

**并发梯度测试**（找服务的吞吐拐点与最大承载）：

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai --api-key sk-xxxxxxxx \
  --dataset random --tokenizer-path Qwen/Qwen3-32B \
  --min-prompt-length 1024 --max-prompt-length 1024 \
  --min-tokens 512 --max-tokens 512 \
  --extra-args '{"ignore_eos": true}' \
  --parallel 1 4 8 16 \
  --number 20 40 80 160
```

### 二、自定义数据集（使用 ModelScope 官方数据集）

随机数据是无语义的合成文本。若要贴近真实业务负载，应使用真实语料。EvalScope 的真实数据集**统一从 ModelScope 官方下载**，有「自动下载」和「先手动下载再指定路径」两种用法。

#### 方式一：自动从 ModelScope 下载（最简单）

指定 `--dataset` 为内置数据集名，EvalScope 会自动从 ModelScope 拉取，无需手动准备：

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai --api-key sk-xxxxxxxx \
  --parallel 4 --number 40 \
  --dataset openqa
```

常用内置真实数据集（均托管在 ModelScope 官方）：

| `--dataset` 取值 | 数据集来源 | 特点与适用场景 |
| --- | --- | --- |
| `openqa` | [AI-ModelScope/HC3-Chinese](https://www.modelscope.cn/datasets/AI-ModelScope/HC3-Chinese/summary) | 中文问答，prompt 较短（一般 < 100 token），适合短输入高并发场景 |
| `longalpaca` | [AI-ModelScope/LongAlpaca-12k](https://www.modelscope.cn/datasets/AI-ModelScope/LongAlpaca-12k) | 长文本，prompt 较长（一般 > 6000 token），适合长上下文压测 |
| `share_gpt_zh` / `share_gpt_en` | [swift/sharegpt](https://www.modelscope.cn/datasets/swift/sharegpt) | 真实对话语料（约 70k 条），中英文可选，最贴近聊天类业务 |
| `flickr8k` / `kontext_bench` | ModelScope 官方多模态数据集 | 构建图文输入，用于压测多模态模型 |

> 默认下载源即为 ModelScope（`--data-source modelscope`），国内网络无需额外配置即可直接拉取。

#### 方式二：先从 ModelScope 手动下载，再用 `--dataset-path` 指定

适用于以下场景：需要提前把数据缓存到本地、实例出公网受限、或希望多次压测复用同一份数据。

**第 1 步：从 ModelScope 官方下载数据集**

实例已随 evalscope 安装了 ModelScope 命令行工具，直接使用：

```bash
# 创建数据目录
mkdir -p /opt/evalscope/datasets

# 从 ModelScope 官方下载数据集（以 HC3-Chinese 为例）
/opt/evalscope/venv/bin/modelscope download \
  --dataset AI-ModelScope/HC3-Chinese \
  --local_dir /opt/evalscope/datasets/HC3-Chinese

# 查看下载得到的数据文件
ls -lh /opt/evalscope/datasets/HC3-Chinese
```

也可以在 [ModelScope 数据集页面](https://www.modelscope.cn/datasets)搜索所需数据集，复制其数据集 ID（形如 `组织名/数据集名`）后替换上面的 `--dataset` 参数。

**第 2 步：压测时通过 `--dataset-path` 指定本地路径**

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai --api-key sk-xxxxxxxx \
  --parallel 4 --number 40 \
  --dataset openqa \
  --dataset-path /opt/evalscope/datasets/HC3-Chinese/open_qa.jsonl
```

注意：`--dataset` 仍需指定为对应的解析模式，它决定了 EvalScope 从文件中读取哪个字段：

- `--dataset openqa` → 读取 jsonl 的 `question` 字段
- `--dataset longalpaca` → 读取 jsonl 的 `instruction` 字段

#### 方式三：使用自己的纯文本语料（`line_by_line`）

若您有自己的业务 prompt，整理成一行一条的 txt 文件即可直接压测：

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai --api-key sk-xxxxxxxx \
  --parallel 4 --number 40 \
  --dataset line_by_line \
  --dataset-path /opt/evalscope/datasets/my_prompts.txt
```

#### 用真实数据做定长对照压测

真实语料长度参差不齐，若想在真实语料上做**固定输入长度**的对照压测，用 `--dataset-args` 的 `target_input_len`（需配合 `--tokenizer-path`）：

```bash
evalscope perf \
  --model Qwen/Qwen3-32B \
  --url http://192.168.1.100:8000/v1/chat/completions \
  --api openai --api-key sk-xxxxxxxx \
  --parallel 4 --number 40 \
  --dataset share_gpt_zh \
  --tokenizer-path Qwen/Qwen3-32B \
  --dataset-args '{"target_input_len": 2048}'
```

> `--min/max-prompt-length` 只做**筛选**（不满足长度的样本被丢弃，结果仍长短不一）；`target_input_len` 会把每条 prompt **截断改写**到指定长度。真实数据集想要"每条恰好 N token"只能用后者。

### 三、结果查看与 SwanLab 上报

压测结束后，终端会打印汇总报告（成功率、RPS、TTFT、TPOT、吞吐等），同时在 `outputs/<时间戳>/<模型名>/` 目录下生成：

- `perf_report.html`：可视化 HTML 报告
- `performance_summary.txt`：文本摘要
- `benchmark_data.db`：每条请求的明细数据（sqlite3，可自行分析）

若需将压测结果上报到 SwanLab 进行多次实验对比，追加两个参数即可：

```bash
evalscope perf \
  ... \
  --visualizer swanlab \
  --swanlab-api-key <您的SwanLab API Key> \
  --name evalscope-qwen3-32b-test
```

SwanLab API Key 可在 [SwanLab 官网](https://swanlab.cn)登录后于个人设置中获取。上报成功后命令行会输出实验链接，打开即可查看图表。

---

## 问题排查

| 现象 | 原因与解决办法 |
| --- | --- |
| SSH 连接超时 `Operation timed out` | 实例默认无公网 IP 且安全组未放行。改用 ECS Workbench 远程连接，或按[登录方式二](#一登录-evalscope-实例)绑定 EIP 并放行 22 端口。 |
| `curl` 模型地址超时不返回 | 模型侧安全组未放行端口，或模型未监听 `0.0.0.0`，或两者不在同一 VPC。参见[私网访问注意事项](#私网访问推荐)。 |
| 返回 `{"error":"Unauthorized"}` / 401 | 模型服务开启了鉴权，压测命令需补上正确的 `--api-key`。 |
| 报错 model not found | `--model` 与模型服务实际的模型 id 不一致。先用 `curl /v1/models` 查到 `id` 字段再填写。 |
| `ModuleNotFoundError: No module named 'uvicorn'` | 压测依赖缺失（仅早期版本实例可能出现）。执行 `/opt/evalscope/venv/bin/pip install uvicorn fastapi sse_starlette` 修复。 |
| `--dataset random` 报错缺少 tokenizer | `random` 模式必须指定 `--tokenizer-path`。 |
| 数据集下载缓慢或失败 | 实例需具备公网出口（EIP 或 NAT 网关）才能访问 ModelScope。也可参照[方式二](#方式二先从-modelscope-手动下载再用---dataset-path-指定)在有网络的环境下载后上传到实例。 |
| 各请求输出长度不一致、吞吐数据波动大 | 压测命令未加 `--extra-args '{"ignore_eos": true}'`，模型提前结束生成所致。 |

更多用法与参数详见官方文档：[EvalScope 压测参数说明](https://evalscope.readthedocs.io/zh-cn/latest/user_guides/stress_test/parameters.html)

## 联系我们

- EvalScope 官方文档：[https://evalscope.readthedocs.io/zh-cn/latest/](https://evalscope.readthedocs.io/zh-cn/latest/)
- 社区版开源地址：[https://github.com/modelscope/evalscope](https://github.com/modelscope/evalscope)
- ModelScope 数据集广场：[https://www.modelscope.cn/datasets](https://www.modelscope.cn/datasets)
- SwanLab 官网：[https://swanlab.cn](https://swanlab.cn)
