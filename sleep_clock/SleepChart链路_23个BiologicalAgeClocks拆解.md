# SleepChart链路下 23 个 Biological Age Clocks 拆解

## 1. 文档目的

这份文档不是复述 `Sleep chart of biological ageing clocks in middle and late life` 一篇文章，而是沿着它引用的前作、补充材料和 GitHub 仓库，尽量回答下面几个对建模最关键的问题：

- 这 `23` 个 clock 分别是什么。
- 每个 clock 的定义是什么。
- 每个 clock 纳入了哪些变量。
- 训练时用了什么方法。
- 哪些信息是公开可追溯的，哪些还没有公开到“最终权重/最终模型文件”这一层。

这份文档更适合用作你后续做“大规模多器官/多组学 biological age clock”时的参考底稿。

## 2. 我实际核对过的来源

### 主论文

- `Sleep chart of biological ageing clocks in middle and late life`
  - DOI: [10.1038/s41586-026-10524-5](https://doi.org/10.1038/s41586-026-10524-5)
  - 作用：说明 SleepChart 如何把既有 `23` 个 clock 拿来做睡眠关联、疾病风险、生存和中介分析。

### 前作论文

- `MRI-based multi-organ clocks for healthy aging and disease assessment`
  - DOI: [10.1038/s41591-025-03999-8](https://doi.org/10.1038/s41591-025-03999-8)
  - 作用：`7` 个 `MRIBAG` 的来源。

- `Refining the generation, interpretation and application of multi-organ, multi-omics biological aging clocks`
  - DOI: [10.1038/s43587-025-00928-9](https://doi.org/10.1038/s43587-025-00928-9)
  - 作用：`11` 个 `ProtBAG` 的蛋白列表、基准性能和解释框架。

- `Multi-organ metabolome biological age implicates cardiometabolic conditions and mortality risk`
  - DOI: [10.1038/s41467-025-59964-z](https://doi.org/10.1038/s41467-025-59964-z)
  - 作用：`5` 个 `MetBAG` 的代谢物来源、基准性能和器官注释表。

### GitHub / Portal

- `SleepChart` repo: [anbai106/SleepChart](https://github.com/anbai106/SleepChart)
  - 公开了 `GAM` 关联分析脚本，不公开个体级 `BAG` 数据。

- `MLNI` repo: [anbai106/mlni](https://github.com/anbai106/mlni)
  - 公开了回归建模框架，支持 `SVR`、`LASSO`、`Elastic Net`、`NN`。

- `SleepChart portal`: [labs-laboratory.com/sleepchart](https://labs-laboratory.com/sleepchart)
- `MEDICINE portal`: [labs-laboratory.com/medicine](https://labs-laboratory.com/medicine/)

### 我本地额外下载并核对的补充材料

这些文件已经放在当前工作区下：

- [mri_supp_data.xlsx](E:\生物学年龄\模型总结\口腔微生物时钟\tmp_sources\mri_supp_data.xlsx)
- [mri_supp_code.zip](E:\生物学年龄\模型总结\口腔微生物时钟\tmp_sources\mri_supp_code.zip)
- [mri_supp_info.pdf](E:\生物学年龄\模型总结\口腔微生物时钟\tmp_sources\mri_supp_info.pdf)
- [prot_supp_info.pdf](E:\生物学年龄\模型总结\口腔微生物时钟\tmp_sources\prot_supp_info.pdf)
- [met_supp_info.pdf](E:\生物学年龄\模型总结\口腔微生物时钟\tmp_sources\met_supp_info.pdf)

## 3. 先给总图：23 个 clock 是什么

### MRI clocks: 7 个

- `Brain_MRIBAG`
- `Heart_MRIBAG`
- `Adipose_MRIBAG`
- `Kidney_MRIBAG`
- `Liver_MRIBAG`
- `Pancreas_MRIBAG`
- `Spleen_MRIBAG`

### Proteomics clocks: 11 个

- `Brain_ProtBAG`
- `Eye_ProtBAG`
- `Heart_ProtBAG`
- `Pulmonary_ProtBAG`
- `Renal_ProtBAG`
- `Hepatic_ProtBAG`
- `Immune_ProtBAG`
- `Endocrine_ProtBAG`
- `Skin_ProtBAG`
- `Reproductive_female_ProtBAG`
- `Reproductive_male_ProtBAG`

### Metabolomics clocks: 5 个

- `Endocrine_MetBAG`
- `Digestive_MetBAG`
- `Hepatic_MetBAG`
- `Immune_MetBAG`
- `Metabolic_MetBAG`

## 4. 方法总框架：这套 clock 是怎么训练出来的

## 4.1 共同思想

这条链路里，clock 训练不是“直接拿所有人做年龄回归”，而是先在相对健康的人群上学习“正常年龄轨迹”，再把病人放进去算 `predicted age` 和 `BAG`。

例子：

- `CN` = 没有疾病诊断记录的人
- `PT` = 有至少一个疾病诊断记录的人
- 先在 `CN` 上学年龄，再把模型应用到 `PT`

这样做的好处是：

- 训练目标更接近“纯衰老轨迹”
- 后续疾病分析里更容易把 `BAG` 解释成“相对健康年龄轨迹的偏离”

## 4.2 训练/验证框架

从公开补充材料能确定的共同结构是：

- 使用 `nested cross-validation`
- 外层使用 `repeated hold-out`
- 外层重复次数通常是 `50`
- 内层通常是 `10-fold` 做超参数选择
- 单独留出一个 `independent test` 集做最终泛化评估
- 之后做 `age bias correction`

### MRI 链路能直接确认到的脚本细节

`mri_supp_code.zip` 里能直接确认：

- `run_mlni_lasso.py` 调用 `regression_roi(..., 50, n_threads=8)`
- `run_mlni_SVR.py` 调用 `regression_roi(..., 50, n_threads=8)`
- `run_mlni_elastic.py` 调用 `regression_roi(..., 50, n_threads=8)`
- `run_mlni_nn.py` 调用 `regression_roi(..., 50, epochs=2500, n_threads=8, gpu=True)`

可见 MRI clock 至少明确跑了四类模型：

- `SVR`
- `LASSO`
- `Elastic Net`
- `Neural Network`

### MRI 链路能直接确认到的数据切分

`1_prepare_mlni_cv_split.py` 明确写了：

- 从 `CN` 中随机抽 `500` 人做 `independent test`
- 其余 `CN` 进入训练/验证/测试循环
- `PT` 单独导出后用于后续外部应用

### Age bias correction

`3_age_bias_correction.py` 明确写了：

- 先算 `bag = predicted_age - chronological_age`
- 再回归 `bag ~ chronological_age`
- 用回归得到的 `alpha, beta` 对预测年龄做偏差校正

这意味着这里的 bias correction 不是只看简单差值，而是显式做了年龄相关偏差回归。

例子：

- 原始 `predicted_age` 往往会对年轻人高估、对老年人低估
- 他们用 `bag ~ age` 的线性关系把这个系统性偏差扣掉

## 5. 证据分级：哪些信息是“确定的”，哪些只是“高可信推断”

### A 级：公开材料里可直接确认

- 23 个 clock 的名字
- MRI 7 个 clock 的精确输入特征名单
- 11 个 ProtBAG 的精确蛋白名单
- 107 个 organ-associated metabolite 的注释表
- SleepChart 的 `GAM` 协变量与分析流程
- MLNI 里实现了哪些回归模型
- ProtBAG 和 MetBAG 的基准性能表

### B 级：高可信推断，但正文没有公开成“最终模型文件”

- 某个 organ 最终 downstream 用的是哪一种模型
- `Metabolic_MetBAG` 的精确输入子集

### C 级：目前没公开到可复现最终参数的层级

- 各 clock 的最终模型权重
- 完整训练后导出的 joblib/model artifact
- 全部 `23` 个 clock 的统一表格式最终系数文件

## 6. SleepChart 这篇文章本身公开了什么

`SleepChart` 仓库只公开了睡眠和 `23 BAGs` 的关联分析脚本：

- `code/GAM_SleepChart.R`

它明确写了：

- 23 个 `BAG` 的列名
- 分析协变量
- `sleep_duration` 限定在 `4–10 h`
- `mgcv::gam`
- 候选 family:
  - `Gaussian`
  - `scat()` / t-like family
  - `Gamma(log)`
- `k = 3, 5, 10, 15, 20`
- 通过 `AIC` 选最佳 family 和 k
- 估计 female / male 两条曲线
- 输出每个 clock 的拟合曲线与统计量

它没有公开：

- 个体级 `sleep` 数据
- 个体级 `23 BAGs`
- 训练这些 `23 BAGs` 的最终模型文件

## 7. 逐类详细拆解

## 7.1 MRI clocks：7 个 MRIBAG

### 总体定义

`MRIBAG` = 基于 MRI 或 MRI-derived imaging features 训练出的器官年龄预测模型；`BAG` 是 `predicted age - chronological age`，再经过年龄偏差校正。

### 数据来源

- UK Biobank MRI
- 脑：MUSE atlas 的 `T1` 结构 ROI
- 心脏：cardiac MRI derived traits
- 腹部器官：UKBB abdominal MRI derived traits

### 训练方法

根据 `mri_supp_code.zip` 与主文 Methods，可以确认：

- 候选模型：
  - `SVR`
  - `LASSO`
  - `Elastic Net`
  - `NN`
- 标准化：
  - `minmax`
- 外层：
  - `50` 次 repeated hold-out
- 内层：
  - nested CV
- 独立测试集：
  - `CN` 中随机 `500` 人
- 训练人群：
  - `CN`
- 患者外推：
  - `PT`

### 每个 MRIBAG 的定义与变量

#### 1. `Brain_MRIBAG`

- 定义：基于脑结构 MRI 的 `119` 个 MUSE ROI 训练的 brain age clock。
- 输入变量：`119` 个灰质/皮层/皮层下 ROI。
- 变量类型：结构体积或区域量化指标。
- 公开程度：`A 级`，精确名单已公开。

例子：

- `Right Accumbens Area`
- `Left Hippocampus`
- `Right AIns`
- `Left MFG`
- `Right PCu`

#### 2. `Heart_MRIBAG`

- 定义：基于心脏 MRI 量化特征训练的 heart age clock。
- 输入变量：`80` 个 cardiac MRI traits。
- 变量类型：
  - 腔室容积
  - 射血分数
  - 心肌质量
  - 主动脉面积
  - `AHA` 分区壁厚
  - 应变指标
- 公开程度：`A 级`

例子：

- `lv_end_diastolic_volume_f24100_2_0`
- `lv_ejection_fraction_f24103_2_0`
- `rv_end_systolic_volume_f24107_2_0`
- `ascending_aorta_maximum_area_f24118_2_0`
- `lv_circumferential_strain_global_f24157_2_0`

#### 3. `Adipose_MRIBAG`

- 定义：基于脂肪分布、肌肉脂肪浸润和体成分 MRI 指标的 adipose age clock。
- 输入变量：`16` 个。
- 公开程度：`A 级`

关键变量：

- `Visceral_fat_volume_21085-2.0`
- `Pancreas_PDFF_(fat_fraction)_21090-2.0`
- `Anterior_thigh_fat-free_muscle_volume_(right)_22403-2.0`
- `Posterior_thigh_fat-free_muscle_volume_(left)_22406-2.0`
- `Abdominal_subcutaneous_adipose_tissue_volume_(ASAT)_22408-2.0`
- `Total_trunk_fat_volume_22410-2.0`
- `Total_abdominal_adipose_tissue_index_22432-2.0`
- `Abdominal_fat_ratio_22434-2.0`
- `Muscle_fat_infiltration_22435-2.0`
- `Proton_density_fat_fraction_(PDFF)_40061-2.0`

#### 4. `Liver_MRIBAG`

- 定义：基于肝脏体积、脂肪、铁负荷和 `cT1` 的 liver age clock。
- 输入变量：`4` 个。
- 公开程度：`A 级`

变量：

- `Liver_volume_21080-2.0`
- `Liver_PDFF_(fat_fraction)_21088-2.0`
- `Liver_iron_21089-2.0`
- `Liver_iron_corrected_T1_(ct1)_40062-2.0`

#### 5. `Spleen_MRIBAG`

- 定义：基于脾脏体积与铁相关指标的 spleen age clock。
- 输入变量：`3` 个。
- 公开程度：`A 级`

变量：

- `Spleen_volume_21083-2.0`
- `Spleen_iron_-_IDEAL_21170-2.0`
- `Spleen_iron_-_protocol_normalised_21173-2.0`

#### 6. `Kidney_MRIBAG`

- 定义：基于肾体积与形态距离特征的 kidney age clock。
- 输入变量：`3` 个。
- 公开程度：`A 级`

变量：

- `Left_kidney_volume_21081-2.0`
- `Kidney_parenchyma_(right)_21162-2.0`
- `Kidney_distance_21163-2.0`

#### 7. `Pancreas_MRIBAG`

- 定义：基于胰腺体积、脂肪和铁的 pancreas age clock。
- 输入变量：`3` 个。
- 公开程度：`A 级`

变量：

- `Pancreas_volume_21087-2.0`
- `Pancreas_PDFF_(fat_fraction)_21090-2.0`
- `Pancreas_iron_21091-2.0`

### MRI clocks 这支目前还缺什么

- 公开材料里能确认候选模型和训练脚本。
- 但“每个器官最终到底选了哪一个模型”在我当前能稳定解析到的文件里，没有找到和 ProtBAG/MetBAG 同样清晰的最终汇总表。
- 所以 MRI 这支更稳的表述是：
  - `最终框架确定`
  - `精确变量确定`
  - `最终 deployed model type 不能像 ProtBAG/MetBAG 一样逐个百分百确认`

## 7.2 Proteomics clocks：11 个 ProtBAG

### 总体定义

`ProtBAG` = 基于血浆蛋白组中的器官富集蛋白，训练器官/系统层面的 age prediction model。

### 数据来源

- UK Biobank plasma proteomics
- 使用 `HPA` 定义 tissue-enriched proteins
- 公开补充材料列出了每个 ProtBAG 的精确蛋白清单

### 训练/评估框架

从补充材料可直接确认：

- 训练人群：
  - `CN nested CV dataset = 4589`
- 独立测试：
  - `CN ind. test = 500`
- 患者外部应用：
  - `PT = 38,409`
- 外层：
  - `50` 次 repeated hold-out
- 内层：
  - `10-fold CV`
- 比较模型：
  - `svr`
  - `lasso`
  - `nn`

### ProtBAG 的“最佳独立测试表现模型”

这里我用 `Supplementary Table 2` 的 `after age bias correction` 独立测试性能做 practical summary：

- `Reproductive_female_ProtBAG`: `lasso`
- `Pulmonary_ProtBAG`: `svr`
- `Heart_ProtBAG`: `lasso`
- `Brain_ProtBAG`: `nn`
- `Eye_ProtBAG`: `svr`
- `Hepatic_ProtBAG`: `lasso`
- `Renal_ProtBAG`: `lasso`
- `Reproductive_male_ProtBAG`: `lasso`
- `Endocrine_ProtBAG`: `lasso`
- `Immune_ProtBAG`: `lasso`
- `Skin_ProtBAG`: `lasso`

说明：

- 这是基于公开表格中 `Ind. test` 的 `MAE` 和 `r` 做的整理。
- 它非常接近作者最终使用的模型，但因为 PDF 粗文本抽取会丢失“加粗”格式，所以这里最好理解为“最佳公开基准模型”。

### 每个 ProtBAG 的定义、变量数与变量名单

#### 1. `Brain_ProtBAG`

- 定义：基于 brain-enriched plasma proteins 的 brain age clock。
- 输入蛋白数：`53`
- 最佳独立测试模型：`nn`

蛋白：

`PMCH, MOG, OXT, NCAN, MEPE, NRGN, MAG, MDGA1, CNTN2, CNP, VSNL1, GRIN2B, BCAN, CRTAM, GRIK2, CBLN1, CNDP1, NPTXR, LRTM2, SEPTIN8, C1QL2, CA11, SEPTIN3, ABCA2, APLP1, CRH, GFAP, IDS, IGSF21, RTN4R, FGFR2, IGLON5, LHPP, NPTX1, TUBB3, ADAM22, DNM1, ERC2, GP1BB, KLK6, PTPRZ1, SEZ6L, SLITRK1, SNAP25, CEND1, CLIP2, EDIL3, HPCAL1, ICAM5, OMG, PCDH9, SEZ6, TAFA5`

#### 2. `Eye_ProtBAG`

- 定义：基于 eye-enriched proteins 的 eye age clock。
- 输入蛋白数：`8`
- 最佳独立测试模型：`svr`

蛋白：

`CRX, IMPG1, PCARE, CLUL1, CABP2, RTBDN, STX3, WIF1`

#### 3. `Heart_ProtBAG`

- 定义：基于 heart-enriched proteins 的 heart age clock。
- 输入蛋白数：`6`
- 最佳独立测试模型：`lasso`

蛋白：

`NPPB, BMP10, MYL4, PXDNL, CRIP2, FGF12`

#### 4. `Pulmonary_ProtBAG`

- 定义：基于 lung-enriched proteins 的 pulmonary age clock。
- 输入蛋白数：`9`
- 最佳独立测试模型：`svr`

蛋白：

`SFTPA1, SFTPA2, SCGB3A2, SFTPD, AGER, SCGB1A1, LAMP3, CCL18, MSR1`

#### 5. `Renal_ProtBAG`

- 定义：基于 kidney-enriched proteins 的 renal age clock。
- 输入蛋白数：`8`
- 最佳独立测试模型：`lasso`

蛋白：

`UMOD, NPHS2, SOST, REN, PTH1R, SLC13A1, GGACT, PDZK1`

#### 6. `Hepatic_ProtBAG`

- 定义：基于 liver-enriched proteins 的 hepatic age clock。
- 输入蛋白数：`99`
- 最佳独立测试模型：`lasso`

蛋白：

`AHSG, CFHR2, MBL2, F9, CFHR5, A1BG, SERPINC1, APOA2, F2, ITIH1, SERPINA7, HAO1, APOH, CFHR4, FGA, APOF, AGXT, F12, C9, F13B, ORM1, HRG, INHBC, GDF2, C8B, SAA4, FGF21, ITIH3, PON1, F7, SERPINA11, CPB2, APCS, AMBP, LPA, HGFAC, LECT2, ANG, ASGR2, CCL16, PGLYRP2, SERPINA1, CA5A, ASGR1, PROC, ITIH4, PZP, ADH4, C5, KLKB1, FETUB, LBP, LRG1, CFB, PON3, SERPINA6, AKR1C4, GC, SERPINF2, ANGPTL3, BCHE, C4BPB, HSD11B1, AFM, PLG, AFP, MST1, APOC1, C3, CES1, CLEC1B, UPB1, DCXR, AGT, C2, CFH, EPO, FGL1, FTCD, IGFBP1, IL1RAP, APOA1, C1S, CFI, GCHFR, PCSK9, SERPINA4, SERPIND1, SHBG, SULT2A1, THPO, ACADSB, APOM, ARG1, F11, FCN2, FUOM, LEPR, RNASE4`

#### 7. `Immune_ProtBAG`

- 定义：基于 immune-enriched proteins 的 immune age clock。
- 输入蛋白数：`59`
- 最佳独立测试模型：`lasso`

蛋白：

`SH2D1A, GRAP2, SERPINA9, CD8A, TCL1A, CD1C, CR2, CTSV, CXCL9, CD3G, CXCL13, SIT1, SLAMF1, TIGIT, CD72, FCRL3, FCRL5, CCL17, CCL21, CD5L, LILRB1, STAB2, VCAM1, CCL19, CD27, CD79B, FCER2, HMOX1, LY75, NCR1, SLAMF6, MPO, RNASE3, RAB44, AHSP, PRTN3, AZU1, MMP8, CEACAM8, TARM1, SLC4A1, PGLYRP1, PRG3, CLC, EREG, CLEC4D, VSTM1, CCL7, ITGAM, OSM, CST7, CXCL8, RELT, S100A12, CLEC5A, HMBS, PLAUR, RETN, FOLR3`

#### 8. `Endocrine_ProtBAG`

- 定义：基于 endocrine / pancreas / pituitary / thyroid 相关富集蛋白的 endocrine age clock。
- 输入蛋白数：`51`
- 最佳独立测试模型：`lasso`

蛋白：

`CCN3, AKR1B1, DBH, SCARB1, CELA2A, CPA1, CTRB1, CELA3A, PNLIP, AMY2A, CLPS, CTRC, PNLIPRP1, PLA2G1B, CPB1, CTRL, GP2, PRSS2, CPA2, SERPINI2, AMY2B, PNLIPRP2, SPINK1, PPY, REG3G, REG1A, REG1B, GCG, GPHA2, RNASE1, KIRREL2, SEL1L, COCH, IL22RA1, TG, IGFBPL1, PTH, CD109, ALCAM, CHGA, ATP6AP2, TSHB, PRL, GH1, POMC, GHRHR, FSHB, LHB, GAL, CGA, NPTX2`

#### 9. `Skin_ProtBAG`

- 定义：基于 skin-enriched proteins 的 skin age clock。
- 输入蛋白数：`10`
- 最佳独立测试模型：`lasso`

蛋白：

`CDSN, FABP9, CST6, CCL27, SERPINA12, DSG4, CD207, EPPK1, S100A3, WFDC12`

#### 10. `Reproductive_female_ProtBAG`

- 定义：基于 female reproductive-enriched proteins 的女性生殖系统 age clock。
- 输入蛋白数：`13`
- 最佳独立测试模型：`lasso`

蛋白：

`BTN1A1, STC2, MMP10, PSG1, SIGLEC6, PRG2, TFPI2, INSL4, ADAM12, HGF, FCGR2B, FLT1, PAPPA`

#### 11. `Reproductive_male_ProtBAG`

- 定义：基于 male reproductive-enriched proteins 的男性生殖系统 age clock。
- 输入蛋白数：`26`
- 最佳独立测试模型：`lasso`

蛋白：

`LYZL2, DNAJB8, ZNRF4, DKKL1, PDCL2, TMCO5A, GAGE2A, TEX101, BRDT, IL13, ACRV1, CRISP2, ACRBP, IZUMO1, LRRC37A2, IL5, SPESP1, PHOSPHO1, KLK3, KLK4, MSMB, EDDM3B, SPINK2, NPC2, ADGRG2, ENPP5`

## 7.3 Metabolomics clocks：5 个 MetBAG

### 总体定义

`MetBAG` = 基于血浆代谢组构建的器官/系统 age clock。

### 这支最重要的两点

- 他们最后主要依赖 `107` 个 `non-derived raw metabolites`
- 不建议把大量 `composite metabolite` 一起并入训练，因为会引入高共线性并导致过拟合

例子：

- 作者在 Supplementary Note 1 里明确说，`107 original metabolites` 的 ratio/sum feature 会带来明显 overfitting。

### 数据来源

- UKBB Nightingale metabolomics
- 总分析特征 `327`
- 其中原始非派生代谢物 `107`
- Supplementary Table 2 给出了 `107` 个 metabolite 的 organ annotation

### 训练/评估框架

从补充材料可确认：

- 训练/验证/测试：
  - `CN training/validation/test = 29,354`
- 独立测试：
  - `CN ind. test = 5,000`
- 患者外部应用：
  - `PT = 239,893`
- 比较模型：
  - `LASSO`
  - `NN`

### 哪个模型更像最终下游模型

Supplementary Table 3 的标题明确写了：

- `bolded results are used for downstream genetic analyses and prediction analyses`

因为纯文本抽取无法保留加粗，这里给 practical summary：

- `Endocrine_MetBAG`: `LASSO` 明显优于 `NN`
- `Digestive_MetBAG`: `LASSO` 明显优于 `NN`
- `Hepatic_MetBAG`: `LASSO` 明显优于 `NN`
- `Metabolic_MetBAG`: `LASSO` 明显优于 `NN`
- `Immune_MetBAG`: `NN` 的独立测试 `MAE` 略优于 `LASSO`，但差距很小

更稳妥的研究口径是：

- 这篇工作以 `LASSO` 为主干模型非常明确
- `Immune_MetBAG` 是否最终用了 `NN`，单靠纯文本抽取不够稳，最好保留为“`NN` 与 `LASSO` 接近，需以原始加粗表或作者最终结果文件确认”

### 每个 MetBAG 的定义与变量

#### 1. `Endocrine_MetBAG`

- 定义：基于 endocrine-annotated raw metabolites 的 endocrine system clock。
- 输入变量数：`52`
- 变量来源：Supplementary Table 2 中 `Organ = Endocrine`
- 最佳公开基准模型：`LASSO`

变量：

`bOHbutyrate, Acetoacetate, Acetone, Albumin, HDL_size, VLDL_size, XXL_VLDL_CE, L_HDL_CE, L_VLDL_CE, S_VLDL_CE, XL_HDL_CE, XL_VLDL_CE, Citrate, XXL_VLDL_P, L_HDL_P, L_VLDL_P, M_VLDL_P, S_VLDL_P, XL_HDL_P, XL_VLDL_P, XS_VLDL_P, XXL_VLDL_FC, L_HDL_FC, L_VLDL_FC, XL_HDL_FC, XL_VLDL_FC, Gly, GlycA, MUFA, Omega_3, XXL_VLDL_PL, L_HDL_PL, L_VLDL_PL, M_VLDL_PL, S_VLDL_PL, XL_HDL_PL, XL_VLDL_PL, XS_VLDL_PL, SFA, XXL_VLDL_TG, IDL_TG, L_LDL_TG, L_VLDL_TG, M_HDL_TG, M_LDL_TG, M_VLDL_TG, S_HDL_TG, S_LDL_TG, S_VLDL_TG, XL_HDL_TG, XL_VLDL_TG, XS_VLDL_TG`

#### 2. `Hepatic_MetBAG`

- 定义：基于 hepatic-annotated metabolites 的 liver-related clock。
- 输入变量数：`22`
- 最佳公开基准模型：`LASSO`

变量：

`Acetate, Ala, ApoA1, LDL_size, M_HDL_CE, S_HDL_CE, M_HDL_P, S_HDL_P, DHA, M_HDL_FC, S_HDL_FC, His, Lactate, LA, Omega_6, Phosphatidylc, Phosphoglyc, M_HDL_PL, S_HDL_PL, Pyruvate, Sphingomyelins, Cholines`

#### 3. `Digestive_MetBAG`

- 定义：基于 digestive-annotated metabolites 的 digestive clock。
- 输入变量数：`14`
- 最佳公开基准模型：`LASSO`

变量：

`IDL_CE, M_VLDL_CE, XS_VLDL_CE, Unsaturation, IDL_FC, L_LDL_FC, Glucose, Gln, Ile, Leu, Phe, IDL_PL, Tyr, Val`

#### 4. `Immune_MetBAG`

- 定义：基于 immune-annotated metabolites 的 immune clock。
- 输入变量数：`17`
- 最佳公开基准模型：`LASSO/NN 接近`

变量：

`ApoB, L_LDL_CE, M_LDL_CE, S_LDL_CE, Clinical_LDL_C, IDL_P, L_LDL_P, M_LDL_P, S_LDL_P, M_LDL_FC, M_VLDL_FC, S_LDL_FC, S_VLDL_FC, XS_VLDL_FC, L_LDL_PL, M_LDL_PL, S_LDL_PL`

#### 5. `Metabolic_MetBAG`

- 定义：这是 5 个 MetBAG 里最需要谨慎表述的一个。
- 公开证据能确认：
  - 作者确实训练了 `Metabolic_MetBAG`
  - 它在表 3 中有单独性能
  - 全文 repeatedly 强调 `107 original metabolites`
- 但 Supplementary Table 2 并没有给出一个单独名为 `Organ = Metabolic` 的注释子表。

### 我对 `Metabolic_MetBAG` 的最稳妥解释

- **高可信推断**：`Metabolic_MetBAG` 最可能是基于全部 `107` 个 non-derived raw metabolites 训练出来的 broad metabolome clock。
- 理由：
  - 表 2 的器官注释总数正好是 `107`
  - 其中只有 `Endocrine/Hepatic/Digestive/Immune` 四个大类足以单独建器官时钟
  - 还剩 `Heart = 1`、`CNS = 1`
  - `Metabolic_MetBAG` 很自然对应“非器官专属、全局代谢层 clock”

### 这部分应如何写进你的研究里

建议写法：

- “`Metabolic_MetBAG` likely corresponds to a broad metabolome-based clock built from the full 107 non-derived metabolite panel, inferred from Supplementary Table 2 and the model-design description, although the authors did not publish a standalone variable list labelled as ‘Metabolic’.”

## 8. 这条链路给你研究设计上的真正启发

## 8.1 不要只做一个总时钟

这 23 个 clock 最值得学的，不是某个单模型，而是“按器官/组学拆时钟”。

例子：

- MRI 一层
- 蛋白一层
- 代谢一层
- 每层再拆器官或系统

## 8.2 健康轨迹训练 + 患病偏离解释

这条链路默认先学 `CN` 的年龄轨迹，再看 `PT` 的偏离。

例子：

- 如果你后面做口腔微生物时钟、代谢时钟或多模态时钟，都可以先用“无重大疾病/无显著炎症/相对健康”的人群训练基线轨迹。

## 8.3 Bias correction 一定要显式做

这套链路里，`predicted_age - age` 不是最终解释终点。

他们更在意：

- 先看 `bag`
- 再回归 `bag ~ chronological_age`
- 再做校正

这比直接用裸差值稳很多。

## 8.4 先看泛化，再谈解释

ProtBAG 和 MetBAG 的补充材料都把 `training` 和 `independent test` 分开写得很清楚。

例子：

- 有些 `NN` 在训练集上更好
- 但独立测试集明显过拟合
- 作者最后没有盲目追求训练集指标，而是强调独立测试集稳定性

## 8.5 变量筛选要和生物学先验结合

这条链路不是“把全 omics 硬塞进模型”。

而是：

- ProtBAG 先用 `HPA` 做 tissue-enriched protein 过滤
- MetBAG 先做 organ-specific metabolite annotation
- MRI 直接用器官结构/功能指标

例子：

- 如果你自己做新 clock，完全可以先做 `organ-anchored feature preselection`
- 这样比纯黑箱全变量投喂更稳，也更容易解释

## 9. 目前公开性最大的缺口

如果你想 100% 复现作者的最终 clock，还差这些信息：

- 各 clock 的最终系数或网络权重
- 全部 organ 的最终模型文件
- 明确写成一张表的 “clock -> final selected model”
- `Metabolic_MetBAG` 的官方独立变量清单

所以现实可行的复用方式不是“直接复刻作者权重”，而是复用它的：

- 时钟分层设计
- 变量预筛逻辑
- 训练/验证框架
- bias correction
- 下游验证链路

## 10. 一句话结论

如果只看可公开复用的价值，这条链路最核心的不是单个 clock 的神秘参数，而是这一整套范式：

- `先定义器官/系统特异的特征子集`
- `再在健康人里做年龄预测`
- `用 nested CV + independent test 控制过拟合`
- `显式做 age-bias correction`
- `最后把 BAG 连到疾病、死亡和中介机制`

这套范式非常适合迁移到你自己的“大规模 clocks”研究里。

## 11. 你下一步最值得做的三件事

### 方案 1：按这条链路先搭你自己的时钟矩阵

例子：

- 临床检验 clock
- 口腔微生物 clock
- 蛋白/代谢 clock
- 影像 clock

### 方案 2：统一定义健康训练集

例子：

- 先约束无癌症、无严重慢病、无近期急性感染
- 再建 “reference aging trajectory”

### 方案 3：统一下游评估模板

例子：

- 年龄预测性能：`MAE, Pearson r`
- 偏差校正后性能
- 疾病关联
- 生存分析
- 可解释性分析

## 12. 附：我在本次整理中直接核对过的关键文件

- [SleepChart README](https://github.com/anbai106/SleepChart)
- [SleepChart GAM script](https://raw.githubusercontent.com/anbai106/SleepChart/main/code/GAM_SleepChart.R)
- [MLNI README](https://raw.githubusercontent.com/anbai106/mlni/master/README.md)
- [MLNI adml_regression.py](https://raw.githubusercontent.com/anbai106/mlni/master/mlni/adml_regression.py)
- [MLNI adml_regression_lasso.py](https://raw.githubusercontent.com/anbai106/mlni/master/mlni/adml_regression_lasso.py)
- [MLNI adml_regression_elastic.py](https://raw.githubusercontent.com/anbai106/mlni/master/mlni/adml_regression_elastic.py)
- [MLNI adml_regression_nn.py](https://raw.githubusercontent.com/anbai106/mlni/master/mlni/adml_regression_nn.py)

