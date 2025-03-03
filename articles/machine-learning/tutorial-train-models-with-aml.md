---
title: "Tutorial: Train an example Jupyter Notebook"
titleSuffix: Azure Machine Learning
description: Use Azure Machine Learning to train an image classification model with scikit-learn in a cloud-based Python Jupyter Notebook. This tutorial is part one of two. 
services: machine-learning
ms.service: machine-learning
ms.subservice: core
ms.topic: tutorial

author: sdgilley
ms.author: sgilley
ms.date: 04/26/2021
ms.custom: seodec18, devx-track-python, contperf-fy21q4
#Customer intent: As a professional data scientist, I can build an image classification model with Azure Machine Learning by using Python in a Jupyter Notebook.
---

# Tutorial: Train an image classification model with an example Jupyter Notebook 

In this tutorial, you train a machine learning model on remote compute resources. You'll use the training and deployment workflow for Azure Machine Learning in a Python Jupyter Notebook.  You can then use the notebook as a template to train your own machine learning model with your own data. This tutorial is **part one of a two-part tutorial series**.  

This tutorial trains a simple logistic regression by using the [MNIST](http://yann.lecun.com/exdb/mnist/) dataset and [scikit-learn](https://scikit-learn.org) with Azure Machine Learning. MNIST is a popular dataset consisting of 70,000 grayscale images. Each image is a handwritten digit of 28 x 28 pixels, representing a number from zero to nine. The goal is to create a multi-class classifier to identify the digit a given image represents.

Learn how to take the following actions:

> [!div class="checklist"]
> * Set up your development environment.
> * Access and examine the data.
> * Train a simple logistic regression model on a remote cluster.
> * Review training results and register the best model.

You learn how to select a model and deploy it in [part two of this tutorial](tutorial-deploy-models-with-aml.md).

If you don't have an Azure subscription, create a free account before you begin. Try the [free or paid version of Azure Machine Learning](https://aka.ms/AMLFree) today.

>[!NOTE]
> Code in this article was tested with [Azure Machine Learning SDK](/python/api/overview/azure/ml/intro) version 1.13.0.

## Prerequisites

* Complete the [Quickstart: Get started with Azure Machine Learning](quickstart-create-resources.md) to:
    * Create a workspace.
    * Create a cloud-based compute instance to use for your development environment.
    * Create a cloud-based compute cluster to use for training your model.

## <a name="azure"></a>Run a notebook from your workspace

Azure Machine Learning includes a cloud notebook server in your workspace for an install-free and pre-configured experience. Use [your own environment](how-to-configure-environment.md#local) if you prefer to have control over your environment, packages, and dependencies.

 Follow along with this video or use the detailed steps to clone and run the tutorial notebook from your workspace.

> [!VIDEO https://www.microsoft.com/en-us/videoplayer/embed/RE4mTUr]

### <a name="clone"></a> Clone a notebook folder

You complete the following experiment setup and run steps in Azure Machine Learning studio. This consolidated interface includes machine learning tools to perform data science scenarios for data science practitioners of all skill levels.

1. Sign in to [Azure Machine Learning studio](https://ml.azure.com/).

1. Select your subscription and the workspace you created.

1. On the left, select **Notebooks**.

1. At the top, select the **Samples** tab.

1. Open the **Python** folder.

1. Open the folder with a version number on it. This number represents the current release for the Python SDK.

1. Select the **...** button at the right of the **tutorials** folder, and then select **Clone**.

    :::image type="content" source="media/tutorial-1st-experiment-sdk-setup/clone-tutorials.png" alt-text="Screenshot that shows the Clone tutorials folder.":::

1. A list of folders shows each user who accesses the workspace. Select your folder to clone the **tutorials**  folder there.

### <a name="open"></a> Open the cloned notebook

1. Open the **tutorials** folder that was cloned into your **User files** section.

    > [!IMPORTANT]
    > You can view notebooks in the **samples** folder but you can't run a notebook from there. To run a notebook, make sure you open the cloned version of the notebook in the **User Files** section.
    
1. Select the **img-classification-part1-training.ipynb** file in your **tutorials/image-classification-mnist-data** folder.

    :::image type="content" source="media/tutorial-1st-experiment-sdk-setup/expand-user-folder.png" alt-text="Screenshot that shows the Open tutorials folder.":::

1. On the top bar, select your compute instance to use to run the notebook.


The tutorial and accompanying **utils.py** file is also available on [GitHub](https://github.com/Azure/MachineLearningNotebooks/tree/master/tutorials) if you wish to use it on your own [local environment](how-to-configure-environment.md#local). If you aren't using the compute instance, run `pip install azureml-sdk[notebooks] azureml-opendatasets matplotlib` to install dependencies for this tutorial. 

> [!Important]
> The rest of this article contains the same content as you see in the notebook.  
>
> Switch to the Jupyter Notebook now if you want to read along as you run the code. 
> To run a single code cell in a notebook, click the code cell and hit **Shift+Enter**. Or, run the entire notebook by choosing **Run all** from the top toolbar.

## <a name="start"></a>Set up your development environment

All the setup for your development work can be accomplished in a Python notebook. Setup includes the following actions:

* Import Python packages.
* Connect to a workspace, so that your local computer can communicate with remote resources.
* Create an experiment to track all your runs.
* Create a remote compute target to use for training.

### Import packages

Import Python packages you need in this session. Also display the Azure Machine Learning SDK version:

```python
%matplotlib inline
import numpy as np
import matplotlib.pyplot as plt

import azureml.core
from azureml.core import Workspace

# check core SDK version number
print("Azure ML SDK Version: ", azureml.core.VERSION)
```

### Connect to a workspace

Create a workspace object from the existing workspace. `Workspace.from_config()` reads the file **config.json** and loads the details into an object named `ws`.  The compute instance has a copy of this file saved in its root directory.  If you run the code elsewhere, you'll need to [create the file](how-to-configure-environment.md#workspace).

```python
# load workspace configuration from the config.json file in the current folder.
ws = Workspace.from_config()
print(ws.name, ws.location, ws.resource_group, sep='\t')
```

>[!NOTE]
> You may be asked to authenticate to your workspace the first time you run the following code. Follow the on-screen instructions.

### Create an experiment

Create an experiment to track the runs in your workspace. A workspace can have multiple experiments:

```python
from azureml.core import Experiment
experiment_name = 'Tutorial-sklearn-mnist'

exp = Experiment(workspace=ws, name=experiment_name)
```

### Create or attach an existing compute target

By using Azure Machine Learning Compute, a managed service, data scientists can train machine learning models on clusters of Azure virtual machines. Examples include VMs with GPU support. In this tutorial, you create Azure Machine Learning Compute as your training environment. You will submit Python code to run on this VM later in the tutorial. 

The code below creates the compute clusters for you if they don't already exist in your workspace. It sets up a cluster that will scale down to 0 when not in use, and can scale up to a maximum of 4 nodes.

 **Creation of the compute target takes about five minutes.** If the compute resource is already in the workspace, the code uses it and skips the creation process.  

> [!TIP]
> If you created a compute cluster in the quickstart, make sure `compute_name` in the code below uses the same name.

```python
from azureml.core.compute import AmlCompute
from azureml.core.compute import ComputeTarget
import os

# choose a name for your cluster
compute_name = os.environ.get("AML_COMPUTE_CLUSTER_NAME", "cpu-cluster")
compute_min_nodes = os.environ.get("AML_COMPUTE_CLUSTER_MIN_NODES", 0)
compute_max_nodes = os.environ.get("AML_COMPUTE_CLUSTER_MAX_NODES", 4)

# This example uses CPU VM. For using GPU VM, set SKU to STANDARD_NC6
vm_size = os.environ.get("AML_COMPUTE_CLUSTER_SKU", "STANDARD_D2_V2")


if compute_name in ws.compute_targets:
    compute_target = ws.compute_targets[compute_name]
    if compute_target and type(compute_target) is AmlCompute:
        print('found compute target. just use it. ' + compute_name)
else:
    print('creating a new compute target...')
    provisioning_config = AmlCompute.provisioning_configuration(vm_size=vm_size,
                                                                min_nodes=compute_min_nodes,
                                                                max_nodes=compute_max_nodes)

    # create the cluster
    compute_target = ComputeTarget.create(
        ws, compute_name, provisioning_config)

    # can poll for a minimum number of nodes and for a specific timeout.
    # if no min node count is provided it will use the scale settings for the cluster
    compute_target.wait_for_completion(
        show_output=True, min_node_count=None, timeout_in_minutes=20)

    # For a more detailed view of current AmlCompute status, use get_status()
    print(compute_target.get_status().serialize())
```

You now have the necessary packages and compute resources to train a model in the cloud. 

## Explore data

Before you train a model, you need to understand the data that you use to train it. In this section you learn how to:

* Download the MNIST dataset.
* Display some sample images.

### Download the MNIST dataset

Use Azure Open Datasets to get the raw MNIST data files. [Azure Open Datasets](../open-datasets/overview-what-are-open-datasets.md) are curated public datasets that you can use to add scenario-specific features to machine learning solutions for more accurate models. Each dataset has a corresponding class, `MNIST` in this case, to retrieve the data in different ways.

This code retrieves the data as a `FileDataset` object, which is a subclass of `Dataset`. A `FileDataset` references single or multiple files of any format in your datastores or public urls. The class provides you with the ability to download or mount the files to your compute by creating a reference to the data source location. Additionally, you register the Dataset to your workspace for easy retrieval during training.

Follow the [how-to](how-to-create-register-datasets.md) to learn more about Datasets and their usage in the SDK.

```python
from azureml.core import Dataset
from azureml.opendatasets import MNIST

data_folder = os.path.join(os.getcwd(), 'data')
os.makedirs(data_folder, exist_ok=True)

mnist_file_dataset = MNIST.get_file_dataset()
mnist_file_dataset.download(data_folder, overwrite=True)

mnist_file_dataset = mnist_file_dataset.register(workspace=ws,
                                                 name='mnist_opendataset',
                                                 description='training and test dataset',
                                                 create_new_version=True)
```

### Display some sample images

Load the compressed files into `numpy` arrays. Then use `matplotlib` to plot 30 random images from the dataset with their labels above them. This step requires a `load_data` function that's included in an `utils.py` file. This file is included in the sample folder. Make sure it's placed in the same folder as this notebook. The `load_data` function simply parses the compressed files into numpy arrays.

```python
# make sure utils.py is in the same directory as this code
from utils import load_data
import glob


# note we also shrink the intensity values (X) from 0-255 to 0-1. This helps the model converge faster.
X_train = load_data(glob.glob(os.path.join(data_folder,"**/train-images-idx3-ubyte.gz"), recursive=True)[0], False) / 255.0
X_test = load_data(glob.glob(os.path.join(data_folder,"**/t10k-images-idx3-ubyte.gz"), recursive=True)[0], False) / 255.0
y_train = load_data(glob.glob(os.path.join(data_folder,"**/train-labels-idx1-ubyte.gz"), recursive=True)[0], True).reshape(-1)
y_test = load_data(glob.glob(os.path.join(data_folder,"**/t10k-labels-idx1-ubyte.gz"), recursive=True)[0], True).reshape(-1)


# now let's show some randomly chosen images from the traininng set.
count = 0
sample_size = 30
plt.figure(figsize=(16, 6))
for i in np.random.permutation(X_train.shape[0])[:sample_size]:
    count = count + 1
    plt.subplot(1, sample_size, count)
    plt.axhline('')
    plt.axvline('')
    plt.text(x=10, y=-10, s=y_train[i], fontsize=18)
    plt.imshow(X_train[i].reshape(28, 28), cmap=plt.cm.Greys)
plt.show()
```

A random sample of images displays:

![Random sample of images](./media/tutorial-train-models-with-aml/digits.png)

Now you have an idea of what these images look like and the expected prediction outcome.

## Train on a remote cluster

For this task, you submit the job to run on the remote training cluster you set up earlier.  To submit a job you:
* Create a directory
* Create a training script
* Create a script run configuration
* Submit the job

### Create a directory

Create a directory to deliver the necessary code from your computer to the remote resource.

```python
import os
script_folder = os.path.join(os.getcwd(), "sklearn-mnist")
os.makedirs(script_folder, exist_ok=True)
```

### Create a training script

To submit the job to the cluster, first create a training script. Run the following code to create the training script called `train.py` in the directory you just created.

```python
%%writefile $script_folder/train.py

import argparse
import os
import numpy as np
import glob

from sklearn.linear_model import LogisticRegression
import joblib

from azureml.core import Run
from utils import load_data

# let user feed in 2 parameters, the dataset to mount or download, and the regularization rate of the logistic regression model
parser = argparse.ArgumentParser()
parser.add_argument('--data-folder', type=str, dest='data_folder', help='data folder mounting point')
parser.add_argument('--regularization', type=float, dest='reg', default=0.01, help='regularization rate')
args = parser.parse_args()

data_folder = args.data_folder
print('Data folder:', data_folder)

# load train and test set into numpy arrays
# note we scale the pixel intensity values to 0-1 (by dividing it with 255.0) so the model can converge faster.
X_train = load_data(glob.glob(os.path.join(data_folder, '**/train-images-idx3-ubyte.gz'), recursive=True)[0], False) / 255.0
X_test = load_data(glob.glob(os.path.join(data_folder, '**/t10k-images-idx3-ubyte.gz'), recursive=True)[0], False) / 255.0
y_train = load_data(glob.glob(os.path.join(data_folder, '**/train-labels-idx1-ubyte.gz'), recursive=True)[0], True).reshape(-1)
y_test = load_data(glob.glob(os.path.join(data_folder, '**/t10k-labels-idx1-ubyte.gz'), recursive=True)[0], True).reshape(-1)

print(X_train.shape, y_train.shape, X_test.shape, y_test.shape, sep = '\n')

# get hold of the current run
run = Run.get_context()

print('Train a logistic regression model with regularization rate of', args.reg)
clf = LogisticRegression(C=1.0/args.reg, solver="liblinear", multi_class="auto", random_state=42)
clf.fit(X_train, y_train)

print('Predict the test set')
y_hat = clf.predict(X_test)

# calculate accuracy on the prediction
acc = np.average(y_hat == y_test)
print('Accuracy is', acc)

run.log('regularization rate', np.float(args.reg))
run.log('accuracy', np.float(acc))

os.makedirs('outputs', exist_ok=True)
# note file saved in the outputs folder is automatically uploaded into experiment record
joblib.dump(value=clf, filename='outputs/sklearn_mnist_model.pkl')
```

Notice how the script gets data and saves models:

+ The training script reads an argument to find the directory that contains the data. When you submit the job later, you point to the datastore for this argument:
```parser.add_argument('--data-folder', type=str, dest='data_folder', help='data directory mounting point')```

+ The training script saves your model into a directory named **outputs**. Anything written in this directory is automatically uploaded into your workspace. You access your model from this directory later in the tutorial. `joblib.dump(value=clf, filename='outputs/sklearn_mnist_model.pkl')`

+ The training script requires the file `utils.py` to load the dataset correctly. The following code copies `utils.py` into `script_folder` so that the file can be accessed along with the training script on the remote resource.

  ```python
  import shutil
  shutil.copy('utils.py', script_folder)
  ```

### Configure the training job

Create a [ScriptRunConfig](/python/api/azureml-core/azureml.core.scriptrunconfig) object to specify the configuration details of your training job, including your training script, environment to use, and the compute target to run on. Configure the ScriptRunConfig by specifying:

* The directory that contains your scripts. All the files in this directory are uploaded into the cluster nodes for execution.
* The compute target. In this case, you use the Azure Machine Learning compute cluster you created.
* The training script name, **train.py**.
* An environment that contains the libraries needed to run the script.
* Arguments required from the training script.

In this tutorial, this target is AmlCompute. All files in the script folder are uploaded into the cluster nodes for run. The **--data_folder** is set to use the dataset.

First, create the environment that contains: the scikit-learn library, azureml-dataset-runtime required for accessing the dataset, and azureml-defaults which contains the dependencies for logging metrics. The azureml-defaults also contains the dependencies required for deploying the model as a web service later in the part 2 of the tutorial.

Once the environment is defined, register it with the Workspace to re-use it in part 2 of the tutorial.

```python
from azureml.core.environment import Environment
from azureml.core.conda_dependencies import CondaDependencies

# to install required packages
env = Environment('tutorial-env')
cd = CondaDependencies.create(pip_packages=['azureml-dataset-runtime[pandas,fuse]', 'azureml-defaults'], conda_packages=['scikit-learn==0.22.1'])

env.python.conda_dependencies = cd

# Register environment to re-use later
env.register(workspace=ws)
```

Then, create the ScriptRunConfig by specifying the training script, compute target and environment.

```python
from azureml.core import ScriptRunConfig

args = ['--data-folder', mnist_file_dataset.as_mount(), '--regularization', 0.5]

src = ScriptRunConfig(source_directory=script_folder,
                      script='train.py', 
                      arguments=args,
                      compute_target=compute_target,
                      environment=env)
```

### Submit the job to the cluster

Run the experiment by submitting the ScriptRunConfig object:

```python
run = exp.submit(config=src)
run
```

Because the call is asynchronous, it returns a **Preparing** or **Running** state as soon as the job is started.

## Monitor a remote run

In total, the first run takes **about 10 minutes**. But for subsequent runs, as long as the script dependencies don't change, the same image is reused. So the container startup time is much faster.

What happens while you wait:

- **Image creation**: A Docker image is created that matches the Python environment specified by the Azure ML environment. The image is uploaded to the workspace. Image creation and uploading takes **about five minutes**.

  This stage happens once for each Python environment because the container is cached for subsequent runs. During image creation, logs are streamed to the run history. You can monitor the image creation progress by using these logs.

- **Scaling**: If the remote cluster requires more nodes to do the run than currently available, additional nodes are added automatically. Scaling typically takes **about five minutes.**

- **Running**: In this stage, the necessary scripts and files are sent to the compute target. Then datastores are mounted or copied. And then the **entry_script** is run. While the job is running, **stdout** and the **./logs** directory are streamed to the run history. You can monitor the run's progress by using these logs.

- **Post-processing**: The **./outputs** directory of the run is copied over to the run history in your workspace, so you can access these results.

You can check the progress of a running job in several ways. This tutorial uses a Jupyter widget and a `wait_for_completion` method.

### Jupyter widget

Watch the progress of the run with a [Jupyter widget](/python/api/azureml-widgets/azureml.widgets). Like the run submission, the widget is asynchronous and provides live updates every 10 to 15 seconds until the job finishes:

```python
from azureml.widgets import RunDetails
RunDetails(run).show()
```

The widget will look like the following at the end of training:

![Notebook widget](./media/tutorial-train-models-with-aml/widget.png)

If you need to cancel a run, you can follow [these instructions](./how-to-track-monitor-analyze-runs.md).

### Get log results upon completion

Model training and monitoring happen in the background. Wait until the model has finished training before you run more code. Use `wait_for_completion` to show when the model training is finished:

```python
run.wait_for_completion(show_output=False)  # specify True for a verbose log
```

### Display run results

You now have a model trained on a remote cluster. Retrieve the accuracy of the model:

```python
print(run.get_metrics())
```

The output shows the remote model has accuracy of 0.9204:

`{'regularization rate': 0.8, 'accuracy': 0.9204}`

In the next tutorial, you explore this model in more detail.

## Register model

The last step in the training script wrote the file `outputs/sklearn_mnist_model.pkl` in a directory named `outputs` in the VM of the cluster where the job is run. `outputs` is a special directory in that all content in this directory is automatically uploaded to your workspace. This content appears in the run record in the experiment under your workspace. So the model file is now also available in your workspace.

You can see files associated with that run:

```python
print(run.get_file_names())
```

Register the model in the workspace, so that you or other collaborators can later query, examine, and deploy this model:

```python
# register model
model = run.register_model(model_name='sklearn_mnist',
                           model_path='outputs/sklearn_mnist_model.pkl')
print(model.name, model.id, model.version, sep='\t')
```

## Clean up resources

[!INCLUDE [aml-delete-resource-group](../../includes/aml-delete-resource-group.md)]

You can also delete just the Azure Machine Learning Compute cluster. However, autoscale is turned on, and the cluster minimum is zero. So this particular resource won't incur additional compute charges when not in use:

```python
# Optionally, delete the Azure Machine Learning Compute cluster
compute_target.delete()
```

## Next steps

In this Azure Machine Learning tutorial, you used Python for the following tasks:

> [!div class="checklist"]
> * Set up your development environment.
> * Access and examine the data.
> * Train multiple models on a remote cluster using the popular scikit-learn machine learning library
> * Review training details and register the best model.

You're ready to deploy this registered model by using the instructions in the next part of the tutorial series:

> [!div class="nextstepaction"]
> [Tutorial 2 - Deploy models](tutorial-deploy-models-with-aml.md)
