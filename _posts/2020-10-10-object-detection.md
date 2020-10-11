---
title: 使用Tensorflow检测自定义物体
layout: post
---

> 随着深度学习方法的不断发展，越来越多的问题可以借助深度学习的模型来处理。对于一些物体检测特别是人脸、车辆等的检测，使用OpenCV等图像处理工具已经可以很好的处理，而对于自定义的物体，如有些标记，物体的特征等，需要利用自行训练神经网络来进行处理，儿在Tensorflow中给出了一个进行[物体检测模型](https://github.com/tensorflow/models)，可以用来训练自己的神经网络。关于该模型使用的文章和教程虽然已经比较多，但在使用的过程中还是遇到了不少问题，这里简单总结。

#### 1. Tensorflow版本

> 虽然官方的文档中说可以支持Tensorflow2，但实际使用的过程中发现在运行到训练模型的时候，会因为模块tensorflow.contrib在Tensorflow2中被移除而在使用过程中出现错误，并且暂时没有查到可以兼容的方法，以下的代码让人比较无奈。为此暂时只能使用Tensorflow 1.14版本
>
```python
try:
  from tensorflow.contrib import opt as tf_opt  # pylint: disable=g-import-not-at-top
except:  # pylint: disable=bare-except
  pass
```

#### 2. 手动标记数据

> 为了训练神经网络，需要首先手动标记需要识别的目标的数据，这里使用的工具是[LabelImg](https://github.com/tzutalin/labelImg)。使用该工具打开需要进行标记的图片，将图面中需要进行识别的部分用方框选中，并添加对应的标签，每标记一张图并进行保存后会生成一个xml文件记录对应的标记信息。最后 将标记完成后的图片和对应的xml分别放入train和test两个目录中作为训练集和测试集。

#### 3. 生成训练数据

> 之后需要将得到的数据转化为Tensorflow训练中所用的TFRecord格式，尝试了自带的脚本create\_pascal\_tf\_record.py，但由于其没有详细说明具体使用方法，只是给了一个针对2012 PASCAL VOC数据的转换命令，没有个给出参数的具体含义，这里使用了一般的教程中给出的转换方法，即首先将标记的数据转换为csv文件，然后再将csv文件和图片一起转换为TFRecord格式（TFRecord文件中就已经包含了图片本身的信息）。涉及两个转换的脚本如下
>
```python
#XML转csv
import os
import glob
import pandas as pd
import xml.etree.ElementTree as ET
>
def xml_to_csv(path):
    xml_list = []
    for xml_file in glob.glob(path + '/*.xml'):
        print(xml_file)
        tree = ET.parse(xml_file)
        root = tree.getroot()
        i = 0;
        for member in root.findall('object'):
            value = (root.find('filename').text,
                     int(root.find('size')[0].text),
                     int(root.find('size')[1].text),
                     member[0].text,
                     int(member[4][0].text),
                     int(member[4][1].text),
                     int(member[4][2].text),
                     int(member[4][3].text)
                     )
            xml_list.append(value)
    column_name = ['filename', 'width', 'height',
                   'class', 'xmin', 'ymin', 'xmax', 'ymax']
    xml_df = pd.DataFrame(xml_list, columns=column_name)
    return xml_df
>
def main():
    for directory in ['train', 'test']:
        image_path = os.path.join(os.getcwd(), 'images/{}'.format(directory))
        xml_df = xml_to_csv(image_path)
        print(image_path);
        print('data/{}_labels.csv'.format(directory));
        xml_df.to_csv('data/{}_labels.csv'.format(directory), index=None)
        print('Successfully converted xml to csv.')
>
if __name__ == '__main__':
    main()
```
> 其中需要注意将一个xml文件转为value数组时，需要根据实际xml的情况设定member对应的位置，这里使用的是4。转换的图片文件放在images/train和images/test目录下，运行完成后会在data目录下生成train\_label.csv和test\_label.csv。
>
```python
from __future__ import division
from __future__ import print_function
from __future__ import absolute_import
import os
import io
import pandas as pd
import tensorflow.compat.v1 as tf;
>
from PIL import Image
from object_detection.utils import dataset_util
from collections import namedtuple, OrderedDict
>
flags = tf.app.flags
flags.DEFINE_string('csv_input', '', 'Path to the CSV input')
flags.DEFINE_string('output_path', '', 'Path to output TFRecord')
FLAGS = flags.FLAGS
>
# The index of the first class has to be 1!
# Do not use 0. DO NOT!
def class_text_to_int(row_label):
#这里XXXXXX为标记图片时所使用的标签名称
    if row_label == 'XXXXXX':
        return 1
    else:
        return 0;
>
def split(df, group):
    data = namedtuple('data', ['filename', 'object'])
    gb = df.groupby(group)
    return [data(filename, gb.get_group(x)) for filename, x in zip(gb.groups.keys(), gb.groups)]
>
def create_tf_example(group, path):
    with tf.gfile.GFile(os.path.join(path, '{}'.format(group.filename)), 'rb') as fid:
        encoded_jpg = fid.read()
    encoded_jpg_io = io.BytesIO(encoded_jpg)
    image = Image.open(encoded_jpg_io)
    width, height = image.size
>
    filename = group.filename.encode('utf8')
    image_format = b'jpg'
    xmins = []
    xmaxs = []
    ymins = []
    ymaxs = []
    classes_text = []
    classes = []
>
    for index, row in group.object.iterrows():
        xmins.append(row['xmin'] / width)
        xmaxs.append(row['xmax'] / width)
        ymins.append(row['ymin'] / height)
        ymaxs.append(row['ymax'] / height)
        classes_text.append(row['class'].encode('utf8'))
        classes.append(class_text_to_int(row['class']))
>
    tf_example = tf.train.Example(features=tf.train.Features(feature={
        'image/height': dataset_util.int64_feature(height),
        'image/width': dataset_util.int64_feature(width),
        'image/filename': dataset_util.bytes_feature(filename),
        'image/source_id': dataset_util.bytes_feature(filename),
        'image/encoded': dataset_util.bytes_feature(encoded_jpg),
        'image/format': dataset_util.bytes_feature(image_format),
        'image/object/bbox/xmin': dataset_util.float_list_feature(xmins),
        'image/object/bbox/xmax': dataset_util.float_list_feature(xmaxs),
        'image/object/bbox/ymin': dataset_util.float_list_feature(ymins),
        'image/object/bbox/ymax': dataset_util.float_list_feature(ymaxs),
        'image/object/class/text': dataset_util.bytes_list_feature(classes_text),
        'image/object/class/label': dataset_util.int64_list_feature(classes),
    }))
    return tf_example
>
def main(_):
    writer = tf.python_io.TFRecordWriter(FLAGS.output_path)
    #运行前将YYYYYY改为train或test
    path = os.path.join(os.getcwd(), 'images/YYYYY/')
    examples = pd.read_csv(FLAGS.csv_input)
    grouped = split(examples, 'filename')
    for group in grouped:
        tf_example = create_tf_example(group, path)
        writer.write(tf_example.SerializeToString())
    writer.close()
    output_path = os.path.join(os.getcwd(), FLAGS.output_path)
    print('Successfully created the TFRecords: {}'.format(output_path))
>
if __name__ == '__main__':
    tf.app.run()
```
>
> 使用方法如下，运行后将得到train.record和test.record两个文件
```bash
python generate_tfrecord.py --csv_input=data/train_labels.csv  --output_path=train.record
python generate_tfrecord.py --csv_input=data/test_labels.csv  --output_path=test.record
```

#### 4. 定义模型参数

> 模型参数可以参考ssd\_mobilenet\_v2\_coco.config，并对其中一些部分做修改，因为这里只需要检测自定义的一种物体，为此将num\_classes修改为1，另外将fine\_tune\_checkpoint、train\_input\_reader，eval\_input\_reader中的相关路径改为实际对应的路径。
>
> 另外可以根据计算机的性能适当调整batch\_size的大小，更大的batch\_size会导致内存占用的大幅上升，但能够提升计算的收敛速度。

#### 5. 训练和导出模型

> 完成以上准备后可以开始运行训练过程，参数分别为训练使用的配置，训练模型保存的路径，训练的步数和进行效果预估的步数
>
```bash
python model_main.py \
    --pipeline_config_path=training/ssd_mobilenet_v2_coco.config \
    --model_dir=training \
    --num_train_steps=15000 \
    --num_eval_steps=2000 \
    --alsologtostderr
```
> 完成训练后需要对训练的模型进行导出，导出的命令如下，其中model.ckpt选择对应目录中后缀值最大的一个，导出后会得到模型文件result/frozen\_inference\_graph.pb
>
```bash
python3 export_inference_graph.py \
    --input_type=image_tensor \
    --pipeline_config_path=training/ssd_mobilenet_v2_coco.config \
    --trained_checkpoint_prefix=training/model.ckpt-15000 \
    --output_directory=result
```

#### 6. 使用模型进行物体检测

> 在完成以上全部步骤后就可以开始对图片中的物体进行检测了，检测的脚本如下，检测的图片保存为in.jpg，运行后会输出进行标记的文件out.jpg
>
```python
import numpy as np
import tensorflow as tf
from utils import label_map_util
from utils import visualization_utils as vis_util
import cv2
>
PATH_TO_CKPT = 'result/frozen_inference_graph.pb'
PATH_TO_LABELS = 'training/pascal_label_map.pbtxt';
NUM_CLASSES = 1
>
detection_graph = tf.Graph()
with detection_graph.as_default():
  od_graph_def = tf.GraphDef()
  with tf.gfile.GFile(PATH_TO_CKPT, 'rb') as fid:
    serialized_graph = fid.read()
    od_graph_def.ParseFromString(serialized_graph)
    tf.import_graph_def(od_graph_def, name='')
>
label_map = label_map_util.load_labelmap(PATH_TO_LABELS)
categories = label_map_util.convert_label_map_to_categories(label_map, max_num_classes=NUM_CLASSES, use_display_name=True)
category_index = label_map_util.create_category_index(categories)
>
with detection_graph.as_default():
    with tf.Session(graph=detection_graph) as sess:
        image_tensor = detection_graph.get_tensor_by_name('image_tensor:0')
        detection_boxes = detection_graph.get_tensor_by_name('detection_boxes:0')
        detection_scores = detection_graph.get_tensor_by_name('detection_scores:0')
        detection_classes = detection_graph.get_tensor_by_name('detection_classes:0')
        num_detections = detection_graph.get_tensor_by_name('num_detections:0')
>
        image_np = cv2.imread('in.jpg');
        image_np_expanded = np.expand_dims(image_np, axis=0)
        (boxes, scores, classes, num) = sess.run(
            [detection_boxes, detection_scores, detection_classes, num_detections],
            feed_dict={image_tensor: image_np_expanded})
        print(scores);
        vis_util.visualize_boxes_and_labels_on_image_array(image_np, np.squeeze(boxes), np.squeeze(classes).astype(np.int32), np.squeeze(scores), category_index, use_normalized_coordinates=True, line_thickness=8)
        cv2.imwrite('out.jpg', image_np)
```
> 实现了一个检测图面中哆啦A梦的模型，其检测的效果如下，总体上还是可以的。 
>
<img src="//raw.githubusercontent.com/longlong2010/image.longlong2010.github.io/master/202010/doraemon.jpg" width="600"/>
