# Anomaly Detection

Anomaly Detection detects abnormal samples in a given time series. _Chronos_ provides a set of unsupervised anomaly detectors.

View some examples notebooks for [Datacenter AIOps][AIOps].

## 1. ThresholdDetector

ThresholdDetector detects anomaly based on threshold. It can be used to detect anomaly on a given time series ([notebook][AIOps_anomaly_detect_unsupervised]), or used together with [Forecasters](#forecasting) to detect anomaly on new coming samples ([notebook][AIOps_anomaly_detect_unsupervised_forecast_based]).

View [ThresholdDetector API Doc](../../PythonAPI/Chronos/anomaly_detectors.html#chronos-model-anomaly-th-detector) for more details.


## 2. AEDetector

AEDetector detects anomaly based on the reconstruction error of an autoencoder network.

View anomaly detection [notebook][AIOps_anomaly_detect_unsupervised] and [AEDetector API Doc](../../PythonAPI/Chronos/anomaly_detectors.html#chronos-model-anomaly-ae-detector) for more details.

## 3. DBScanDetector

DBScanDetector uses DBSCAN clustering algortihm for anomaly detection.

```eval_rst
.. note::
     Users may install ``scikit-learn-intelex`` to accelerate this detector. Chronos will detect if ``scikit-learn-intelex`` is installed to decide if using it. More details please refer to: https://intel.github.io/scikit-learn-intelex/installation.html
```

View anomaly detection [notebook][AIOps_anomaly_detect_unsupervised] and [DBScanDetector API Doc](../../PythonAPI/Chronos/anomaly_detectors.html#chronos-model-anomaly-dbscan-detector) for more details.


[AIOps]:<https://github.com/intel-analytics/BigDL/tree/main/python/chronos/use-case/AIOps>
[AIOps_anomaly_detect_unsupervised]:<https://github.com/intel-analytics/BigDL/blob/main/python/chronos/use-case/AIOps/AIOps_anomaly_detect_unsupervised.ipynb>
[AIOps_anomaly_detect_unsupervised_forecast_based]:<https://github.com/intel-analytics/BigDL/blob/main/python/chronos/use-case/AIOps/AIOps_anomaly_detect_unsupervised_forecast_based.ipynb>