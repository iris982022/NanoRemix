# NanoRemix 上传与模型下载：请先读这里

整理日期：2026-11-04；推理工具版本仍为 **v0.3.0**。本次补全文档、输入示例、上游参考脚本和校验资料，没有重新训练或更换模型，也没有修改原包的推理实现。

## 1. 上传哪些文件

源码仓库和预训练权重分开发；不再上传一个几百 MiB 的全装 ZIP。网页上传时，源码 ZIP 内含99个文件，仓库首页 `README.md` 和 ZIP 共2个网页上传项，低于100个文件限制。五个模型 ZIP 则作为 GitHub Release 附件。

| 文件 | 用途 | 模型权重 |
|---|---|---|
| `NanoRemix_v0.3.0_GitHub_source_compact_20261104.zip` | 精简源码/说明归档，98个内部文件；上传时是1个ZIP文件 | 不含 |
| `NanoRemix_v0.3.0_models_RNA002_Binary_20261104.zip` | RNA002 Binary 三代 S1/S2 | 18 个 `.pkl` |
| `NanoRemix_v0.3.0_models_RNA002_Multi_20261104.zip` | RNA002 Multi 三代 S1/S2 与原始 manifests | 6 个 `.pkl` |
| `NanoRemix_v0.3.0_models_RNA004_Binary_20261104.zip` | RNA004 Binary 三代 S1/S2 | 30 个 `.pkl` |
| `NanoRemix_v0.3.0_models_RNA004_Multi_20261104.zip` | RNA004 Multi 三代 S1/S2 与原始 manifests | 10 个 `.pkl` |
| `NanoRemix_v0.3.0_models_SecondGen_20261104.zip` | 共享二代序列模型，融合需要；含网络定义和全部25个checkpoint | 25 个 `.pt` |

所有模型附件都放在仓库页 **Releases → Draft a new release** 的 Assets。每个附件只包含相应模型组，解压到源码仓库根目录后合并到 `models/`。每个 ZIP 都小于旧的 434 MiB 完整 ZIP；四组三代加二代总文件集合不变，共 89 个权重。此前的 `NanoRemix_complete_v0.3.0.zip` 与 `NanoRemix_final_code_bundle.zip` 是本地历史包，不需要再次上传；论文分析归档不参与 CLI 推理。

网页上传的 ZIP 会作为归档文件显示在仓库中；GitHub 不会自动展开里面的源文件。需要逐文件浏览源码时，用 GitHub Desktop/Git 解压源码包并推送其中的项目文件（Git 本身没有网页上传器的100文件批次限制）。

## 2. 共有几个版本

“二代/三代”在这里指 **NanoRemix 模型代际**：二代使用连续 81 nt 序列，三代使用 chemistry 对应的序列和电流特征；二代分支不直接读取 Illumina FASTQ/BAM。

| chemistry × task | 三代特征输入 | 三代权重数 | 可选输出 |
|---|---|---:|---|
| RNA002 Binary | 9 nt 上下文＋20 个数值 | 18 | 纯三代；二代＋三代融合 |
| RNA002 Multi | 9 nt 上下文＋20 个数值 | 6 | 同上 |
| RNA004 Binary | 17 nt 上下文＋36 个数值 | 30 | 同上 |
| RNA004 Multi | 17 nt 上下文＋36 个数值 | 10 | 同上 |

因此为 **4 套 chemistry/task、8 种主要工作流**。每种有 S1/S2 两种策略，可细分为 16 种配置；`--strategy both` 分别保存两套结果，不生成共识。旧名 `s3` 对应这里的 `s2`。四套共享 25 个二代网络，总权重数为 64 `.pkl`＋25 `.pt`＝89。

Binary 支持 A/m6A、C/m5C、U/pseudoU 三组。Multi 的 RNA002/RNA004 三代轴分别为 18/16 类，包含正常碱基；共有融合轴分别为 14/13 类，默认 extended 保留三代独有类别的质量。还可单独输出二代结果。全部类别和 S1/S2 模型内聚合参数见完整手册与 JSON 附件。

## 3. 每种模式需要哪些文件

| 模式 | 主输入 | 额外输入 |
|---|---|---|
| 纯三代 | 对应 chemistry 的 prepared CSV/TSV，可 gzip 压缩 | 对应三代权重 |
| 融合，现场运行二代 | 同上 | 同中心、正确 RNA 方向的连续 81 nt 列；或 FASTA＋明确中心坐标/链；以及二代权重 |
| 融合，已有二代概率 | 同上 | 表内或按身份键连接的完整 15 类二代概率，不需再加载二代网络 |
| 已有三代概率 | Binary 为对应 pair 概率；Multi 为完整原生 18/16 类轴 | 融合时仍需上述一种二代来源 |

`9mer_final` 是历史列名：RNA002 为 9 个碱基 token，RNA004 为 17 个。数值列是 `mean_output1..K`、`median_output1..K`、`std_output1..K`、`length_output1..K`，K 分别为 5、9。每行是一条 read 覆盖一个候选中心，不是自动聚合的位点。

81 nt 必须是连续序列，中心为第 41 位；不得把 9 个重叠 9-mer 拼出的 81 token 当成连续序列。独立概率表必须按 `record_key` 或明确复合键连接，不能按行号拼接。输入字段、格式样例、完整类别名和 FASTA/链坐标规则见[完整中文手册](docs/guide_20261104/README_版本输入预处理参数与计算.md)第 3 节及[人工示例](docs/guide_20261104/examples/README.md)。

## 4. 原始数据怎样得到输入

| chemistry | 上游链 | 到达推理输入前需要完成的工作 |
|---|---|---|
| RNA002 | FAST5 → Guppy 6.3.8 → ont-fast5-api 4.1.1 → Tombo 1.0 / GraphMap 0.5.2 → Step4 | 使用原定义归一化的 5-mer 信号统计，按 read 连续重叠构造 9 nt/20 值 prepared；保留同步坐标；融合另取同中心 81 nt |
| RNA004 | POD5 → Dorado 1.3.0 / `rna004_130bps_hac@v5.2.0`，保留 moves → 比对、samtools 过滤 → Remora 3.3.0 → NPZ＋sidecar → Step6 | 保留 read、参考、链与中心，构造正确方向 17 nt/36 值 prepared；融合另取沿转录/剪接路径的 81 nt |

完整版本证据、输出字段、信号统计、候选 region 的坐标语义、samtools 过滤以及分环境命令见[上游命令模板](docs/guide_20261104/PIPELINE_命令与输入衔接.md)和[上游副本索引](upstream_reference/README.md)。随包提供关键脚本及 RNA004 9-mer level 表；Guppy/Dorado 等可执行程序、参考基因组/转录组和原始测序数据需另行准备。

**RNA004 重要边界：v0.3.0 的旧 `prepare --format step6` 对当前按 RNA 方向保存的负链 Step6 存在候选丢失问题；本次没有修复该转换算法。** 附件给出可复现例子。现存 catalog 适配/剪接上下文脚本还依赖已知 catalog/标签，不能原样作为未知样本的一键入口。未知样本应先完成无真值依赖、正确处理链和剪接的适配与中心校验，再交给推理。详见[已知限制](docs/KNOWN_LIMITATIONS.zh-CN.md)。

## 5. 安装和运行

在解压后的 `NanoRemix/` 中，使用 Python 3.10/3.11：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[models,second]' -c constraints-models.txt
python verification/verify_distribution.py
```

然后从 Release 下载适用权重 ZIP。纯三代只下载一个对应的 chemistry/task ZIP；现场用81 nt运行二代融合，再下载共享 `SecondGen` ZIP。进入源码根目录运行：

```bash
unzip -o NanoRemix_v0.3.0_models_RNA004_Multi_20261104.zip -d .
unzip -o NanoRemix_v0.3.0_models_SecondGen_20261104.zip -d .
nanoremix doctor --model-root models --model-checksums models/SHA256.json \
  --chemistry RNA004 --task multi --strategy s2 --data-type biological --strict
```

纯三代请移除第二个解压命令，并把 doctor 的数据类型改成 `synthetic`，或按实际 biological 数据设定 `--mode thirdgen`。已有二代概率表则无需 SecondGen 权重；详见[模型选择表](docs/MODELS.md)。

依赖安装需要网络或事先准备的 wheel 镜像；本包不是离线环境镜像。原始数据预处理使用独立环境，不能把 Tombo、Remora 和推理环境合并安装。模型约束与本次实测环境的区别见[环境说明](environments/README.md)。仅检查包完整性可单独使用标准库脚本，无需先安装模型依赖。

不加载模型的人工概率示例：

```bash
nanoremix predict \
  --input examples/minimal_binary_probabilities.csv \
  --task binary --chemistry RNA004 --mods m6A \
  --input-kind probabilities --data-type synthetic --strategy s2 \
  --output results/NEW_probability_demo
```

自己的生物样本纯三代（002/004、binary/multi 按实际选择）：

```bash
nanoremix predict --input /path/to/prepared004.csv \
  --output results/NEW_RNA004_multi_third \
  --chemistry RNA004 --task multi --mods all --mode thirdgen --strategy s2 \
  --model-root models --model-checksums models/SHA256.json --trust-model-files
```

同一 prepared 含正确的 `sequence81` 列时，输出融合及两个分支：

```bash
nanoremix predict --input /path/to/prepared004_with81.csv \
  --output results/NEW_RNA004_multi_fused \
  --chemistry RNA004 --task multi --mods all --data-type biological --strategy both \
  --sequence-column sequence81 --also-thirdgen --also-secondgen \
  --model-root models --model-checksums models/SHA256.json --trust-model-files
```

真实表若使用 `reference_sequence_81bp` 等列名，应同步替换 `--sequence-column`。每次使用新的输出目录。`--trust-model-files` 是原 CLI 加载可信序列化权重的显式选项。

## 6. 条件、筛选、参数和计算

默认 `--data-type biological` 走融合；纯三代生物样本用 `--mode thirdgen`。`--data-type` 与 `--mode` 互斥。默认 `--strategy s2`，Binary `--feature-set full`；包内没有 mean/std/length 消融权重。

工具先检查输入、中心/方向、所选类别与家族、概率有效性及选择质量，再按阈值标记并按输出策略写结果。`--min-confidence`、`--min-selected-mass`、`--min-second-mass` 默认都为 0。confidence 是胜出类别的条件概率；selected mass 是所选集合占对应参考家族的质量，均不等于位点修饰率。

Binary 默认融合：`sigmoid(logit(p2) + logit(p3))`，例如 0.8 和 0.6 得 0.857142857。Multi 先按参考碱基家族和所选类别条件化，在两代共有类别中做归一化概率幂乘积，再乘回这部分三代质量；三代独有类别保留三代质量。Multi 三代内部使用 manifest 冻结的几何聚合权重和温度，不能擅自等权平均。

**全部 41 个 predict/validate 长选项、默认值、条件、拒绝原因、精确公式和数值例子**集中在[完整中文手册](docs/guide_20261104/README_版本输入预处理参数与计算.md)第 7–8 节；[CLI_help.txt](docs/guide_20261104/CLI_help.txt)为实际解析器输出。工具没有自动 motif/coverage/MAPQ/位点阈值过滤、S1/S2 共识或 site 聚合选项；这些应在合适的上游或下游显式执行。

## 7. 目录与核验

- `src/`、`pyproject.toml`：统一 CLI 和安装入口；保留模型兼容模块。
- `models/`：运行时从 GitHub Release 下载对应分包后组装；无需上传到 Git 源码仓库。
- `docs/guide_20261104/`：详细中文手册、pipeline、10 个代表性人工输入数据、类别/聚合参数。
- `upstream_reference/`：16 个上游文件的原样副本、原路径/新路径/哈希对应关系。
- 论文分析归档不是运行依赖，不放在源码仓库或模型 Release 包中。
- `verification/`：保留包完整性核验脚本和核验范围说明；自动生成的测试日志与核验报告未放入精简上传包。
- `MANIFEST.json`、`SHA256SUMS.txt`、`DISTRIBUTION.json`：当前分发包的逐文件清单、哈希和分发类型；[上传方法](docs/UPLOAD.zh-CN.md)。

推理代码版本未升级。人工输入的成功运行只能证明软件路径和输出契约，不代表新样本准确率、预处理等价性或论文结果复现。源码文件可用 `python verification/verify_distribution.py` 按包内清单检查。

原始 [LICENSE](LICENSE) 仍为 **LICENSE PENDING RIGHTS-HOLDER CONFIRMATION**。本次本地打包没有重新授予任何许可证；公开分发前需要权利人明确源码和模型的发布许可。新 ZIP 已在本地准备，尚未上传。
