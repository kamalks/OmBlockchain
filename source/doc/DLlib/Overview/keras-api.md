# Keras-Like API

## 1. Introduction
[DLlib](dllib.md) provides __Keras-like API__ based on [__Keras 1.2.2__](https://faroit.github.io/keras-docs/1.2.2/) for distributed deep learning on Apache Spark. Users can easily use the Keras-like API to create a neural network model, and train, evaluate or tune it in a distributed fashion on Spark.

To define a model in Scala using the Keras-like API, one just needs to import the following packages:

```scala
import com.intel.analytics.bigdl.dllib.keras.layers._
import com.intel.analytics.bigdl.dllib.keras.models._
import com.intel.analytics.bigdl.dllib.utils.Shape
```

One of the highlighted features with regard to the new API is __shape inference__. Users only need to specify the input shape (a `Shape` object __excluding__ batch dimension, for example, `inputShape=Shape(3, 4)` for 3D input) for the first layer of a model and for the remaining layers, the input dimension will be automatically inferred.

---
## 2. LeNet Example
Here we use the Keras-like API to define a LeNet CNN model and train it on the MNIST dataset:

```scala
import com.intel.analytics.bigdl.numeric.NumericFloat
import com.intel.analytics.bigdl.dllib.keras.layers._
import com.intel.analytics.bigdl.dllib.keras.models._
import com.intel.analytics.bigdl.dllib.utils.Shape

val model = Sequential()
model.add(Reshape(Array(1, 28, 28), inputShape = Shape(28, 28, 1)))
model.add(Convolution2D(6, 5, 5, activation = "tanh").setName("conv1_5x5"))
model.add(MaxPooling2D())
model.add(Convolution2D(12, 5, 5, activation = "tanh").setName("conv2_5x5"))
model.add(MaxPooling2D())
model.add(Flatten())
model.add(Dense(100, activation = "tanh").setName("fc1"))
model.add(Dense(10, activation = "softmax").setName("fc2"))

model.getInputShape().toSingle().toArray // Array(-1, 28, 28, 1)
model.getOutputShape().toSingle().toArray // Array(-1, 10)
```
---
## 3. Shape
Input and output shapes of a model in the Keras-like API are described by the `Shape` object in Scala, which can be classified into `SingleShape` and `MultiShape`.

`SingleShape` is just a list of Int indicating shape dimensions while `MultiShape` is essentially a list of `Shape`.

Example code to create a shape:
```scala
// create a SingleShape
val shape1 = Shape(3, 4)
// create a MultiShape consisting of two SingleShape
val shape2 = Shape(List(Shape(1, 2, 3), Shape(4, 5, 6)))
```
You can use method `toSingle()` to cast a `Shape` to a `SingleShape`. Similarly, use `toMulti()` to cast a `Shape` to a `MultiShape`.

---
## 4. Define a model
You can define a model either using [Sequential API](#sequential-api) or [Functional API](#functional-api). Remember to specify the input shape for the first layer.

After creating a model, you can call the following __methods__:

```scala
getInputShape()
```
```scala
getOutputShape()
```
* Return the input or output shape of a model, which is a [`Shape`](#2-shape) object. For `SingleShape`, the first entry is `-1` representing the batch dimension. For a model with multiple inputs or outputs, it will return a `MultiShape`.

```scala
setName(name)
```
* Set the name of the model.

---
## 5. Sequential API
The model is described as a linear stack of layers in the Sequential API. Layers can be added into the `Sequential` container one by one and the order of the layers in the model will be the same as the insertion order.

To create a sequential container:
```scala
Sequential()
```

Example code to create a sequential model:
```scala
import com.intel.analytics.bigdl.dllib.keras.layers.{Dense, Activation}
import com.intel.analytics.bigdl.dllib.keras.models.Sequential
import com.intel.analytics.bigdl.dllib.utils.Shape

val model = Sequential[Float]()
model.add(Dense[Float](32, inputShape = Shape(128)))
model.add(Activation[Float]("relu"))
```

---
## 6. Functional API
The model is described as a graph in the Functional API. It is more convenient than the Sequential API when defining some complex model (for example, a model with multiple outputs).

To create an input node:
```scala
Input(inputShape = null, name = null)
```
Parameters:

* `inputShape`: A [`Shape`](#shape) object indicating the shape of the input node, not including batch.
* `name`: String to set the name of the input node. If not specified, its name will by default to be a generated string.

To create a graph container:
```scala
Model(input, output)
```
Parameters:

* `input`: An input node or an array of input nodes.
* `output`: An output node or an array of output nodes.

To merge a list of input __nodes__ (__NOT__ layers), following some merge mode in the Functional API:
```scala
import com.intel.analytics.bigdl.dllib.keras.layers.Merge.merge

merge(inputs, mode = "sum", concatAxis = -1) // This will return an output NODE.
```

Parameters:

* `inputs`: A list of node instances. Must be more than one node.
* `mode`: Merge mode. String, must be one of: 'sum', 'mul', 'concat', 'ave', 'cos', 'dot', 'max'. Default is 'sum'.
* `concatAxis`: Int, axis to use when concatenating nodes. Only specify this when merge mode is 'concat'. Default is -1, meaning the last axis of the input.

Example code to create a graph model:
```scala
import com.intel.analytics.bigdl.dllib.keras.layers.{Dense, Input}
import com.intel.analytics.bigdl.dllib.keras.layers.Merge.merge
import com.intel.analytics.bigdl.dllib.keras.models.Model
import com.intel.analytics.bigdl.dllib.utils.Shape

// instantiate input nodes
val input1 = Input[Float](inputShape = Shape(8))
val input2 = Input[Float](inputShape = Shape(6))
// call inputs() with an input node and get an output node
val dense1 = Dense[Float](10).inputs(input1)
val dense2 = Dense[Float](10).inputs(input2)
// merge two nodes following some merge mode
val output = merge(inputs = List(dense1, dense2), mode = "sum")
// create a graph container
val model = Model[Float](Array(input1, input2), output)
```

## 7. Persistence
This section describes how to save and load the Keras-like API.

### 7.1 save
To save a Keras model, you call the method `saveModel(path)`.

**Scala:**
```scala
import com.intel.analytics.bigdl.dllib.keras.layers.{Dense, Activation}
import com.intel.analytics.bigdl.dllib.keras.models.Sequential

val model = Sequential[Float]()
model.add(Dense[Float](32, inputShape = Shape(128)))
model.add(Activation[Float]("relu"))
model.saveModel("/tmp/seq.model")
```
**Python:**
```python
import bigdl.dllib.keras.Sequential
from bigdl.dllib.keras.layer import Dense

model = Sequential()
model.add(Dense(input_shape=(32, )))
model.saveModel("/tmp/seq.model")
```

### 7.2 load
To load a saved Keras model, you call the method `load_model(path)`.

**Scala:**
```scala
import com.intel.analytics.bigdl.dllib.keras.Models

val model = Models.loadModel[Float]("/tmp/seq.model")
```

**Python:**
```python
from bigdl.dllib.keras.models
model = load_model("/tmp/seq.model")
```
