#  通用的KBQA问答系统

**查询效果**

> Query:  加拿大最大城市有哪些机场？
>
> Answer: 多伦多皮尔逊国际机场,比利·毕晓普多伦多市机场

Response:

```python
{
"score": 0.9515,
"query": "加拿大最大城市有哪些机场",
"entity": "加拿大",
"mention": "加拿大",
"intent": "最大城市\t机场",
"answer": "多伦多皮尔逊国际机场,比利·毕晓普多伦多市机场",
"template": "2a",
}
```



**整体结构**

主要包含以下五个组件：**指称识别、实体链接、模版查询、路径排序、结果归一化**

**指称识别**

抽取出query中的人名、地名、机构名、专有名词等实体，通用kbqa中抽取到的指称范围较大，基本大多数通用名次都可能包含在内。所以为了能够不遗漏大多数实体，则需要利用多种抽取方法。例如：中国首都的人口，可以抽取为“中国首都”，“中国”，在图谱中需要的是中国这个实体。这里利用了两种抽取方法：

序列标注模型：bert+crf

词典抽取：jieba的关键词抽取，jieba.analyse.extract_tag ，根据多种名词词性做的关键词抽取

**实体链接**

指称识别可能会抽取出多个实体，此处实体链接主要有两个作用：

1、将实体链接到图谱中的标准实体（粗排）

2、选择出多个实体中的焦点实（精排）

实体链接粗排包括候选实体的生成、同义实体消岐两个部分。首先需要构建候选实体词典，通过百度百科语料，找到词条的别名、中文名、英文名、简称等构建候选实体词典。例如“小巨人”-“姚明”，“马云”-“马云（阿里巴巴创始人）”。

实体链接精排，对于每个候选实体，挖掘多级特征，利用模型选择出topk个实体。

**模版查询**

构建了多种neo4j图谱查询模版，包括一度关系查询和二度关系查询

| 模版代号 | 查询模版                          | 示例                                                         | 子模板                           |
| -------- | --------------------------------- | ------------------------------------------------------------ | -------------------------------- |
| 1a       | 实体a-关系->实体b                 | 1 姚明有多高；2瓦妮莎是科比的什么人？                        | 1 返回实体b ；2 返回关系         |
| 2a       | 实体a-关系r1->实体b-关系r2->实体c | 1发明显微镜的人是什么职业；2 汉初三杰中被封为留侯的是谁？    | 1 返回实体c；2返回实体b          |
| 2d       | 实体a<-关系r1-实体b-关系r2->实体c | 1 拜仁的西班牙球员都有谁？;唐玄宗的出生地是哪里？            | 1 返回实体b；2返回实体a或者实体c |
| 2c       | 实体a-关系r1->实体b<-关系r2-实体c | 1 郭靖和黄蓉都会什么武功？；《三个音乐家》的作者是什么派别的画家？ | 1 返回实体b；返回实体a或者实体c  |

部分query也需要同时中间实体b和结果实体c，例如：《卧虎藏龙》的主演都来自于哪些国家，对应的是模版2a，需要同时返回主演和主演的国籍

在模版2a中，实体b需要链接到图谱中的标准实体

由于部分实体查询结果数量巨大，中国、北京、上海等实体的查询结果数量在几十万以上。不进行处理，会对排序结果和查询效率造成很大影响。对于不同的模版需要执行对应的剪枝策略。对于1a模版，可以限制关系的数量。2度查询可以排除某些特殊的关系或者关系组合。

**路径排序**

基于查询模版得到的众多查询路径进行排序，构建多级特征，利用分类模型或者排序模型选择最优答案。

在数据质量高的情况下可以选择lightgbm作为分类模型，否则选择 lambdamart作为排序模型。特征主要包括：

问题与路径的语义特征、问题与路径的字符特征、多级路径内部的字符特征、问题与答案的类型匹配特征、实体链接概率、查询模版分类特征。

1 问题与路径的语义特征

基于bert训练语义相似度模型，由于二度关系查询路径多大几十万，查询效率低

2 问题与路径的字符特征

计算路径与问题的各种字符相似度，包括：编辑距离、Jaccard距离、difflib（字符相似度模版）等

3 问题与答案的类型匹配特征

利用规则对问题进行分类，利用jieba对答案进行实体抽取分类；主要分为人物、地名、时间、数量等

**结果归一化**

由于图谱的数据来源较多，同一条路径可能查询多多个结果，部分结果表达的是同一个含义，所以需要进行结果归一化。使用相似度计算方法对多个结果进行相似度计算，表达相同含义的保留一个。

例如1：中国的首都？

answer：北京；北京（中国四大直辖市之一）

将上述多个查询结果表述相同的归一化。

例如2：发明显微镜的人是什么职业？

answer：物理学家_（探索、研究物理学的科学家）；发明家；博物学家

表述不同的保留结果

