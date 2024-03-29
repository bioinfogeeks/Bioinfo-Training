## 目录
+ [Introduction to data format](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#introduction-to-data-format)
  + [关于质控](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#%E5%85%B3%E4%BA%8E%E8%B4%A8%E6%8E%A7)
  + [数据格式转换](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#%E6%95%B0%E6%8D%AE%E6%A0%BC%E5%BC%8F%E8%BD%AC%E6%8D%A2)
+ [Step1.trios phasing](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#step1trios-phasing)
+ [Step2.缺失基因型推断（missing genotype imputation）](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#step2%E7%BC%BA%E5%A4%B1%E5%9F%BA%E5%9B%A0%E5%9E%8B%E6%8E%A8%E6%96%ADmissing-genotype-imputation)
+ [Step3.风险评分(Genetic risk score)](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#step3%E9%A3%8E%E9%99%A9%E8%AF%84%E5%88%86genetic-risk-score)
+ [Step4.祖源分析(Ancestry)](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#step4%E7%A5%96%E6%BA%90%E5%88%86%E6%9E%90ancestry)
  + [radmixture使用指南](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#1radmixture%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97)
  + [祖源计算原理](https://github.com/bioinfogeeks/Bioinfo-Training/tree/master/Train1-SNP%20Array%20Analysis%20pipeline#2%E7%A5%96%E6%BA%90%E8%AE%A1%E7%AE%97%E5%8E%9F%E7%90%86)

##  Introduction to data format
果壳那边可以直接出 **`.PED`** 和 **`.MAP`** 格式的数据，所以对于我们下游处理来说更加方便了
![图](https://github.com/bioinfogeeks/Bioinfo-Training/blob/master/Train1-SNP%20Array%20Analysis%20pipeline/pic/4.png)

#### 关于质控
从果壳拿到的数据已经经过了质控，因此我们直接进行下一步的基因型数据填充即可

#### 数据格式转换
+ 果壳提供.PED/ .MAP 格式的文件， （如果是VCF，需要用VCF tools转换成 plink的输入数据格式），所以现在直接使用 **`Gtools`** 将Plink格式的文件转为 **`IMPUTES2`** 的输入文件

```shell
gtool -P --ped test.ped --map test.map --og test.gen --os test.sample
```

---

## Step1.trios phasing
trios phasing的目的是区分母系和父系遗传的等位基因，即，一个人的某个基因型是AT，通过phasing的步骤，就可以了解A和T分别遗传自母亲还是父亲
【此步骤对后续祖源分析有用】
trios phasing分为两类：有家系数据和没有家系数据

+ **`有家系数据`**
如果能够获取家庭成员的基因检测数据，那么就可以以父母的数据作为背景，进行phasing，这样准确率是更高的

+ **`无家系数据`**
但是大多数情况无法获取用户的父母数据，因此，只能以大人群队列的数据作为参考
由于目前也没有东亚人群的数据库，暂定 *HapMap* 和 *1000 Genomes*的数据作为reference population

+ **`使用IMPUTES2完成phasing`**

```bash
## 使用IMPUTE2进行phasing
# imputes2的phase选项可用于完成phasing
# Example
./impute2 \
-phase \
-m data/test.map \
-g data/test.gen \
-int 0 50000 \
-Ne 20000 \
-o data/test.phasing.impute2
```
+ 输出结果
```shell
test.phasing.impute2_info_by_sample
test.phasing.impute2_summary
test.phasing.impute2_haps 
test.phasing.impute2_warnings
test.phasing.impute2_haps_confidence
test.phasing.impute2_info
```

---

## Step2.缺失基因型推断（missing genotype imputation）
这里仍使用IMPUTES2完成imputation工作

```bash
# Example
./impute2 \
-use_prephased_g \
-m data/test.map \
-h  ALL_1000G_phase.haps \
-l ALL_1000G_phase.legend \
-known_haps_g data/test.phasing.impute2_haps \
-int 0 50000 \
-Ne 20000 \
-o data/imputated_result \
```
+ **`-use_prephased_g`**  使用phased得到的数据来做Imputation
+ **`m`** 我们之前的 .map文件
+ **`h`** 参考基因集的单倍体型文件
+ **`l`**  参考基因集的单体型文件对应的Legend文件，保存的是对每个SNP位点的描述信息
+ **`known_haps_g`** pre-phase环节得到的单体型文件
+ **`int`** 指定滑动窗口填补的碱基对位置的上下区间 
+ **`Ne`**

IMPUTES可以完成多个种类的imputation工作，具体使用指南可以参考：[IMPUTES2手册](https://mathgen.stats.ox.ac.uk/impute/impute_v2.html#examples)

---

## Step3.风险评分(Genetic risk score)
PRS(polygenic risk score)是一个比较大的概念，称作多基因风险评分，它还有个名字叫做 Genetic Risk Score(GRS)。GRS分为多个种类：简单GRS，加权GRS等等，目前基因检测公司所用的风险预测模型大多都是加权GRS(wGRS) ， 因此我们在计算得时候也采用这个模型。
#### wGRS模型定义
+ wGRS=ΣβiSi(βi 为第 i 个 SNPs 的权重， Si 为第 i 个 SNPs)。该算法认为每个风险等位基因对疾病的影响不同，通过给每个风险等位基因赋予一个相应的权重来显示不同SNPs 对疾病的影响程度不同 
+ βi 值来源于已有的GWAS研究中的OR值(离散表型为OR值，例如单眼皮或者双眼皮，逻辑回归)或者β值(连续表型为β，例如身高体重，线性回归)

#### 一个计算的例子
以我们运动基因<肩袖损伤可能性>这个项目为例，它给出的文献是 Genome-wide association study identifies a locus associated with rotator cuff injury
假设现在有一个人，它的这个位点的携带基因型是AA，我们要计算他的风险概率，(其实就可以视作我们已有OR值后，利用逻辑回归的公式反推概率P)
+ **`第一步，设定计算标准：`**

含有风险等位基因纯合子（有两个高风险等位基因）——记2分

杂合子——记1分

没有风险等位基因——记0分


+ **`第二步，查看GWAS文献中给出的统计系数（OR值）`**
**文献给出了风险snp和对应的OR值，风险碱基是A**

![文献](https://github.com/bioinfogeeks/Bioinfo-Training/blob/master/Train1-SNP%20Array%20Analysis%20pipeline/pic/1.png)

+ **`第三步，计算GRS`**
  + 因为这个人携带2个风险碱基(AA)，因此 GRS=ΣβiSi=1.25*2=2.5
  + 逻辑回归预测公式： p=1/(1+e^(-ΣβiSi))， 大写的P代表pheno, 小写的p代表概率

![公式](https://github.com/bioinfogeeks/Bioinfo-Training/blob/master/Train1-SNP%20Array%20Analysis%20pipeline/pic/2.jpg)

  + 计算风险概率 = 1/(1+2.71828^(-2.5)) = 0.9241417 ≈ 92.4%
  + **注意** GWAS研究中，可能还给了性别，年龄等协变量的OR值，我们在计算GRS时，如果可以获得个体样本的年龄，性别信息，也可以纳入模型一起计算。为简化计算，此处我们没有纳入常量B0。

+ **`总结`**
（1）疾病风险预测的模型有很多，我们此处用的是最基本的logistic回归模型。可以看到SNP如果越多，纳入的项目越多，那么我们在预测一个人的风险的时候，会更加综合和平均。
（2）有些GWAS研究会自己构建的PRS模型，进行预测验证，这个时候如果要追求预测准确，最佳的方法是，我们直接拿到GWAS研究者所构建的预测模型来做预测。但是通常情况下，以运动基因为例，GWAS研究比较少，构建了预测模型并给出源码的 就更少了。
（3） **我们有好多运动基因其实没拿到OR值(因为没有GWAS研究去研究他们和表型的关联性)，因此此时我们预测风险高低的时候，就用最简单的SNP加和(ΣSi，分越高 风险越大)**

#### **`关于PRSice`**


> 参考链接

---

## Step4.祖源分析(Ancestry)
+ 虽然现在有一些公开的祖源分析网站，但是大多是国外的网站，所以对于中国人的祖源分析没有太多的参考意义
+ **`WeGene`** 公司使用的是基于美国加州大学洛杉矶分校Admixture工具改进的算法来完成的祖源相似性比较
+ 我们目前暂时使用R包radmixture来做祖源分析，但是我们使用的是公共参考数据库(上海马普所的一个老师好像建立了一个汉族人群队列的数据库，之后可以探究使用)，因此准确性有待提高

#### 1.radmixture使用指南

**`安装radmixture包`**
```r
## 可以通过github安装
devtools::install_github("wegene-llc/radmixture")
## 也可以通过CRAN安装
install.packages("radmixture")
```
**`输入数据格式`**
输入radmixture包进行祖源分析的数据要满足一定的格式，类似于这种：

+ 第一列是rsid号
+ 第二列是chr号（染色体）
+ 第三列是snp位点的position
+ 第四列是genotype(基因型)

**`使用公共数据集，计算祖源相似性`**
+ 第一步，读入snp芯片数据
```r
library(radmixture)
genotype <- read.table(file = '/path/to/file')
# genotype <- read.csv(file = 'path/to/file')
```
+ 第二步，下载公共数据集，并载入数据
```r
## 首先下载数据集
download.file(url = 'https://github.com/wegene-llc/radmixture/raw/master/data/globe4.alleles.RData', destfile = '/path/to/globe4.alleles.RData')
download.file(url = 'https://github.com/wegene-llc/radmixture/raw/master/data/globe4.4.F.RData', destfile = '/path/to/globe4.4.F.RData')
## 加载数据集
load('/path/to/globe4.alleles.RData')
load('/path/to/globe4.4.F.RData')
```
+ 第三步，使用`tfrdpub()`函数将我们的raw data转变为radmixture包可以读懂的数据格式
使用`fFixQN()`计算种族分层结果

```r
# Use K4
res <- tfrdpub(genotype, 4, globe4.alleles, globe4.4.F)
ances <- fFixQN(res$g, res$q, res$f, tol = 1e-4, method = "BR", pubdata = "K4")
## 取出计算结果
ances$q
##        European   Asian African Amerindian
## result    1e-05 0.94662   1e-05    0.05336

# Use K7b
res <- tfrdpub(genotype, 7, K7b.alleles, K7b.7.F)
ances <- fFixQN(res$g, res$q, res$f, tol = 1e-4, method = "BR", pubdata = "K7b")
```
用的是果壳芯片东亚人种的测试数据，可以看到结果中94%都是亚洲人血统，说明还是比较准

#### 2.祖源计算原理
+ 祖源计算，简单而言就是进行群体分层，探寻某个人究竟属于哪个人种（举个不恰当的例子，可以参考动物育种里面的品种分类，例如鉴定某某牛，是荷兰牛，还是美国进口牛的品种）。
+ 它的原理是，每个杂交后代（父代是不同的品种），他们的基因组的一些基因型频率和父代的基因型频率之间具有一定的相关性。所以我们可以通过一些统计方法，计算某个品种父代对子代个体基因组的遗传贡献率。
+ 目前使用snp进行祖源分析预测主要有两类方法：
  + 基于admixture的方法（多群体基因频率混合分布）
  + 线性回归分析的方法
+ 常用的计算软件有：
  + 估计亲缘关系： ADMIXTURE,  ANCESTRY
   + 估计群体结构：STRUCTURE， MENDEL和EigenStrait
   + 使用线性回归方法预测： R语言







