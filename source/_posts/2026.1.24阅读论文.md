---
title: 2026.1.24研读论文

date: 2026-1-25
tags: 
  - 文章阅读
  - 美赛
categories: 
  - 文章阅读 
对应赛题: 2023 ICM D、2023 ICM C
keywords: 
  - ICM
  - 美赛O奖论文分析
top_img: /image/16.jpg

---

# 论文1：《Revitalizing a Classic Game: Uncovering the Secrets ofWordle through Data Analysis》

通读本文，发现以下几点特色值得学习：

## 一、核心亮点：构建“数据科学”研究的清晰叙事线

这篇论文成功地将一个开放性数据分析问题，组织成了一个经典的研究流程，其叙事结构堪称范本：

**数据获取与清洗 → 探索性数据分析（EDA）与假设建立 → 构建预测模型 → 构建分类模型 → 深入挖掘有趣特征 → 总结与致信**

这个做法和2025D题论文《A Multi-Methodological Framework for Post-Disaster Traffic Optimization and Sustainable Mobility Planning in Baltimore》趋同，但是增加了构建预测模型和深入挖掘有趣特征

## 📈 与之前D题论文的对比与融合思考

|维度|本篇C题（Wordle）|2025D题（交通/水文）|**对你的启示**|
| ------| ------------------------------------| ----------------------------------------| --------------------------------------------------------------------------------------------|
|**问题本质**|**数据科学问题**：从数据中挖掘模式、预测未知。|**复杂系统问题**：对系统进行建模、仿真与优化。|识别赛题类型。C题思维重在**发现相关性、预测准确性**；D题思维重在**理清因果关系、寻求最优平衡**。|
|**模型核心**|以**机器学习算法**（GRU, RF, K-Means）为驱动引擎。|以**机理模型**（网络流、均衡理论）为基础框架。|掌握不同“武器”的适用场景。论文手要能清晰阐述：我们为什么用“这个”模型，而不是“那个”。|
|**叙事重心**| **“发现-预测”之旅**：着重展示从数据中提取了哪些洞见。| **“诊断-优化”之旅**：着重展示对复杂系统的理解与改进方案。|根据问题类型，调整叙事节奏。数据分析题要突出**洞察**；系统优化题要突出**逻辑**。|
|**共同精髓**|**极强的结构感、清晰的指标定义、多层次的结果验证、对数据/假设的透明处理**。||**这些是无论何种题型，作为论文手都必须锤炼的通用核心技能**。|

**学习点**：无论题目多开放，你都需要为论文设计一个**步步为营、逻辑递进**的故事框架。这个故事框架应让读者（评委）感觉你是在系统地“探索问题”，而非零散地“回答问题”。

‍

## 二、可视化层面可以学习

### 1. **与叙事结构紧密耦合，图随文走**

论文的可视化完美遵循了其“探索性数据分析（EDA）→ 预测 → 分类 → 深入洞察”的研究流程。每一阶段的图表都服务于该阶段的核心结论。作为论文手，应**为论文的每个关键结论或步骤，提前规划至少一张核心图表**。图表不是装饰，而是论据的直观呈现。

**典型示例**： **（GRU预测结果）**  是这种做法的巅峰。它将**实际值、预测值、时间序列**以及至关重要的  **“偏差”（Deviation）曲线** 整合在同一图中。右侧坐标轴的“偏差”曲线，直观揭示了模型在圣诞节（2022-12-25）出现巨大预测误差（近50%）的特殊情况，并引导正文对此进行解释（节假日效应）。

![image](assets/image-20260124224503-azgwab9.png)

​**学习点**​：不要满足于绘制原始数据。尝试在图中​**增加辅助线、阴影区间、异常点标注、次级坐标轴**，以揭示数据背后的模式和故事。

### 2. **善用对比，直观揭示差异与关系**

论文大量使用对比性可视化来支撑其统计结论，让读者“一眼看懂”。

（1）**箱线图对比**：分别用于比较“有无字母重复”和“不同词性”对 `score` 的影响。通过并置箱线图，中位数差异、数据分布范围的区别一目了然，有力支撑了“词性无显著影响”的结论。

![image](assets/image-20260124224735-7btqmrw.png)

（2）**双散点图对比**：并排展示了“词频”和“字母频率”与 `score` 的回归关系。相同的坐标轴尺度让读者能直观比较两个相关系数（-0.32 vs -0.42）的强弱。

![image](assets/image-20260124224751-js818xm.png)

**学习点**：当需要证明“A与B有关”或“X组与Y组不同”时，**并排的对比图表可以是有说服力的武器**。

### 4、**定性资料与定量图表结合，解释力倍增**

论文在分析一个极端案例时，采用了创新的可视化方法：在分析为何“PARER”是地狱级难度单词时，论文不仅给出了其极高的难度系数（0.98）和X百分比（48%）等数据，还直接截取了Twitter用户的评论截图作例图。用户评论指出“存在许多更高频的相似单词（如PAPER, PARED, PARES）”，这为数据异常提供了生动、可信的现实解释。

![image](assets/image-20260124224937-3566gh4.png)

这样的可视化也在2025D同样出现过，其中他直接使用地图进行标记，也是十分直观

​**学习点**​：当数据出现无法用常规模型解释的“故事点”时，可以尝试引入​**定性证据（如文本、图片）作为辅助可视化**。这展示了你们深入思考、联系实际的能力，是论文的亮点。

# 论文2：《Methods of Measuring Priority via Graph Models: Who Is theTop 1?》2024D

通读本文，发现其核心特色在于构建了一个​**层次分明、逻辑自洽、从静态分析到动态推演的复杂系统模型体系**。其叙事结构是典型的问题导向型建模流程：

**问题拆解 → 数据与假设 → 静态关系建模（图模型） → 优先级决策建模（综合评价） → 动态发展建模（时序仿真） → 敏感性分析 → 应用与建议**

## 一、核心亮点分析：如何构建并讲述一个“复杂系统”的故事

这篇论文最出色的地方在于，它的三个核心模型不是孤立的，而是**层层递进、环环相扣**的，形成了一个完整的研究闭环。

**关系图模型**（**认知**）：识别目标间关系

**优先级模型**（**决策**）：确定优先顺序

**时间香槟塔模型**（**推演**预测）：模拟资源分配与发展

**学习点**：对于涉及多因素、动态性的赛题，**设计一个“认知-决策-推演”的模型框架是强有力的叙事策略**。它向评委展示了你们不仅会分析现状，还能做出规划，并预测未来，体现了思维的深度和系统性。

## 二、**结果解释与建议**

**不只展示数据**：对每个重要结果都进行解释（如“目标3和4总在前5，说明健康和教育是各国共性重点”）**阐明“为什么”** 

**提出具体建议**：对联合国：调整目标7标准，加强国际合作。对国家：给予更多自主选择权。对企业：根据规模采用不同策略

### 结果解释的原文例证

#### 1.**第一层：描述性解释**

> “We find that Goal 3 and Goal 4 are in the top5 lists of all countries.”  
> （我们发现目标3和目标4在所有国家的top5列表中。）

*这句话纯粹陈述了一个从模型中得出的客观事实。*

#### 2. **第二层：逻辑性解释（连接现实）**

> “Goal 3 is about health, while goal 4 is about education. In reality, health and education are great concerns of all countries. They are closely related to the economic and social development of a country. They have important impacts on the welfare of the people and the long-term development of the country.”  
> （目标3是关于健康的，目标4是关于教育的。现实中，健康和教育是所有国家高度关注的议题。它们与一个国家的经济和社会发展紧密相关，对人民福祉和国家的长远发展具有重要影响。）

*这句话完美地将数学结果（排名高）与普遍认知（健康和教育重要）和社会经济学原理（它们是发展的基石）联系起来，回答了“为什么这个结果合理”。*

> “Goal 7 is ‘affordable and clean energy’. The cost of clean energy is much higher than fossil energy. And it needs a lot of investment for technological improvement. It also occupies a lot of human and material resources. It makes sense that goal 7 and other goals are trade-offs.”  
> （目标7是“负担得起的清洁能源”。清洁能源的成本远高于化石能源，并且需要大量投资进行技术改进。它还占据大量的人力和物力资源。因此，目标7与其他目标是权衡关系是有道理的。）

*这句话解释了模型中“负相关”结果背后的现实资源竞争逻辑。*

#### 3. **第三层：发现性解释（提炼新规律）**

> “Besides, priorities of goals differ from country to country.”  
> （此外，目标的优先级因国家而异。）

*这是一个从三国数据对比中提炼出的、超越具体数字的普遍性发现。它直接导向了一个重要的政策建议。*

#### 4. **第四层：机制性解释（动态过程）**

> “From 2025 to 2026, Goal 3 is achieved, and the remaining resources are allocated to Goal 9. Hence, we can notice a huge leap of Goal 9 between 2025 and 2026... Afterwards, the structure of network in Temporal Champagne Tower Model is changed. Goals which are positively correlated to Goal 9 increase faster. This impact comes from the achievement of Goal 3.”  
> （从2025年到2026年，目标3完成，剩余资源被分配给目标9。因此，我们可以注意到目标9在2025到2026年间有一个巨大的跃升……之后，时间香槟塔模型中的网络结构改变了。与目标9正相关的目标增长更快。这一影响来源于目标3的完成。）

*这清晰地阐述了动态模型中，一个事件（目标完成）如何通过模型内置的机制（资源溢出、网络关联）引发连锁反应，就像讲述一个故事。*

### 政策建议的原文例证

#### 1. **针对性建议（直接回应发现）**

> “We suggest that the UN should lower the standard of goal 7 for countries like Indonesia, whose synthetic national power is not strong. We also recommend these countries to put goal 7 in low priority. Moreover, it is a good idea to have some developed countries help these countries develop the technologies of affordable and clean energy.”  
> （我们建议联合国应为像印尼这样综合国力不强的国家降低目标7的标准。我们还建议这些国家将目标7置于低优先级。此外，让一些发达国家帮助这些国家发展可负担的清洁能源技术是个好主意。）

*这三条建议直接、具体、分层，完全源于“目标7是资源黑洞，与发展中国家其他目标冲突”这一模型发现。*

#### 2. **战略性建议（提升到新理念）**

> “Therefore, we propose Goal 18: International Cooperation. Help other countries achieve their goals with the correlation benefits of complete goals. For example, since Goal 3 is achieved, we emphasize international cooperation on health.”  
> （因此，我们提出目标18：国际合作。利用已达成目标的关联效益帮助其他国家实现其目标。例如，既然目标3（健康）已实现，我们应强调在健康领域的国际合作。）

*这是从模型机制（协同效益）中孕育出的创造性战略构想，极具洞察力和高度。*

#### 3. **差异化建议（区分受众）**

**对联合国：**

> “We suggest the United Nation organize some cooperation plans on health and education.”  
> “We suggest the United Nation offer more free choice to countries.”  
> “We would appreciate it if the United Nation could recommend our model to other countries.”

**对成员国（印尼等）：**

> “We also recommend these countries to put goal 7 in low priority.”  
> “We think it a good idea to enable the country to decide priorities on their own.”

**对企业/组织**：

> “7.3 Model Impact on Companies and Organizations”  
> *该节专门分析了模型对不同规模组织的应用方式。*

#### 4. **量化支撑的建议（用数据说话）**

> “By 2033, Indonesia can achieve 82 percent of SDGs, according to our model. However, it can achieve 78 percent of SDGs, according to its current plan. Therefore, our model can enable Indonesia to achieve 4 more percent of SDGs, which shows the superiority of our model.”  
> （根据我们的模型，到2033年，印尼可以实现82%的SDGs。然而，根据其现有计划，它只能实现78%。因此，我们的模型能使印尼多实现4%的SDGs，这显示了我们模型的优越性。）

*用自己模型的量化优势，证明其值得被采纳*

‍

## 三、**学术规范与研究深度**

### **文献引用恰当**

引用联合国官方文件、Nature等重要期刊

在引言中梳理已有研究，明确自己的创新点

#### 1. **在引言中构建“学术坐标系”**

> 原文引用 [4] David Le Blanc, [5] Ranjula Bali Swain, [6] Pradhan P 等人的工作。
>
> ​**确立问题合法性**：表明“研究SDG间关系”是一个受到学界关注的真问题。
>
> ​**梳理方法脉络**：指出现有研究主要使用“关联网络”和“相关性分析”方法（Spearman相关）。
>
> ​**定位自身创新**：在承认前人使用相关性分析效果良好的基础上，为后文自己更复杂的模型（结合图中心性、TOPSIS、动态模拟）埋下伏笔。潜台词是：“我们站在巨人的肩膀上，走得更远。”

#### 2. **在方法部分提供“权威依据”**

> ​**原文（遍布各方法小节）** ：
>
> ​**数据标准化与相关性**：引用 [10] Hauke J, Kossowski T 来论证选择Spearman相关而非Pearson相关的理由（“better at capturing non-linear correlation and is less sensible to outliers”）。
>
> ​**图中心性度量**：分别为三种中心性引用其经典文献（[11] Freeman L, [12] Sabidussi G, [13] Bonacich P）。
>
> ​**TOPSIS方法**​：引用其原始提出者 [14] Hwang C L 的著作。  
> ​**功能**：
>
> ​**避免方法赘述**：对于成熟方法，无需详细展开公式推导，引用权威文献即可，节省篇幅。
>
> ​**彰显专业性**：表明作者了解所用方法的根源与谱系，不是盲目使用“黑箱”工具。
>
> ​**增强方法可信度**：表明所选方法是学界公认的，而非作者主观臆造。

#### 3. **在模型调整部分引入“外部证据”**

> **原文**“Based on the study of Robin Naidoo[7], we get some adjustment to our model.”  
> 当需要修改模型以应对新冠疫情时，作者没有自己“拍脑袋”设定参数，而是引用了  **《自然》期刊** 上关于疫情如何影响SDGs的权威研究。
>
> **将主观调整客观化**：将模型参数的修改依据从一个“假设”转变为基于顶级学术研究的“推论”，使整个调整过程无可挑剔，极大地增强了模型扩展部分的说服力。

#### 4. **在背景介绍中锚定“核心概念”**

> 引用 [1] 布伦特兰报告，给出“可持续发展”的权威定义。  
> **功能**：为整个研究奠定基石，确保讨论的起点是学界和官方公认的，避免在基本概念上产生歧义。

#### 总结引用技巧的细节分析

**引用与行文的自然融合**

本文在介绍熵权法-TOPSIS子模型时，将引用 [14] 自然地融入对方法本身的描述流程中，使其成为论述的一部分，而不是括号里的孤立标注。

![image](assets/image-20260124235638-btohefg.png)

**引用的精确性与针对性**

不是模糊地引用“某本教科书”，而是​**精确引用到提出该具体指标或方法的原始文献**​（如三种中心性各自有引）。引用 [9] 联合国数据库，明确了​**数据来源**，这是模型可信的生命线

**参考文献列表的规范性**

​**格式统一**：尽管截图中未完整展示，但通常此类优秀论文的参考文献列表格式（作者、标题、期刊、卷期、页码、年份）极为规范。

**类型全面**：包含报告[1]、联合国文件[3]、期刊论文[4,5,6,7]、统计学方法研究[10]、经典算法著作[14]、官方数据库[9]等，显示出信息来源的多样性和可靠性。

![image](assets/image-20260125000122-c213fj2.png)![image](assets/image-20260125000126-08bfjlo.png)

在一篇论文中，可以尝试构建一个类似的功能性引用矩阵：

|引用场景|应寻找的文献类型|功能<br />||
| ----------| ------------------------------| -------------------------------------------------------| ----------------------|
|**定义核心概念**|开创性报告/纲领性文件|确立研究范畴<br />||
|**综述研究现状**|近3-5年的高水平综述/实证研究|展示问题价值，定位创新点<br />||
|**证明数据可信**|官方统计机构/权威数据库|为模型提供坚实基础<br />||
|**论证方法选择**|方法比较类论文/教科书|解释为何选A而非B<br />||
|**支撑关键模型**|算法原始论文/权威解释|避免赘述，展示根基<br />||
|**应对外部变化**|顶级期刊的最新相关研究|将主观调整客观化<br />||

引用部分的话语还有待学习

‍
