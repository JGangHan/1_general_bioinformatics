### 示例数据如下
| 项目             | 内容                                              |
| -------------- | ----------------------------------------------- |
| GEO 样本编号   | GSM3633643                                      |
| 样本名称       | D85N1                                           |
| 物种         | Ovis aries（绵羊）                                  |
| 组织来源       | 骨骼肌（longissimus dorsi）                          |
| 样本处理       | 使用 Trizol 提取 RNA，构建 Illumina TruSeq 小RNA文库      |
| 测序平台       | Illumina HiSeq 2500                             |
| **测序类型**       | RNA-Seq（SINGLE-end 单端测序）                        |
| 数据归属研究     | PRJNA524671 / SRP187052（BioProject 和 SRA Study） |
| 对应样本       | SAMN11029864（BioSample 编号）                      |
| **SRA Run 编号** | **SRR8644988 和 SRR8644989**               |

### 1. 下载方式：wget 或 prefetch
- **prefetch 首选**：下载 .sra 文件是完整格式，包含：原始 reads，质量分数（Phred+33）等关键信息
- wget：下载SRA 的精简版 .sralite 文件，通常用于网页预览、快速展示（不用于正式分析），可能缺少质量值（如你前面看到的全是 ?），在转换成 fastq 时会出现 warning，如 READLEN < 1
- 内容比较
```
# 1. 两种下载方式
prefetch SRR8644988
wget https://sra-downloadb.be-md.ncbi.nlm.nih.gov/sos5/sra-pub-zq-16/SRR008/644/SRR8644988.sralite.1

# 2. 存在文件大小差异
SRR8644988.sra        # 249MB
SRR8644988.sralite.1  # 192MB

# 3. 转换的 fastq 文件存在差异
# 两种解压方式对 .sra 文件解压，解压后文件大小一致
fasterq-dump --split-files --threads 16 SRR8644988.sra      # 2.68 GB 
fastq-dump --split-files SRR8644988.sra                     # 2.68 GB
# 对 .sralite.1 文件解压，解压后文件大小不同
fasterq-dump --split-files --threads 16 SRR8644988.sralite.1  # 2.97GB
fastq-dump --split-files SRR8644988.sralite.1                 # 2.68GB

# 4. prefetch 常用参数
prefetch --progress --transport ascp --output-directory ./datasets SRR8644988
# 文件会输出到 ./datasets/SRR8644988 文件夹下
# --transport ascp 利用 ascp 进行下载，理论上速度会快，但NCBI目前关闭了 ascp 网络端口
```


### 2. 单样本下载、属性和转换
**SRR5852272 为例**
```
# 1. 下载
prefetch --progress --transport ascp --output-directory ./datasets/sra_cache SRR5852272
# 生成 ./datasets/sra_cache/SRR5852272/SRR5852272.sra 文件

# 2. 属性
# vdb-dump 工具
# SRR5852272.sra 是高度压缩文件，无法通过文本编辑工具进行查看

vdb-dump -R1 -C READ_LEN,READ_TYPE,SPOT_LEN SRR5852272.sra

# 双端测序，PE150模式
# READ_LEN: 150, 150
# READ_TYPE: SRA_READ_TYPE_BIOLOGICAL, SRA_READ_TYPE_BIOLOGICAL
# SPOT_LEN: 300

# 3. 解压/转为 fastq 格式
# fasterq-dump（首选）：多线程，快速，稳定，但不可输出为压缩文件
# fastq-dump：单线程，速度慢，但可输出为压缩文件

fasterq-dump --split-files -e 32 -O ./fastq SRR5852272.sra

# --split-files 自动识别单端或双端测序
# -e 线程
# -O 输出路径

```


### 3. 大批量样本下载和转换
**一般情况：单样本对应单个SRR ID**
**实现批量下载、判断单双端、转为fastq格式、重命名为样品名、压缩为 .gz 格式，并检查各步骤完成情况，若失败SRR ID单独保存，并转移至下一个SRR ID**
- 准备文件一：sample_list.txt
```
# 第一列 SRR ID，第二列 Sample ID
SRR23955350	Hu_1
SRR23955317	Hu_2
SRR23955313	Hu_3
................
SRR23955349	Hu_4
SRR23955346	Hu_5
SRR23955343	Hu_6
```
- 准备文件二：batch_download.sh
```
#!/bin/bash

# 设置线程数
THREADS=32

# 输入 SRR ID 与 sample ID 的映射文件
INPUT_FILE="sample_list.txt"

# 日志文件
LOG_FILE="download_process_$(date +%Y%m%d_%H%M%S).log"
FAILED_FILE="failed_ids.txt"

# 输出目录
SRA_CACHE_DIR="/data/hanjiangang/Sheep_Muscle_Saline/datasets/sra_cache"
FASTQ_DIR="/data/hanjiangang/Sheep_Muscle_Saline/datasets/fastq"
mkdir -p "$SRA_CACHE_DIR"
mkdir -p "$FASTQ_DIR"

# 日志输出函数
log_msg() {
    echo "$(date +'%Y-%m-%d %H:%M:%S') $1" | tee -a "$LOG_FILE"
}

# 逐行读取 SRR ID 和 SAMPLE ID
while IFS=$'\t' read -r SRR_ID SAMPLE_ID
do
    [[ -z "$SRR_ID" ]] && continue  # 跳过空行

    log_msg "开始处理 $SRR_ID -> $SAMPLE_ID"

    # 下载 .sra 文件到指定目录
    if prefetch --transport ascp --progress --output-directory "$SRA_CACHE_DIR" "$SRR_ID"; then
        log_msg "$SRR_ID 下载成功"
    else
        log_msg "$SRR_ID 下载失败，跳过"
        echo "$SRR_ID" >> "$FAILED_FILE"
        continue
    fi

    # 转换为 fastq 格式，输出至临时中间目录（和 SRR_ID 文件夹同级）
	# 不需要精确指向 .sra 文件，使用 SRR_ID 即可
    if fasterq-dump "$SRA_CACHE_DIR/$SRR_ID" -e "$THREADS" -O "$SRA_CACHE_DIR/$SRR_ID"; then
        log_msg "$SRR_ID 成功转换为 FASTQ 格式"
    else
        log_msg "$SRR_ID 转换失败，跳过"
        echo "$SRR_ID" >> "$FAILED_FILE"
        continue
    fi

    # 判断单双端
    FQ1="$SRA_CACHE_DIR/$SRR_ID/${SRR_ID}_1.fastq"
    FQ2="$SRA_CACHE_DIR/$SRR_ID/${SRR_ID}_2.fastq"
    FQ_SINGLE="$SRA_CACHE_DIR/$SRR_ID/${SRR_ID}.fastq"

    if [[ -f "$FQ2" ]]; then
        log_msg "$SRR_ID 为双端测序"

        # 重命名并压缩
        mv "$FQ1" "$FASTQ_DIR/${SAMPLE_ID}_1.fastq"
        mv "$FQ2" "$FASTQ_DIR/${SAMPLE_ID}_2.fastq"
        gzip "$FASTQ_DIR/${SAMPLE_ID}_1.fastq"
        gzip "$FASTQ_DIR/${SAMPLE_ID}_2.fastq"
        log_msg "$SRR_ID 重命名并压缩为 ${SAMPLE_ID}_1.fastq.gz 和 ${SAMPLE_ID}_2.fastq.gz"

    elif [[ -f "$FQ_SINGLE" ]]; then
        log_msg "$SRR_ID 为单端测序"

        mv "$FQ_SINGLE" "$FASTQ_DIR/${SAMPLE_ID}.fastq"
        gzip "$FASTQ_DIR/${SAMPLE_ID}.fastq"
        log_msg "$SRR_ID 重命名并压缩为 ${SAMPLE_ID}.fastq.gz"

    else
        log_msg "$SRR_ID 未检测到 fastq 文件，跳过"
        echo "$SRR_ID" >> "$FAILED_FILE"
        continue
    fi

    log_msg "$SRR_ID 全流程完成"

done < "$INPUT_FILE"
```
- 输出文件一：日志文件 .log
```
# 若成功
2025-07-31 16:00:01 开始处理 SRR23955350 -> Hu_1
2025-07-31 16:05:23 SRR23955350 下载成功
2025-07-31 16:12:40 SRR23955350 成功转换为 FASTQ 格式
2025-07-31 16:12:41 SRR23955350 为双端测序
2025-07-31 16:12:45 SRR23955350 重命名并压缩为 Hu_1_1.fastq.gz 和 Hu_1_2.fastq.gz
2025-07-31 16:12:45 SRR23955350 全流程完成

# 若失败
2025-07-31 16:25:14 开始处理 SRR23955313 -> Hu_3
2025-07-31 16:28:55 SRR23955313 下载失败，跳过
```
- 输出文件二：FASTQ_DIR="/data/hanjiangang/Sheep_Muscle_Saline/datasets/fastq"路径下的 rename.fastq.gz 文件
- 输出文件三：failed_ids.txt，包含失败的SRR ID




### 4. 特殊情况：单样本对应多个SRR ID
**Y2_6 样本对应	ERR2074457 ERR2074458 ERR2074459 ERR2074460 ERR2074461 ERR2074462 等多个 SRR ID**  
原因：一个样本多次上机，或在多个泳道进行测序，从而产生多个测序文件
处理流程：下载每个 SRR ID，转换为 fastq，合并 fastq，重命名并压缩

```






```








### 5. 更加特殊情况：单样本对应多个SRR ID，但


E135_1	SRR8645004	SRR8645005
E135_2	SRR8645002	SRR8645003
E135_3	SRR8645000	SRR8645001

**后续分析怎么做？**
对于下游分析（如差异表达、聚类等），应将两个 SRR 的数据合并使用，因为它们共同代表一个样本。
```
cat SRR8644988.fastq SRR8644989.fastq > D85N1.fastq
```























