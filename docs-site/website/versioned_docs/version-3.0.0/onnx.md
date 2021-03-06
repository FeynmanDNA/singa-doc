---
id: version-3.0.0-onnx
title: ONNX
original_id: onnx
---

<!--- Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements.  See the NOTICE file distributed with this work for additional information regarding copyright ownership.  The ASF licenses this file to you under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License.  You may obtain a copy of the License at http://www.apache.org/licenses/LICENSE-2.0 Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.  See the License for the specific language governing permissions and limitations under the License.  -->

[ONNX](https://onnx.ai/) is an open representation format for machine learning
models, which enables AI developers to use models across different libraries and
tools. SINGA supports loading ONNX format models for training and inference, and
saving models defined using SINGA APIs (e.g., [Module](./module)) into ONNX
format.

SINGA has been tested with the following
[version](https://github.com/onnx/onnx/blob/master/docs/Versioning.md) of ONNX.

| ONNX version | File format version | Opset version ai.onnx | Opset version ai.onnx.ml | Opset version ai.onnx.training |
| ------------ | ------------------- | --------------------- | ------------------------ | ------------------------------ |
| 1.6.0        | 6                   | 11                    | 2                        | -                              |

## General usage

### Loading an ONNX Model into SINGA

After loading an ONNX model from disk by `onnx.load`, you need to update the
model's batchsize, since for most models, they use a placeholder to represent
its batchsize. We give an example here, as `update_batch_size`. You only need to
update the batchsize of input and output, the shape of internal tensors will be
inferred automatically.

Then, you can prepare the SINGA model by using `sonnx.prepare`. This function
iterates and translates all the nodes within the ONNX model's graph into SINGA
operators, loads all stored weights and infers each intermediate tensor's shape.

```python3
import onnx
from singa import device
from singa import sonnx

# if the input has multiple tensors? can put this function inside prepare()?
def update_batch_size(onnx_model, batch_size):
    model_input = onnx_model.graph.input[0]
    model_input.type.tensor_type.shape.dim[0].dim_value = batch_size
    model_output = onnx_model.graph.output[0]
    model_output.type.tensor_type.shape.dim[0].dim_value = batch_size
    return onnx_model


model_path = "PATH/To/ONNX/MODEL"
onnx_model = onnx.load(model_path)

# set batch size
onnx_model = update_batch_size(onnx_model, 1)

# convert onnx graph nodes into SINGA operators
dev = device.create_cuda_gpu()
sg_ir = sonnx.prepare(onnx_model, device=dev)
```

### Inference SINGA model

Once the model is created, you can do inference by calling `sg_ir.run`. The
input and output must be SINGA `Tensor` instances. Since SINGA model returns the
output as a list, if there is only one output, you just need to take the first
element from the output.

```python3
# can warp the following code in prepare()
# and provide a flag training=True/False?

class Infer:


    def __init__(self, sg_ir):
        self.sg_ir = sg_ir

    def forward(self, x):
        return sg_ir.run([x])[0]


data = get_dataset()
x = tensor.Tensor(device=dev, data=data)

model = Infer(sg_ir)
y = model.forward(x)
```

### Saving SINGA model into ONNX Format

Given the input tensors and the output tensors generated by the operators the
model, you can trace back all internal operations. Therefore, a SINGA model is
defined by the input and outputs tensors. To export a SINGA model into ONNX
format, you just need to provide the input and output tensor list.

```python3
# x is the input tensor, y is the output tensor
sonnx.to_onnx([x], [y])
```

### Re-training an ONNX model

To train (or refine) an ONNX model using SINGA, you need to set the internal
tensors to be trainable

```python3
class Infer:

    def __init__(self, sg_ir):
        self.sg_ir = sg_ir
        ## can wrap these codes in sonnx?
        for idx, tens in sg_ir.tensor_map.items():
            # allow the tensors to be updated
            tens.requires_grad = True
            tens.stores_grad = True

    def forward(self, x):
        return sg_ir.run([x])[0]

autograd.training = False
model = Infer(sg_ir)

autograd.training = True
# then you training the model like normal
# give more details??
```

### Transfer-learning an ONNX model

You also can append some layers to the end of ONNX model to do
transfer-learning. The `last_layers` means you cut the ONNX layers from [0,
last_layers]. Then you can append more layers by the normal SINGA model.

```python3
class Trans:

    def __init__(self, sg_ir, last_layers):
        self.sg_ir = sg_ir
        self.last_layers = last_layers
        self.append_linear1 = autograd.Linear(500, 128, bias=False)
        self.append_linear2 = autograd.Linear(128, 32, bias=False)
        self.append_linear3 = autograd.Linear(32, 10, bias=False)

    def forward(self, x):
        y = sg_ir.run([x], last_layers=self.last_layers)[0]
        y = self.append_linear1(y)
        y = autograd.relu(y)
        y = self.append_linear2(y)
        y = autograd.relu(y)
        y = self.append_linear3(y)
        y = autograd.relu(y)
        return y

autograd.training = False
model = Trans(sg_ir, -1)

# then you training the model like normal
```

## A Full Example

This part introduces the usage of SINGA ONNX by using the mnist example. In this
section, the examples of how to export, load, inference, re-training, and
transfer-learning the minist model are displayed. You can try this part
[here](https://colab.research.google.com/drive/1-YOfQqqw3HNhS8WpB8xjDQYutRdUdmCq).

### Load dataset

Firstly, you need to import some necessary libraries and define some auxiliary
functions for downloading and preprocessing the dataset:

```python
import os
import urllib.request
import gzip
import numpy as np
import codecs

from singa import device
from singa import tensor
from singa import opt
from singa import autograd
from singa import sonnx
import onnx


def load_dataset():
    train_x_url = 'http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz'
    train_y_url = 'http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz'
    valid_x_url = 'http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz'
    valid_y_url = 'http://yann.lecun.com/exdb/mnist/t10k-labels-idx1-ubyte.gz'
    train_x = read_image_file(check_exist_or_download(train_x_url)).astype(
        np.float32)
    train_y = read_label_file(check_exist_or_download(train_y_url)).astype(
        np.float32)
    valid_x = read_image_file(check_exist_or_download(valid_x_url)).astype(
        np.float32)
    valid_y = read_label_file(check_exist_or_download(valid_y_url)).astype(
        np.float32)
    return train_x, train_y, valid_x, valid_y


def check_exist_or_download(url):

    download_dir = '/tmp/'

    name = url.rsplit('/', 1)[-1]
    filename = os.path.join(download_dir, name)
    if not os.path.isfile(filename):
        print("Downloading %s" % url)
        urllib.request.urlretrieve(url, filename)
    return filename


def read_label_file(path):
    with gzip.open(path, 'rb') as f:
        data = f.read()
        assert get_int(data[:4]) == 2049
        length = get_int(data[4:8])
        parsed = np.frombuffer(data, dtype=np.uint8, offset=8).reshape(
            (length))
        return parsed


def get_int(b):
    return int(codecs.encode(b, 'hex'), 16)


def read_image_file(path):
    with gzip.open(path, 'rb') as f:
        data = f.read()
        assert get_int(data[:4]) == 2051
        length = get_int(data[4:8])
        num_rows = get_int(data[8:12])
        num_cols = get_int(data[12:16])
        parsed = np.frombuffer(data, dtype=np.uint8, offset=16).reshape(
            (length, 1, num_rows, num_cols))
        return parsed


def to_categorical(y, num_classes):
    y = np.array(y, dtype="int")
    n = y.shape[0]
    categorical = np.zeros((n, num_classes))
    categorical[np.arange(n), y] = 1
    categorical = categorical.astype(np.float32)
    return categorical
```

### MNIST model

Then you can define a class called **CNN** to construct the mnist model which
consists of several convolution, pooling, fully connection and relu layers. You
can also define a function to calculate the **accuracy** of our result. Finally,
you can define a **train** and a **test** function to handle the training and
prediction process.

```python
class CNN:
    def __init__(self):
        self.conv1 = autograd.Conv2d(1, 20, 5, padding=0)
        self.conv2 = autograd.Conv2d(20, 50, 5, padding=0)
        self.linear1 = autograd.Linear(4 * 4 * 50, 500, bias=False)
        self.linear2 = autograd.Linear(500, 10, bias=False)
        self.pooling1 = autograd.MaxPool2d(2, 2, padding=0)
        self.pooling2 = autograd.MaxPool2d(2, 2, padding=0)

    def forward(self, x):
        y = self.conv1(x)
        y = autograd.relu(y)
        y = self.pooling1(y)
        y = self.conv2(y)
        y = autograd.relu(y)
        y = self.pooling2(y)
        y = autograd.flatten(y)
        y = self.linear1(y)
        y = autograd.relu(y)
        y = self.linear2(y)
        return y


def accuracy(pred, target):
    y = np.argmax(pred, axis=1)
    t = np.argmax(target, axis=1)
    a = y == t
    return np.array(a, "int").sum() / float(len(t))


def train(model,
          x,
          y,
          epochs=1,
          batch_size=64,
          dev=device.get_default_device()):
    batch_number = x.shape[0] // batch_size

    for i in range(epochs):
        for b in range(batch_number):
            l_idx = b * batch_size
            r_idx = (b + 1) * batch_size

            x_batch = tensor.Tensor(device=dev, data=x[l_idx:r_idx])
            target_batch = tensor.Tensor(device=dev, data=y[l_idx:r_idx])

            output_batch = model.forward(x_batch)
            # onnx_model = sonnx.to_onnx([x_batch], [y])
            # print('The model is:\n{}'.format(onnx_model))

            loss = autograd.softmax_cross_entropy(output_batch, target_batch)
            accuracy_rate = accuracy(tensor.to_numpy(output_batch),
                                     tensor.to_numpy(target_batch))

            sgd = opt.SGD(lr=0.001)
            for p, gp in autograd.backward(loss):
                sgd.update(p, gp)
            sgd.step()

            if b % 1e2 == 0:
                print("acc %6.2f loss, %6.2f" %
                      (accuracy_rate, tensor.to_numpy(loss)[0]))
    print("training completed")
    return x_batch, output_batch

def test(model, x, y, batch_size=64, dev=device.get_default_device()):
    batch_number = x.shape[0] // batch_size

    result = 0
    for b in range(batch_number):
        l_idx = b * batch_size
        r_idx = (b + 1) * batch_size

        x_batch = tensor.Tensor(device=dev, data=x[l_idx:r_idx])
        target_batch = tensor.Tensor(device=dev, data=y[l_idx:r_idx])

        output_batch = model.forward(x_batch)
        result += accuracy(tensor.to_numpy(output_batch),
                           tensor.to_numpy(target_batch))

    print("testing acc %6.2f" % (result / batch_number))
```

### Train mnist model and export it to onnx

Now, you can train the mnist model and export its onnx model by calling the
**soonx.to_onnx** function.

```python
def make_onnx(x, y):
    return sonnx.to_onnx([x], [y])

# create device
dev = device.create_cuda_gpu()
#dev = device.get_default_device()
# create model
model = CNN()
# load data
train_x, train_y, valid_x, valid_y = load_dataset()
# normalization
train_x = train_x / 255
valid_x = valid_x / 255
train_y = to_categorical(train_y, 10)
valid_y = to_categorical(valid_y, 10)
# do training
autograd.training = True
x, y = train(model, train_x, train_y, dev=dev)
onnx_model = make_onnx(x, y)
# print('The model is:\n{}'.format(onnx_model))

# Save the ONNX model
model_path = os.path.join('/', 'tmp', 'mnist.onnx')
onnx.save(onnx_model, model_path)
print('The model is saved.')
```

### Inference

After you export the onnx model, you can find a file called **mnist.onnx** in
the '/tmp' directory, this model, therefore, can be imported by other libraries.
Now, if you want to import this onnx model into singa again and do the inference
using the validation dataset, you can define a class called **Infer**, the
forward function of Infer will be called by the test function to do inference
for validation dataset. By the way, you should set the label of training to
**False** to fix the gradient of autograd operators.

When import the onnx model, you need to call **onnx.load** to load the onnx
model firstly. Then the onnx model will be fed into the **soonx.prepare** to
parse and initiate to a singa model(**sg_ir** in the code). The sg_ir contains a
singa graph within it, and then you can run an step of inference by feeding
input to its run function.

```python
class Infer:
    def __init__(self, sg_ir):
        self.sg_ir = sg_ir
        for idx, tens in sg_ir.tensor_map.items():
            # allow the tensors to be updated
            tens.requires_grad = True
            tens.stores_grad= True
            sg_ir.tensor_map[idx] = tens

    def forward(self, x):
        return sg_ir.run([x])[0] # we can run one step of inference by feeding input

# load the ONNX model
onnx_model = onnx.load(model_path)
sg_ir = sonnx.prepare(onnx_model, device=dev) # parse and initiate to a singa model

# inference
autograd.training = False
print('The inference result is:')
test(Infer(sg_ir), valid_x, valid_y, dev=dev)
```

### Re-training

Assume after import the model, you want to re-train the model again, we can
define a function called **re_train**. Before we call this re_train function, we
should set the label of training to **True** to make the autograde operators
update their gradient. And after we finish the training, we set it as **False**
again to call the test function doing inference.

```python
def re_train(sg_ir,
             x,
             y,
             epochs=1,
             batch_size=64,
             dev=device.get_default_device()):
    batch_number = x.shape[0] // batch_size

    new_model = Infer(sg_ir)

    for i in range(epochs):
        for b in range(batch_number):
            l_idx = b * batch_size
            r_idx = (b + 1) * batch_size

            x_batch = tensor.Tensor(device=dev, data=x[l_idx:r_idx])
            target_batch = tensor.Tensor(device=dev, data=y[l_idx:r_idx])

            output_batch = new_model.forward(x_batch)

            loss = autograd.softmax_cross_entropy(output_batch, target_batch)
            accuracy_rate = accuracy(tensor.to_numpy(output_batch),
                                     tensor.to_numpy(target_batch))

            sgd = opt.SGD(lr=0.01)
            for p, gp in autograd.backward(loss):
                sgd.update(p, gp)
            sgd.step()

            if b % 1e2 == 0:
                print("acc %6.2f loss, %6.2f" %
                      (accuracy_rate, tensor.to_numpy(loss)[0]))
    print("re-training completed")
    return new_model

# load the ONNX model
onnx_model = onnx.load(model_path)
sg_ir = sonnx.prepare(onnx_model, device=dev)

# re-training
autograd.training = True
new_model = re_train(sg_ir, train_x, train_y, dev=dev)
autograd.training = False
test(new_model, valid_x, valid_y, dev=dev)
```

### Transfer learning

Finally, if we want to do transfer-learning, we can define a function called
**Trans** to append some layers after the onnx model. For demonstration, the
code only appends several linear(fully connection) and relu after the onnx
model. You can define a transfer_learning function to handle the training
process of the transfer-learning model. And the label of training is the same as
the previous one.

```python
class Trans:
    def __init__(self, sg_ir, last_layers):
        self.sg_ir = sg_ir
        self.last_layers = last_layers
        self.append_linear1 = autograd.Linear(500, 128, bias=False)
        self.append_linear2 = autograd.Linear(128, 32, bias=False)
        self.append_linear3 = autograd.Linear(32, 10, bias=False)

    def forward(self, x):
        y = sg_ir.run([x], last_layers=self.last_layers)[0]
        y = self.append_linear1(y)
        y = autograd.relu(y)
        y = self.append_linear2(y)
        y = autograd.relu(y)
        y = self.append_linear3(y)
        y = autograd.relu(y)
        return y

def transfer_learning(sg_ir,
             x,
             y,
             epochs=1,
             batch_size=64,
             dev=device.get_default_device()):
    batch_number = x.shape[0] // batch_size

    trans_model = Trans(sg_ir, -1)

    for i in range(epochs):
        for b in range(batch_number):
            l_idx = b * batch_size
            r_idx = (b + 1) * batch_size

            x_batch = tensor.Tensor(device=dev, data=x[l_idx:r_idx])
            target_batch = tensor.Tensor(device=dev, data=y[l_idx:r_idx])
            output_batch = trans_model.forward(x_batch)

            loss = autograd.softmax_cross_entropy(output_batch, target_batch)
            accuracy_rate = accuracy(tensor.to_numpy(output_batch),
                                     tensor.to_numpy(target_batch))

            sgd = opt.SGD(lr=0.07)
            for p, gp in autograd.backward(loss):
                sgd.update(p, gp)
            sgd.step()

            if b % 1e2 == 0:
                print("acc %6.2f loss, %6.2f" %
                      (accuracy_rate, tensor.to_numpy(loss)[0]))
    print("transfer-learning completed")
    return trans_mode

# load the ONNX model
onnx_model = onnx.load(model_path)
sg_ir = sonnx.prepare(onnx_model, device=dev)

# transfer-learning
autograd.training = True
new_model = transfer_learning(sg_ir, train_x, train_y, dev=dev)
autograd.training = False
test(new_model, valid_x, valid_y, dev=dev)
```

## ONNX model zoo

The [ONNX Model Zoo](https://github.com/onnx/models) is a collection of
pre-trained, state-of-the-art models in the ONNX format contributed by community
members. SINGA has supported several CV and NLP models now. More models are
going to be supported soon.

### Image Classification

This collection of models take images as input, then classifies the major
objects in the images into 1000 object categories such as keyboard, mouse,
pencil, and many animals.

| Model Class                                                                                    | Reference                                          | Description                                                                                                                                                                              | Link                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <b>[MobileNet](https://github.com/onnx/models/tree/master/vision/classification/mobilenet)</b> | [Sandler et al.](https://arxiv.org/abs/1801.04381) | Light-weight deep neural network best suited for mobile and embedded vision applications. <br>Top-5 error from paper - ~10%                                                              | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1HsixqJMIpKyEPhkbB8jy7NwNEFEAUWAf) |
| <b>[ResNet18](https://github.com/onnx/models/tree/master/vision/classification/resnet)</b>     | [He et al.](https://arxiv.org/abs/1512.03385)      | A CNN model (up to 152 layers). Uses shortcut connections to achieve higher accuracy when classifying images. <br> Top-5 error from paper - ~3.6%                                        | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1u1RYefSsVbiP4I-5wiBKHjsT9L0FxLm9) |
| <b>[VGG16](https://github.com/onnx/models/tree/master/vision/classification/vgg)</b>           | [Simonyan et al.](https://arxiv.org/abs/1409.1556) | Deep CNN model(up to 19 layers). Similar to AlexNet but uses multiple smaller kernel-sized filters that provides more accuracy when classifying images. <br>Top-5 error from paper - ~8% | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/14kxgRKtbjPCKKsDJVNi3AvTev81Gp_Ds) |

### Object Detection

Object detection models detect the presence of multiple objects in an image and
segment out areas of the image where the objects are detected.

| Model Class                                                                                                       | Reference                                             | Description                                                                                                                        | Link                                                                                                                                                    |
| ----------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <b>[Tiny YOLOv2](https://github.com/onnx/models/tree/master/vision/object_detection_segmentation/tiny_yolov2)</b> | [Redmon et al.](https://arxiv.org/pdf/1612.08242.pdf) | A real-time CNN for object detection that detects 20 different classes. A smaller version of the more complex full YOLOv2 network. | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/11V4I6cRjIJNUv5ZGsEGwqHuoQEie6b1T) |

### Face Analysis

Face detection models identify and/or recognize human faces and emotions in
given images.

| Model Class                                                                                               | Reference                                          | Description                                                                                                                         | Link                                                                                                                                                    |
| --------------------------------------------------------------------------------------------------------- | -------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <b>[ArcFace](https://github.com/onnx/models/tree/master/vision/body_analysis/arcface)</b>                 | [Deng et al.](https://arxiv.org/abs/1801.07698)    | A CNN based model for face recognition which learns discriminative features of faces and produces embeddings for input face images. | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1qanaqUKGIDtifdzEzJOHjEj4kYzA9uJC) |
| <b>[Emotion FerPlus](https://github.com/onnx/models/tree/master/vision/body_analysis/emotion_ferplus)</b> | [Barsoum et al.](https://arxiv.org/abs/1608.01041) | Deep CNN for emotion recognition trained on images of faces.                                                                        | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1XHtBQGRhe58PDi4LGYJzYueWBeWbO23r) |

### Machine Comprehension

This subset of natural language processing models that answer questions about a
given context paragraph.

| Model Class                                                                                           | Reference                                             | Description                                                                     | Link                                                                                                                                                    |
| ----------------------------------------------------------------------------------------------------- | ----------------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <b>[BERT-Squad](https://github.com/onnx/models/tree/master/text/machine_comprehension/bert-squad)</b> | [Devlin et al.](https://arxiv.org/pdf/1810.04805.pdf) | This model answers questions based on the context of the given input paragraph. | [![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1kud-lUPjS_u-TkDAzihBTw0Vqr0FjCE-) |

## Supported operators

The following operators are supported:

- Conv
- Relu
- Constant
- MaxPool
- AveragePool
- Softmax
- Sigmoid
- Add
- MatMul
- BatchNormalization
- Concat
- Flatten
- Add
- Gemm
- Reshape
- Sum
- Cos
- Cosh
- Sin
- Sinh
- Tan
- Tanh
- Acos
- Acosh
- Asin
- Asinh
- Atan
- Atanh
- Selu
- Elu
- Equal
- Less
- Sign
- Div
- Sub
- Sqrt
- Log
- Greater
- HardSigmoid
- Identity
- Softplus
- Softsign
- Mean
- Pow
- Clip
- PRelu
- Mul
- Transpose
- Max
- Min
- Shape
- And
- Or
- Xor
- Not
- Neg
- Reciprocal
- LeakyRelu
- GlobalAveragePool
- ConstantOfShape
- Dropout
- ReduceSum
- ReduceMean
- LeakyRelu
- GlobalAveragePool
- Squeeze
- Unsqueeze
- Slice
- Ceil
- Split
- Gather
- Tile
- NonZero
- Cast
- OneHot

### Special comments for ONNX backend

- Conv, MaxPool and AveragePool

  Input must be 1d`(N*C*H)` and 2d(`N*C*H*W`) shape and `dilation` must be 1.

- BatchNormalization

  `epsilon` is 1e-05 and cannot be changed.

- Cast

  Only support float32 and int32, other types are casted to these two types.

- Squeeze and Unsqueeze

  If you encounter errors when you `Squeeze` or `Unsqueeze` between `Tensor` and
  Scalar, please report to us.

- Empty tensor Empty tensor is illegal in SINGA.

## Implementation

The code of SINGA ONNX locates at `python/singa/soonx.py`. There are three main
class, `SingaFrontend` and `SingaBackend` and `SingaRep`. `SingaFrontend`
translates a SINGA model to ONNX model; `SingaBackend` translates a ONNX model
to `SingaRep` object which stores all SINGA operators and tensors(the tensor in
this doc means SINGA `Tensor`); `SingaRep` can be run like a SINGA model.

### SingaFrontend

The entry function of `SingaFrontend` is `singa_to_onnx_model` which also is
called `to_onnx`. `singa_to_onnx_model` creates the ONNX model, and it also
create a ONNX graph by using `singa_to_onnx_graph`.

`singa_to_onnx_graph` accepts the output of the model, and recursively iterate
the SINGA model's graph from the output to get all operators to form a queue.
The input and intermediate tensors, i.e, trainable weights, of the SINGA model
is picked up at the same time. The input is stored in `onnx_model.graph.input`;
the output is stored in `onnx_model.graph.output`; and the trainable weights are
stored in `onnx_model.graph.initializer`.

Then the SINGA operator in the queue is translated to ONNX operators one by one.
`_rename_operators` defines the operators name mapping between SINGA and ONNX.
`_special_operators` defines which function to be used to translate the
operator.

In addition, some operators in SINGA has different definition with ONNX, that
is, ONNX regards some attributes of SINGA operators as input, so
`_unhandled_operators` defines which function to handle the special operator.

Since the bool type is regarded as int32 in SINGA, `_bool_operators` defines the
operators to be changed as bool type.

### SingaBackend

The entry function of `SingaBackend` is `prepare` which checks the version of
ONNX model and call `_onnx_model_to_singa_net` then.

The purpose of `_onnx_model_to_singa_net` is to get SINGA tensors and operators.
The tensors are stored in a dictionary by their name in ONNX, and operators are
stored in queue by the form of
`namedtuple('SingaOps', ['name', 'op', 'handle', 'forward'])`. For each
operator, `name` is its ONNX node name; `op` is the ONNX node; `forward` is the
SINGA operator's forward function; `handle` is prepared for some special
operators such as Conv and Pooling which has `handle` object.

The first step of `_onnx_model_to_singa_net` is to call `_init_graph_parameter`
to get all tensors within the model. For trainable weights, it can init SINGA
`Tensor` from `onnx_model.graph.initializer`. Please note, the weights may also
be stored within graph's input or a ONNX node called `Constant`, SINGA can also
handle these.

Though all weights are stored within ONNX model, the input of the model is
unknown but its shape and type. So SINGA support two ways to init input, 1,
generate random tensor by its shape and type, 2, allow the user to assign the
input. The first way works fine for most models, however, for some model such as
bert, the indices of matrix cannot be random generated otherwise it will incurs
errors.

Then, `_onnx_model_to_singa_net` iterators all nodes within ONNX graph to
translate it to SIGNA operators. Also, `_rename_operators` defines the operators
name mapping between SINGA and ONNX. `_special_operators` defines which function
to be used to translate the operator. `_run_node` runs the generated SINGA model
by its input tensors and store its output tensors for being used by later
operators.

This class finally return a `SingaRep` object and stores all SINGA tensors and
operators within it.

### SingaRep

`SingaBackend` stores all SINGA tensors and operators. `run` accepts the input
of the model and run the SINGA operators one by one following the operators
queue. The user can use `last_layers` to decide to run the model till the last
few layers. Set `all_outputs` as `False` to get only the final output, `True` to
also get all the intermediate output.
