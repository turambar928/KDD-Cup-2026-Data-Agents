# KDD Cup 2026 DataAgent-Bench 快速上手

## 前提条件

- 已克隆官方仓库到本地
- 已安装 `uv`（Python 包管理器）
- 有一个支持 OpenAI-compatible 接口的模型 API

---

## 第一步：安装依赖

```bash
cd /home/taozifu2025/kdd_cup
uv sync
```

---

## 第二步：下载数据集

### 下载地址

官方 Demo Dataset（Phase 1，共 50 个任务）：

**Google Drive：** https://drive.google.com/file/d/1c6u5WlFw4KV7CBRyXh5BvFYbKqxhBSbL/view

> 也可以在官方仓库 README 的 "Demo Dataset" 徽章中找到该链接。

### 下载并解压

**方法一：浏览器下载后上传到服务器**

1. 在本地浏览器打开上方链接，下载 zip 文件
2. 用 `scp` 或 `rsync` 上传到服务器：
   ```bash
   scp demo_samples_0417.zip taozifu2025@CIP-GPUSERVER-16:~/kdd_cup/
   ```

**方法二：服务器直接用 gdown 下载（需要能访问 Google）**

```bash
pip install gdown
gdown 1c6u5WlFw4KV7CBRyXh5BvFYbKqxhBSbL
```

### 解压到 public/ 目录

```bash
cd ~/kdd_cup
unzip demo_samples_0417.zip -d public/
```

解压后确认目录结构如下：

```
public/
├── input/
│   ├── task_11/
│   │   ├── task.json       # 题目（task_id、difficulty、question）
│   │   └── context/        # 数据文件（CSV、JSON、SQLite、文本等）
│   ├── task_24/
│   └── ...（共 50 个任务）
└── output/
    ├── task_11/
    │   └── gold.csv        # 标准答案
    └── ...
```

> `public/` 已加入 `.gitignore`，数据集不会被提交到 git。

---

## 第三步：创建本地配置文件

复制示例配置并修改：

```bash
cp configs/react_baseline.example.yaml configs/react_baseline.local.yaml
```

编辑 `configs/react_baseline.local.yaml`，填入你的模型信息：

```yaml
dataset:
  root_path: /home/taozifu2025/kdd_cup/public/input  # 数据集绝对路径

agent:
  model: Qwen3-32B-no-thinking   # 模型名称
  api_base: http://api.cipsup.cn/v1  # API 地址
  api_key: YOUR_API_KEY          # 替换为你的 key
  max_steps: 16
  temperature: 0.0

run:
  output_dir: artifacts/runs
  run_id:           # 留空则自动用时间戳命名
  max_workers: 4
  task_timeout_seconds: 600
```

**模型选择说明：**

| 模型 | 适用场景 |
|------|---------|
| `Qwen3-32B-no-thinking` | 推荐首选，指令遵循好，输出格式稳定，价格低 |
| `claude-sonnet-4-6` | 效果更强，但价格高 10 倍 |
| `Qwen3-32B`（thinking 版） | 不推荐，thinking 标签可能干扰 JSON 解析 |
| `bge-m3` | 不可用，这是 embedding 模型不是对话模型 |

---

## 第四步：验证配置

```bash
uv run dabench status --config configs/react_baseline.local.yaml
```

正常输出应显示 `dataset_root: present` 且 `Public tasks: 50`。

---

## 第五步：跑单个任务（测试）

```bash
uv run dabench run-task task_11 --config configs/react_baseline.local.yaml
```

成功后会输出：

```
Run output: artifacts/runs/20260513T083634Z
Task output: artifacts/runs/20260513T083634Z/task_11
Prediction CSV: artifacts/runs/20260513T083634Z/task_11/prediction.csv
```

查看结果：

```bash
cat artifacts/runs/20260513T083634Z/task_11/prediction.csv
# 对比标准答案
cat public/output/task_11/gold.csv
```

---

## 第六步：跑全量 benchmark

先用 `--limit` 快速验证几个任务：

```bash
uv run dabench run-benchmark --config configs/react_baseline.local.yaml --limit 5
```

没问题后跑全部 50 个任务：

```bash
uv run dabench run-benchmark --config configs/react_baseline.local.yaml
```

结果目录结构：

```
artifacts/runs/<run_id>/
├── summary.json          # 整体统计
├── task_11/
│   ├── prediction.csv    # 模型预测结果
│   └── trace.json        # Agent 推理过程（每一步的 thought/action/observation）
├── task_24/
└── ...
```

---

## 其他常用命令

查看某个任务的题目和数据文件：

```bash
uv run dabench inspect-task task_11 --config configs/react_baseline.local.yaml
```

---

## 注意事项

- `configs/react_baseline.local.yaml` 含有 API key，不要提交到 git
- `run_id` 留空时每次运行会生成新目录；填写固定值时若目录已存在会报错
- `max_workers: 4` 表示并行跑 4 个任务，API 有速率限制时可以调小
- thinking 模式的模型（如 `Qwen3-32B`、`claude-opus-4-6-thinking`）输出的 `<think>` 标签在有 ` ```json ` 围栏时能被正确解析，但稳定性不如 no-thinking 版本
