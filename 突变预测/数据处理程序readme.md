# 1、数据筛选
## HGMD目录
(1)：从HGMD中筛选出可能包含inframe突变的DM样本
代码：HGMD2vcf.py，
输入为Indel.csv
输出为hgmd_dm_Nogross.vcf和hgmd_dm_Onlygross.vcf

(2)：上一步输出的两个文件使用vep官网注释，注释突变类型，根据注释筛选出inframe突变。
Vep官网为[https://asia.ensembl.org/Tools/VEP](https://asia.ensembl.org/Tools/VEP)。注释时使用默认选项，仅增加一项筛选的设置：“Return results for variants in coding regions only”，
输入为hgmd_dm_Nogross.vcf和hgmd_dm_Onlygross.vcf
注释后的文件命名为hgmd_dm_Onlygross_ vep.vcf和hgmd_dm_Nogross_vep.vcf

(3)处理带有注释的文件，只要注释的不同转录本上的结果中包含inframe突变就选择保留，得到Gross种类中的inframe突变。
代码：gross2inframe.py，
输入：hgmd_dm_Onlygross_ vep.vcf和hgmd_dm_Nogross_vep.vcf
输出为hgmd_dm_Onlygross_vep_inframe.vcf ，hgmd_dm_Nogross_vep_inframe.vcf

(4)合并hgmd_dm_Onlygross_vep_inframe.vcf 和hgmd_dm_Nogross_vep_inframe.vcf即可得到最后的数据“hgmd_dm_inframe.vcf”. 

代码：vcf2bed.py
输入为hgmd_dm_Onlygross_vep_inframe.vcf ，hgmd_dm_Nogross_vep_inframe.vcf
输出为hgmd_dm_inframe.vcf

## ClinVar目录

代码：clinvar2vcf.py。

输出为：“TwoStar_p.vcf”，“TwoStar_n.vcf”，“oneStar_p.vcf”和“oneStar_n.vcf”

## gnomAD目录

主要思路：从注释过的gnomAD中粗筛出inframe突变，并通过程序对VCF文件的filter字段筛选，最终获得filter字段为PASS的inframe样本。

(1)：使用linux的命令行代码, 从gnomADv2版本的原始数据中先选出有一条转录本上注释结果为inframe的行。输出

zcat gnomad.exomes.r2.1.1.sites.liftover_grch38.vcf.gz | grep "inframe" > gnomAD_inframe.vcf

输入：zcat gnomad.exomes.r2.1.1.sites.liftover_grch38.vcf.gz 
输出：gnomAD_inframe.vcf。

(2)：从上一步的文件中筛选出filter字段为PASS的样本，去除体细胞的突变并且删除掉原先的INFO注释字段

程序：gnomAD2vcf_passNoVep.py

输入：gnomAD_inframe.vcf
输出：gnomAD_inframe_PASS_noVEP.vcf

## test/DDD目录

从原始数据中，获得vcf文件的突变表示。包含DDD测试集的正样本和负样本数据。
DDD2vcf.py 
输入：DDD.xlsx
输出文件：Pos_DDD.vcf和Neg_DDD.vcf

# 2、序列信息提取
以vcf文件为输入，使用bedtools工具和pandas包，最终获得包含突变前后上下文序列的csv文件。(在以上四个目录都运行一次)

(1)：将vcf文件转化为bedtools输入要求的bed格式。

代码：每个数据来源目录下的vcf2bed.py
输入：xxx.vcf
输出: xxx_200.bed
补充说明：代码中设置获取突变起始点前后200bp的序列。后续处理中根据此序列进行截取上下文序列。

(2)：使用bedtools的下面命令在服务器获取突变位点在基因上对应的碱基序列

例子：/data4/linming/bedtools2/bin/bedtools getfasta -fi GRCh38.fa -bed gnomAD_inframe_PASS_noVEP_200.bed -name > gnomAD_inframe_PASS_noVEP_200.out
输入：xxx_200.bed 
输出: xxx_200.out

补充说明：其中GRCh38.fa为参考基因组，bed为上一步得到的bed格式文件，out为输出文件名。

(3)：根据vcf和bed文件，获取突变上下文100bp的序列，输出为csv文件。代码文件：每个数据来源目录下的bed2data.py

输入：xxx_200.out
输出: xxx.csv
补充说明：这一步依据突变前后序列进行了该数据来源内部的去重（使用”CHROM", "POS", "REF", "ALT"关键字），并且去除了插入缺失长度不能被三整除的突变。设置context=True，获取插入缺失突变上下文序列。设置context=False，获取突变起始点上下文序列。

# 3、不同来源数据处理
(1)：对不同来源的数据集的筛选插入缺失长度，去除重复和矛盾的样本,对数据集做close-by，可以得到处理好的数据，方便从中分理出测试集

准备：需要将训练集的csv文件放在 “/context/100/”目录下，100为截取的序列上下文长度，如果设置别的长度需要相应更改。

代码：progress_content.py
输入：inframe_context_hgmd_dm_inframe100.csv，gnomAD_inframe_PASS_noVEP_AC100.csv  
inframe_context_TwoStar_p_inframe100_seq.csv
inframe_context_TwoStar_n_inframe100_seq.csv"
inframe_context_oneStar_p_inframe100_seq.csv"
inframe_context_oneStar_n_inframe100_seq.csv
../test/DDD/test_contexts100.csv

输出: 当前文件夹下输出 closeBy_hgmd.vcf 、closeBy_gnomAD.vcf  作为分离出测试集的文件。
withgross的文件夹下输出数据的vcf文件 clivar_pos.vcf 、clinvar_neg.vcf
result文件夹下输出五折交叉验证的数据的tsv文件，区分突变前后存放
fea文件夹下输出所有训练集和测试集的tsv文件，区分突变前后存放， dev1.tsv表示DDD测试集



(2)：对比2018.2pro版本的HGMD数据得到更新数据，从更新的数据中，以基因为依据，每个基因随机取一条组成新的测试集。输出新测试集的vcf文件

代码：get_new_testdataset.ipynb，
输入：closeBy_hgmd.vcf 、closeBy_gnomAD.vcf 、HGMD/hgmd_pro_2018.2_hg38.vcf、HGMD/hgmd_dm_Nogross.vcf、HGMD/hgmd_dm_Onlygross.vcf
输出: hgmd_test_gene.csv、gnomAD_test_gene.csv


(3)：和progress_content.py功能类似，但是考虑新测试集。

准备：需要将训练集的csv文件放在 “/context/100/”目录下，100为截取的序列上下文长度，如果设置别的长度需要相应更改。

代码：progress_content_newtest.py
输入：inframe_context_hgmd_dm_inframe100.csv，gnomAD_inframe_PASS_noVEP_AC100.csv  
inframe_context_TwoStar_p_inframe100_seq.csv
inframe_context_TwoStar_n_inframe100_seq.csv"
inframe_context_oneStar_p_inframe100_seq.csv"
inframe_context_oneStar_n_inframe100_seq.csv
../test/DDD/test_contexts100.csv
输出: 
result_newtest_文件夹下输出 划分出新测试集后的五折交叉验证的数据的tsv文件，区分突变前后存放
fea文件夹下输出所有训练集和测试集的tsv文件，区分突变前后存放 dev1.tsv表示DDD测试集，dev2.tsv表示新的分离出的测试集
