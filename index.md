## Introduction

The hyperparameters are the parameters that not originate from the training data. In many scenarios, depending on the algorithm used you may need to specify hyperparameters to configure how the model is trained. For example, the logistic regression algorithm uses a regularization rate hyperparameter to counteract overfitting; and deep learning techniques for convolutional neural networks (CNNs) use hyperparameters like learning rate to control how weights are adjusted during training, and batch size to determine how many data items are included in each training batch.

The choice of hyperparameter values can significantly affect the resulting model, making it important to select the best possible values for your particular data and predictive performance goals.

For example, the rating rate on training model:

![image](https://user-images.githubusercontent.com/71245576/116608219-f296df00-a900-11eb-83a4-965d5bddf5bc.png)

In Azure Machine Learning, you achieve this through an experiment that consists of a hyperdrive run, which initiates a child run for each hyperparameter combination to be tested. Each child run uses a training script with parameterized hyperparameter values to train a model, and logs the target performance metric achieved by the trained model.

## 1. Define a hyperparameter search space

The set of hyperparameter values tried during hyperparameter tuning is known as the search space. The definition of the range of possible values that can be chosen depends on the type of hyperparameter.

There are two types of hyperparameters: discrete hyperparameters and continuous parameters. In using discrete hyperparameters, you must select the value from a particular set of possibilities. You can define a search space for a discrete parameter using a choice from a list of explicit values, which you can define as a Python list (choice([10,20,30])), a range (choice(range(1,10))), or an arbitrary set of comma-separated values (choice(30,50,100)) 

You can also select discrete values from any of the following discrete distributions: qnormal, quniform, qlognormal and qloguniform.

Some hyperparameters are continuous - in other words you can use any value along a scale. To define a search space for these kinds of value, you can use any of the following distribution types: normal, uniform, lognormal and loguniform.

Now let's define a search space for hyperparameter tuning, create a dictionary with the appropriate parameter expression for each named hyperparameter. For example, the following search space indicates that the batch_size hyperparameter can have the value 16, 32, or 64, and the learning_rate hyperparameter can have any value from a normal distribution with a mean of 10 and a standard deviation of 3.

```python
from azureml.train.hyperdrive import choice, normal

param_space = {
                 '--batch_size': choice(16, 32, 64),
                 '--learning_rate': normal(10, 3)
              }
```        

## 2. Configuring sampling

The specific values used in a hyperparameter tuning run depend on the type of sampling used.

Grid sampling can only be employed when all hyperparameters are discrete, and is used to try every possible combination of parameters in the search space.

For example, in the following code example, grid sampling is used to try every possible combination of discrete batch_size and learning_rate value:

```python
from azureml.train.hyperdrive import GridParameterSampling, choice

param_space = {
                 '--batch_size': choice(16, 32, 64),
                 '--learning_rate': choice(0.01, 0.1, 1.0)
              }

param_sampling = GridParameterSampling(param_space)
```
Random sampling is used to randomly select a value for each hyperparameter, which can be a mix of discrete and continuous values as shown in the following code example:
```python
from azureml.train.hyperdrive import RandomParameterSampling, choice, normal

param_space = {
                 '--batch_size': choice(16, 32, 64),
                 '--learning_rate': normal(10, 3)
              }

param_sampling = RandomParameterSampling(param_space)
```
Bayesian sampling chooses hyperparameter values based on the Bayesian optimization algorithm, which tries to select parameter combinations that will result in improved performance from the previous selection. The following code example shows how to configure Bayesian sampling:
```python
from azureml.train.hyperdrive import BayesianParameterSampling, choice, uniform

param_space = {
                 '--batch_size': choice(16, 32, 64),
                 '--learning_rate': uniform(0.05, 0.1)
              }

param_sampling = BayesianParameterSampling(param_space)
```
Note that you can only use Bayesian sampling with choice, uniform, and quniform parameter expressions, and you can't combine it with an early-termination policy.

## 3. Configuring early termination

With a sufficiently large hyperparameter search space, it could take many iterations (child runs) to try every possible combination. Typically, you set a maximum number of iterations, but this could still result in a large number of runs that don't result in a better model than a combination that has already been tried.

To help prevent wasting time, you can set an early termination policy that abandons runs that are unlikely to produce a better result than previously completed runs. The policy is evaluated at an evaluation_interval you specify, based on each time the target performance metric is logged. You can also set a delay_evaluation parameter to avoid evaluating the policy until a minimum number of iterations have been completed.

Early termination is particularly useful for deep learning scenarios where a deep neural network (DNN) is trained iteratively over a number of epochs. The training script can report the target metric after each epoch, and if the run is significantly underperforming previous runs after the same number of intervals, it can be abandoned.

### 3.1 Bandit policy

You can use a bandit policy to stop a run if the target performance metric underperforms the best run so far by a specified margin.
```python
from azureml.train.hyperdrive import BanditPolicy

early_termination_policy = BanditPolicy(slack_amount = 0.2,
                                        evaluation_interval=1,
                                        delay_evaluation=5)
```
This example applies the policy for every iteration after the first five, and abandons runs where the reported target metric is 0.2 or more worse than the best performing run after the same number of intervals.

You can also apply a bandit policy using a slack factor, which compares the performance metric as a ratio rather than an absolute value.

### 3.2 Median stopping policy

A median stopping policy abandons runs where the target performance metric is worse than the median of the running averages for all runs.
```python
from azureml.train.hyperdrive import MedianStoppingPolicy

early_termination_policy = MedianStoppingPolicy(evaluation_interval=1,
                                                delay_evaluation=5)
```
### 3.3 Truncation selection policy
A truncation selection policy cancels the lowest performing X% of runs at each evaluation interval based on the truncation_percentage value you specify for X.

```python
from azureml.train.hyperdrive import TruncationSelectionPolicy

early_termination_policy = TruncationSelectionPolicy(truncation_percentage=10,
                                                     evaluation_interval=1,
                                                     delay_evaluation=5)
```                                                     

## 4. Running tuning experiment

To run a hyperdrive experiment, you need to create a training script just the way you would do for any other training experiment, except that your script must: include an argument for each hyperparameter you want to vary and log the target performance metric. This enables the hyperdrive run to evaluate the performance of the child runs it initiates, and identify the one that produces the best performing model.

For example, the following example script trains a logistic regression model using a --regularization argument to set the regularization rate hyperparameter, and logs the accuracy metric with the name Accuracy:

```python
import argparse
import joblib
from azureml.core import Run
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split
from sklearn.linear_model import LogisticRegression

# Get regularization hyperparameter
parser = argparse.ArgumentParser()
parser.add_argument('--regularization', type=float, dest='reg_rate', default=0.01)
args = parser.parse_args()
reg = args.reg_rate

# Get the experiment run context
run = Run.get_context()

# load the training dataset
data = run.input_datasets['training_data'].to_pandas_dataframe()

# Separate features and labels, and split for training/validatiom
X = data[['feature1','feature2','feature3','feature4']].values
y = data['label'].values
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.30)

# Train a logistic regression model with the reg hyperparameter
model = LogisticRegression(C=1/reg, solver="liblinear").fit(X_train, y_train)

# calculate and log accuracy
y_hat = model.predict(X_test)
acc = np.average(y_hat == y_test)
run.log('Accuracy', np.float(acc))

# Save the trained model
os.makedirs('outputs', exist_ok=True)
joblib.dump(value=model, filename='outputs/model.pkl')

run.complete()
```

Now configure and run a hyperdrive experiment:

To prepare the hyperdrive experiment, you must use a HyperDriveConfig object to configure the experiment run, as shown in the following example code:

```python
from azureml.core import Experiment
from azureml.train.hyperdrive import HyperDriveConfig, PrimaryMetricGoal

# Assumes ws, script_config and param_sampling are already defined

hyperdrive = HyperDriveConfig(run_config=script_config,
                              hyperparameter_sampling=param_sampling,
                              policy=None,
                              primary_metric_name='Accuracy',
                              primary_metric_goal=PrimaryMetricGoal.MAXIMIZE,
                              max_total_runs=6,
                              max_concurrent_runs=4)

experiment = Experiment(workspace = ws, name = 'hyperdrive_training')
hyperdrive_run = experiment.submit(config=hyperdrive)
```

Monitor hyperdrive experiments in Azure studio or by using the Jupyter Notebooks RunDetails widget.

The experiment will initiate a child run for each hyperparameter combination to be tried, and you can retrieve the logged metrics these runs using the following code:

```python
for child_run in run.get_children():
    print(child_run.id, child_run.get_metrics())
```
You can also list all runs in descending order of performance like this:
```python
for child_run in hyperdrive_run.get_children_sorted_by_primary_metric():
    print(child_run)
```
To retrieve the best performing run, you can use the following code:
```python
best_run = hyperdrive_run.get_best_run_by_primary_metric()
```

## Reference

Build AI solutions with Azure Machine Learning, retrieved from https://docs.microsoft.com/en-us/learn/paths/build-ai-solutions-with-azure-ml-service/
