

### 1. 思路

首先, 对于每个实体的所有评论, 进行`中文分词`、`词性标注`, 并且做`依存句法分析`。然后, 根据每个句子中的`依存关系`, 抽取关键`标签`, 构成此实体的`标签库`, 并且对标签库进行显式`语义去重`。最后通过 K-Means 聚类以及LDA主题模型将每个标签 映射到语义独立的主题空间,再根据每个标签相对该主题的置信度进行排序。


### 2. 考虑点

怎样评价一个评论的质量

同义词词典　口味儿　味道



### 3. 

产品特征提取：　

- 人工定义
- 自动提取　(词性标注　句法分析　文本模式)　准确率比较低　
- 

### 4. 步骤

#### 4.1 词法句法分析

每个评论的每个句子进行词法和句法分析。将句子分词, 并且进行词性标注, 而且需要将词与词之间的修饰关系描述出来。本文中我们使用`依存句法分析`来确定词之间的关系。例如, 评 论:"服务员很漂亮。饭菜很好吃,味道不错。"

依存句法分析之后, 就可以将有用的三元组
#### 4.2 候选标签的挖掘抽取

#### 4.3 标签去重和语义独立

#### 4.4 代表标签的选取以及排序






















