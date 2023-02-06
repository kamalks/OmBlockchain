# BigDL-Nano PyTorch ONNXRuntime Acceleration Quickstart

**In this guide we will describe how to apply ONNXRuntime Acceleration on inference pipeline with the APIs delivered by BigDL-Nano in 4 simple steps**

### Step 0: Prepare Environment
We recommend using [conda](https://docs.conda.io/projects/conda/en/latest/user-guide/install/) to prepare the environment. Please refer to the [install guide](../../UserGuide/python.md) for more details.

```bash
conda create -n py37 python==3.7.10 setuptools==58.0.4
conda activate py37
# nightly bulit version
pip install --pre --upgrade bigdl-nano[pytorch]
# set env variables for your conda environment
source bigdl-nano-init
```

Before you start with ONNXRuntime accelerator, you need to install some ONNX packages as follows to set up your environment with ONNXRuntime acceleration.
```bash
pip install onnx onnxruntime
```
### Step 1: Load the data
```python
import torch
from torchvision.io import read_image
from torchvision import transforms
from torchvision.datasets import OxfordIIITPet
from torch.utils.data.dataloader import DataLoader

train_transform = transforms.Compose([transforms.Resize(256),
                                      transforms.RandomCrop(224),
                                      transforms.RandomHorizontalFlip(),
                                      transforms.ColorJitter(brightness=.5, hue=.3),
                                      transforms.ToTensor(),
                                      transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])])
val_transform = transforms.Compose([transforms.Resize([224, 224]), transforms.ToTensor(), transforms.Normalize([0.485, 0.456, 0.406], [0.229, 0.224, 0.225])])
# Apply data augmentation to the tarin_dataset
train_dataset = OxfordIIITPet(root = ".", transform=train_transform)
val_dataset = OxfordIIITPet(root=".", transform=val_transform)
# obtain training indices that will be used for validation
indices = torch.randperm(len(train_dataset))
val_size = len(train_dataset) // 4
train_dataset = torch.utils.data.Subset(train_dataset, indices[:-val_size])
val_dataset = torch.utils.data.Subset(val_dataset, indices[-val_size:])
# prepare data loaders
train_dataloader = DataLoader(train_dataset, batch_size=32)
```

### Step 2: Prepare the Model
```python
import torch
from torchvision.models import resnet18
from bigdl.nano.pytorch import Trainer
model_ft = resnet18(pretrained=True)
num_ftrs = model_ft.fc.in_features

# Here the size of each output sample is set to 37.
model_ft.fc = torch.nn.Linear(num_ftrs, 37)
loss_ft = torch.nn.CrossEntropyLoss()
optimizer_ft = torch.optim.SGD(model_ft.parameters(), lr=0.01, momentum=0.9, weight_decay=5e-4)

# Compile our model with loss function, optimizer.
model = Trainer.compile(model_ft, loss_ft, optimizer_ft)
trainer = Trainer(max_epochs=5)
trainer.fit(model, train_dataloader=train_dataloader)

# Inference/Prediction
x = torch.stack([val_dataset[0][0], val_dataset[1][0]])
model_ft.eval()
y_hat = model_ft(x)
y_hat.argmax(dim=1)
```

### Step 3: Apply ONNXRumtime Acceleration
When you're ready, you can simply append the following part to enable your ONNXRuntime acceleration.
```python
# trace your model as an ONNXRuntime model
# The argument `input_sample` is not required in the following cases:
# you have run `trainer.fit` before trace
# Model has `example_input_array` set
# Model is a LightningModule with any dataloader attached.
from bigdl.nano.pytorch import InferenceOptimizer
ort_model = InferenceOptimizer.trace(model_ft, accelerator="onnxruntime", input_sample=torch.rand(1, 3, 224, 224))

# The usage is almost the same with any PyTorch module
y_hat = ort_model(x)
y_hat.argmax(dim=1)
```

```eval_rst
.. note:: 
     ``ort_model`` is not trainable any more, so you cannot use it in ``fit`` such as ``trainer.fit(ort_model, dataloader)``.
```