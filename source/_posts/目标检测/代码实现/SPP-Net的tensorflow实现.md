---
title: SPP-Net的tensorflow实现
date: 2021-01-05 11:50:12
tags:
- 目标检测
- SPP-Net
- tensorflow
updated:
type:
comments:
description:
keywords: 目标检测金字塔网络
cover: /img/SPP.png
top_img:
mathjax: 
katex: true
aside:
aplayer:
highlight_shrink:
categories: 
- 人工智能学习
- 目标检测
---
代码来源：https://github.com/peace195/sppnet


# 代码解读
## 导入包
```python
import numpy as np
import os
import sys
import tarfile
from six.moves.urllib.request import urlretrieve
from six.moves import cPickle as pickle
from PIL import Image
import math
import random
import re
import scipy.io
import PIL
from numpy import *
from pylab import *
from PIL import Image
from collections import defaultdict
import tensorflow as tf
import matplotlib.pyplot as plt
```
## 参数设置
```python
DROPOUT = 0.5 #随机失活概率
LEARNING_RATE  = 0.1 #
VALIDATION_SIZE = 0 #
TRAINING_ITERATIONS = 50000 #训练次数
WEIGHT_DECAY = 0.00005 #正则化系数
```
## 加载数据
```python
net_data = load("bvlc_alexnet.npy",allow_pickle=True).item()

out_pool_size = [8, 6, 4] #设置金字塔池化的长度，一共三个尺度
hidden_dim = 0
for item in out_pool_size:
  hidden_dim = hidden_dim + item * item
  
data_folder = './jpg'
labels = scipy.io.loadmat('imagelabels.mat')
setid = scipy.io.loadmat('setid.mat')

labels = labels['labels'][0] - 1
trnid = np.array(setid['tstid'][0]) - 1
tstid = np.array(setid['trnid'][0]) - 1
valid = np.array(setid['valid'][0]) - 1

num_classes = 102
data_dir = list() #加载文件名
for img in os.listdir(data_folder):
  data_dir.append(os.path.join(data_folder, img))

data_dir.sort()
```
## 子函数
### 打印迭代训练输出
```python
def print_activations(t):
  print(t.op.name, ' ', t.get_shape().as_list())
```
### 整理标签
```python
def dense_to_one_hot(labels_dense, num_classes):
  num_labels = labels_dense.shape[0]
  index_offset = np.arange(num_labels) * num_classes
  labels_one_hot = np.zeros((num_labels, num_classes))
  labels_one_hot.flat[index_offset + labels_dense.ravel()] = 1
  return labels_one_hot
```
### 读取训练图像
```python
def read_images_from_disk(input_queue):
  label = input_queue[1]
  file_contents = tf.read_file(input_queue[0])
  example = tf.image.decode_jpeg(file_contents, channels=3)
  # example = tf.cast(example, tf.float32 )
  return example, label
```
### 通过函数产生权重数据
```python
def weight_variable(shape, name):
  initial = tf.truncated_normal(shape, stddev=0.01, name=name)
  return tf.Variable(initial)
```
### 通过函数产生偏置数据
```python
def bias_variable(shape, name):
  initial = tf.constant(0.1, shape=shape, name=name)
  return tf.Variable(initial)
```
### 实现caffe多通道卷积，当窗口不够时，舍弃
```python
def conv(input, kernel, biases, k_h, k_w, c_o, s_h, s_w, padding = "VALID", group = 1):
  '''From https://github.com/ethereon/caffe-tensorflow
  '''
  c_i = input.get_shape()[-1]
  assert c_i % group == 0
  assert c_o % group == 0
  convolve = lambda i, k: tf.nn.conv2d(i, k, [1, s_h, s_w, 1], padding=padding)

  if group == 1:
    conv = convolve(input, kernel)
  else:
    input_groups = tf.split(axis=3, num_or_size_splits=group, value=input)
    kernel_groups = tf.split(axis=3, num_or_size_splits=group, value=kernel)
    output_groups = [convolve(i, k) for i, k in zip(input_groups, kernel_groups)]
    conv = tf.concat(axis=3, values=output_groups)
  return tf.reshape(tf.nn.bias_add(conv, biases), [-1] + conv.get_shape().as_list()[1:])
```
### 单通道卷积，当窗口不够时，填充
```python
def conv2d(x, W, stride_h, stride_w, padding='SAME'):
  return tf.nn.conv2d(x, W, strides=[1, stride_h, stride_w, 1], padding=padding)
```
### 2*2池化操作，当窗口不够时，填充
```python
def max_pool_2x2(x):
  return tf.nn.max_pool(x, ksize=[1, 2, 2, 1], strides=[1, 2, 2, 1], padding='SAME')
```
### 3*3池化操作，当窗口不够时，填充
```python
def max_pool_3x3(x):
  return tf.nn.max_pool(x, ksize=[1, 3, 3, 1], strides=[1, 2, 2, 1], padding='SAME')
```
### 4*4池化操作，当窗口不够时，填充
```python
def max_pool_4x4(x):
  return tf.nn.max_pool(x, ksize=[1, 4, 4, 1], strides=[1, 4, 4, 1], padding='SAME')
```
### 金字塔池化操作

将输入的batch_size\*height\*width\*channels的图像池化成batch_size\*1*out_pool_size大小
```python
# Spatial Pyramid Pooling block
# https://arxiv.org/abs/1406.4729
def spatial_pyramid_pool(previous_conv, num_sample, previous_conv_size, out_pool_size):
  """
  previous_conv: a tensor vector of previous convolution layer
  num_sample: an int number of image in the batch
  previous_conv_size: an int vector [height, width] of the matrix features size of previous convolution layer
  out_pool_size: a int vector of expected output size of max pooling layer
  
  returns: a tensor vector with shape [1 x n] is the concentration of multi-level pooling
  """
  for i in range(len(out_pool_size)):
    h_strd = h_size = math.ceil(float(previous_conv_size[0]) / out_pool_size[i])
    w_strd = w_size = math.ceil(float(previous_conv_size[1]) / out_pool_size[i])
    pad_h = int(out_pool_size[i] * h_size - previous_conv_size[0])
    pad_w = int(out_pool_size[i] * w_size - previous_conv_size[1])
    new_previous_conv = tf.pad(previous_conv, tf.constant([[0, 0], [0, pad_h], [0, pad_w], [0, 0]]))
    max_pool = tf.nn.max_pool(new_previous_conv,
                   ksize=[1,h_size, h_size, 1],
                   strides=[1,h_strd, w_strd,1],
                   padding='SAME')
    if (i == 0):
      spp = tf.reshape(max_pool, [num_sample, -1])
    else:
      spp = tf.concat(axis=1, values=[spp, tf.reshape(max_pool, [num_sample, -1])])
  
  return spp
```
--------------------------------------------------------------------------
## 训练模型
### 设置batch
将某一维尺寸相差不超过10个像素点值的部分聚集在一起，成为一个batch
```python
size_cluster = defaultdict(list)
for tid in trnid:
  img = Image.open(data_dir[tid])
  key = (img.size[0] - img.size[0] % 10, img.size[1] - img.size[1] % 10)
  size_cluster[key].append(tid)
  
size_cluster_keys = size_cluster.keys()
```
### 初始化变量
```python
train_accuracies = []
train_cost = []
validation_accuracies = []
x_range = []
batch_size = 20
print('Training ...')
```
### 训练部分
#### 迭代计数部分
```python
# Training block
# 1. Combime all iamges have the same size to a batch.
# 2. Then, train parameters in a batch
# 3. Transfer trained parameters to another batch
it = 0 #迭代计数器
while it < TRAINING_ITERATIONS:
  graph = tf.Graph()
  with graph.as_default():
```
#### 循环主体
##### 设置batch
每一个batch的数量不相同，图片尺寸从小到大进行训练，在进行训练图像输入时，先将图像的长宽缩小2倍
```python
y_train = labels[size_cluster[size_cluster_keys[it%len(size_cluster_keys)]]]
if len(y_train) < 50:
    batch_size = len(y_train)

y_train = dense_to_one_hot(y_train, num_classes)
x_train = [data_dir[i] for i in size_cluster[size_cluster_keys[it%len(size_cluster_keys)]]]

input_queue_train = tf.train.slice_input_producer([x_train, y_train],
                        num_epochs=None,
                        shuffle=True)

x_train, y_train = read_images_from_disk(input_queue_train)

print(size_cluster_keys[it%len(size_cluster_keys)]) #输出聚类图像尺寸字典键

x_train = tf.image.resize_images(x_train,
                    [size_cluster_keys[it%len(size_cluster_keys)][1]/2,
                    size_cluster_keys[it%len(size_cluster_keys)][0]/2],
                    method=1, align_corners=False)

x_train, y_train = tf.train.batch([x_train, y_train], batch_size = batch_size)
```
##### 加载预训练的Alexnet模型
舍弃了alexnet第六层全连接的W，重新初始化变量，修改第八层全连接参数匹配训练集
```python
x = tf.placeholder('float', shape = x_train.get_shape()) #定义输入参数
y_ = tf.placeholder('float', shape = [None, num_classes]) #定义输出参数

conv1W = tf.Variable(net_data["conv1"][0])
conv1b = tf.Variable(net_data["conv1"][1])
conv2W = tf.Variable(net_data["conv2"][0])
conv2b = tf.Variable(net_data["conv2"][1])
conv3W = tf.Variable(net_data["conv3"][0])
conv3b = tf.Variable(net_data["conv3"][1])
conv4W = tf.Variable(net_data["conv4"][0])
conv4b = tf.Variable(net_data["conv4"][1])
conv5W = tf.Variable(net_data["conv5"][0])
conv5b = tf.Variable(net_data["conv5"][1])
fc6W = weight_variable([hidden_dim * 256, 4096], 'fc6W')
fc6b = tf.Variable(net_data["fc6"][1])
fc7W = tf.Variable(net_data["fc7"][0])
fc7b = tf.Variable(net_data["fc7"][1])
fc8W = weight_variable([4096, num_classes], 'W_fc8')
fc8b = bias_variable([num_classes], 'b_fc8')
keep_prob = tf.placeholder('float')
```
##### 前向传播通路
第一层

    conv1 卷积核11*11 卷积步长4 填充卷积 relu函数激活
    lrn1 正则化
    maxpool1 3*3池化 池化步长2 舍弃池化
第二层

    conv2 卷积核5*5 卷积步长1 填充卷积 relu函数激活
    lrn2 正则化
    maxpool2 3*3池化 池化步长为2 舍弃池化

第三层

    conv3 卷积核3*3 卷积步长1 填充卷积 relu函数激活

第四层

    conv4 卷积核3*3 卷积步长1 填充卷积 relu函数激活

第五层

    conv5 卷积核3*3 卷积步长1 填充卷积 relu函数激活
    spp5 金字塔池化

第六层

    fc6 全连接 relu激活 随机失活训练

第七层

    fc7 全连接 relu激活 随机失活训练

第八层

    fc8 全连接 不使用激活函数

```python
def model(x):
    # conv1 卷积核11*11 卷积步长4 填充卷积 relu函数激活
    conv1 = tf.nn.relu(conv(x, conv1W, conv1b, 11, 11, 96, 4, 4, padding="SAME", group=1))
    # lrn1 正则化
    # lrn(2, 2e-05, 0.75, name='norm1')
    lrn1 = tf.nn.local_response_normalization(conv1,
                        depth_radius=5,
                        alpha=0.0001,
                        beta=0.75,
                        bias=1.0)
    # maxpool1 3*3池化 池化步长2 舍弃池化
    maxpool1 = tf.nn.max_pool(lrn1, ksize=[1, 3, 3, 1], strides=[1, 2, 2, 1], padding='VALID')
    # conv2 卷积核5*5 卷积步长1 填充卷积 relu函数激活
    conv2 = tf.nn.relu(conv(maxpool1, conv2W, conv2b, 5, 5, 256, 1, 1, padding="SAME", group=2))
    # lrn2 正则化
    # lrn(2, 2e-05, 0.75, name='norm2')
    lrn2 = tf.nn.local_response_normalization(conv2,
                        depth_radius=5,
                        alpha=0.0001,
                        beta=0.75,
                        bias=1.0)
    # maxpool2 3*3池化 池化步长为2 舍弃池化
    maxpool2 = tf.nn.max_pool(lrn2, ksize=[1, 3, 3, 1], strides=[1, 2, 2, 1], padding='VALID')
    # conv3 卷积核3*3 卷积步长1 填充卷积 relu函数激活
    conv3 = tf.nn.relu(conv(maxpool2, conv3W, conv3b, 3, 3, 384, 1, 1, padding="SAME", group=1))
    # conv4 卷积核3*3 卷积步长1 填充卷积 relu函数激活
    conv4 = tf.nn.relu(conv(conv3, conv4W, conv4b, 3, 3, 384, 1, 1, padding="SAME", group=2))
    # conv5 卷积核3*3 卷积步长1 填充卷积 relu函数激活
    conv5 = tf.nn.relu(conv(conv4, conv5W, conv5b, 3, 3, 256, 1, 1, padding="SAME", group=2))
    print(int(conv5.get_shape()[0]), int(conv5.get_shape()[1]), int(conv5.get_shape()[2]))
    # spp5 金字塔池化
    maxpool5 = spatial_pyramid_pool(conv5,
                    int(conv5.get_shape()[0]),
                    [int(conv5.get_shape()[1]), int(conv5.get_shape()[2])],
                    out_pool_size)
    # fc6 全连接 relu激活 随机失活训练
    fc6 = tf.nn.relu_layer(tf.reshape(maxpool5, [-1, int(prod(maxpool5.get_shape()[1:]))]), fc6W, fc6b)
    fc6_drop = tf.nn.dropout(fc6, keep_prob)
    # fc7 全连接 relu激活 随机失活训练
    fc7 = tf.nn.relu_layer(fc6_drop, fc7W, fc7b)
    fc7_drop = tf.nn.dropout(fc7, keep_prob)
    # fc8 全连接 不使用激活函数
    fc8 = tf.nn.xw_plus_b(fc7_drop, fc8W, fc8b)
    return fc8
```
##### 损失函数计算反向传播
使用交叉熵softmax函数计算score，损失函数为：交叉熵损失+正则化系数*所有权重的L2，动态调整学习率
```python
logits = model(x)
cross_entropy = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits=logits, labels=y_))
regularizers = tf.nn.l2_loss(conv1W) + tf.nn.l2_loss(conv1b) + \
            tf.nn.l2_loss(conv2W) + tf.nn.l2_loss(conv2b) + \
            tf.nn.l2_loss(conv3W) + tf.nn.l2_loss(conv3b) + \
            tf.nn.l2_loss(conv4W) + tf.nn.l2_loss(conv4b) + \
            tf.nn.l2_loss(conv5W) + tf.nn.l2_loss(conv5b) + \
            tf.nn.l2_loss(fc6W) + tf.nn.l2_loss(fc6b) + \
            tf.nn.l2_loss(fc7W) + tf.nn.l2_loss(fc7b) + \
            tf.nn.l2_loss(fc8W) + tf.nn.l2_loss(fc8b)

loss = tf.reduce_mean(cross_entropy + WEIGHT_DECAY * regularizers)

cross_entropy = tf.reduce_mean(tf.nn.softmax_cross_entropy_with_logits(logits=logits, labels=y_))
# optimisation loss function
global_step = tf.Variable(0)
learning_rate = tf.train.exponential_decay(LEARNING_RATE, global_step, 1000, 0.9, staircase=True) #动态学习率
train_step = tf.train.AdagradOptimizer(learning_rate).minimize(loss)
```
##### 评价模型
计算模型正确率
```python
# evaluation
correct_prediction = tf.equal(tf.argmax(logits, 1), tf.argmax(y_, 1))
accuracy = tf.reduce_mean(tf.cast(correct_prediction, 'float'))
predict = tf.argmax(logits, 1)
saver = tf.train.Saver({v.op.name: v for v in [conv1W, conv1b,
                            conv2W, conv2b,
                            conv3W, conv3b,
                            conv4W, conv4b,
                            conv5W, conv5b,
                            fc6W, fc6b,
                            fc7W, fc7b,
                            fc8W, fc8b]})
```
##### 训练设置
```python
with tf.Session(graph=graph) as sess:
init = tf.global_variables_initializer()
sess.run(init)
coord = tf.train.Coordinator()
threads = tf.train.start_queue_runners(coord=coord)
if os.path.exists('./alex_model_spp.ckpt'):
    saver.restore(sess, './alex_model_spp.ckpt')

cnt_tmp = 0
xtrain, ytrain = sess.run([x_train, y_train])
for i in range(10):
    it = it + 1
    _, train_accuracy, cost = sess.run([train_step, accuracy, cross_entropy], 
                    feed_dict = {x: xtrain,
                            y_: ytrain, 
                            keep_prob: 1.0})
    
    print('training_accuracy => %.4f, cost value => %.4f for step %d'
        %(train_accuracy, cost, it))

    if (train_accuracy > 0.95):
    cnt_tmp = cnt_tmp + 1

    if (cnt_tmp > 10):
    break

    train_accuracies.append(train_accuracy)
    x_range.append(it)
    train_cost.append(cost)

saver.save(sess, './alex_model_spp.ckpt')
coord.request_stop()
coord.join(threads)
sess.close()
del sess
```
##### 输出评价图
```python
# Plot accuracy and loss curve
plt.plot(x_range, train_cost,'-b')
plt.ylabel('spp_cost')
plt.xlabel('step')
plt.savefig('spp_cost.png')
plt.close()
plt.plot(x_range, train_accuracies,'-b')
plt.ylabel('spp_accuracies')
plt.ylim(ymax = 1.1)
plt.xlabel('step')
plt.savefig('spp_accuracy.png')
```
--------------------------------------------------------------------------
## 测试模型
```python
# Testing block
# 1. Gather all images have the same size into a batch
# 2. Feed to Alexnet_SPP to predict the expected labels
it = 0
result = list()
f = open('result_spp.txt', 'w')
while it < len(tstid):
  if (it % 10 == 0):
    print(it)
  graph = tf.Graph()
  with graph.as_default():
    # with tf.device('/cpu:0'):
    img = Image.open(data_dir[tstid[it]])
    filename_queue = tf.train.string_input_producer([data_dir[tstid[it]]])
    reader = tf.WholeFileReader()
    key, value = reader.read(filename_queue)
    my_img = tf.image.decode_jpeg(value, channels = 3)
    # my_img = tf.cast(my_img, tf.float32)
    my_img = tf.image.resize_images(my_img,
                    [img.size[1] / 2,
                    img.size[0] / 2],
                    method = 1,
                    align_corners = False)

    my_img = tf.expand_dims(my_img, 0)

    x = tf.placeholder('float', shape=my_img.get_shape())
    print(my_img.get_shape())
    conv1W = tf.Variable(net_data["conv1"][0])
    conv1b = tf.Variable(net_data["conv1"][1])
    conv2W = tf.Variable(net_data["conv2"][0])
    conv2b = tf.Variable(net_data["conv2"][1])
    conv3W = tf.Variable(net_data["conv3"][0])
    conv3b = tf.Variable(net_data["conv3"][1])
    conv4W = tf.Variable(net_data["conv4"][0])
    conv4b = tf.Variable(net_data["conv4"][1])
    conv5W = tf.Variable(net_data["conv5"][0])
    conv5b = tf.Variable(net_data["conv5"][1])
    fc6W = weight_variable([hidden_dim * 256, 4096], 'fc6W')
    fc6b = tf.Variable(net_data["fc6"][1])
    fc7W = tf.Variable(net_data["fc7"][0])
    fc7b = tf.Variable(net_data["fc7"][1])
    fc8W = weight_variable([4096, num_classes], 'W_fc8')
    fc8b = bias_variable([num_classes], 'b_fc8')
    keep_prob = tf.placeholder('float')


    def model(x):
      # conv1
      conv1 = tf.nn.relu(conv(x, conv1W, conv1b, 11, 11, 96, 4, 4, padding="SAME", group=1))
      # lrn1
      # lrn(2, 2e-05, 0.75, name='norm1')
      lrn1 = tf.nn.local_response_normalization(conv1,
                            depth_radius=5,
                            alpha=0.0001,
                            beta=0.75,
                            bias=1.0)
      # maxpool1
      maxpool1 = tf.nn.max_pool(lrn1, ksize=[1, 3, 3, 1], strides=[1, 2, 2, 1], padding='VALID')
      # conv2
      conv2 = tf.nn.relu(conv(maxpool1, conv2W, conv2b, 5, 5, 256, 1, 1, padding="SAME", group=2))
      # lrn2
      # lrn(2, 2e-05, 0.75, name='norm2')
      lrn2 = tf.nn.local_response_normalization(conv2,
                            depth_radius=5,
                            alpha=0.0001,
                            beta=0.75,
                            bias=1.0)
      # maxpool2
      maxpool2 = tf.nn.max_pool(lrn2, ksize=[1, 3, 3, 1], strides=[1, 2, 2, 1], padding='VALID')
      # conv3
      conv3 = tf.nn.relu(conv(maxpool2, conv3W, conv3b, 3, 3, 384, 1, 1, padding="SAME", group=1))
      # conv4
      conv4 = tf.nn.relu(conv(conv3, conv4W, conv4b, 3, 3, 384, 1, 1, padding="SAME", group=2))
      # conv5
      conv5 = tf.nn.relu(conv(conv4, conv5W, conv5b, 3, 3, 256, 1, 1, padding="SAME", group=2))
      maxpool5 = spatial_pyramid_pool(conv5,
                      int(conv5.get_shape()[0]),
                       [int(conv5.get_shape()[1]), int(conv5.get_shape()[2])],
                       out_pool_size)
      # fc6
      fc6 = tf.nn.relu_layer(tf.reshape(maxpool5, [-1, int(prod(maxpool5.get_shape()[1:]))]), fc6W, fc6b)
      fc6_drop = tf.nn.dropout(fc6, keep_prob)
      # fc7
      fc7 = tf.nn.relu_layer(fc6_drop, fc7W, fc7b)
      fc7_drop = tf.nn.dropout(fc7, keep_prob)
      # fc8
      fc8 = tf.nn.xw_plus_b(fc7_drop, fc8W, fc8b)
      prob = tf.nn.softmax(fc8)
      return prob

    logits = model(x)
    predict = tf.argmax(logits, 1)
    saver = tf.train.Saver({v.op.name: v for v in [conv1W, conv1b,
                             conv2W, conv2b,
                             conv3W, conv3b,
                             conv4W, conv4b,
                             conv5W, conv5b,
                             fc6W, fc6b,
                             fc7W, fc7b,
                             fc8W, fc8b]})

  with tf.Session(graph=graph) as sess:
    init = tf.global_variables_initializer()
    sess.run(init)
    coord = tf.train.Coordinator()
    threads = tf.train.start_queue_runners(coord=coord)
    saver.restore(sess, './alex_model_spp.ckpt')
    image = sess.run(my_img)
    predict = predict.eval(feed_dict={x: image, keep_prob: 1.0})
    result.append(predict[0])
    f.write(data_dir[tstid[it]] + '\t' + str(predict[0]) + '\t' + str(labels[tstid[it]]))
    f.write('\n')
    coord.request_stop()
    coord.join(threads)
  sess.close()
  del sess
  it = it + 1

print('Test accuracy: %f' %(sum(np.array(result) == np.array(labels[tstid])).astype('float')/len(tstid)))
f.close()
```