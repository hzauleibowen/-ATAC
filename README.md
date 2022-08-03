
# 1 下载数据
用诺禾的软件下载 ./lnd ogin -u 用户名 -p 密码登录
/lnd cp oss:// 目录/文件 本地目录下载，直接下载目录就ok，软件会统计文件完整度.

# 2 数据质控

```
cat config.raw  |while read id;
do echo $id
arr=($id)
fq2=${arr[2]}
fq1=${arr[1]}
sample=${arr[0]}
nohup  trim_galore -q 25 --phred33 --length 35 -e 0.1 --stringency 4 --paired -o  ../clean/  $fq1   $fq2  &   ##注意这里最好加上--length 15 因为
done                                                                                                       ##如果保留了空reads，软件不能识别+号 
```
# cell ranger ATAC-seq
[10Xgenome的教程]{https://support.10xgenomics.com/single-cell-atac/software/pipelines/latest/using/mkfastq}
# Read预处理
这一步包括三小步——Demultiplexing、Adaptor Trimming和比对（Alignment）
1 为了节省成本，我们往往将多个样本（Sample）混合同时测序，并用Index Adaptor Sequence标记出样本。因此，Demultiplexing要做的就是把样本的ID和细胞的ID（Barcode）拷贝到这个细胞下的每个Read中。我们通常用Illumina的bcl2fastq进行Demultiplexing。
2Adapter Trimming的目的就是把测序中添加的Adaptor Sequence和Primer序列切除掉，工具是AdapterRemoval和Trimmomatic。
3Alignment的作用是把测序Read比对到参考基因组上，常用工具是Bowtie2、BWA和STAR。并不是所有Read都能比对到基因组上，比对成功的一对Read称为片段（Fragment）。sci-ATAC-Seq数据预处理后得到的是二进制BAM文件（参考文献），10X Genomics Chromium Single Cell ATAC数据预处理后的得到的是Fragments文本文档。10X Genomics Chromium Single Cell ATAC配套有“御用”预处理工具箱CellRanger，可以完成所有Read的预处理步骤

# QC
这一步最主流的工具是
工具箱下的Signac软件，目标是通过五个标准删减质量低的细胞。
1. Nucleosome Banding Pattern
染色质是缠绕在组蛋白上形成核小体（Nucleosome）的。核小体的体积是固定的，因此转座酶切割染色质的时候是避开核小体区域的，所以核小体附近切割下来的Fragment长度（Insert Size）是200 bp的倍数，也就是200，400和600 bp。Signac能够根据这些模式识别核小体附近的Fragment和染色质开放区域的Fragment。对每一个细胞，Signac计算核小体附近Fragment的数量与开放区域Fragment的数量的比值。这个比值越小越好。
2. 转录起始位点丰度得分（Transcription Start Site Enrichment Score，TSS Enrichment Score）
ENCODE数据库提供了一个得分，计算TSS附近区域Fragment的数量和TSS侧翼区域Fragment的数量的比值，作为TSS丰度得分。通常，测序质量低的ATAC Fragment的TSS丰度得分会很低。因此，TSS丰度得分越高越好。
3/4. Peak中的Fragment数量/比例
这一步在完成Peak注释后才能做。类似地，我们也可以计算Peak中的Read数量/比例。比例越大越好，数量适中即可。如果Fragment或Read的数量太低，表示这个细胞中的Read没有被充分测得，是低质量细胞；如果Fragment或Read的数量太高，表示我们可能把多个细胞核质错当作一个核质对待了，这种情况也要去除。
5. Blacklist区域
ENCODE数据库提供了Blacklist区域列表（人类、小鼠、果蝇和线虫）。这些区域倾向于被大量的Read覆盖，是需要被去除的技术假阳性。这些区域也被移植到了Signac工具箱中。



#四、Peak注释。
Peak注释不等于Peak Calling，而是包含Peak Calling。这一步是为第四步的矩阵构建做准备。Peak注释包括以下7种：

TF模体（Motif）：也就是转录因子在Peak上的结合位点。斯坦福大学的William Greenleaf开发的chromVAR工具就做了这种注释。
k-mer：即4k种ACGT的字符串。chromVAR中也采用了这种注释。
TSS：基因的转录起始位点承载了较多的信息，因此有些研究顺式（cis-）转录调控的工具（例如Trapnell的Cicero）会采用这种注释。
Bin/Window：把基因组切割成不重叠的固定长度的区间（通常是5 kb），在下游分析中做降维或者聚类。UCSD的任兵开发的snapATAC和Greenleaf开发的ArchR都采用了这种注释，只是Bin的长度不同，ArchR用的长度是500 bp。这种小的Bin能够更精确地检测到TF的结合位点的位置范围（300-500 bp）。
Peak：这是最主流、最复杂的注释方式，需要Peak Calling工具（Peak Caller）完成。多数软件（cisTopic、snapATAC、SCALE、scATAC-pro、Signac和ArchR）都采用了这种注释。
基因：与TSS类似，区别就在于TSS只考虑聚集在TSS附近的Fragment，而基因则考虑整个基因的区域（尤其是编码区）检测到的Fragment。
Topic：类似于双聚类（Bicluster），与双聚类的区别就在于它描述的是与Peak或者细胞的概率上的关系，而不是硬性的0-1关系。目前只有cisTopic采用这种注释。
1-4）和6-7）等方法比较简单。但是，由于单细胞中Read数量太少，ChIP-Seq用的Peak Caller无法检测到Peak。因此Peak Calling都是在Bulk规模上进行，也就是大量细胞的混合物。

