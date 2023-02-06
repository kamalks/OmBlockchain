# Scale Keras 2.3 Applications

---

![](../../../../image/colab_logo_32px.png)[Run in Google Colab](https://colab.research.google.com/github/intel-analytics/BigDL/blob/main/python/orca/colab-notebook/quickstart/keras_lenet_mnist.ipynb) &nbsp;![](../../../../image/GitHub-Mark-32px.png)[View source on GitHub](https://github.com/intel-analytics/BigDL/blob/main/python/orca/colab-notebook/quickstart/keras_lenet_mnist.ipynb)

---

**In this guide we will describe how to scale out _Keras 2.3_ programs using Orca in 4 simple steps.**


### Step 0: Prepare Environment

We recommend using [conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/) to prepare the environment. Please refer to the [install guide](../Overview/install.md) for more details.

```bash
conda create -n py37 python=3.7  # "py37" is conda environment name, you can use any name you like.
conda activate py37

pip install bigdl-orca
pip install tensorflow==1.15.0
pip install tensorflow-datasets==2.1.0
pip install psutil
pip install pandas
pip install scikit-learn
```

### Step 1: Init Orca Context
```python
from bigdl.orca import init_orca_context, stop_orca_context

if cluster_mode == "local":  # For local machine
    init_orca_context(cluster_mode="local", cores=4, memory="4g")
elif cluster_mode == "k8s":  # For K8s cluster
    init_orca_context(cluster_mode="k8s", num_nodes=2, cores=2, memory="4g", master=..., container_image=...)
elif cluster_mode == "yarn":  # For Hadoop/YARN cluster
    init_orca_context(cluster_mode="yarn", num_nodes=2, cores=2, memory="4g")
```

This is the only place where you need to specify local or distributed mode. View [Orca Context](../Overview/orca-context.md) for more details.

Please check the tutorials if you want to run on [Kubernetes](../Tutorial/k8s.md) or [Hadoop/YARN](../Tutorial/yarn.md) clusters.

### Step 2: Define the Model

You may define your model, loss and metrics in the same way as in any standard (single node) Keras program.

```python
from tensorflow import keras

model = keras.Sequential(
    [keras.layers.Conv2D(20, kernel_size=(5, 5), strides=(1, 1), activation='tanh',
                         input_shape=(28, 28, 1), padding='valid'),
     keras.layers.MaxPooling2D(pool_size=(2, 2), strides=(2, 2), padding='valid'),
     keras.layers.Conv2D(50, kernel_size=(5, 5), strides=(1, 1), activation='tanh',
                         padding='valid'),
     keras.layers.MaxPooling2D(pool_size=(2, 2), strides=(2, 2), padding='valid'),
     keras.layers.Flatten(),
     keras.layers.Dense(500, activation='tanh'),
     keras.layers.Dense(10, activation='softmax'),
     ]
)

model.compile(optimizer=keras.optimizers.RMSprop(),
              loss='sparse_categorical_crossentropy',
              metrics=['accuracy'])
```
### Step 3: Define Train Dataset

You can define the dataset using standard [tf.data.Dataset](https://www.tensorflow.org/api_docs/python/tf/data/Dataset). Orca also supports [Spark DataFrame](./spark-dataframe.md) and [Orca XShards](./xshards-pandas.md).

```python
import tensorflow as tf
import tensorflow_datasets as tfds

def preprocess(data):
    data['image'] = tf.cast(data["image"], tf.float32) / 255.
    return data['image'], data['label']

# get DataSet
mnist_train = tfds.load(name="mnist", split="train", data_dir=...)
mnist_test = tfds.load(name="mnist", split="test", data_dir=...)

mnist_train = mnist_train.map(preprocess)
mnist_test = mnist_test.map(preprocess)
```

### Step 4: Fit with Orca Estimator

First, create an Orca Estimator for TensorFlow.

```python
from bigdl.orca.learn.tf.estimator import Estimator

est = Estimator.from_keras(keras_model=model)
```

Next, fit and evaluate using the Estimator.
```python
est.fit(data=mnist_train,
        batch_size=320,
        epochs=5,
        validation_data=mnist_test)

result = est.evaluate(mnist_test)
print(result)
```

**Note:** You should call `stop_orca_context()` when your program finishes.

That's it, the same code can run seamlessly on your local laptop and scale to [Kubernetes](../Tutorial/k8s.md) or [Hadoop/YARN](../Tutorial/yarn.md) clusters.
