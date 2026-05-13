# 《Oral microbiome signatures predict biological age and host health》方法与代码解读

> 资料来源：本地 PDF《Oral microbiome signatures predict biological age and host health.pdf》、本地代码仓库 `Oral-Microbiome-Clock-main/Oral-Microbiome-Clock-main`。  
> 仓库地址：https://github.com/zhaojiajun0110/Oral-Microbiome-Clock

## 1. 一句话讲清楚这篇文章

这篇文章的核心思路是：**用口腔 16S rRNA 微生物数据先筛出随年龄稳定变化的菌属，再训练一个随机森林年龄预测模型；模型预测年龄减去真实年龄得到 OMAA Score，用它表示“口腔微生物年龄加速”，最后验证这个分数是否能预测死亡、衰弱、肾功能和慢病风险。**

举例：

- 某人实际 50 岁，模型根据口腔微生物预测为 58 岁，说明口腔微生物状态比同龄人“更老”。
- 这个“预测年龄 - 真实年龄”的偏差，就是文章想用来表达的生物学年龄加速信号。
- 如果这个偏差还能预测死亡、衰弱、肾功能下降或癌症/心血管风险，就说明它不只是一个年龄拟合模型，而是有健康风险含义。

## 2. 研究总体流程

文章可以拆成 6 步：

1. **收集数据**：使用 NHANES 2009-2010 和 2011-2012 两个队列的口腔微生物与临床资料。
2. **微生物预处理**：16S V4 区测序，得到 ASV，再聚合到 genus 水平；过滤低出现率和低丰度菌属；对菌属表做 CLR 转换。
3. **筛选年龄相关菌属**：用 GAM 模型找与年龄相关的菌属，并在发现队列和验证队列中交叉确认，得到 64 个年龄相关菌属。
4. **构建口腔微生物年龄模型**：用随机森林预测真实年龄，得到 oral microbiome age。
5. **计算 OMAA Score**：用预测年龄相对真实年龄的残差表示口腔微生物年龄加速。
6. **验证临床意义**：看 OMAA Score 是否关联死亡、衰弱、肾功能、慢病风险，并评估饮食和药物是否能解释 OMAA。

这套路线的关键不是“把年龄预测得极准”，而是把年龄预测残差变成一个风险指标。

## 3. 用了什么数据

### 3.1 发现队列和验证队列

文章使用两个 NHANES 队列：

| 数据集 | 角色 | 样本量 | 年龄范围 | 用途 |
|---|---:|---:|---:|---|
| NHANES 2009-2010 | Discovery cohort | 2,029 | 30-70 岁 | 筛菌、训练模型、内部交叉验证 |
| NHANES 2011-2012 | Validation cohort | 2,646 | 30-70 岁 | 独立验证模型和临床关联 |
| 外部多国家数据集 | External validation | 1,293 | 30-70 岁 | 检验模型跨人群泛化 |

纳入/排除条件：

- 纳入 30-70 岁人群。
- 排除协变量缺失者。
- 排除牙周病、妊娠、过去 1 个月使用抗生素者。

这样做的目的，是尽量提取“非明显口腔疾病驱动”的年龄信号。

### 3.2 微生物数据

样本类型是 **口腔冲洗液 oral rinse sample**，不是粪便，也不是单纯唾液。

测序与生信流程：

| 环节 | 方法 |
|---|---|
| 采样 | 10 mL 口腔冲洗液 |
| 测序区域 | 16S rRNA V4 区 |
| 测序平台 | Illumina HiSeq 2500, 125 bp paired-end |
| 原始数据处理 | QIIME 1 demultiplex |
| ASV 推断 | DADA2 |
| 系统发育树 | QIIME 2 |
| 物种注释 | SILVA v123 |
| 分析层级 | 聚合到 genus 水平 |

注意：文章明确说 16S 主要到 genus 水平，不能很好分到 species/strain，也不能直接推断功能通路。这是方法局限。

### 3.3 临床与结局数据

文章还合并了这些变量：

- 死亡结局：通过 National Death Index 追踪到 2019-12-31，最多约 10 年随访。
- 衰弱：49 项 deficit accumulation frailty index，FI >= 0.25 定义为 frail。
- 慢病：银屑病、关节炎、痛风、心衰、冠心病、心绞痛、心梗、卒中、肺气肿、肝病、慢性支气管炎、癌症、甲状腺疾病、糖尿病、高血压。
- 生化指标：血细胞、电解质、糖代谢、脂代谢、肾功能、肝功能、甲状腺功能等 45 个指标。
- 饮食：NHANES 24 小时膳食回顾，使用第一天 recall。
- 药物：过去 30 天处方药，选取至少 30 人使用的 37 种药物。
- 协变量：年龄、性别、种族、教育、婚姻、PIR、吸烟、饮酒、睡眠、BMI、体力活动、牙齿数等。

## 4. 数据经过什么处理

### 4.1 微生物过滤

文章先把 ASV 聚合成 genus 表，然后过滤：

- 出现率低于 10% 的菌属删除。
- 平均相对丰度低于 0.01% 的菌属删除。
- 最终保留 101 个 genus。

举例：

如果某个菌属只在 3% 的样本里出现，即使在少数人中丰度高，也会被过滤掉。原因是这种特征太稀疏，进入人群模型后不稳定。

### 4.2 CLR 转换

微生物丰度是组成型数据，所有菌的相对丰度加起来为 1。直接用相对丰度建模容易产生组成偏倚，所以文章对过滤后的 genus count table 做 **centered log-ratio, CLR** 转换。

直观理解：

- 相对丰度回答的是“这个菌占总菌群的多少”。
- CLR 更接近回答“这个菌相对整体菌群中心水平偏高还是偏低”。

代码仓库中的 `gam_data.xlsx` 已经是类似“每个 RSV_genus 一列”的建模表，数值包含负数，符合 CLR 后数据的形态。

### 4.3 年龄分组和多样性分析

文章把人群分成：

- younger：< 45 岁
- older：>= 45 岁

然后比较口腔微生物整体结构：

| 分析 | 方法 |
|---|---|
| alpha diversity | Observed ASVs, Faith's PD, Shannon, Inverse Simpson |
| alpha diversity 组间比较 | Wilcoxon rank-sum test |
| beta diversity | Bray-Curtis distance |
| 可视化 | PCoA |
| beta diversity 显著性 | PERMANOVA, 999 permutations |

结果显示 older 组 alpha diversity 下降，PCoA 中 younger/older 有显著但效应量较小的分离。

## 5. 用了哪些统计学方法和模型

### 5.1 GAM：筛选年龄相关菌属

目标：找出哪些菌属和年龄有关，而且允许非线性关系。

模型形式：

```text
CLR(Genus abundance) = beta0 + s(Age) + beta1 * Gender + beta2 * Race + error
```

其中：

- `s(Age)` 是年龄的平滑函数，可以捕捉非线性。
- `gender` 和 `race` 是协变量。
- 发现队列用 FDR < 0.05。
- 验证队列用 P < 0.05。

最终得到 **64 个年龄相关菌属**。

举例：

- Rothia 在老年组更高，文章把它和衰弱相关文献联系起来。
- Scardovia 随年龄下降，可能反映口腔生态或糖代谢相关变化。
- Stomatobaculum、Oribacterium 的 EDF 较高，说明与年龄的关系更非线性。

### 5.2 Random Forest：预测真实年龄

目标：用 64 个年龄相关菌属预测 chronological age。

模型：

- R 包：`randomForest`
- 输入：64 个 CLR 转换后的菌属特征
- 输出：预测年龄 oral microbiome age
- 训练集：NHANES 2009-2010
- 验证集：NHANES 2011-2012
- 外部验证：多国家 16S 数据集，使用重叠的 37 个菌属

评估指标：

| 指标 | 含义 |
|---|---|
| Spearman rho | 预测年龄与真实年龄的秩相关 |
| MAE | 预测年龄和真实年龄的平均绝对误差 |

主要结果：

| 数据集 | 相关性 | MAE |
|---|---:|---:|
| 发现队列 10-fold CV | rho = 0.44 | 8.69 年 |
| NHANES 验证队列 | rho = 0.35 | 9.16 年 |
| 外部验证集 | rho = 0.22 | 12.63 年 |

这个性能不如 DNA methylation clock 强，但文章强调它的价值在于非侵入、低成本、能预测健康结局。

### 5.3 OMAA Score：用残差表达年龄加速

OMAA Score 的定义：

```text
OMAA Score = predicted oral microbiome age - chronological age
```

文章 Methods 里表述为预测年龄与真实年龄之间的残差。

解释：

- OMAA > 0：口腔微生物年龄比真实年龄更老，代表加速老化。
- OMAA < 0：口腔微生物年龄比真实年龄更年轻。

举例：

| 真实年龄 | 模型预测年龄 | OMAA Score | 解释 |
|---:|---:|---:|---|
| 50 | 58 | +8 | 口腔微生物状态偏老 |
| 50 | 45 | -5 | 口腔微生物状态偏年轻 |

严格借鉴时，建议不要只做简单相减，还可以用：

```text
predicted_age ~ chronological_age
OMAA = residual
```

这样能减少分数和真实年龄的机械相关。

### 5.4 SHAP：解释哪些菌推动年龄预测

文章用 SHAP 解释随机森林模型，排序每个 genus 对年龄预测的贡献。

主要发现：

- Capnocytophaga
- Actinobacillus
- Stomatobaculum

是重要的年龄预测特征。

解释方式：

- SHAP 排名高，说明这个菌对模型判断“更老/更年轻”贡献大。
- 但 SHAP 不是因果证据，只是模型解释。

### 5.5 Cox 回归、线性回归、Logistic 回归：验证临床意义

文章不是停在年龄预测，而是继续验证 OMAA 的健康含义。

| 结局 | 方法 | 调整变量 |
|---|---|---|
| 全因死亡 | Cox proportional hazards model | 年龄、性别、种族、教育、婚姻、PIR、吸烟、饮酒、睡眠、BMI、体力活动、牙齿数 |
| 生存曲线 | Kaplan-Meier + Log-rank test | 按 OMAA > 0 / < 0 分组 |
| 衰弱 FI 连续值 | 多变量线性回归 | 同上 |
| 衰弱 FI >= 0.25 | Logistic 回归 | 同上 |
| 临床生化指标 | 多变量线性回归 + FDR | 同上 |

主要结果：

- 全因死亡：OMAA 每升高 1 单位，死亡风险升高约 4-5%。
- 发现队列：HR = 1.05, P = 0.024。
- 验证队列：HR = 1.04, P = 0.046。
- 衰弱：OMAA 与 FI 正相关；二分类 frailty 中 OR 约 1.04-1.05。
- 肾功能：OMAA 越高，eGFR 越低。

### 5.6 慢病增量预测：Model A vs Model B

文章想证明：OMAA 不只是年龄替代物，而是能在传统风险因素之外提供新增信息。

因此构建两个随机森林分类模型：

| 模型 | 输入变量 |
|---|---|
| Model A | 年龄 + 性别 + 种族 + 教育 + 婚姻 + PIR + 饮酒 + 吸烟 + BMI + 睡眠 + 体力活动 + 牙齿数 |
| Model B | Model A + OMAA Score |

评估：

- 在发现队列训练。
- 在验证队列测试。
- 用 ROC/AUC 比较分类能力。
- 用 2,000 次 bootstrap 比较 AUC 差异。

主要结果：

| 疾病 | Model A AUC | Model B AUC | P |
|---|---:|---:|---:|
| Cancer | 0.67 | 0.70 | 0.009 |
| Heart attack | 0.76 | 0.79 | 0.016 |
| Emphysema | 0.84 | 0.86 | 0.085，趋势性 |

### 5.7 饮食和药物贡献分析

文章最后问：OMAA 是不是只是饮食或吃药造成的？

方法：

- 用随机森林回归预测 OMAA。
- 输入一组饮食变量，或一组药物变量。
- 训练集做 10-fold CV，验证集外部测试。
- 用 Spearman 相关评价预测能力。
- 药物模型再用 SHAP 排序药物贡献。

结果：

- 饮食预测 OMAA 的能力很弱，验证集中不显著。
- 药物预测 OMAA 的能力也弱，但可重复。
- clopidogrel、atenolol、lisinopril 等药物在 SHAP 中排名靠前。
- 文章解释更倾向于“用药代表基础疾病/虚弱状态”，不一定是药物直接导致口腔微生物衰老。

## 6. 文章主要结论

1. 口腔微生物会随年龄发生系统性改变，但整体效应不算特别大。
2. 通过 GAM 筛选得到 64 个稳定年龄相关菌属。
3. 用这些菌属可以构建口腔微生物年龄模型。
4. OMAA Score 能预测全因死亡、衰弱、肾功能下降。
5. OMAA Score 加入传统风险因素后，可以提升癌症和心梗风险预测。
6. 饮食和药物只能解释很少一部分 OMAA 变化。
7. 口腔冲洗样本具有非侵入、容易采集、适合大规模筛查的优势。

## 7. 代码仓库结构

本地仓库路径：

```text
Oral-Microbiome-Clock-main/Oral-Microbiome-Clock-main
```

文件结构：

```text
README.md
annotation.xlsx
1. GAM_Screening/
  GAM_Screening.R
  gam_data.xlsx
2. OMAA_Construction/
  OMAA_Score_Construction.R
  external_validation.xlsx
3. OMAA_Disease_Incremental_Value/
  OMAA_Disease_Incremental_Value.R
  merge_data.xlsx
4. OMAA_Predictors_Diet_and_Medication/
  OMAA_Predictors_Diet_and_Medication.R
  contribution_data.xlsx
```

注意：仓库代码是论文主流程的复现脚本，不是从 NHANES 原始文件开始下载、清洗、合并的一键全流程。部分脚本假定你已经有 `data/`、`results/` 或预先加载好的对象。

## 8. 每个脚本用了什么工具、R 包和方法

### 8.1 `1. GAM_Screening/GAM_Screening.R`

作用：筛选年龄相关菌属。

使用 R 包：

| R 包 | 用途 |
|---|---|
| `mgcv` | 拟合 GAM 模型 |
| `dplyr` | 数据筛选、合并、整理 |
| `openxlsx` | 读取/写入 xlsx |
| `tidyverse` | 管道和数据整理 |

输入数据：

脚本中写的是：

```r
df_09 <- read.xlsx("data/dat_09.xlsx")
df_11 <- read.xlsx("data/dat_11.xlsx")
```

仓库实际给出的示例表是：

```text
1. GAM_Screening/gam_data.xlsx
```

该表结构：

- 2030 行。
- 109 列。
- 主要字段：`SEQN`、大量 `RSV_genus...`、`RIDAGEYR`、`year`、`gender`、`race`。
- `RSV_genus...` 是 CLR 转换后的菌属特征。

方法：

1. 把所有 genus 列作为候选特征。
2. 对每个 genus 拟合：

```r
Genus ~ s(RIDAGEYR) + gender + race
```

3. 提取年龄平滑项 `s(RIDAGEYR)` 的 P 值和 EDF。
4. 发现队列做 BH-FDR 校正。
5. 保留发现队列 FDR < 0.05 且验证队列 P < 0.05 的 genus。
6. 导出显著菌属和过滤后的特征矩阵。

输出：

```text
results/Significant_Genera_Stats.xlsx
results/df_09_filtered_features.xlsx
results/df_11_filtered_features.xlsx
```

可借鉴点：

- 如果你要做“检验指标年龄时钟”，可以把每个检验指标替换成这里的 genus。
- 用 GAM 先筛掉与年龄无关或关系不稳定的指标。
- EDF 可以帮助识别线性和非线性年龄轨迹。

举例：

```text
ALT ~ s(age) + sex
creatinine ~ s(age) + sex
lymphocyte_pct ~ s(age) + sex
```

如果某个指标在训练集 FDR 显著，在验证集也显著，就可以作为候选年龄特征。

### 8.2 `2. OMAA_Construction/OMAA_Score_Construction.R`

作用：训练口腔微生物年龄模型，并做内部和外部验证。

使用 R 包：

| R 包 | 用途 |
|---|---|
| `tidyverse` | 数据处理 |
| `randomForest` | 训练随机森林回归模型 |
| `caret` | 生成交叉验证 folds |
| `openxlsx` | 读写 Excel |

输入数据：

脚本读取：

```r
discovery_df <- read.xlsx("results/df_09_filtered_features.xlsx")
validation_df_1 <- read.xlsx("results/df_11_filtered_features.xlsx")
ext_val_2_raw <- read.xlsx("data/external_validation.xlsx")
```

仓库中提供了：

```text
2. OMAA_Construction/external_validation.xlsx
```

该外部验证表结构：

- 2439 行。
- 39 列。
- 字段：`ID`、`age`、37 个 `RSV_genus...` 重叠菌属。

方法：

1. 在 discovery set 中构建 X 和 y。
2. 用 `caret::createFolds` 做 10-fold CV。
3. 每一折训练 `randomForest`：

```r
randomForest(
  x = X_train,
  y = y_train,
  ntree = 500,
  mtry = floor(sqrt(n_features)),
  importance = TRUE
)
```

4. 保存 discovery 内部交叉验证预测结果。
5. 用完整 discovery set 训练最终模型。
6. 在 NHANES 2011-2012 验证集预测。
7. 对外部验证集，只取 discovery 和外部数据共有的 genus，重新训练共同特征模型。
8. 输出 MAE。

输出：

```text
results/OMAA_Discovery_CV_Results.xlsx
results/OMAA_Validation_1_Results.xlsx
results/OMAA_External_Validation_2_Results.xlsx
```

代码注意点：

- 脚本注释说输入包含 `age`，但第一个脚本输出字段是 `RIDAGEYR`。实际跑时需要统一年龄列名。
- 脚本没有在这里显式计算 `aa`/OMAA Score，只输出预测年龄结果；后续 `merge_data.xlsx` 中已经包含 `oral_age` 和 `aa`。
- README 提到 SHAP，但这个脚本本身没有写 `fastshap` 计算 SHAP 的代码。

可借鉴点：

- 先用训练集交叉验证拿到 out-of-fold predicted age。
- 再在独立验证集测试模型。
- 年龄加速分数必须用没有数据泄露的预测年龄来算。

举例：

在医院检验数据里可以这样映射：

```text
输入 X：血常规 + 肝肾功能 + 糖脂代谢 + 炎症指标
目标 y：真实年龄
模型：randomForest / XGBoost / Elastic Net
输出：predicted biological age
年龄加速：predicted biological age - chronological age
```

### 8.3 `3. OMAA_Disease_Incremental_Value/OMAA_Disease_Incremental_Value.R`

作用：评估 OMAA Score 对慢病风险预测有没有增量价值。

使用 R 包：

| R 包 | 用途 |
|---|---|
| `caret` | 训练随机森林分类模型、交叉验证 |
| `pROC` | ROC、AUC、AUC 差异检验 |
| `dplyr` | 数据处理 |
| `randomForest` | caret 后端随机森林 |
| `openxlsx` | 导出 xlsx |
| `tidyverse` | 数据整理 |

输入数据：

仓库提供：

```text
3. OMAA_Disease_Incremental_Value/merge_data.xlsx
```

表结构：

- 2030 行。
- 85 列。
- 关键字段：`SEQN`、`age`、`oral_age`、`aa`、`gender`、`race`、`BMI`、`tooth_count`、多种生化指标、疾病变量。

脚本中直接使用了 `merge_09` 和 `merge_11`：

```r
data_train <- merge_09 %>% ...
data_val <- merge_11 %>% ...
```

也就是说，脚本假定你已经提前把发现队列和验证队列对象加载到 R 环境中；仓库没有在脚本里写 `merge_09 <- read.xlsx(...)` 这一步。

疾病结局：

```text
Psoriasis, Arthritis, Gout, Congestive_heart_failure,
Coronary_heart_disease, Angina, Heart_disease, Stroke,
Emphysema, Liver, Chronic_bronchitis, Cancer, thyroid,
diabetes, hpd
```

协变量：

```text
age, gender, race, educational_level, marriage_status,
PIR, drinking_status, smoking_status, sleep, BMI, met, tooth_count
```

方法：

1. 对每个疾病建两个模型。
2. Model A：传统风险因素。
3. Model B：传统风险因素 + `aa`，其中 `aa` 是 OMAA Score。
4. 使用 `caret::train(method = "rf")` 训练随机森林分类模型。
5. 训练控制为 repeated 10-fold CV，重复 5 次。
6. 在验证队列预测疾病概率。
7. 用 `pROC::roc` 计算 AUC。
8. 用 `pROC::roc.test(method = "bootstrap", boot.n = 2000)` 比较 AUC 差异。

输出：

```text
results/Validation_AUC_Comparison.xlsx
```

可借鉴点：

这是评估“生物学年龄分数是否有临床增量价值”的标准做法。

举例：

如果你用血液检验数据训练出一个 `BA_acceleration`，可以比较：

```text
Model A：年龄 + 性别 + BMI + 吸烟 + 饮酒
Model B：年龄 + 性别 + BMI + 吸烟 + 饮酒 + BA_acceleration
```

如果 Model B 的 AUC 明显提高，说明你的生物学年龄分数不是“年龄的重复表达”，而是提供了额外风险信息。

### 8.4 `4. OMAA_Predictors_Diet_and_Medication/OMAA_Predictors_Diet_and_Medication.R`

作用：评估饮食和药物能不能解释 OMAA Score。

使用 R 包：

| R 包 | 用途 |
|---|---|
| `randomForest` | 随机森林模型 |
| `caret` | 交叉验证训练 |
| `fastshap` | SHAP 解释 |
| `openxlsx` | 读写 xlsx |
| `tidyverse` | 数据整理 |

输入数据：

脚本写的是：

```r
df_09 <- read.xlsx("data/contribution_09.xlsx")
df_11 <- read.xlsx("data/contribution_11.xlsx")
```

仓库提供的示例表是：

```text
4. OMAA_Predictors_Diet_and_Medication/contribution_data.xlsx
```

表结构：

- 2019 行。
- 79 列。
- 关键字段：`SEQN`、`age`、`oral_age`、`aa`。
- 药物字段：如 `CARVEDILOL`、`AMLODIPINE`、`METFORMIN`、`CLOPIDOGREL` 等，0/1 编码。
- 饮食字段：以 `DR1T` 开头，如水果、蔬菜、谷物、蛋白、乳制品、油脂、添加糖、酒精饮品等。

方法：

1. 用 `grep("^DR1T", colnames(df_09))` 找饮食变量。
2. 其他非 `SEQN/aa/diet_vars` 的变量作为药物变量。
3. 饮食变量训练随机森林回归预测 `aa`。
4. 药物变量训练随机森林回归预测 `aa`。
5. discovery set 中做 10-fold CV。
6. validation set 中做外部预测。
7. 用 Spearman 相关评价预测能力。
8. 对药物模型用 `fastshap::explain` 计算 SHAP。
9. 计算每个药物的 mean absolute SHAP 作为全局重要性。

输出：

```text
results/OMAA_Contribution_Performance.xlsx
results/Medication_SHAP_Importance.xlsx
```

可借鉴点：

这个脚本不是为了提高年龄模型性能，而是为了回答“分数到底被什么外部因素驱动”。

举例：

如果你做血液生物学年龄模型，可以用类似思路评估：

```text
输入：药物使用、慢病史、生活方式、饮食、睡眠
目标：BA_acceleration
问题：这些因素能解释多少 BA_acceleration？
```

如果生活方式模型预测力很弱，而疾病模型预测力强，说明分数可能更接近疾病负担或系统性衰老信号。

## 9. 代码和论文之间的对应关系

| 论文分析模块 | 仓库脚本是否覆盖 | 对应脚本 |
|---|---|---|
| 16S 原始数据处理、DADA2、QIIME、SILVA 注释 | 未覆盖 | 论文 Methods |
| genus 过滤、CLR 转换 | 脚本使用处理后结果，未完整覆盖 | `gam_data.xlsx` 等 |
| alpha/beta diversity、PCoA、PERMANOVA | 未覆盖 | 论文 Results/Methods |
| GAM 筛选年龄相关菌属 | 覆盖 | `GAM_Screening.R` |
| 随机森林年龄预测 | 覆盖 | `OMAA_Score_Construction.R` |
| OMAA Score 计算 | 脚本输出预测年龄，后续表已含 `aa` | `merge_data.xlsx` |
| SHAP 解释年龄模型 | README/论文提到，脚本未完整给出 | 可用 `fastshap` 补 |
| 死亡 Cox、衰弱回归 | 论文有，仓库脚本未覆盖 | 论文 Methods |
| 慢病增量 AUC | 覆盖 | `OMAA_Disease_Incremental_Value.R` |
| 饮食/药物解释 OMAA | 覆盖 | `OMAA_Predictors_Diet_and_Medication.R` |

## 10. 可以怎么借鉴到你的生物学年龄项目

### 10.1 可以直接借鉴的核心框架

最值得借鉴的是这条链：

```text
数据清洗
-> 年龄相关特征筛选
-> 年龄预测模型
-> 年龄加速残差
-> 临床结局/疾病风险验证
-> SHAP 解释
-> 外部因素贡献分析
```

这条链适合从口腔微生物迁移到医院检验数据。

### 10.2 如果换成医院检验数据，可以这样做

| 文章中的变量 | 你的项目可替换为 |
|---|---|
| genus CLR abundance | 血常规、肝肾功能、糖脂代谢、炎症指标等标准化检验值 |
| 16S 预处理 | 单位统一、异常值处理、同日聚合、缺失值处理 |
| GAM 筛年龄相关菌属 | GAM 筛年龄相关检验指标 |
| Random forest oral age | XGBoost/Random Forest/Elastic Net 生物学年龄 |
| OMAA Score | 检验生物学年龄加速分数 |
| SHAP genus importance | SHAP 检验指标贡献 |
| 慢病增量 AUC | 疾病风险/住院风险/体检异常风险增量预测 |

举例：

```text
真实年龄 = 55 岁
模型预测检验生物学年龄 = 63 岁
BA acceleration = +8 岁

报告解释：
你的检验指标组合显示出比同龄人更高的生物学年龄，主要由 eGFR 偏低、HbA1c 偏高、CRP 偏高和 HDL-C 偏低推动。
```

### 10.3 建议你借鉴但要改进的地方

1. **年龄加速最好用年龄残差校正**  
   不只做 `predicted_age - age`，建议做：

```text
predicted_age ~ chronological_age
residual = age acceleration
```

这样能更好地避免分数仍然强烈受真实年龄影响。

2. **模型可以从随机森林换成 XGBoost/Elastic Net 对比**

文章用随机森林，优点是稳健、易解释、适合非线性；但你做医院检验数据时可以同时比较：

| 模型 | 适合场景 |
|---|---|
| Elastic Net | 线性、特征较多、需要稳定系数 |
| Random Forest | 非线性、交互关系、基线模型 |
| XGBoost | 表格数据强基线，适合 SHAP 解释 |

3. **一定要保留独立验证**

文章的可信度来自 discovery-validation 设计。你的项目也应避免只在同一批数据内交叉验证。

最低配置：

```text
训练集：70%
内部验证：10-fold CV
测试集：30%
如果有不同月份/不同医院数据，再做外部验证
```

4. **不仅看预测年龄，还要看健康结局**

文章最值得学的地方是：它没有只说“我能预测年龄”，而是继续证明 OMAA 能关联死亡、衰弱、肾功能、慢病。

你的项目也应加一个验证层：

```text
生物学年龄加速
-> 慢病诊断
-> 肾功能异常
-> 肝功能异常
-> 炎症异常
-> 未来复查恶化
```

### 10.4 最适合形成产品报告的表达

这篇文章的报告表达可以借鉴为：

```text
1. 你的口腔/检验生物学年龄是多少
2. 相对真实年龄偏老还是偏年轻
3. 主要由哪些指标推动
4. 与哪些健康风险相关
5. 哪些外部因素可能影响这个分数
```

对应到你的小程序或报告，可以写成：

```text
综合生物学年龄：58 岁
真实年龄：50 岁
年龄加速：+8 岁
主要贡献：肾功能、糖代谢、炎症免疫、血脂
风险提示：当前模式与同龄人相比提示代谢和肾功能压力偏高
```

## 11. 复现实操提醒

如果你要直接跑仓库代码，需要注意：

1. 代码中的路径是相对路径，需要在对应脚本目录或项目根目录下整理 `data/` 和 `results/`。
2. `GAM_Screening.R` 里读取的是 `data/dat_09.xlsx`、`data/dat_11.xlsx`，但仓库示例给的是 `gam_data.xlsx`。
3. `OMAA_Score_Construction.R` 里使用 `age` 列，但第一步脚本中年龄字段叫 `RIDAGEYR`，需要统一列名。
4. `OMAA_Disease_Incremental_Value.R` 使用 `merge_09`、`merge_11`，脚本没有写读取步骤，需要你自己读入并拆分。
5. README 提到的 SHAP 年龄模型解释，在当前脚本里不完整；药物贡献脚本里有 `fastshap` 示例，可以照着补到年龄模型上。
6. 仓库提供的是处理后表和分析脚本，不包含从 NHANES 原始 XPT 文件到最终分析表的完整 ETL。

## 12. 最终可借鉴模板

你可以把这篇文章的方法压缩成一个自己的方案模板：

```text
Step 1 数据治理：
统一样本、年龄、性别、检验指标、疾病结局，处理缺失和异常值。

Step 2 年龄相关特征筛选：
每个指标建 GAM：indicator ~ s(age) + sex + covariates。
保留训练集 FDR 显著、验证集可重复的指标。

Step 3 生物学年龄建模：
用筛选指标训练 XGBoost/Random Forest/Elastic Net 预测真实年龄。

Step 4 年龄加速分数：
用预测年龄对真实年龄回归，取残差作为 BA acceleration。

Step 5 可解释分析：
用 SHAP 给出推动偏老/偏年轻的指标。

Step 6 临床验证：
检验 BA acceleration 是否关联慢病、肾功能、炎症、代谢异常或未来风险。

Step 7 增量价值：
比较传统模型 A 与加入 BA acceleration 的模型 B，使用 AUC、Delta AUC 和 bootstrap P 值。
```

最简例子：

```text
Model A = age + sex + BMI + smoking
Model B = age + sex + BMI + smoking + BA_acceleration

如果 Model B 的 AUC 从 0.70 提高到 0.75，且 bootstrap P < 0.05，
就可以说 BA_acceleration 在传统风险因素之外提供了额外预测价值。
```

## 13. 一句话评价

这篇文章真正值得学习的不是随机森林本身，而是它的研究闭环：**年龄相关特征筛选 -> 生物学年龄预测 -> 残差定义年龄加速 -> 多结局验证 -> 增量预测价值 -> 外部因素解释**。这套框架可以直接迁移到医院检验数据、生化指标时钟或多模块生物学年龄报告中。
