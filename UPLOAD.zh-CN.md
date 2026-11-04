# GitHub 上传与模型安装

## 源码仓库网页上传

精简源码归档内部共98个文件。若仓库网页还限制一次最多100个上传项，在空仓库页面选择 **Add file → Upload files**，提交 `README.md` 和 `NanoRemix_v0.3.0_GitHub_source_compact_20261104.zip` 两项即可。ZIP会作为一个可下载归档保存，GitHub不会自动展开其中的项目文件。

源码包不含模型、研究数据、结果或自动生成的核验日志。需要在仓库页面逐个浏览源文件时，应使用GitHub Desktop/Git推送解压后的项目文件；网页上传器不会替ZIP解压。

## 创建Release并上传五个权重包

进入 **Releases → Draft a new release**，创建标签`v0.3.0`，把下面五个ZIP逐个添加为Release附件。不要将其解压后上传到源码仓库。

| Release附件 | 内容 | 用途 |
|---|---|---|
| `NanoRemix_v0.3.0_models_RNA002_Binary_20261104.zip` | 18个RNA002 Binary `.pkl` | RNA002 Binary S1/S2 |
| `NanoRemix_v0.3.0_models_RNA002_Multi_20261104.zip` | 6个RNA002 Multi `.pkl`及S1/S2 manifests | RNA002 Multi S1/S2 |
| `NanoRemix_v0.3.0_models_RNA004_Binary_20261104.zip` | 30个RNA004 Binary `.pkl` | RNA004 Binary S1/S2 |
| `NanoRemix_v0.3.0_models_RNA004_Multi_20261104.zip` | 10个RNA004 Multi `.pkl`及S1/S2 manifests | RNA004 Multi S1/S2 |
| `NanoRemix_v0.3.0_models_SecondGen_20261104.zip` | 25个共享`.pt` checkpoint和网络定义 | RNA002/RNA004二代序列推理 |

纯三代预测下载相应chemistry/task包。融合且现场计算二代时，额外下载SecondGen；如果已提供按身份键匹配的完整二代概率表，则不需要SecondGen权重。Binary消融权重未分发。

把本地`NanoRemix_v0.3.0_release_assets_SHA256SUMS_20261104.txt`的内容复制到Release说明，供下载者检查附件。各ZIP解压到项目根目录后会合并到`models/`。

## 安装和检查

使用Python 3.10/3.11，在解压的NanoRemix项目根目录运行：

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -e '.[models,second]' -c constraints-models.txt
python verification/verify_distribution.py
nanoremix doctor --model-root models --model-checksums models/SHA256.json --strict
```

`verify_distribution.py`检查源码文件哈希，不读取模型。`doctor`检查所选模型资产和运行依赖；融合路线需下载对应三代模型以及SecondGen，或选择预计算二代概率输入。

完整的输入类型、上游软件版本、RNA002/RNA004预处理、参数、筛选和融合公式见[中文入口](../START_HERE.zh-CN.md)及[完整技术手册](guide_20261104/README_版本输入预处理参数与计算.md)。RNA004旧Step6转换器对当前RNA方向负链有已知限制，不能将其宣传为适用于任意样本的通用预处理pipeline。

公开分发前需由权利人确认源码、模型和上游材料的许可；本包不会自行授予许可证。
