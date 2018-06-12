---
title: tensorflow-autoencoder
date: 2017-03-26 19:08:57
categories:
    - Machine Learning
tags:
    - tensorflo
    - unsupervised
---
# 加性高斯噪声自编码器

---

先上代码

```python
import numpy as np
import sklearn.preprocessing as prep
import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data


def xavier_init(fan_in, fan_out, constant=1):
    high = constant * np.sqrt(6.0 / (fan_in + fan_out))
    low = -high
    return tf.random_uniform((fan_in, fan_out), minval=low, maxval=high, dtype=tf.float32)


def standard_scale(x_train, x_test):
    preprocessor = prep.StandardScaler().fit(x_train)
    x_train = preprocessor.transform(x_train)
    x_test = preprocessor.transform(x_test)
    return x_train, x_test


def get_random_block_from_data(data, batch_size):
    start_index = np.random.randint(0, len(data) - batch_size)
    return data[start_index:(start_index + batch_size)]


class AdditiveGaussianNoiseAutoenvoder(object):
    def __init__(self, n_input, n_hidden, transfer_function=tf.nn.softplus, optimizer=tf.train.AdamOptimizer(),
                 scale=0.1):
        self.n_input = n_input
        self.n_hidden = n_hidden
        self.transfer = transfer_function
        self.scale = tf.placeholder(tf.float32)
        self.training_scale = scale
        self.weights = self._initialize_weights()

        self.x = tf.placeholder(tf.float32, [None, self.n_input])
        # x * w + b
        self.hidden = self.transfer(
            tf.add(tf.matmul(self.x + scale * tf.random_normal((n_input,)), self.weights['w1']), self.weights['b1']))
        self.reconstruction = tf.add(tf.matmul(self.hidden, self.weights['w2']), self.weights['b2'])

        self.cost = 0.5 * tf.reduce_sum(tf.pow(tf.subtract(self.reconstruction, self.x), 2.0))
        self.optimizer = optimizer.minimize(self.cost)

        init = tf.global_variables_initializer()
        self.sess = tf.Session()
        self.sess.run(init)

    def _initialize_weights(self):  # 初始化权重
        all_weights = dict()
        all_weights['w1'] = tf.Variable(xavier_init(self.n_input, self.n_hidden))
        all_weights['b1'] = tf.Variable(tf.zeros([self.n_hidden], dtype=tf.float32))
        all_weights['w2'] = tf.Variable(tf.zeros([self.n_hidden, self.n_input], dtype=tf.float32))
        all_weights['b2'] = tf.Variable(tf.zeros([self.n_input], dtype=tf.float32))
        return all_weights

    def partial_fit(self, x):
        cost, opt = self.sess.run((self.cost, self.optimizer), feed_dict={self.x: x, self.scale: self.training_scale})
        return cost

    def calc_total_cost(self, x):
        return self.sess.run(self.cost, feed_dict={self.x: x, self.scale: self.training_scale})

    def transform(self, x):
        return self.sess.run(self.hidden, feed_dict={self.x: x, self.scale: self.training_scale})

    def generate(self, hidden=None):
        if hidden is None:
            hidden = np.random.normal(size=self.weights['b1'])
        return self.sess.run(self.reconstruction, feed_dict={self.hidden: hidden})

    def reconstruct(self, x):
        return self.sess.run(self.reconstruction, feed_dict={self.x: x, self.scale: self.training_scale})

    def getWeights(self):
        return self.sess.run(self.weights['w1'])

    def getBiases(self):
        return self.sess.run(self.weights['b1'])


if __name__ == '__main__':
    mnist = input_data.read_data_sets('mnist', one_hot=True)  # 读取数据
    x_train, x_test = standard_scale(mnist.train.images, mnist.test.images) # 数据标准化
    n_samples = int(mnist.train.num_examples) 
    training_epoches = 20
    batch_size = 128
    display_step = 1
    
    # 初始化autoencoder
    autoencoder = AdditiveGaussianNoiseAutoenvoder(n_input=784, n_hidden=200, transfer_function=tf.nn.relu,
                                                   optimizer=tf.train.AdamOptimizer(learning_rate=0.001), scale=0.02)
    print(n_samples)
    # train
    for epoch in range(training_epoches):
        avg_cost = 0.
        total_batch = int(n_samples / batch_size)
        for i in range(total_batch):
            batch_xs = get_random_block_from_data(x_train, batch_size)
            cost = autoencoder.partial_fit(batch_xs)
            avg_cost += cost / n_samples * batch_size
        if epoch % display_step == 0:
            print("Epoch: {:<4} cost= {:.9}".format(epoch + 1, avg_cost))
    print("Total cost: {}".format(autoencoder.calc_total_cost(x_test)))

```

1 xavier_init: 初始化方法
![image_1bc53v7e51prc1m0318lf15go1mlq9.png-14.9kB][1]

2 _initialize_weights 权重、偏置初始化

3 partial_fit： 执行2个计算图的节点，分别是损失cost和训练过程optimizer，从输入层到隐层。

4 calc_total_cost：只计算损失cost

5 transform: 隐层计算，输出隐层计算结果
 
6 generate： 隐层到输出层

7 reconstruct： 整个自编码过程， 包括transform和generate两块。


  [1]: http://static.zybuluo.com/luzhongqiu/ai9nsj6oslhxbn729f25kzhe/image_1bc53v7e51prc1m0318lf15go1mlq9.png
  
8 最后输出cost大约在7000左右

## summary

    自编码器和单隐层神经网络差不多，只不过在数据输入时候做了标准化，并加上高斯噪声。
    自编码器是无监督，用于提取高阶特征。
    个人觉得，如果在监督学习中效果不好，可以在前期进行无监督学习提取高阶特征。