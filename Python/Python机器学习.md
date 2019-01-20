机器学习之数学基础

标量：

向量：

张量：

#### 矩阵

- 奇异矩阵
- 行列式

方差、标准差、协方差

范数，衡量向量或矩阵的大小。

$$x=\frac{-b\pm\sqrt{b^2-4ac}}{2a}$$

:joy_cat:
$$
\sum_{i=1}^n a_i=0
$$
:apple:

机器学习之垃圾短信识别

先将训练数据通过pandas导入进来，通过jieba进行分词分析：

```python
import pandas as pd
import jieba

data = pd.read_csv(r"./data/80w.txt",encoding='utf-8',sep='    ',header=None)
data.head()

# 垃圾短信
garbage = data[data[1] == 1]
garbage[2] = garbage[2].map(lambda x:' '.join(jieba.cut(x)))
garbage.head()
# 正常短信
normal = data[data[1] == 0]
normal[2] = normal[2].map(lambda x:' '.join(jieba.cut(x)))
normal.head()

garbage.to_csv('garbage.csv',encoding='utf-8',header=False,index=False,columns=[2])
normal.to_csv('normal.csv',encoding='utf-8',header=False,index=False,columns=[2])

```

通过自然语言处理库NLTK来进行分类处理。

```python
import nltk.classify.util
from nltk.classify import NaiveBayesClassifier
from nltk.classify import accuracy as nltk_accuracy
from nltk.corpus import PlaintextCorpusReader

# 加载短信消息语料库
message_corpus = PlaintextCorpusReader('./',['garbage.csv','normal.csv'])
all_message = message_corpus.words()

# 特征函数，生成特征
def massage_feature(word,num_letter=1):
    return {'feature':word[-num_letter:]}

# 对短信特征进行标记提取
labels_name = ([(massage,'垃圾') for massage in message_corpus.words('soam.csv')]+[(massage,'正常') for massage in message_corpus.words('normal.csv')])
random.seed(7)
random.shuffle(labels_name)

# 训练并预测模型
featuresets = [(massage_feature(n),massage) for (n,massage) in labels_name]
train_set,test_set = featuresets[2000:],featuresets[:2000]
classifier = NaiveBayesClassifier.train(train_set)
```


