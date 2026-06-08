# AmpliconSuite-pipeline 中文新手指南

![GitHub](https://img.shields.io/github/license/jluebeck/AmpliconSuite-pipeline)

AmpliconSuite-pipeline 是一个面向肿瘤基因组扩增分析的端到端命令行流程。它把 [AmpliconArchitect](https://github.com/jluebeck/AmpliconArchitect) 以及配套的数据准备、种子区域筛选和结果解释工具串联起来，帮助用户从测序数据中寻找并解析局灶性扩增结构，例如 ecDNA（extrachromosomal DNA，染色体外 DNA）和 BFB（breakage-fusion-bridge，断裂-融合-桥）等模式。

这个项目以前叫 **PrepareAA**。如果你在旧教程、脚本或论文中看到 PrepareAA，通常指的就是现在的 AmpliconSuite-pipeline。

**当前版本：0.1546.0**

> 版本号规则：`主版本号.距离首次提交的天数.次版本号`。首次提交日期为 2019 年 3 月 5 日。

如果你已经熟悉英文文档，也推荐阅读项目的 [详细英文指南](https://github.com/jluebeck/PrepareAA/blob/master/GUIDE.md)，其中包含最佳实践和常见问题。

---

## 目录

- [这个工具能做什么](#这个工具能做什么)
- [适合谁使用](#适合谁使用)
- [支持的参考基因组](#支持的参考基因组)
- [开始前需要准备什么](#开始前需要准备什么)
- [安装方式](#安装方式)
  - [方式 A：无需本地安装](#方式-a无需本地安装)
  - [方式 B：使用 Conda 安装（即将支持）](#方式-b使用-conda-安装即将支持)
  - [方式 C：使用安装脚本进行本地安装](#方式-c使用安装脚本进行本地安装)
  - [方式 D：使用 Singularity 或 Docker 容器](#方式-d使用-singularity-或-docker-容器)
- [第一次运行：推荐流程](#第一次运行推荐流程)
- [常见运行示例](#常见运行示例)
- [命令行参数说明](#命令行参数说明)
- [如何理解输出结果](#如何理解输出结果)
- [常见问题和排错](#常见问题和排错)
- [打包输出用于 AmpliconRepository](#打包输出用于-ampliconrepository)
- [引用方式](#引用方式)
- [附加分析脚本](#附加分析脚本)

---

## 这个工具能做什么

AmpliconSuite-pipeline 主要负责把运行 AmpliconArchitect 前后需要的步骤自动化，包括：

1. **数据准备**
   - 从 FASTQ 开始时，可以进行比对。
   - 从 BAM 开始时，可以检查和准备已比对数据。
2. **拷贝数分析和种子区域识别**
   - 可以调用 CNVkit 进行 CNV 分析。
   - 也可以使用你自己准备好的 CNV BED 或 CNVkit `.cns` 文件。
3. **筛选适合 AmpliconArchitect 分析的扩增区域**
   - 根据拷贝数阈值、区域长度和参考基因组的过滤区域筛选 seed intervals。
4. **运行 AmpliconArchitect（可选）**
   - 构建扩增区域的断点图和结构模型。
5. **运行 AmpliconClassifier（可选）**
   - 对 AA 结果进行分类，例如识别 ecDNA、BFB 等结构类别。
6. **从中间步骤继续运行**
   - 你不一定要从 FASTQ 开始；也可以从排序后的 BAM、CNV 结果，甚至已完成的 AA 输出开始。

---

## 适合谁使用

本 README 面向以下用户：

- 第一次接触 AmpliconSuite、AmpliconArchitect 或 ecDNA 分析的新手。
- 有肿瘤 WGS/WES/靶向测序数据，希望分析局灶性扩增结构的研究者。
- 希望在本地服务器、集群、Docker 或 Singularity 中运行流程的用户。

如果你只想少量测试非敏感样本，可以优先考虑 GenePattern Web 界面。如果你需要处理大量样本、受保护健康信息（PHI）或想使用高级参数，建议本地安装或容器运行。

---

## 支持的参考基因组

AmpliconSuite-pipeline 支持以下参考基因组名称：

| 名称 | 说明 |
| --- | --- |
| `hg19` | 人类 hg19 |
| `GRCh37` | 人类 GRCh37 |
| `GRCh38` / `hg38` | 人类 GRCh38/hg38 |
| `GRCh38_viral` | 项目提供的人类-病毒混合参考基因组，可用于检测肿瘤病毒相关的混合扩增和 ecDNA |
| `mm10` / `GRCm38` | 小鼠 mm10/GRCm38 |

使用 `GRCh38_viral` 时请注意：

- 如果从 FASTQ 开始，需要指定 `--ref GRCh38_viral`。
- 如果从 BAM 开始，BAM 必须已经比对到 `$AA_DATA_REPO/GRCh38_viral` 对应的参考基因组。

---

## 开始前需要准备什么

### 1. 操作系统和基础环境

推荐使用现代 Unix/Linux 环境，例如：

- Ubuntu 18.04 或更新版本
- CentOS 7 或更新版本
- macOS
- HPC/服务器上的 Linux 环境

本地安装至少需要：

- `python3`
- `git`
- `wget` 或类似下载工具
- 足够的磁盘空间保存参考基因组和分析输出
- 多线程 CPU；建议每个样本使用 12 个或更多线程

### 2. 输入数据

至少需要以下输入之一：

- 一对双端 FASTQ：`sample_R1.fq.gz sample_R2.fq.gz`
- 一个坐标排序后的 BAM：`sample.cs.bam`
- 已完成的 AmpliconArchitect 输出目录

如果从 BAM 开始，建议确认：

- BAM 是 coordinate-sorted（按坐标排序）。
- BAM 有索引文件，例如 `sample.cs.bam.bai`。
- BAM 使用的参考基因组与你后续指定的 `--ref` 一致。

### 3. Mosek 许可证

AmpliconArchitect 依赖 Mosek 优化工具。AA 没有 Mosek 许可证将无法正常工作。

- 学术用户通常可以申请免费许可证。
- 许可证文件需要放在 `$HOME/mosek/` 目录下。

---

## 安装方式

### 方式 A：无需本地安装

这是最方便的方式，适合小规模、非敏感数据测试。但它不适合大量样本、受保护健康信息（PHI）或需要复杂命令行参数的场景。

#### GenePattern Web 界面

1. 打开 [GenePattern](https://genepattern.ucsd.edu/gp)。
2. 注册或登录账号。
3. 在模块搜索框中搜索 `AmpliconSuite`。
4. 上传输入文件并按照网页参数说明运行。

该模块由 GenePattern 团队成员 Edwin Huang、Ted Liefeld、Michael Reich 与本项目协作构建。

#### Nextflow

也可以通过 Nextflow 使用 [nf-core/circdna](https://nf-co.re/circdna) 流程运行 AmpliconSuite-pipeline。该流程由 [Daniel Schreyer](https://github.com/DSchreyer) 构建，适合已经熟悉 Nextflow/nf-core 的用户。

---

### 方式 B：使用 Conda 安装（即将支持）

```bash
conda create -n ampsuite
conda activate ampsuite
conda install -c bioconda -c mosek ampliconsuite
wget https://raw.githubusercontent.com/AmpliconSuite/AmpliconSuite-pipeline/bioconda/install.sh
bash install.sh --finalize
```

执行 `install.sh --finalize` 时，脚本会确认数据仓库路径和 Mosek 许可证目录。

完成后，请继续阅读 [方式 C 的第 2 步](#第-2-步下载-aa-参考数据) 来下载参考数据。

---

### 方式 C：使用安装脚本进行本地安装

这种方式适合希望直接在服务器或工作站上运行的用户。

#### 第 1 步：下载源码并运行安装脚本

```bash
git clone https://github.com/AmpliconSuite/AmpliconSuite-pipeline
cd AmpliconSuite-pipeline

# 可选：先查看安装帮助
./install.sh -h

# 安装 AmpliconArchitect、AmpliconClassifier 和依赖项
./install.sh
```

默认情况下，安装脚本会把 AA 数据仓库放在你的 `$HOME` 目录下。安装完成后，通常会设置或提示你设置 `AA_DATA_REPO` 等环境变量。

#### 第 2 步：下载 AA 参考数据

如果你通过 Conda 安装，也从这里继续。

1. 打开可用注释数据列表：
   <https://datasets.genepattern.org/?prefix=data/module_support_files/AmpliconArchitect/>
2. 复制你需要的参考基因组压缩包 URL。
3. 进入 `$AA_DATA_REPO` 并下载、解压。

```bash
cd $AA_DATA_REPO
wget [reference_build_url]
tar -xzf [reference_build].tar.gz
rm [reference_build].tar.gz
```

文件名中带 `_indexed` 的版本包含 BWA index，只有当你要从 FASTQ 开始运行时才需要。

#### 第 3 步：准备 Mosek 许可证

获取 Mosek 许可证后，把许可证放到：

```bash
$HOME/mosek/
```

如果 AA 报错提示找不到 Mosek license，优先检查这个目录和许可证文件权限。

---

### 方式 D：使用 Singularity 或 Docker 容器

容器方式可以减少依赖冲突，适合服务器、集群和可复现分析。

#### 第 1 步：获取镜像

**Singularity**

- 安装说明：<https://docs.sylabs.io/guides/3.0/user-guide/installation.html>
- 需要 Singularity 3.6 或更高版本。
- 拉取镜像：

```bash
singularity pull library://jluebeck/ampliconsuite-pipeline/ampliconsuite-pipeline
```

**Docker**

- 安装说明：<https://docs.docker.com/install/>
- 拉取镜像：

```bash
docker pull jluebeck/prepareaa
```

可选：把当前用户加入 docker 组，避免每次都使用 `sudo`：

```bash
sudo usermod -a -G docker $USER
```

执行后需要退出并重新登录。

#### 第 2 步：下载执行脚本并配置数据仓库

```bash
git clone https://github.com/AmpliconSuite/AmpliconSuite-pipeline
cd AmpliconSuite-pipeline

# 可选：查看帮助
./install.sh -h

# 配置数据仓库路径和 Mosek 许可证目录
./install.sh --finalize
```

#### 第 3 步：准备 Mosek 许可证

容器运行同样需要 Mosek 许可证。请确认许可证目录能够被容器挂载并在运行时访问。

---

## 第一次运行：推荐流程

如果你是新手，推荐按下面顺序做一次小样本测试：

1. **确认参考基因组**
   - 例如样本比对到 GRCh38，就使用 `--ref GRCh38`。
2. **确认输入类型**
   - 已有 BAM：使用 `--bam sample.cs.bam`。
   - 只有 FASTQ：使用 `--fastqs sample_R1.fq.gz sample_R2.fq.gz`，并确保参考数据包含 BWA index。
3. **先运行完整流程但保守设置输出目录**
   - 使用单独目录保存结果，避免和其他样本混在一起。
4. **同时运行 AA 和 AC**
   - 加上 `--run_AA --run_AC`，这样可以得到结构重建和分类结果。
5. **检查日志和输出**
   - 如果失败，先看报错是否与参考基因组、BAM 索引、Mosek license、CNVkit/R 版本有关。

---

## 常见运行示例

下面示例中的方括号 `[]` 表示可选参数；实际运行时不要输入方括号。

### 示例 1：从排序后的 BAM 开始，自动调用 CNVkit

```bash
AmpliconSuite-pipeline.py \
  -s sample_name \
  -t 12 \
  --bam sample.cs.bam \
  --ref GRCh38 \
  --run_AA \
  --run_AC
```

适用场景：你已经有一个比对到 GRCh38 的 coordinate-sorted BAM，希望流程自动完成 CNV 分析、seed 筛选、AA 和 AC。

### 示例 2：从一对 FASTQ 开始

```bash
AmpliconSuite-pipeline.py \
  -s sample_name \
  -t 16 \
  --fastqs sample_R1.fq.gz sample_R2.fq.gz \
  --ref GRCh38 \
  --run_AA \
  --run_AC
```

适用场景：你只有原始双端 FASTQ。请确保 `GRCh38_indexed` 或对应带 BWA index 的参考数据已经下载。

### 示例 3：使用自己准备的 CNV 结果

```bash
AmpliconSuite-pipeline.py \
  -s sample_name \
  -t 12 \
  --bam sample.cs.bam \
  --cnv_bed your_cnvs.bed \
  --ref GRCh38 \
  --run_AA \
  --run_AC
```

`--cnv_bed` 可以是普通 BED，也可以是 CNVkit 的 `sample_name.cns` 文件。

普通 BED 至少需要以下信息：

```text
chr    start    end    copy_number
```

也允许 `end` 和 `copy_number` 之间存在额外列，但 **copy_number 必须是最后一列**。

### 示例 4：多个相关样本联合分析

如果多个样本来自同一患者、同一细胞系或同一来源，建议使用分组分析流程统一 seed intervals，提高样本之间的可比性。请见下方 [`GroupedAnalysisAmpSuite.py`](#-grouped-analysis-of-related-samples-groupedanalysisampsuitepy) 的说明。

### 示例 5：分析肿瘤病毒相关样本

```bash
AmpliconSuite-pipeline.py \
  -s sample_name \
  -t 16 \
  --fastqs sample_R1.fq.gz sample_R2.fq.gz \
  --ref GRCh38_viral \
  --cnsize_min 10000 \
  --run_AA \
  --run_AC
```

如果从 BAM 开始，该 BAM 必须已经比对到 `$AA_DATA_REPO/GRCh38_viral` 对应参考。

### 示例 6：从已完成的 AA 结果开始，只运行 AmpliconClassifier

```bash
AmpliconSuite-pipeline.py \
  -s project_name \
  --completed_AA_runs /path/to/location_of_all_AA_results/ \
  --completed_run_metadata run_metadata_file.json \
  -t 1 \
  --ref GRCh38
```

这种模式要求所有 AA 结果都使用同一个参考基因组版本生成。

---

## 命令行参数说明

### 必需或常用参数

| 参数 | 是否必需 | 说明 |
| --- | --- | --- |
| `-o`, `--output_directory {outdir}` | 可选 | 输出目录。默认是当前目录。建议每个项目或样本使用独立输出目录。 |
| `-s`, `--sample_name {sname}` | 必需 | 样本名，会用于输出文件命名。建议只使用字母、数字、下划线和短横线。 |
| `-t`, `--nthreads {int}` | 必需 | BWA 和 CNVkit 使用的线程数。推荐 12 个或更多线程。 |
| `--bam`, `--sorted_bam {sample.cs.bam}` | 三选一 | 坐标排序后的 BAM。 |
| `--fastqs {R1.fq.gz R2.fq.gz}` | 三选一 | 双端 FASTQ 文件。 |
| `--completed_AA_runs {/path/to/AA_outputs}` | 三选一 | 已完成的 AA 输出目录，可包含一个或多个样本结果。 |

### CNV 和输入相关参数

| 参数 | 说明 |
| --- | --- |
| `--cnv_bed {cnvfile.bed}` | 使用你自己的 CNV calls。可以是最后一列为 copy number 的 BED，也可以是 CNVkit `.cns` 文件。未提供时流程会调用 CNVkit。 |
| `--cnvkit_dir {/path/to/cnvkit.py}` | 如果 CNVkit 是从源码安装，且没有提供 `--cnv_bed`，需要指定包含 `cnvkit.py` 的目录。 |
| `--normal_bam {matched_normal.bam}` | 给 CNVkit 使用的匹配正常样本 BAM。AA 本身不会使用该文件。 |
| `--purity {0 到 1 之间的浮点数}` | 给 CNVkit 使用的肿瘤纯度估计。AA 本身不使用。低纯度样本校正后可能产生很多高拷贝 seed，必要时提高 `--cngain`。 |
| `--ploidy {float}` | 给 CNVkit 使用的倍性估计。AA 本身不使用。 |
| `--cnvkit_segmentation {str}` | CNVkit 分段方法，默认 `cbs`。可选：`cbs`、`haar`、`hmm`、`hmm-tumor`、`hmm-germline`、`none`。 |

### 是否运行 AA/AC

| 参数 | 说明 |
| --- | --- |
| `--run_AA` | 在准备步骤结束后运行 AmpliconArchitect。 |
| `--run_AC` | 在 AA 之后运行 AmpliconClassifier。只有设置 `--run_AA` 时才有实际作用。 |

### 参考基因组和阈值参数

| 参数 | 说明 |
| --- | --- |
| `--ref {ref name}` | 参考基因组名称，可选 `hg19`、`GRCh37`、`GRCh38`、`GRCh38_viral`、`mm10`、`GRCm38`。如果不设置，流程会尝试自动检测。 |
| `--cngain {float}` | AA 考虑的拷贝数增益阈值，默认 `4.5`。 |
| `--cnsize_min {int}` | AA 考虑的 CN 区间最小长度，默认 `50000`。病毒相关分析中有时会设置更小，例如 `10000`。 |
| `--downsample {float}` | AA 内部 BAM 覆盖度下采样阈值，默认 `10`。不影响 AA 外其他分析的覆盖度。 |
| `--no_filter` | 不调用 `amplified_intervals.py` 对扩增 seed 区域进行过滤。 |
| `--no_QC` | 跳过 BAM 的 QC。 |

### 解释器、外部程序和 AA 高级参数

| 参数 | 说明 |
| --- | --- |
| `--rscript_path {/path/to/Rscript}` | 当使用 CNVkit 且系统 Rscript 版本低于 3.5 时，指定较新 Rscript 的路径。 |
| `--python3_path {/path/to/python3}` | 使用 CNVkit 时，如需指定自定义 Python 3 路径，可设置此参数。 |
| `--aa_python_interpreter {/path/to/python}` | 指定运行 AA 的 Python 解释器。默认使用系统 `python`。 |
| `--use_old_samtools` | 如果 SAMtools 版本低于 1.0，设置此标志。 |
| `--samtools_path {/path/to/samtools}` | 指定特定 samtools 二进制文件路径；默认使用系统 PATH 中的 samtools。 |
| `--AA_runmode {FULL, BPGRAPH, CYCLES, SVVIEW}` | AA 运行模式，默认 `FULL`。详情见 AA 文档。 |
| `--AA_extendmode {EXPLORE, CLUSTERED, UNCLUSTERED, VIRAL}` | AA 扩展模式，默认 `EXPLORE`。详情见 AA 文档。 |
| `--AA_insert_sdevs {float}` | 默认 `3.0`。如果文库插入片段大小控制较差、properly-paired reads 比例较低，可考虑提高到 `8` 或 `9`。 |

### 元数据参数

| 参数 | 说明 |
| --- | --- |
| `--sample_metadata {sample_metadata.json}` | 样本元数据 JSON。建议从 `sample_metadata_skeleton.json` 模板复制后填写。 |
| `--completed_run_metadata {run_metadata.json}` | 仅在从 `--completed_AA_runs` 开始时需要。指定之前 AA 结果的 run metadata。若没有，可设置为 `None`。 |

---

## 如何理解输出结果

AmpliconSuite-pipeline 可能生成多类输出，具体取决于你是否运行 AA 和 AC：

- **准备阶段输出**：包括 BAM 检查、CNV 结果、seed intervals 等。
- **AmpliconArchitect 输出**：包括图文件、cycles 文件和可视化结果。
- **AmpliconClassifier 输出**：包括扩增结构分类表，例如 ecDNA、BFB 或其他类别。

更多解释请查看：

- AmpliconClassifier 输出说明：<https://github.com/AmpliconSuite/AmpliconClassifier#3-output>
- AA cycles 文件解释：<https://github.com/jluebeck/AmpliconArchitect#interpreting-the-aa-cycles-files>

---

## 常见问题和排错

### 1. 不确定应该从 FASTQ 还是 BAM 开始？

- 如果你已经有质量可靠、坐标排序并带索引的 BAM，建议从 BAM 开始。
- 如果你希望流程统一完成比对，或者使用 `GRCh38_viral` 重新比对，则从 FASTQ 开始。

### 2. `--ref` 应该怎么选？

`--ref` 必须和输入 BAM 使用的参考基因组一致。常见错误是 BAM 比对到 hg19，但运行时指定 GRCh38，这会导致坐标和注释不匹配。

### 3. 没有 CNV 文件怎么办？

可以不提供 `--cnv_bed`，流程会调用 CNVkit。不过你需要确保 CNVkit 及其依赖可用。如果你已经在外部运行了 CNVkit，推荐提供 `.cns` 文件。

### 4. CNVkit 对 R 有要求吗？

CNVkit 需要 R 3.5 或更高版本。老 Linux 系统可能默认 R 版本过低。此时可以用：

```bash
--rscript_path /path/to/Rscript
```

指定较新的 Rscript。

### 5. 低纯度样本有什么注意事项？

如果设置了较低的 `--purity`，CNVkit 校正后可能产生很多高拷贝数 seed 区域。可以考虑提高拷贝数阈值，例如：

```bash
--cngain 8
```

### 6. AA 报 Mosek 相关错误怎么办？

请检查：

- Mosek 许可证是否已获取。
- 许可证是否放在 `$HOME/mosek/`。
- 容器运行时是否正确挂载了许可证目录。

### 7. 想看更多最佳实践怎么办？

请阅读英文 [GUIDE.md](https://github.com/jluebeck/PrepareAA/blob/master/GUIDE.md)。

---

## 打包输出用于 AmpliconRepository

AmpliconRepository 是用于保存和分享 AmpliconSuite-pipeline 输出的在线平台。相关功能将逐步发布。

如需打包一组 AA 输出，可按以下步骤准备：

1. **推荐：准备样本元数据**
   - 复制 `sample_metadata_skeleton.json`。
   - 为每个样本填写一个元数据 JSON。
   - 运行流程时使用：

   ```bash
   --sample_metadata sample_metadata.json
   ```

2. **在 AmpliconClassifier 输出目录中生成结果表**

   ```bash
   cd [directory_of_classification_files]
   $AC_SRC/make_results_table.py \
     -i samples.input \
     --summary_map samples_summary_map.txt \
     --classification_file samples_amplicon_classification_profiles.tsv \
     --ref [hg19/hg38/...]
   ```

   `samples.input` 和 `samples_summary_map.txt` 通常由 `make_input.sh` 创建。

3. **压缩 AA 输出**

   ```bash
   tar -czf my_collection.tar.gz /path/to/AA_outputs/
   ```

   使用 `.zip` 也可以。

4. **登录 GenePattern**
   - 打开 <https://genepattern.ucsd.edu/gp>。
   - 登录账号。

5. **运行 AmpliconSuiteAggregator**
   - 在左上角搜索框中搜索 `AmpliconSuiteAggregator`。
   - 上传压缩后的 AA 输出集合。
   - 运行模块并下载聚合后的 `.tar.gz` 结果。

6. **上传到 AmpliconRepository**
   - 平台发布后，按网站说明上传聚合结果。

---

## 引用方式

如果你在论文或报告中使用 AmpliconSuite-pipeline，请引用你实际使用到的模块。项目在 [CITATIONS.md](https://github.com/jluebeck/AmpliconSuite-pipeline/blob/master/CITATIONS.md) 中总结了相关引用信息。

---

## 附加分析脚本

本仓库还提供若干辅助脚本，用于分组分析、图文件转换、cycles 文件处理和伪影清理。

### - Grouped analysis of related samples：`GroupedAnalysisAmpSuite.py`

当多个样本来自共同来源时，例如同一患者的纵向样本、多区域样本或同一细胞系，建议在运行 AA 前统一 seed intervals。这样可以提高不同样本 AA 结果之间的可比性。

`GroupedAnalysisAmpSuite.py` 会自动完成这类分组分析。它接受大多数与 `PrepareAA.py` 类似的参数，但额外需要一个输入列表文件。

输入文件格式：

```text
sample_name    bamfile    "tumor"/"normal"    [CNV_calls]    [sample_metadata_json]
```

说明：

- `CNV_calls` 和 `sample_metadata_json` 是可选列。
- 这两个可选列是位置相关的；如果跳过 `CNV_calls`，请写成 `NA` 或 `None`。
- 默认会运行 AA 和 AC。
- 如果不想运行 AA，可使用 `--no_AA`。

示例命令：

```bash
GroupedAnalysisAmpSuite.py \
  -i inputs.txt \
  -o output_dir \
  -t 12
```

### - Candidate Amplicon Path Enumerator：`CAMPER.py`

`CAMPER.py` 用于在 AA graph 文件中穷举搜索最长路径，包括环状和非环状路径。它可以帮助识别候选 ecDNA 结构。

基本思路：

- 需要提供扩增子的中位拷贝数；如果不提供，脚本会尝试自行估计。
- 脚本会根据中位拷贝数缩放 segment copy number，估计每个片段在扩增子中的 multiplicity。
- 然后搜索能解释这些 multiplicity 的合理最长路径。
- 输出是 AA 格式 cycles 文件，并附带长度和质量控制注释。

质量控制指标包括：

- `RMSR`：copy number residual 的均方根，越低越好。
- `DBI`：Davies-Bouldin index，用于衡量 copy-number 到 multiplicity 聚类的质量。

更多方法细节可参考相关论文方法部分：<https://www.nature.com/articles/s41588-022-01190-0>。

注意：该脚本适用于最多包含一个 ecDNA 的 AA amplicon；不支持多物种/多结构重建。

示例命令：

```bash
AmpliconSuite-pipeline/scripts/plausible_paths.py \
  -g sample_amplicon1_graph.txt \
  --scaling_factor [CN_estimate_value] \
  --remove_short_jumps \
  --keep_all_LC \
  --max_length [value_in_kbp]
```

### - `breakpoints_to_bed.py`

功能：把 AA graph 中的 discordant edges（断点连接）写成 pseudo-BED 文件。

依赖：需要预先安装 `intervaltree` Python 包。

### - `convert_cns_to_bed.py`

很多用户会先在 AmpliconSuite-pipeline 外部运行 CNVkit，然后希望把 CNVkit 结果提供给 AA。推荐使用 CNVkit 的 `.cns` 文件作为 seed 来源。

注意：不推荐使用 `.call.cns` 作为 seed 来源，因为它通常经过更激进的合并。

该脚本可以把 CNVkit `.cns` 文件转换成 AmpliconSuite-pipeline 可用的 BED 文件。

用法：

```bash
scripts/convert_cns_to_bed.py sample.cns
```

输出的 BED 文件可以作为 `--cnv_bed` 输入。

### - `cycles_to_bed.py`

功能：把 AA cycles 文件转换成一组 BED 文件，每个 decomposition 输出一个 BED。

说明：

- segments 会被合并并排序。
- segment 的原始顺序和方向信息会丢失。
- 需要预先安装 `intervaltree` Python 包。

### - `graph_cleaner.py`

功能：清理 AA graph 文件中可能由测序伪影导致的短断点边。

典型伪影包括：

- 很短的 everted orientation edges。
- 在 AA amplicon 图中表现为大量短的棕色 “spikes”。

用法一：处理单个 graph 文件。

```bash
scripts/graph_cleaner.py \
  -g /path/to/sample_ampliconx_graph.txt \
  --max_hop_size 4000
```

用法二：处理 graph 文件列表。

```bash
scripts/graph_cleaner.py \
  --graph_list /path/to/list_of_graphfiles.txt \
  --max_hop_size 4000
```

输出文件名通常类似：

```text
/path/to/my_sample_ampliconX_cleaned_graph.txt
```

### - `graph_to_bed.py`

功能：从 AA graph 文件生成：

- graph segments 的 BED 文件
- discordant graph edges 的 BEDPE 文件

可选功能：

- 使用 `--min_cn` 只输出 copy number 高于阈值的 segments。
- 使用 `--unmerged` 保留未合并的相邻 graph segments，并在最后一列输出 segment copy number。
- 使用 `--add_chr_tag` 添加 `chr` 前缀。

用法：

```bash
scripts/graph_to_bed.py \
  -g sample_amplicon_graph.txt \
  --unmerged \
  --min_cn 0 \
  --add_chr_tag
```

### - `bfb_foldback_detection.py`（已弃用）

**该脚本已弃用且不再维护，仅为兼容旧分析保留。更稳健的 BFB 检测请使用 [AmpliconClassifier](https://github.com/jluebeck/AmpliconClassifier)。**

依赖：需要预先安装 `intervaltree` Python 包。

如果确实需要对旧 AA 输出运行该脚本，请创建一个两列文件：

```text
graph_name    /path/to/graph_file
```

必需参数包括：

| 参数 | 说明 |
| --- | --- |
| `--exclude [path_to_$AA_DATA_REPO/[ref]/[mappability_excludable_file]]` | 排除区域文件。 |
| `-o [output_filename_prefix]` | 输出文件名前缀。 |
| `--ref [hg19, GRCh37, GRCh38]` | 参考基因组。 |
| `--AA_graph_list [two-column_file_listing_AA_graphs]` | 两列 graph 列表文件。 |

---

## 给新手的最后建议

- 先用一个小样本跑通，再扩展到批量样本。
- 每个样本或项目使用独立输出目录。
- 记录完整命令、软件版本、参考基因组版本和参数。
- 如果结果异常，优先检查参考基因组是否匹配、BAM 是否排序和索引、Mosek 许可证是否可用、CNV 阈值是否适合样本纯度。
- 对论文或正式报告，务必保留 AA、AC 原始输出和日志，方便复现。
