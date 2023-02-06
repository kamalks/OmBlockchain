# DLLib Scala Getting Start Guide

## 1. Creating dev environment

#### Scala project (maven & sbt)

- **Maven**

  To use BigDL DLLib to build your own deep learning application, you can use maven to create your project and add bigdl-dllib to your dependency. Please add below code to your pom.xml to add BigDL DLLib as your dependency:
  ```
  <dependency>
      <groupId>com.intel.analytics.bigdl</groupId>
      <artifactId>bigdl-dllib-spark_2.4.6</artifactId>
      <version>0.14.0</version>
  </dependency>
  ```

- **SBT**
  ```
  libraryDependencies += "com.intel.analytics.bigdl" % "bigdl-dllib-spark_2.4.6" % "0.14.0"
  ```
  For more information about how to add BigDL dependency, please refer [scala docs](../../UserGuide/scala.md#build-a-scala-project)

#### IDE (Intelij)
Open up IntelliJ and click File => Open

Navigate to your project. If you have add BigDL DLLib as dependency in your pom.xml.
The IDE will automatically download it from maven and you are able to run your application.

For more details about how to setup IDE for BigDL project, please refer [IDE Setup Guide](../../UserGuide/develop.html#id2)


## 2. Code initialization
```NNContext``` is the main entry for provisioning the dllib program on the underlying cluster (such as K8s or Hadoop cluster), or just on a single laptop.

It is recommended to initialize `NNContext` at the beginning of your program:
```
import com.intel.analytics.bigdl.dllib.NNContext
import com.intel.analytics.bigdl.dllib.keras.Model
import com.intel.analytics.bigdl.dllib.keras.models.Models
import com.intel.analytics.bigdl.dllib.keras.optimizers.Adam
import com.intel.analytics.bigdl.dllib.nn.ClassNLLCriterion
import com.intel.analytics.bigdl.dllib.utils.Shape
import com.intel.analytics.bigdl.dllib.keras.layers._
import com.intel.analytics.bigdl.numeric.NumericFloat
import org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.sql.SQLContext
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types.DoubleType

val sc = NNContext.initNNContext("dllib_demo")
```
For more information about ```NNContext```, please refer to [NNContext](../Overview/dllib.md#initialize-nn-context)

## 3. Distributed Data Loading

#### Using Spark Dataframe APIs
DLlib supports Spark Dataframes as the input to the distributed training, and as
the input/output of the distributed inference. Consequently, the user can easily
process large-scale dataset using Apache Spark, and directly apply AI models on
the distributed (and possibly in-memory) Dataframes without data conversion or serialization

We create Spark session so we can use Spark API to load and process the data
```
val spark = new SQLContext(sc)
```

1. We can use Spark API to load the data into Spark DataFrame, eg. read csv file into Spark DataFrame
   ```
   val path = "pima-indians-diabetes.data.csv"
   val df = spark.read.options(Map("inferSchema"->"true","delimiter"->",")).csv(path)
         .toDF("num_times_pregrant", "plasma_glucose", "blood_pressure", "skin_fold_thickness", "2-hour_insulin", "body_mass_index", "diabetes_pedigree_function", "age", "class")
   ```

   If the feature column for the model is a Spark ML Vector. Please assemble related columns into a Vector and pass it to the model. eg.
   ```
   val assembler = new VectorAssembler()
     .setInputCols(Array("num_times_pregrant", "plasma_glucose", "blood_pressure", "skin_fold_thickness", "2-hour_insulin", "body_mass_index", "diabetes_pedigree_function", "age"))
     .setOutputCol("features")
   val assembleredDF = assembler.transform(df)
   val df2 = assembleredDF.withColumn("label",col("class").cast(DoubleType) + lit(1))
   ```

2. If the training data is image, we can use DLLib api to load image into Spark DataFrame. Eg.
   ```
   val createLabel = udf { row: Row =>
   if (new Path(row.getString(0)).getName.contains("cat")) 1 else 2
   }
   val imagePath = "cats_dogs/"
   val imgDF = NNImageReader.readImages(imagePath, sc)
   ```

   It will load the images and generate feature tensors automatically. Also we need generate labels ourselves. eg:
   ```
   val df = imgDF.withColumn("label", createLabel(col("image")))
   ```

   Then split the Spark DataFrame into traing part and validation part
   ```
   val Array(trainDF, valDF) = df.randomSplit(Array(0.8, 0.2))
   ```

## 4. Model Definition

#### Using Keras-like APIs

To define a model, you can use the [Keras Style API](../Overview/keras-api.md).
```
val x1 = Input(Shape(8))
val dense1 = Dense(12, activation="relu").inputs(x1)
val dense2 = Dense(8, activation="relu").inputs(dense1)
val dense3 = Dense(2).inputs(dense2)
val dmodel = Model(x1, dense3)
```

After creating the model, you will have to decide which loss function to use in training.

Now you can use `compile` function of the model to set the loss function, optimization method.
```
dmodel.compile(optimizer = new Adam(), loss = ClassNLLCriterion())
```

Now the model is built and ready to train.

## 5. Distributed Model Training
Now you can use 'fit' begin the training, please set the label columns. Model Evaluation can be performed periodically during a training.
1. If the dataframe is generated using Spark apis, you also need set the feature columns. eg.
   ```
   model.fit(x=trainDF, batchSize=4, nbEpoch = 2,
     featureCols = Array("feature1"), labelCols = Array("label"), valX=valDF)
   ```
   Note: Above model accepts single input(column `feature1`) and single output(column `label`).

   If your model accepts multiple inputs(eg. column `f1`, `f2`, `f3`), please set the features as below:
   ```
   model.fit(x=dataframe, batchSize=4, nbEpoch = 2,
     featureCols = Array("f1", "f2", "f3"), labelCols = Array("label"))
   ```

   Similarly, if the model accepts multiple outputs(eg. column `label1`, `label2`), please set the label columns as below:
   ```
   model.fit(x=dataframe, batchSize=4, nbEpoch = 2,
     featureCols = Array("f1", "f2", "f3"), labelCols = Array("label1", "label2"))
   ```

2. If the dataframe is generated using DLLib `NNImageReader`, we don't need set `featureCols`, we can set `transform` to config how to process the images before training. Eg.
   ```
   val transformers = transforms.Compose(Array(ImageResize(50, 50),
     ImageMirror()))
   model.fit(x=dataframe, batchSize=4, nbEpoch = 2,
     labelCols = Array("label"), transform = transformers)
   ```
   For more details about how to use DLLib keras api to train image data, you may want to refer [ImageClassification](https://github.com/intel-analytics/BigDL/blob/main/scala/dllib/src/main/scala/com/intel/analytics/bigdl/dllib/example/keras/ImageClassification.scala)

## 6. Model saving and loading
When training is finished, you may need to save the final model for later use.

BigDL allows you to save your BigDL model on local filesystem, HDFS, or Amazon s3.
- **save**
  ```
  val modelPath = "/tmp/demo/keras.model"
  dmodel.saveModel(modelPath)
  ```

- **load**
  ```
  val loadModel = Models.loadModel(modelPath)

  val preDF2 = loadModel.predict(valDF, featureCols = Array("features"), predictionCol = "predict")
  ```

You may want to refer [Save/Load](../Overview/keras-api.html#save)

## 7. Distributed evaluation and inference
After training finishes, you can then use the trained model for prediction or evaluation.

- **inference**
  1. For dataframe generated by Spark API, please set `featureCols`
     ```
     dmodel.predict(trainDF, featureCols = Array("features"), predictionCol = "predict")
     ```
  2. For dataframe generated by `NNImageReader`, no need to set `featureCols` and you can set `transform` if needed
     ```
     model.predict(imgDF, predictionCol = "predict", transform = transformers)
     ```

- **evaluation**

  Similary for dataframe generated by Spark API, the code is as below:
  ```
  dmodel.evaluate(trainDF, batchSize = 4, featureCols = Array("features"),
    labelCols = Array("label"))
  ```

  For dataframe generated by `NNImageReader`:
  ```
  model.evaluate(imgDF, batchSize = 1, labelCols = Array("label"), transform = transformers)
  ```

## 8. Checkpointing and resuming training
You can configure periodically taking snapshots of the model.
```
val cpPath = "/tmp/demo/cp"
dmodel.setCheckpoint(cpPath, overWrite=false)
```
You can also set ```overWrite``` to ```true``` to enable overwriting any existing snapshot files

After training stops, you can resume from any saved point. Choose one of the model snapshots to resume (saved in checkpoint path, details see Checkpointing). Use Models.loadModel to load the model snapshot into an model object.
```
val loadModel = Models.loadModel(path)
```

## 9. Monitor your training

- **Tensorboard**

  BigDL provides a convenient way to monitor/visualize your training progress. It writes the statistics collected during training/validation. Saved summary can be viewed via TensorBoard.

  In order to take effect, it needs to be called before fit.
  ```
  dmodel.setTensorBoard("./", "dllib_demo")
  ```
  For more details, please refer [visulization](../Overview/visualization.md)`

## 10. Transfer learning and finetuning

- **freeze and trainable**

  BigDL DLLib supports exclude some layers of model from training.
  ```
  dmodel.freeze(layer_names)
  ```
  Layers that match the given names will be freezed. If a layer is freezed, its parameters(weight/bias, if exists) are not changed in training process.

  BigDL DLLib also support unFreeze operations. The parameters for the layers that match the given names will be trained(updated) in training process
  ```
  dmodel.unFreeze(layer_names)
  ```
  For more information, you may refer [freeze](../../PythonAPI/DLlib/freeze.md)

## 11. Hyperparameter tuning
- **optimizer**

  DLLib supports a list of optimization methods.
  For more details, please refer [optimization](../../PythonAPI/DLlib/optim-Methods.md)

- **learning rate scheduler**

  DLLib supports a list of learning rate scheduler.
  For more details, please refer [lr_scheduler](../../PythonAPI/DLlib/learningrate-Scheduler.md)

- **batch size**

  DLLib supports set batch size during training and prediction. We can adjust the batch size to tune the model's accuracy.

- **regularizer**

  DLLib supports a list of regularizers.
  For more details, please refer [regularizer](../../PythonAPI/DLlib/regularizers.md)

- **clipping**

  DLLib supports gradient clipping operations.
  For more details, please refer [gradient_clip](../../PythonAPI/DLlib/clipping.md)

## 12. Running program
You can run a bigdl-dllib program as a standard Spark program (running on either a local machine or a distributed cluster) as follows:
```
# Spark local mode
${BIGDL_HOME}/bin/spark-submit-with-dllib.sh \
  --master local[2] \
  --class class_name \
  jar_path

# Spark standalone mode
## ${SPARK_HOME}/sbin/start-master.sh
## check master URL from http://localhost:8080
${BIGDL_HOME}/bin/spark-submit-with-dllib.sh \
  --master spark://... \
  --executor-cores cores_per_executor \
  --total-executor-cores total_cores_for_the_job \
  --class class_name \
  jar_path

# Spark yarn client mode
${BIGDL_HOME}/bin/spark-submit-with-dllib.sh \
 --master yarn \
 --deploy-mode client \
 --executor-cores cores_per_executor \
 --num-executors executors_number \
 --class class_name \
 jar_path

# Spark yarn cluster mode
${BIGDL_HOME}/bin/spark-submit-with-dllib.sh \
 --master yarn \
 --deploy-mode cluster \
 --executor-cores cores_per_executor \
 --num-executors executors_number \
 --class class_name
 jar_path
```
For more detail about how to run BigDL scala application, please refer to [Scala UserGuide](../../UserGuide/scala.md)
