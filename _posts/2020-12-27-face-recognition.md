---
title: 人脸识别及匹配
layout: post
---
> 基于特征以及使用深度学习进行人脸识别已经是非常成熟的算法，应用广泛的计算机视觉库opencv中就提供了人脸识别的相关算法，能够很好的识别图片或视频中的人脸部分。而实际应用中除了要识别出人脸，还需要对人脸的特征进行比对，如进行身份认证或比对是否为同一人等。如果直接使用计算图片相似度的方法，不仅运行速度较慢，同时由于包含人脸的矩形区域内还会夹杂一些干扰信息，对比的精度也会收到影响。
>
> 这里使用基于深度学习库[dlib](http://dlib.net/)的人脸识别工具face\_recognition，首先识别图片中的人脸，同时提取人脸的特征向量，对比时可以直接对提取到的不同人脸的特征向量进行对比，这里使用经典的余弦相似度算法，即计算两个特征向量夹角的余弦值，其值越接近1则相似度越高。

#### 1. face\_recognition的安装
> [face\_recognition](https://github.com/ageitgey/face_recognition)为python编写，可以直接使用python的pip包管理工具进行安装，其主要依赖cblas（要用cblas而不是openblas）、numpy、matplotlib、dlib，安装过程中会自动编译dlib，过程中需要至少4GB以上的可用内存。
```bash
pip install face_recognition
```

#### 2. 基本使用方法

> 使用load\_image\_file方法可以读入图片文件，该方法返回一个图片的多维数组和opencv中的格式一致。face\_locations方法传入一个图片多维数组，可以返回其中的人脸所在的举行区域，其中矩形区域的顺序为上、右、下、左。使用face\_encodings方法传入一个图片的多维数组，可以返回一个128维的人脸特征向量。
>
```python
import face_recognition;
if __name__ == '__main__':
    im = face_recognition.load_image_file('test.jpg');
    boxes = face_recognition.face_locations(im);
    encodings = face_recognition.face_encodings(im);
```
> 另外face\_distance和compare\_faces可以计算多个已知的人脸特征和一个未知的人脸特征的相似度和对比结果，默认的阈值会取0.6。其中compare\_faces本质就是在调用face\_distance。
>
```python
    im1 = face_recognition.load_image_file('1/l.jpg');
    enc1 = face_recognition.face_encodings(im1)[0];
    im2 = face_recognition.load_image_file('2/1.jpg');
    enc2 = face_recognition.face_encodings(im2)[0];
    #注意这里第一个参数是多个已知人脸的特征向量
    face_recognition.face_distance([enc1], enc2);
    face_recognition.compare_faces([enc1], enc2)
```
#### 3. 进行人脸的匹配
>
> 不清楚为什么该工具默认会使用欧式距离的方法来计算相似度，实际测试下来使用欧式距离计算的相似度波动较大，整体的阈值选择较为困难。
>
> 2个目录中各有6张图片，其中目录1中的图片，分别为蔡徐坤、李宇春、刘翔、马云、姚明、王怀南。目录2中依次为蔡徐坤、马云、王怀南、姚明、李宇春、刘翔。如使用命令行工具进行两个目录之间图片的对比
```bash
face_recognition 1 2 --show-distance=1
>
#输出结果
2/1.jpg,cxk,0.33241630184812043
2/1.jpg,lyc,0.5966439550889049
2/6.jpg,lx,0.4915904587201082
2/4.jpg,lx,0.5309307486036086
2/4.jpg,my,0.5750886946477217
2/4.jpg,ym,0.38126356882352785
2/2.jpg,my,0.34452493549783725
2/3.jpg,lx,0.5901235869708715
2/3.jpg,whn,0.3484571564371876
2/3.jpg,my,0.5870175925341533
2/5.jpg,lyc,0.3480285326046423
```
> 一些图片识别出了多组结果，如果选择距离最小的一个，则可以全部识别正确。但图6的距离有0.49，这种情况有可能该人脸仅仅是和刘翔的相似度最高，而其实并不是其中的任何一个人。
>
> 如果使用余弦相似度，则只用face\_recognition计算得到人脸的特征向量，然后自行计算相似度，其的代码如下
```python
import face_recognition;
import numpy;
from numpy import linalg;
import os;
>
if __name__ == '__main__':
    dir1 = '1/';   
    files1 = os.listdir(dir1);
    size = len(files1);
    N = 128;
    #已知人脸特征向量矩阵
    A = numpy.zeros((size, N));
    i = 0;
    for f in files1:
        im = face_recognition.load_image_file(dir1 + f);
        enc = face_recognition.face_encodings(im)[0];
        #特征向量长度归一化
        enc /= linalg.norm(enc);
        A[i] = enc;
        i += 1;
    dir2 = '2/';
    files2 = os.listdir(dir2);
    for f in files2:
        im = face_recognition.load_image_file(dir2 + f);
        enc = face_recognition.face_encodings(im)[0];
        enc /= linalg.norm(enc);
        #计算相似度 
        r = A @ enc;
        #取相似度最高的一个
        idx = numpy.argmax(r);
        print("%s %s %f" % (f, files1[idx], r[idx]));
```
> 使用同样的例子运行结果如下
```bash
1.jpg cxk.jpg 0.974498
6.jpg lx.jpg 0.935104
4.jpg ym.jpg 0.959824
2.jpg my.jpg 0.968253
3.jpg whn.jpg 0.967666
5.jpg lyc.jpg 0.969720
```
> 同样完全计算正确，同时相似度都达到了较高的0.9以上，相比距离更容易设定阈值。
>
> 在实际使用中已知的人脸特征向量矩阵A可以提前算好并保存，同时也方便进行增量计算，实际一次匹配的过程仅需要进行一次矩阵和向量的乘法即可完成，计算的效率相对还是比较高的。
