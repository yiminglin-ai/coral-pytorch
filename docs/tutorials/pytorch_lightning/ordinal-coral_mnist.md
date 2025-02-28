# A Convolutional Neural Net for Ordinal Regression using CORAL -- MNIST Dataset

In this tutorial, we implement a convolutional neural network for ordinal regression based on the CORAL method. To learn more about CORAL, please have a look at our paper:

- Wenzhi Cao, Vahid Mirjalili, and Sebastian Raschka (2020): 
[Rank Consistent Ordinal Regression for Neural Networks with Application to Age Estimation](https://www.sciencedirect.com/science/article/pii/S016786552030413X). 
Pattern Recognition Letters. 140, 325-331


Please note that **MNIST is not an ordinal dataset**. The reason why we use MNIST in this tutorial is that it is included in the PyTorch's `torchvision` library and is thus easy to work with, since it doesn't require extra data downloading and preprocessing steps.

## General settings and hyperparameters

- Here, we specify some general hyperparameter values and general settings
- Note that for small datatsets, it is not necessary and better not to use multiple workers as it can sometimes cause issues with too many open files in PyTorch. So, if you have problems with the data loader later, try setting `NUM_WORKERS = 0` instead.


```python
BATCH_SIZE = 256
NUM_EPOCHS = 20
LEARNING_RATE = 0.005
NUM_WORKERS = 4

DATA_BASEPATH = "./data"
```

## Converting a regular classifier into a CORAL ordinal regression model

Changing a classifier to a CORAL model for ordinal regression is actually really simple and only requires a few changes:

**1)**
We replace the output layer  

```python
output_layer = torch.nn.Linear(hidden_units[-1], num_classes)
```

by a CORAL layer (available through `coral_pytorch`):

```python
output_layer = CoralLayer(size_in=hidden_units[-1], num_classes=num_classes)`
```

**2)**

Convert the integer class labels into the extended binary label format using the `levels_from_labelbatch` provided via `coral_pytorch`:

```python
levels = levels_from_labelbatch(class_labels, 
                                num_classes=num_classes)
```

**3)** 

Swap the cross entropy loss from PyTorch,

```python
torch.nn.functional.cross_entropy(logits, true_labels)
```

with the CORAL loss (also provided via `coral_pytorch`):

```python
loss = coral_loss(logits, levels)
```

**4)**

In a regular classifier, we usually obtain the predicted class labels as follows:

```python
predicted_labels = torch.argmax(logits, dim=1)
```

Replace this with the following code to convert the predicted probabilities into the predicted labels:

```python
predicted_labels = proba_to_label(probas)
```

## Implementing a `ConvNet` using PyTorch Lightning's `LightningModule`

- In this section, we set up the main model architecture using the `LightningModule` from PyTorch Lightning.
- We start with defining our convolutional neural network `ConvNet` model in pure PyTorch, and then we use it in the `LightningModule` to get all the extra benefits that PyTorch Lightning provides.
- Given a multilayer perceptron classifier with cross-entropy loss, it is very easy to change this classifier into a ordinal regression model using CORAL as explained in the previous section. In the code example below, we use "1) the `CoralLayer`".


```python
import torch
from coral_pytorch.layers import CoralLayer


# Regular PyTorch Module
class ConvNet(torch.nn.Module):
    def __init__(self, in_channels, num_classes):
        super().__init__()

        # num_classes is used by the corn loss function
        self.num_classes = num_classes

        # Initialize CNN layers
        all_layers = [
            torch.nn.Conv2d(in_channels=in_channels, out_channels=3, 
                            kernel_size=(3, 3), stride=(1, 1), 
                            padding=1),
            torch.nn.MaxPool2d(kernel_size=(2, 2),  stride=(2, 2)),
            torch.nn.Conv2d(in_channels=3, out_channels=6, 
                            kernel_size=(3, 3), stride=(1, 1), 
                            padding=1),
            torch.nn.MaxPool2d(kernel_size=(2, 2),  stride=(2, 2)),
            torch.nn.Flatten()
        ]

        # CORAL: output layer -------------------------------------------
        # Regular classifier would use the following output layer:
        # output_layer = torch.nn.Linear(294, num_classes)
        
        # We replace it by the CORAL layer:
        output_layer = CoralLayer(size_in=294,
                                  num_classes=num_classes)
        # ----------------------------------------------------------------
        
        all_layers.append(output_layer)
        self.model = torch.nn.Sequential(*all_layers)
        
    def forward(self, x):
        x = self.model(x)
        return x
```

- In our `LightningModule` we use loggers to track mean absolute errors for both the training and validation set during training; this allows us to select the best model based on validation set performance later.
- Note that we make changes 2) (`levels_from_labelbatch`), 3) (`coral_loss`), and 4) (`proba_to_label`) to implement a CORAL model instead of a regular classifier:


```python
from coral_pytorch.losses import coral_loss
from coral_pytorch.dataset import levels_from_labelbatch
from coral_pytorch.dataset import proba_to_label

import pytorch_lightning as pl
import torchmetrics


# LightningModule that receives a PyTorch model as input
class LightningCNN(pl.LightningModule):
    def __init__(self, model, learning_rate):
        super().__init__()

        self.learning_rate = learning_rate
        # The inherited PyTorch module
        self.model = model

        # Save settings and hyperparameters to the log directory
        # but skip the model parameters
        self.save_hyperparameters(ignore=['model'])

        # Set up attributes for computing the MAE
        self.train_mae = torchmetrics.MeanAbsoluteError()
        self.valid_mae = torchmetrics.MeanAbsoluteError()
        self.test_mae = torchmetrics.MeanAbsoluteError()
        
    # Defining the forward method is only necessary 
    # if you want to use a Trainer's .predict() method (optional)
    def forward(self, x):
        return self.model(x)
        
    # A common forward step to compute the loss and labels
    # this is used for training, validation, and testing below
    def _shared_step(self, batch):
        features, true_labels = batch
        logits = self(features)

        # Convert class labels for CORAL ------------------------
        levels = levels_from_labelbatch(
            true_labels, num_classes=self.model.num_classes).type_as(logits)
        # -------------------------------------------------------

        logits = self(features)

        # CORAL Loss --------------------------------------------
        # A regular classifier uses:
        # loss = torch.nn.functional.cross_entropy(logits, true_labels)
        loss = coral_loss(logits, levels)
        # -------------------------------------------------------

        # CORAL Prediction to label -----------------------------
        # A regular classifier uses:
        # predicted_labels = torch.argmax(logits, dim=1)
        probas = torch.sigmoid(logits)
        predicted_labels = proba_to_label(probas)
        # -------------------------------------------------------
        return loss, true_labels, predicted_labels

    def training_step(self, batch, batch_idx):
        loss, true_labels, predicted_labels = self._shared_step(batch)
        self.log("train_loss", loss)
        self.train_mae(predicted_labels, true_labels)
        self.log("train_mae", self.train_mae, on_epoch=True, on_step=False)
        return loss  # this is passed to the optimzer for training

    def validation_step(self, batch, batch_idx):
        loss, true_labels, predicted_labels = self._shared_step(batch)
        self.log("valid_loss", loss)
        self.valid_mae(predicted_labels, true_labels)
        self.log("valid_mae", self.valid_mae,
                 on_epoch=True, on_step=False, prog_bar=True)

    def test_step(self, batch, batch_idx):
        loss, true_labels, predicted_labels = self._shared_step(batch)
        self.test_mae(predicted_labels, true_labels)
        self.log("test_mae", self.test_mae, on_epoch=True, on_step=False)

    def configure_optimizers(self):
        optimizer = torch.optim.Adam(self.parameters(), lr=self.learning_rate)
        return optimizer
```


    ---------------------------------------------------------------------------

    ModuleNotFoundError                       Traceback (most recent call last)

    Input In [4], in <cell line: 5>()
          2 from coral_pytorch.dataset import levels_from_labelbatch
          3 from coral_pytorch.dataset import proba_to_label
    ----> 5 import pytorch_lightning as pl
          6 import torchmetrics
          9 # LightningModule that receives a PyTorch model as input


    File ~/conda/lib/python3.8/site-packages/pytorch_lightning/__init__.py:20, in <module>
         17 _PACKAGE_ROOT = os.path.dirname(__file__)
         18 _PROJECT_ROOT = os.path.dirname(_PACKAGE_ROOT)
    ---> 20 from pytorch_lightning.callbacks import Callback  # noqa: E402
         21 from pytorch_lightning.core import LightningDataModule, LightningModule  # noqa: E402
         22 from pytorch_lightning.trainer import Trainer  # noqa: E402


    File ~/conda/lib/python3.8/site-packages/pytorch_lightning/callbacks/__init__.py:14, in <module>
          1 # Copyright The PyTorch Lightning team.
          2 #
          3 # Licensed under the Apache License, Version 2.0 (the "License");
       (...)
         12 # See the License for the specific language governing permissions and
         13 # limitations under the License.
    ---> 14 from pytorch_lightning.callbacks.base import Callback
         15 from pytorch_lightning.callbacks.device_stats_monitor import DeviceStatsMonitor
         16 from pytorch_lightning.callbacks.early_stopping import EarlyStopping


    File ~/conda/lib/python3.8/site-packages/pytorch_lightning/callbacks/base.py:26, in <module>
         23 from torch.optim import Optimizer
         25 import pytorch_lightning as pl
    ---> 26 from pytorch_lightning.utilities.types import STEP_OUTPUT
         29 class Callback(abc.ABC):
         30     r"""
         31     Abstract base class used to build new callbacks.
         32 
         33     Subclass this class and override any of the relevant hooks
         34     """


    File ~/conda/lib/python3.8/site-packages/pytorch_lightning/utilities/__init__.py:18, in <module>
         14 """General utilities."""
         16 import numpy
    ---> 18 from pytorch_lightning.utilities.apply_func import move_data_to_device  # noqa: F401
         19 from pytorch_lightning.utilities.distributed import AllGatherGrad, rank_zero_info, rank_zero_only  # noqa: F401
         20 from pytorch_lightning.utilities.enums import (  # noqa: F401
         21     AMPType,
         22     DeviceType,
       (...)
         26     ModelSummaryMode,
         27 )


    File ~/conda/lib/python3.8/site-packages/pytorch_lightning/utilities/apply_func.py:30, in <module>
         28 if _TORCHTEXT_AVAILABLE:
         29     if _compare_version("torchtext", operator.ge, "0.9.0"):
    ---> 30         from torchtext.legacy.data import Batch
         31     else:
         32         from torchtext.data import Batch


    ModuleNotFoundError: No module named 'torchtext.legacy'


## Setting up the dataset

- In this section, we are going to set up our dataset.
- Please note that **MNIST is not an ordinal dataset**. The reason why we use MNIST in this tutorial is that it is included in the PyTorch's `torchvision` library and is thus easy to work with, since it doesn't require extra data downloading and preprocessing steps.

### Inspecting the dataset


```python
import torch

from torchvision import datasets
from torchvision import transforms
from torch.utils.data import DataLoader


train_dataset = datasets.MNIST(root=DATA_BASEPATH, 
                               train=True, 
                               transform=transforms.ToTensor(),
                               download=True)

train_loader = DataLoader(dataset=train_dataset, 
                          batch_size=BATCH_SIZE, 
                          num_workers=NUM_WORKERS,
                          drop_last=True,
                          shuffle=True)

test_dataset = datasets.MNIST(root=DATA_BASEPATH, 
                              train=False,
                              transform=transforms.ToTensor())

test_loader = DataLoader(dataset=test_dataset, 
                         batch_size=BATCH_SIZE,
                         num_workers=NUM_WORKERS,
                         drop_last=False,
                         shuffle=False)

# Checking the dataset
all_train_labels = []
all_test_labels = []

for images, labels in train_loader:  
    all_train_labels.append(labels)
all_train_labels = torch.cat(all_train_labels)
    
for images, labels in test_loader:  
    all_test_labels.append(labels)
all_test_labels = torch.cat(all_test_labels)
```


```python
print('Training labels:', torch.unique(all_train_labels))
print('Training label distribution:', torch.bincount(all_train_labels))

print('\nTest labels:', torch.unique(all_test_labels))
print('Test label distribution:', torch.bincount(all_test_labels))
```

- Above, we can see that the dataset consists of 8 features, and there are 998 examples in total.
- The labels are in range from 1 (weakest) to 5 (strongest), and we normalize them to start at zero (hence, the normalized labels are in the range 0 to 4). 
- Notice also that the dataset is quite imbalanced.

### Performance baseline

- Especially for imbalanced datasets, it's quite useful to compute a performance baseline.
- In classification contexts, a useful baseline is to compute the accuracy for a scenario where the model always predicts the majority class -- you want your model to be better than that!
- Note that if you are intersted in a single number that minimized the dataset mean squared error (MSE), that's the mean; similary, the median is a number that minimzes the mean absolute error (MAE).
- So, if we use the mean absolute error, $\mathrm{MAE}=\frac{1}{N} \sum_{i=1}^{N}\left|y_{i}-\hat{y}_{i}\right|$, to evaluate the model, it is useful to compute the MAE pretending the predicted label is always the median:


```python
all_test_labels = all_test_labels.float()
avg_prediction = torch.median(all_test_labels)  # median minimizes MAE
baseline_mae = torch.mean(torch.abs(all_test_labels - avg_prediction))
print(f'Baseline MAE: {baseline_mae:.2f}')
```

- In other words, a model that would always predict the dataset median would achieve a MAE of 2.52. A model that has an MAE of > 2.52 is certainly a bad model.

### Setting up a `DataModule`

- There are three main ways we can prepare the dataset for Lightning. We can
  1. make the dataset part of the model;
  2. set up the data loaders as usual and feed them to the fit method of a Lightning Trainer -- the Trainer is introduced in the next subsection;
  3. create a LightningDataModule.
- Here, we are going to use approach 3, which is the most organized approach. The `LightningDataModule` consists of several self-explanatory methods as we can see below:



```python
import os

from torch.utils.data.dataset import random_split
from torch.utils.data import DataLoader


class DataModule(pl.LightningDataModule):
    def __init__(self, data_path='./'):
        super().__init__()
        self.data_path = data_path
        
    def prepare_data(self):
        datasets.MNIST(root=self.data_path,
                       download=True)
        return

    def setup(self, stage=None):
        # Note transforms.ToTensor() scales input images
        # to 0-1 range
        train = datasets.MNIST(root=self.data_path, 
                               train=True, 
                               transform=transforms.ToTensor(),
                               download=False)

        self.test = datasets.MNIST(root=self.data_path, 
                                   train=False, 
                                   transform=transforms.ToTensor(),
                                   download=False)

        self.train, self.valid = random_split(train, lengths=[55000, 5000])

    def train_dataloader(self):
        train_loader = DataLoader(dataset=self.train, 
                                  batch_size=BATCH_SIZE, 
                                  drop_last=True,
                                  shuffle=True,
                                  num_workers=NUM_WORKERS)
        return train_loader

    def val_dataloader(self):
        valid_loader = DataLoader(dataset=self.valid, 
                                  batch_size=BATCH_SIZE, 
                                  drop_last=False,
                                  shuffle=False,
                                  num_workers=NUM_WORKERS)
        return valid_loader

    def test_dataloader(self):
        test_loader = DataLoader(dataset=self.test, 
                                 batch_size=BATCH_SIZE, 
                                 drop_last=False,
                                 shuffle=False,
                                 num_workers=NUM_WORKERS)
        return test_loader
```

- Note that the `prepare_data` method is usually used for steps that only need to be executed once, for example, downloading the dataset; the `setup` method defines the the dataset loading -- if you run your code in a distributed setting, this will be called on each node / GPU. 
- Next, lets initialize the `DataModule`; we use a random seed for reproducibility (so that the data set is shuffled the same way when we re-execute this code):


```python
torch.manual_seed(1) 
data_module = DataModule(data_path=DATA_BASEPATH)
```

## Training the model using the PyTorch Lightning Trainer class

- Next, we initialize our CNN (`ConvNet`) model.
- Also, we define a call back so that we can obtain the model with the best validation set performance after training.
- PyTorch Lightning offers [many advanced logging services](https://pytorch-lightning.readthedocs.io/en/latest/extensions/logging.html) like Weights & Biases. Here, we will keep things simple and use the `CSVLogger`:


```python
from pytorch_lightning.callbacks import ModelCheckpoint
from pytorch_lightning.loggers import CSVLogger


pytorch_model = ConvNet(
    in_channels=1,
    num_classes=torch.unique(all_test_labels).shape[0])

lightning_model = LightningCNN(
    model=pytorch_model,
    learning_rate=LEARNING_RATE)

callbacks = [ModelCheckpoint(
    save_top_k=1, mode='min', monitor="valid_mae")]  # save top 1 model 
logger = CSVLogger(save_dir="logs/", name="cnn-coral-mnist")
```

- Now it's time to train our model:


```python
import time


trainer = pl.Trainer(
    max_epochs=NUM_EPOCHS,
    callbacks=callbacks,
    progress_bar_refresh_rate=50,  # recommended for notebooks
    accelerator="auto",  # Uses GPUs or TPUs if available
    devices="auto",  # Uses all available GPUs/TPUs if applicable
    logger=logger,
    deterministic=True,
    log_every_n_steps=10)

start_time = time.time()
trainer.fit(model=lightning_model, datamodule=data_module)

runtime = (time.time() - start_time)/60
print(f"Training took {runtime:.2f} min in total.")
```

## Evaluating the model

- After training, let's plot our training MAE and validation MAE using pandas, which, in turn, uses matplotlib for plotting (you may want to consider a [more advanced logger](https://pytorch-lightning.readthedocs.io/en/latest/extensions/logging.html) that does that for you):


```python
import pandas as pd


metrics = pd.read_csv(f"{trainer.logger.log_dir}/metrics.csv")

aggreg_metrics = []
agg_col = "epoch"
for i, dfg in metrics.groupby(agg_col):
    agg = dict(dfg.mean())
    agg[agg_col] = i
    aggreg_metrics.append(agg)

df_metrics = pd.DataFrame(aggreg_metrics)
df_metrics[["train_loss", "valid_loss"]].plot(
    grid=True, legend=True, xlabel='Epoch', ylabel='Loss')
df_metrics[["train_mae", "valid_mae"]].plot(
    grid=True, legend=True, xlabel='Epoch', ylabel='MAE')
```

- It's hard to tell what the best model (based on the lowest validation set MAE) is in this case, but no worries, the `trainer` saved this model automatically for us, we which we can load from the checkpoint via the `ckpt_path='best'` argument; below we use the `trainer` instance to evaluate the best model on the test set:


```python
trainer.test(model=lightning_model, datamodule=data_module, ckpt_path='best')
```

- The MAE of our model is quite good, especially compared to the 2.52 MAE baseline earlier.

## Predicting labels of new data

- You can use the `trainer.predict` method on a new `DataLoader` or `DataModule` to apply the model to new data.
- Alternatively, you can also manually load the best model from a checkpoint as shown below:


```python
path = trainer.checkpoint_callback.best_model_path
print(path)
```


```python
lightning_model = LightningCNN.load_from_checkpoint(
    path, model=pytorch_model)
lightning_model.eval();
```

- Note that our `ConvNet`, which is passed to `LightningCNN` requires input arguments. However, this is automatically being taken care of since we used `self.save_hyperparameters()` in `LightningCNN`'s `__init__` method.
- Now, below is an example applying the model manually. Here, pretend that the `test_dataloader` is a new data loader.


```python
test_dataloader = data_module.test_dataloader()

all_predicted_labels = []
for batch in test_dataloader:
    features, _ = batch
    logits = lightning_model(features)
    probas = torch.sigmoid(logits)
    predicted_labels = proba_to_label(probas)
    all_predicted_labels.append(predicted_labels)
    
all_predicted_labels = torch.cat(all_predicted_labels)
all_predicted_labels[:5]
```
