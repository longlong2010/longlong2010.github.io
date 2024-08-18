---
title: 使用PyTorch进行数据拟合
layout: post
---
>  对于较为复杂的曲线进行拟合，除了传统的使用多项式、分段线性等拟合方式，使用神经网络进行拟合也称为越来越流行的一种方法，特别是输入变量为多维，并且在拟合数据样本量较大的情况，使用神经网络会让拟合的过程更为简单。
>
> 目前最主流的神经网络框架已经由Tensorflow转向了PyTorch，相比Tensorflow，PyTorch更为轻量化，其类似Numpy提供了多维数组（张量）的相关操作，并提供了一套基于自动微分的优化框架，而训练神经网络的过程本质上也是一个优化问题，相对其他框架提供了更为通用的功能，同时目前Keras也提供了PyTorch的版本，进行代码的迁移应该也会比较简单。

#### 1. 拟合基本流程
> 使用PyTorch进行拟合的流程与使用Keras非常类似，只是优化过程的循环需要自行编写，其主要代码如下：
>
```python
model = torch.nn.Sequential();
#创建第一层网络和激活函数，网络的输入为模型输入参数的维度
model.append(torch.nn.Linear(6, 200));
model.append(torch.nn.Sigmoid());
#创建第二层网络和激活函数
model.append(torch.nn.Linear(200, 200));
model.append(torch.nn.ReLU());
#创建第三层网络和激活函数，网络的输出为模型输出参数的维度
model.append(torch.nn.Linear(200, 1));
model.append(torch.nn.ReLU());
>
#设置训练模型的优化器和损失函数
optimizer = torch.optim.Adam(model.paramenters(), lr=learn_rate);
loss_fn = torch.nn.MSELoss();
>
for i in range(0, epoch):
    #进行预测，并计算误差
    y_p = model(x);
    loss = loss_fn(y_p, y);
    #计算梯度，并进行优化
    optimizer.zero_grad();
    loss.backward();
    optimizer.step();
```
> 可见除了需要给出神经网络的构造外，其余过程和一般的优化问题是完全一样的。

#### 2. 预测基本流程

> 预测的过程和训练中调用模型进行预测是完全一样的，但需要注意传入的参数应该与训练时的使用的结构完全相同的tensor，PyTorch和Keras一样可以保存训练好的模型，并从文件中读取模型以便直接使用，其主要代码如下：
```python
model = torch.load('Model.pt');
y = model(x);
```
> 需要注意的是无论模型返回的结果有几个维度，返回值都将是一个tensor，为此在使用的时候需要进行相关的转换，特别是返回结果是一个值的时候。

#### 3. Python读写Excel表格

> 在进行模型训练时可能会遇到训练集是Excel文件的情况，同时也会遇到需要将预测的结果回写到Excel文件的情况，目前大多数Python的Excel库都支持能支持从Excel文件中读取数据，经过测试读写操作均比较好用的是xlwinds，但xlwinds要求必须在Windows环境并且安装了Excel的情况下使用，其工作流程就是调用了Excel API直接进行文件的操作，特别是在将visible设为True时，甚至可以看到Excel文件的操作过程。
>
> 其读写操作的代码如下：
```python
app = xw.App(visibl=True, add_book=False);
#打开一个已经存在的Excel文件
book app.books.open('Data.xlsx');
#获取第一个表格
sheet = book.sheets[0];
#获取表格B2及其右下方区域的全部数据
data = sheet.range('B2').expand().value;
>
#更新i行，j列的数据
sheet[i, j].value = c;
#对结果进行另存为，并关闭文件
book.save('Data-New.xlsx');
book.close();
```

#### 4. 拟合实例

> 对于一电池的电量进行随使用次数及每次的充放电时间进行拟合，其拟合结果曲线如所示： 
<img src="//raw.githubusercontent.com/longlong2010/image.longlong2010.github.io/master/prediction.png" width="600"/>
>
> 可见拟合效果还是不错的，并且在数据有较强的波动的情况下也并不存在明显的过拟合现象。
