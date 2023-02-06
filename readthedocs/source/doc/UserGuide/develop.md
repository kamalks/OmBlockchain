# Developer Guide

---

BigDL source code is available at [GitHub](https://github.com/intel-analytics/BigDL):

```bash
git clone https://github.com/intel-analytics/BigDL.git
```

By default, `git clone` will download the development version of BigDL. If you want a release version, you can use the command `git checkout` to change the specified version.


### 1. Python

#### 1.1 Build

To generate a new [whl](https://pythonwheels.com/) package for pip install, you can run the following script:

```bash
cd BigDL/python/dev
bash release_default_linux_spark246.sh default false false false  # build on Spark 2.4.6 for linux
# Use release_default_linux_spark313.sh to build on Spark 3.1.3 for linux
# Use release_default_mac_spark246.sh to build on Spark 2.4.6 for mac
# Use release_default_mac_spark313.sh to build on Spark 3.1.3 for mac
```

**Arguments:**

- The first argument is the BigDL __version__ to build for. 'default' means the default version (`BigDL/python/version.txt`) for the current branch. You can also specify a different version if you wish, e.g., '0.14.0.dev1'.
- The second argument is whether to __quick build__ BigDL Scala dependencies. You need to set it to be 'true' for the first build. In later builds, if you don't make any changes in BigDL Scala, you can set it to be 'false' so that the Scala dependencies would not be re-built.
- The third argument is whether to __upload__ the packages to pypi. Set it to 'false' if you are simply developing BigDL for your own usage.
- The fourth argument is whether to add __spark suffix__ (i.e. -spark2 or -spark3) to BigDL package names. Just set this to be 'false' if you are simply developing BigDL for your own usage.
- You can also add other Maven profiles to build the package (if any) after the fourth argument, for example '-Ddata-store-url=..', etc.


After running the above command, you will find a `whl` file for each submodule of BigDL and you can then directly pip install them to your local Python environment:
```bash
# Install bigdl-nano
cd BigDL/python/nano/src/dist
pip install bigdl_nano-*.whl

# Install bigdl-dllib
cd BigDL/python/dllib/src/dist
pip install bigdl_dllib-*.whl

# Install bigdl-orca, which depends on bigdl-dllib and you need to install bigdl-dllib first
cd BigDL/python/orca/src/dist
pip install bigdl_orca-*.whl

# Install bigdl-friesian, which depends on bigdl-orca and you need to install bigdl-dllib and bigdl-orca first
cd BigDL/python/friesian/src/dist
pip install bigdl_friesian-*.whl

# Install bigdl-chronos, which depends on bigdl-orca and bigdl-nano. You need to install bigdl-dllib, bigdl-orca and bigdl-nano first
cd BigDL/python/chronos/src/dist
pip install bigdl_chronos-*.whl

# Install bigdl-serving
cd BigDL/python/serving/src/dist
pip install bigdl_serving-*.whl
```

See [here](./python.md) for more instructions to run BigDL after pip install.


#### 1.2 IDE Setup
Any IDE that support Python should be able to run BigDL. PyCharm works fine for us.

You need to do the following preparations before starting the IDE to successfully run a BigDL Python program in the IDE:

- Build BigDL; see [here](#build) for more instructions.
- Prepare Spark environment by either setting `SPARK_HOME` as the environment variable or `pip install pyspark`. Note that the Spark version should match the one you build BigDL on.
- Check the jars under `BigDL/dist/lib` and set the environment variable `BIGDL_CLASSPATH`. Modify SPARKVERSION and BIGDLVERSION(Scala) as appropriate:
  ```bash
  export BIGDL_CLASSPATH=BigDL/dist/lib/bigdl-dllib-spark_SPARKVERSION-BIGDLVERSION-jar-with-dependencies.jar:BigDL/dist/lib/bigdl-orca-spark_SPARKVERSION-BIGDLVERSION-jar-with-dependencies.jar:BigDL/dist/lib/bigdl-friesian-spark_SPARKVERSION-BIGDLVERSION-jar-with-dependencies.jar
  ```
- Configure BigDL source files to the Python interpreter:

  You can easily do this after launching PyCharm by right clicking the folder `BigDL/python/dllib/src` -> __Mark Directory As__ -> __Sources Root__ (also do this for `BigDL/python/nano/src`, `BigDL/python/orca/src`, `BigDL/python/friesian/src`, `BigDL/python/chronos/src`, `BigDL/python/serving/src` if necessary).

  Alternatively, you can add BigDL source files to `PYTHONPATH`:
  ```bash
  export PYTHONPATH=BigDL/python/dllib/src:BigDL/python/nano/src:BigDL/python/orca/src:BigDL/python/friesian/src:BigDL/python/chronos/src:BigDL/python/serving/src:$PYTHONPATH
  ```

- Add `spark-bigdl.conf` to `PYTHONPATH`:
  ```bash
  export PYTHONPATH=BigDL/python/dist/conf/spark-bigdl.conf:$PYTHONPATH
  ```

- Install and add `tflibs` to `TF_LIBS_PATH`:
  ```bash
  # Install bigdl-tf and bigdl-math
  pip install bigdl-tf bigdl-math

  # Configure TF_LIBS_PATH
  export TF_LIBS_PATH=$(python -c 'import site; print(site.getsitepackages()[0])')/bigdl/share/tflibs
  ```


The above environment variables should be available when running or debugging code in the IDE. When running applications in PyCharm, you can add runtime environment variables by clicking  __Run__ -> __Edit Configurations__; then in the __Run/Debug Configurations__ panel, you can add necessary environment variables to your applications.


#### 1.3 Terminal Setup

Besides setting the environment variables mentioned above manually for Linux users, we also provide a solution to set them with a script:

```bash
# Install bigdl-tf and bigdl-math
pip install bigdl-tf bigdl-math

cd BigDL/python/friesian
source dev/prepare_env.sh
```

You can verify the BigDL environment by running the following example.

```bash
python BigDL/python/dllib/examples/autograd/custom.py
```

Note that this approach will only work temporarily for this terminal. 


### 2. Scala

#### 2.1 Build

Maven 3 is needed to build BigDL, you can download it from the [maven website](https://maven.apache.org/download.cgi).

After installing Maven 3, please set the environment variable MAVEN_OPTS as follows:
```bash
$ export MAVEN_OPTS="-Xmx2g -XX:ReservedCodeCacheSize=512m"
```

**Build using `make-dist.sh`**

It is highly recommended that you build BigDL using the [make-dist.sh script](https://github.com/intel-analytics/BigDL/blob/branch-2.0/scala/make-dist.sh) with **Java 8**.

You can build BigDL with the following commands:
```bash
$ cd scala
$ bash make-dist.sh
```
After that, you can find a `dist` folder, which contains all the needed files to run a BigDL program. The files in `dist` include:

* **dist/lib/bigdl-VERSION-jar-with-dependencies.jar**: This jar package contains all dependencies except Spark classes.
* **dist/lib/bigdl-VERSION-python-api.zip**: This zip package contains all Python files of BigDL.

The instructions above will build BigDL with Spark 2.4.6. To build with other spark versions, for example building analytics-zoo with spark 2.2.0, you can use `bash make-dist.sh -Dspark.version=2.2.0`.  

**Build with JDK 11**

Spark starts to supports JDK 11 and Scala 2.12 at Spark 3.0. You can use `-P spark_3.x` to specify Spark3 and scala 2.12. Additionally, `make-dist.sh` default uses Java 8. To compile with Java 11, it is required to specify building opts `-Djava.version=11 -Djavac.version=11`. You can build with `make-dist.sh`.

It's recommended to download [Oracle JDK 11](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html). This will avoid possible incompatibilities with maven plugins. You should update `PATH` and make sure your `JAVA_HOME` environment variable is set to Java 11 if you're running from the command line. If you're running from an IDE, you need to make sure it is set to run maven with your current JDK. 

Build with `make-dist.sh`:
 
```bash
$ bash make-dist.sh -P spark_3.x -Djava.version=11 -Djavac.version=11
```

#### 2.2 IDE Setup

BigDL uses maven to organize project. You should choose an IDE that supports Maven project and scala language. IntelliJ IDEA works fine for us.

In IntelliJ, you can open BigDL project root directly, and the IDE will import the project automatically. If not imported automatically, right click `scala/pom.xml` and choose `Add as Maven Project`.

We set the scopes of spark related libraries to `provided` in the maven pom.xml, which, however, will cause a problem in IDE  (throwing `NoClassDefFoundError` when you run applications). You can easily change the scopes using the `all-in-one` profile.

* In Intellij, go to View -> Tools Windows -> Maven Projects. Then in the Maven Projects panel, Profiles -> click "all-in-one".
