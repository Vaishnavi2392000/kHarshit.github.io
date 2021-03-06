---
layout: post
title: "Transfer learning: How to build accurate models"
date: 2018-08-10
categories: [Deep Learning, Computer Vision, Python]
---

A good Covolutional Neural Network model requires a large dataset and good amount of training, which is often not possible in practice. Transfer learning provides a turn around it. It's a method to use pre-trained models to obtain better results. A pre-trained model has been previously trained on a dataset and contains the weights and biases that represent the features of whichever dataset it was trained on. There are two ways to achieve this:

* Extract features from the pre-trained model and use them in your model
* Fine-tune the pre-trained ConvNet model

The following table summarizes the method to be adopted according to your dataset properties:

| Size of dataset || Compared to original dataset || Method                                                                  |
|-----------------|------------------------------|-------------------------------------------------------------------------|
| small           || similar                      || train a linear classifier on CNN nodes                                  |
| small           || different                     || train classifier from activations somewhere earlier in the network |
| large           || similar                      || fine-tune the model                                                     |
| large           || different                    || can build model from scratch, initialize weight from pre-trained model  |
{:.mbtablestyle}


**Case I: Small dataset, similar data** 

* slice off the end of the neural network <abbr title="Since the data sets are similar, images from each data set will have similar higher level features">only</abbr>
* add a new fully connected layer that matches the number of classes in the new data set
* randomize the weights of the new fully connected layer; <abbr title="To avoid overfitting on the small data set, the weights of the original network will be held constant rather than re-training the weights. ">freeze all the weights</abbr> from the pre-trained network *(so that the network behaves as fixed feature-extractor)*
* train the network to update the weights of the new fully connected layer

**Case II: Small dataset, Different data**

* slice off <abbr title="But the original training set and the new data set do not share higher level features. In this case, the new network will only use the layers containing lower level features.">all</abbr> but some of the pre-trained layers near the beginning of the network
* add to the remaining pre-trained layers a new fully connected layer that matches the number of classes in the new data set
* randomize the weights of the new fully connected layer; <abbr title="to combat overfitting while re-training">freeze</abbr> all the weights from the pre-trained network
* train the network to update the weights of the new fully connected layer

**Case III: Large dataset, Similar data**

* remove the last fully connected layer and replace with a layer matching the number of classes in the new data set
* randomly initialize the weights in the new fully connected layer
* initialize the rest of the weights using the *pre-trained* weights
* re-train the <abbr title="overfitting no longer concern due to large dataset">entire</abbr> neural network

**Case IV: Large dataset, Different data**

* remove the last fully connected layer and replace with a layer matching the number of classes in the new data set
* retrain the network from scratch with randomly initialized weights
* alternatively, you could just use the same strategy as the "large and similar" data case


The following guide used ResNet50<sup id="a1">[1](#myfootnote1)</sup> as pre-trained model and uses it as feature extractor for building a ConvNet for CIFAR10<sup id="a2">[2](#myfootnote2)</sup> dataset. The CIFAR-10 dataset consists of 60000 32x32 colour images in 10 classes, with 6000 images per class. There are 50000 training images and 10000 test images. 

{% highlight python %}
import numpy as np
import matplotlib.pyplot as plt
from keras.datasets import cifar10
from keras.utils import np_utils
from keras.models import Sequential
from keras.layers import Dense, Conv2D, MaxPooling2D
from keras.layers import Dropout, Flatten, GlobalAveragePooling2D
from keras.applications.resnet50 import ResNet50, preprocess_input

(X_train, y_train), (X_test, y_test) = cifar10.load_data() 
X_train.shape, X_test.shape, np.unique(y_train).shape[0]
# one-hot encoding
n_classes = 10
y_train = np_utils.to_categorical(y_train, n_classes)
y_test = np_utils.to_categorical(y_test, n_classes)
{% endhighlight %}

Now, extract features from ResNet50 and save them.

{% highlight python %}
# load model
model_tl = ResNet50(weights='imagenet',
                    include_top=False,  # remove top FC layers
                   input_shape=(200, 200, 2))

# reshape as min size of image to fed into ResNet is (197, 197, 3)
X_train_new = np.array([imresize(X_train[i], (200, 200, 3)) for i in range(0, len(X_train))]).astype('float32')
# preprocess data 
resnet_train_input = preprocess_input(X_train_new)
# create bottleneck features for training data
train_features = model.predict(resnet_train_input)
# save the bottleneck features
np.savez('resnet_features_train', features=train_features)

# reshape testing data
X_test_new = np.array([imresize(X_test[i], (200, 200,  3)) for i in range(0, len(X_test))]).astype('float32')
# preprocess to fed it in pre-trained ResNet50
restnet_test_input = preprocess_input(X_test_new)
# extract features
test_featues = model.predict(restnet_test_input)
# save features
np.savez('resnet_features_test', features=test_featues)
{% endhighlight %}

Finally, build the model in Keras using the extracted features.

{% highlight python %}
# create model
model = Sequential()
model.add(GlobalAveragePooling2D(input_shape=train_featues.shape[1:]))
model.add(Dropout(0.3))
model.add(Dense(10, activation='softmax'))
model.summary()

model.compile(loss='categorical_crossentropy',
             optimizer='adam',
             metrics=['accuracy'])

model.fit(train_features, y_train,
          batch_size=32, epochs=10,
         validation_split=0.2, callbacks=[checkpointer],
         verbose=True, shuffle=True)

# model evaluation
score = model.evaluate(test_features, y_test)
print('Accuracy on test set: {}'.format(score[1]))
{% endhighlight %}

The use of transfer learning is possible because the features that ConvNets learn in the first layers are independent of the dataset, so are often transferable to different dataset.

**Update:** Also, check implementation in PyTorch on GitHub at *[kHarshit/transfer-learning](https://github.com/kHarshit/transfer-learning)*.

**Footnotes:**  

<a name="myfootnote1"></a>1: [ResNet-50](https://www.kaggle.com/keras/resnet50) [↩](#a1)  
<a name="myfootnote2"></a>2: [CIFAR10 dataset](https://www.cs.toronto.edu/~kriz/cifar.html) [↩](#a2)

**References:**  

1. [Transfer Learning - CS231n](http://cs231n.github.io/transfer-learning/)
2. [How transferable are features in deep neural networks?](https://arxiv.org/abs/1411.1792)
3. [Machine learning - Hackerearth](https://www.hackerearth.com/practice/machine-learning/)
4. [PyTorch Deep Learning Nanodegree - Udacity](https://in.udacity.com/course/deep-learning-nanodegree--nd101)
