---
title: 使用Keras实现文本分类
layout: post
---

> 目前已经广泛使用深度学习的方法进行数据分类，包括文本、图像、声音和视频等。其中Tensorflow是使用最为广泛的一种深度学习框架，其能够实现分布式以及使用GPU提高建立模型的速度，但其缺点是代码流程较为复杂，而Keras实现了对Tensorflow API的高层封装，使其相比Tensorflow更加简单易用，目前Keras已经作为Tensorflow中的一部分与Tensorflow一起发布。

#### 1. 数据来源

> 为了进行模型的训练过程，需要获得一定量的样本数据，本文中的样本数据来自某问答平台，将用户提问标题的文本和问题所属的分类作为学习的训练集。目前Python的pyquery库很好的模拟了jQuery的风格，相比使用正则表达式，其能够更便利的从html文本中提取到所需位置的数据，获取数据的脚本如下
>
```python
k = 0;
#对于每个分类的地址将获取到的问题写入一个文件
for url in urls:
    k += 1;
    fp = open(str(k) + '.txt', 'w');
    #获取该分类下的问题总数
    request = urllib.request.Request(url = url,  headers = headers);
    try:
        res = urllib.request.urlopen(request, timeout=3);
        html = res.read().decode('gbk');
    except Exception as e:
        print(e);
        continue;
    #计算页数
    num = int(jQuery(html).find('span.count-num')[0].text);
    page = num // size;
>
    #按页进行数据获取
    for i in range(0, page):
        request = urllib.request.Request(url = url + '&rn=' + str(size) + '&pn=' + str(i * size),  headers = headers);
        #如果出现错误就放弃当页的数据，防止整个程序退出
        try:
            res = urllib.request.urlopen(request, timeout=3);
            html = res.read().decode('gbk');
        except Exception as e:
            print(e);
            continue;
        #通过pyquery获取保存问题标题的a标签
        for x in jQuery(html).find('a.title-link'):
            fp.write(x.text.strip() + "\n");
        time.sleep(2);
    fp.close();
```

#### 2. 通过数据生成训练模型

> 生成训练模型的思路为使用经典和简单的TF-IDF方法（这里没有使用目前更加流行的word2vec基于卷积神经网络的方法），对每个问题的标题进行分词，并计算出主要词汇的TF-IDF值生成对应每个问题的向量，问题所属的分类则作为该问题的分类值。
>
> 首先需要根据全部的问题生成总的词汇表——每个词汇对应一个唯一的索引值(0~词汇表长度-1)，同时也决定了对于每一个问题对应的向量的长度，即train\_x数组的维度为“问题个数x词汇表的长度”，train\_y数组的维度则为“问题个数x1”，其每一个元素为该个问题的分类索引值。训练过程的代码如下
>
```python
import jieba.analyse;
import numpy;
import pickle;
import tensorflow.keras;
import scipy.sparse;
from tensorflow.keras.layers import Dense, Activation;
if __name__ == '__main__':
    category = 20;
    k = 0;
    j = 0;
    #生成词汇表
    words = dict();
    for i in range(0, category):
        #对于每个文件（该分类）中的问题
        fp = open(str(i + 1) + '.txt');
        for x in fp:
            j += 1;
            #进行TF-IDF分词，IDF使用分词引擎中的数据
            for y in jieba.analyse.extract_tags(x, withWeight=True):
                #如果是新词，则更新词汇表
                if y[0] not in words:
                    words[y[0]] = k;
                    k += 1;
        fp.close();
    #将词汇表保存，以便于预测时使用
    pickle.dump(words, open('words.bin', 'wb'));
    #确定了两个数组的维度，这里使用稀疏矩阵可以节约一定的内存空间
    train_x = scipy.sparse.lil_matrix((j, len(words)));
    train_y = numpy.zeros((j, 1));
    j = 0;
    #将分类索引值和TF-IDF值填入对应的数组中
    for i in range(0, category):
        fp = open(str(i + 1) + '.txt');
        for x in fp:
            #填入分类索引值
            train_y[j] = i;
            for y in jieba.analyse.extract_tags(x, withWeight=True):
                #填入TF-IDF
                train_x[j, words[y[0]]] = y[1];
            j = j + 1;
    #初始化模型
    model = tensorflow.keras.Sequential();
    #设置网络
    model.add(Dense(32, activation='relu', input_dim=len(words)));
    model.add(Dense(category, activation='sigmoid'));
    #设置分类的优化器
    model.compile(optimizer='rmsprop',
        loss='categorical_crossentropy',
        metrics=['accuracy']);
    #处理分类索引值数组
    train_y = tensorflow.keras.utils.to_categorical(train_y, num_classes=category);
    #开始训练
    model.fit(train_x, train_y, epochs=10, batch_size=32);
    #保存训练得到的模型
    model.save('1.h5');
```
> 运行训练过程的输出结果如下，可见10轮训练后的准确率就比较高了
>
```
Epoch 1/10
14921/14921 [==============================] - 4s 289us/sample - loss: 2.9050 - accuracy: 0.2525
Epoch 2/10
14921/14921 [==============================] - 4s 269us/sample - loss: 2.3243 - accuracy: 0.6018
Epoch 3/10
14921/14921 [==============================] - 4s 273us/sample - loss: 1.5976 - accuracy: 0.7354
Epoch 4/10
14921/14921 [==============================] - 4s 275us/sample - loss: 1.1616 - accuracy: 0.7962
Epoch 5/10
14921/14921 [==============================] - 4s 274us/sample - loss: 0.8692 - accuracy: 0.8380
Epoch 6/10
14921/14921 [==============================] - 4s 271us/sample - loss: 0.6639 - accuracy: 0.8674
Epoch 7/10
14921/14921 [==============================] - 4s 272us/sample - loss: 0.5143 - accuracy: 0.8899
Epoch 8/10
14921/14921 [==============================] - 4s 267us/sample - loss: 0.4068 - accuracy: 0.9089
Epoch 9/10
14921/14921 [==============================] - 4s 267us/sample - loss: 0.3283 - accuracy: 0.9220
Epoch 10/10
14921/14921 [==============================] - 4s 267us/sample - loss: 0.2700 - accuracy: 0.9335
```

#### 3. 进行预测

> 训练得到模型后就可以使用模型对新的问题进行分类预测了，预测的脚本如下
>
```python
#读取词库
words = pickle.load(open('words.bin', 'rb'));
#加载模型
model = load_model('1.h5');
#需要预测的文本信息，由命令行参数给出
text = sys.argv[1];
#输入向量的维度为1x词汇表长度
data = numpy.zeros((1, len(words)));
#分词并计算TF-IDF
for x in jieba.analyse.extract_tags(text, withWeight=True):
    #如果词汇表中有此词汇则将TF-IDF填入输入向量中
    if x[0] in words:
        data[0, words[x[0]]] = x[1];
#预测，并取出可能性最高的一个分类的索引值
i = numpy.argmax(model.predict(data));
print(labels[i]);
```
> 尝试了几个问题，得到的分类结果如下，总体感觉似乎还可以。
>
```
健康猫爪子会不会沾染狂犬病毒，请看下面的补充评论谢谢。 -> 生活
平板网络很好，为什么有的软件显示网络不可用？早上还可以！ -> 家电数码
普耐尔250二氧化碳焊机手工焊起弧电流什么凋？ -> 生产制造
中国的地标城市是在那个地方？ -> 旅游
```
