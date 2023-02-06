# Detect Anomaly Point in Real Time Traffic Data

---

![](../../../../image/colab_logo_32px.png)[Run in Google Colab][chronos_minn_traffic_anomaly_detector_colab] &nbsp;![](../../../../image/GitHub-Mark-32px.png)[View source on GitHub][chronos_minn_traffic_anomaly_detector]

---

**In this guide we will demonstrate how to use _Chronos Anomaly Detector_ for time seires anomaly detection in 3 simple steps.**

### Step 0: Prepare Environment

We recommend using [conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/) to prepare the environment. Please refer to the [install guide](../Overview/chronos.html#install) for more details.

```bash
conda create -n my_env python=3.7 # "my_env" is conda environment name, you can use any name you like.
conda activate my_env
pip install bigdl-chronos
```

## Step 1: Prepare dataset
For demonstration, we use the publicly available real time traffic data from the Twin Cities Metro area in Minnesota, collected by the Minnesota Department of Transportation. The detailed information can be found [here](https://github.com/numenta/NAB/blob/master/data/realTraffic/speed_7578.csv)

Now we need to do data cleaning and preprocessing on the raw data. Note that this part could vary for different dataset. 
For the machine_usage data, the pre-processing contains 2 parts: <br>
1. Change the time interval from irregular to 5 minutes.<br>
2. Check missing values and handle missing data.

```python
from bigdl.chronos.data import TSDataset

tsdata = TSDataset.from_pandas(df, dt_col="timestamp", target_col="value")
df = tsdata.resample("5min")\
           .impute(mode="linear")\
           .to_pandas()
```

## Step 2: Use Chronos Anomaly Detector
Chronos provides many anomaly detector for anomaly detection, here we use DBScan as an example. More anomaly detector can be found [here](../../PythonAPI/Chronos/anomaly_detectors.html).

```python
from bigdl.chronos.detector.anomaly import DBScanDetector

ad = DBScanDetector(eps=0.3, min_samples=6)
ad.fit(df['value'].to_numpy())
anomaly_indexes = ad.anomaly_indexes()
```

[chronos_minn_traffic_anomaly_detector_colab]: <https://colab.research.google.com/github/intel-analytics/BigDL/blob/main/python/chronos/colab-notebook/chronos_minn_traffic_anomaly_detector.ipynb>
[chronos_minn_traffic_anomaly_detector]: <https://github.com/intel-analytics/BigDL/blob/main/python/chronos/colab-notebook/chronos_minn_traffic_anomaly_detector.ipynb>