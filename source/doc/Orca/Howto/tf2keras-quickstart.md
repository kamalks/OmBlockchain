# Scale TensorFlow 2 Applications

---

![](../../../../image/colab_logo_32px.png)[Run in Google Colab](https://colab.research.google.com/github/intel-analytics/BigDL/blob/main/python/orca/colab-notebook/quickstart/tf2_keras_lenet_mnist.ipynb) &nbsp;![](../../../../image/GitHub-Mark-32px.png)[View source on GitHub](https://github.com/intel-analytics/BigDL/blob/main/python/orca/colab-notebook/quickstart/tf2_keras_lenet_mnist.ipynb)

---

**In this guide we will describe how to to scale out _TensorFlow 2_ programs using Orca in 4 simple steps.**

### Step 0: Prepare Environment

We recommend using [conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/) to prepare the environment. Please refer to the [install guide](../Overview/install.md) for more details.

```bash
conda create -n py37 python=3.7  # "py37" is conda environment name, you can use any name you like.
conda activate py37

pip install bigdl-orca[ray]
pip install tensorflow
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

You can then define and compile the Keras model in the _Creator Function_ using the standard TensorFlow 2 Keras APIs.

```python
import tensorflow as tf

def model_creator(config):
    model = tf.keras.Sequential(
        [tf.keras.layers.Conv2D(20, kernel_size=(5, 5), strides=(1, 1), activation='tanh',
                                input_shape=(28, 28, 1), padding='valid'),
         tf.keras.layers.MaxPooling2D(pool_size=(2, 2), strides=(2, 2), padding='valid'),
         tf.keras.layers.Conv2D(50, kernel_size=(5, 5), strides=(1, 1), activation='tanh',
                                padding='valid'),
         tf.keras.layers.MaxPooling2D(pool_size=(2, 2), strides=(2, 2), padding='valid'),
         tf.keras.layers.Flatten(),
         tf.keras.layers.Dense(500, activation='tanh'),
         tf.keras.layers.Dense(10, activation='softmax'),
         ]
    )

    model.compile(optimizer=tf.keras.optimizers.RMSprop(),
                  loss='sparse_categorical_crossentropy',
                  metrics=['accuracy'])
    return model
```
### Step 3: Define the Dataset

You can define the dataset in the _Creator Function_ using standard [tf.data.Dataset](https://www.tensorflow.org/api_docs/python/tf/data/Dataset) APIs. Orca also supports [Spark DataFrame](./spark-dataframe.md) and [Orca XShards](./xshards-pandas.md).


```python
def preprocess(x, y):
    x = tf.cast(tf.reshape(x, (28, 28, 1)), dtype=tf.float32) / 255.0
    return x, y

def train_data_creator(config, batch_size):
    (train_feature, train_label), _ = tf.keras.datasets.mnist.load_data()
    dataset = tf.data.Dataset.from_tensor_slices((train_feature, train_label))
    dataset = dataset.repeat()
    dataset = dataset.map(preprocess)
    dataset = dataset.shuffle(1000)
    dataset = dataset.batch(batch_size)
    return dataset

def val_data_creator(config, batch_size):
    _, (val_feature, val_label) = tf.keras.datasets.mnist.load_data()
    dataset = tf.data.Dataset.from_tensor_slices((val_feature, val_label))
    dataset = dataset.repeat()
    dataset = dataset.map(preprocess)
    dataset = dataset.batch(batch_size)
    return dataset
```

### Step 4: Fit with Orca Estimator

First, create an Orca Estimator for TensorFlow 2.

```python
from bigdl.orca.learn.tf2 import Estimator

est = Estimator.from_keras(model_creator=model_creator, workers_per_node=2)
```

Next, fit and evaluate using the Estimator. 
```python
batch_size = 320
train_stats = est.fit(train_data_creator,
                      epochs=5,
                      batch_size=batch_size,
                      steps_per_epoch=60000 // batch_size,
                      validation_data=val_data_creator,
                      validation_steps=10000 // batch_size)

eval_stats = est.evaluate(val_data_creator, num_steps=10000 // batch_size)
print(eval_stats)
```

### Step 5: Save and Load the Model

Orca TensorFlow 2 Estimator supports two formats to save and load the entire model (**TensorFlow SavedModel and Keras H5 Format**). The recommended format is SavedModel, which is the default format when you use `estimator.save()`.

You could also save the model to Keras H5 format by passing `save_format='h5'` or a filename that ends in `.h5` or `.keras` to `estimator.save()`.

**Note that if you run on Apache Hadoop/YARN cluster, you are recommended to save the model to HDFS and load from HDFS as well.**

**1. SavedModel Format**

```python
# save model in SavedModel format
est.save("lenet_model")

# load model
est.load("lenet_model")
```

**2. HDF5 format**

```python
# save model in H5 format
est.save("lenet_model.h5", save_format='h5')

# load model
est.load("lenet_model.h5")
```

**Note:** You should call `stop_orca_context()` when your program finishes.

That's it, the same code can run seamlessly on your local laptop and scale to [Kubernetes](../Tutorial/k8s.md) or [Hadoop/YARN](../Tutorial/yarn.md) clusters.
