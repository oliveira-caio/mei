[![Test](https://github.com/sinzlab/mei/actions/workflows/test.yml/badge.svg)](https://github.com/sinzlab/mei/actions/workflows/test.yml)
[![Black](https://github.com/sinzlab/mei/actions/workflows/black.yml/badge.svg)](https://github.com/sinzlab/mei/actions/workflows/black.yml)
[![Isort](https://github.com/sinzlab/mei/actions/workflows/isort.yml/badge.svg)](https://github.com/sinzlab/mei/actions/workflows/isort.yml)

Generate most exciting inputs (MEIs).

## Installation

The easiest way to install the package is via pip:

```python
pip install neuro-mei
```

If you want to install from source from a different branch, for example `inception_loop`, you can also install by doing:

```python
pip install git+https://github.com/sinzlab/mei.git@inception_loop
```

## Usage

This section describes the general usage of the MEI framework. Due to the fact that this framework uses 
[DataJoint](https://github.com/datajoint/datajoint-python/) and [NNFabrik](https://github.com/sinzlab/nnfabrik), general
familiarity with these two packages is assumed.

### 1. Table Setup

This section describes how the tables used in the MEI generation process have to be set up.

#### 1.1 Trained Ensemble Model Table

This table contains ensembles of previously trained models. During the MEI generation process, all models in an ensemble
will be given the same input and their output will be averaged. The framework provides a template class that you can use
to create a trained ensemble model table by declaring a new class that inherits from the template. Afterwards you have 
to link up your class with your NNFabrik-style dataset and trained model tables via class attributes.

##### Example

```python
from mei.main import TrainedEnsembleModelTemplate


@schema
class TrainedEnsembleModel(TrainedEnsembleModelTemplate):
    dataset_table = Dataset
    trained_model_table = TrainedModel
```



#### 1.2 Objective Table

This table has two jobs:

1. Provide a method that can be used to get the to-be-optimized objective.
2. Contain the information that is needed to come up with the aforementioned objective.

Note that you will have to implement your own objective table because the exact implementation is heavily dependent on 
the structure of the data you want to use and the architecture of your models. 

##### Example

The objective table implemented below contains information that can be used to map the ID of a real neuron (`neuron_id`)
to the index of its corresponding output unit (`output_unit`) in the model's output. The `get_objective`
method uses this information to constrain a given model to a single output unit and therefore to a single real neuron.

```python
from mei.modules import ConstrainedOutputModel


@schema
class Objective(dj.Computed):
    definition = """
    -> self.dataset_table
    neuron_id:		int
    ---
    output_unit:	int
    """
    
    dataset_table = Dataset
    
    def make(self, key):
        """Fills the table."""
        
    def get_objective(self, model, key):
        output_unit = (self & key).fetch1("output_unit")
        return ConstrainedOutputModel(model, output_unit)
```

Your implementation must provide a method  called `get_objective` that has a PyTorch module (`model`) and a
dictionary (`key`) as its only parameters and that must return a PyTorch module representing the objective. The returned
module must itself return
a scalar value when called.

#### 1.4 MEI Table

This table contains generated MEIs. You can create your own MEI table by inheriting from the provided template class.
Afterwards you have to link up your table with your trained (ensemble) model and objective tables via class attributes.

##### Example

```python
from mei.main import MEITemplate


@schema
class MEI(MEITemplate):
    trained_model_table = TrainedEnsembleModel
    objective_table = Objective
```

### 2. Generating MEIs

This section lays out the general steps one would execute when generating MEIs.

#### 2.1 Optional: Creating an Ensemble Model

Note that this step is only required if you are using the trained ensemble model table.

You can create a new ensemble model by calling the `create_ensemble` method of the trained ensemble model table with a
DataJoint restriction (`key`). The passed restriction is used to restrict the trained model table and all models still
present in the restricted table will be made part of the new ensemble. Note that the provided restriction must be able
to restrict the dataset table down to a single entry because creating an ensemble consisting of models trained on 
different datasets is currently not supported. For your own reference you can also pass a comment when creating a new
ensemble.

##### Example

```python
TrainedEnsembleModel().create_ensemble(key, comment="My ensemble")
```

#### 2.2 Populating the Objective Table

Before generating MEIs you have to populate the objective table by either calling its `populate` method if your
implementation provides it or by manually inserting entries.

##### Example

```python
Objective().populate()
```

#### 2.3 Configuring the Generation Process

Each MEI is generated according to a user-configurable method. You can specify a new method by adding it to the MEI
method table using its  `add_method` method (see example below). This method expects the name of a method function
(`method_fn`) and method configuration object (`method_config`).

The function name needs to be the absolute path to a callable object. A function that can be used to generate MEIs using
gradient ascent is provided with the framework and its path is `mei.methods.gradient_ascent`. You can also implement
your own function and use it with the framework. Further information on how to do that can be found [here](#method).

The  configuration object will be passed to the function by the framework and should contain information that will be
used by the method function to alter its behavior.

In the case of the provided function the configuration object is a dictionary. It contains information about which
components to use when generating MEIs and how to configure those components. A component must be a callable object that
must return another callable object when called. The configuration dictionary contains the absolute path (`"path"`) to
the corresponding component and can additionally contain a set of keyword arguments (`"kwargs"`) that will be passed to
the corresponding component when it is initially called. Below is a list of supported components:

* `"device"`: Required, must be either `"cpu"` or `"cuda"`. The MEI will be generated on the CPU or the GPU depending on
    this value 
* `"initial"`: Required component used to generate an initial guess from which the MEI generation process will start
* `"optimizer"`: Required component used to optimize the input to the model and in turn generate the MEI. Must be a 
    PyTorch-style optimizer class
* `"stopper"`: Required component used to determine whether or not to stop the MEI generation process in each iteration
    based on a user-defined condition
* `"transform"`: Optional component used to transform the input before passing it through the model
* `"regularization"`: Optional component used to compute a regularization term from the (transformed) input that is
    added to the model's output before taking the optimization step
* `"precondition"`: Optional component used to precondition the gradient
* `"postprocessing"`: Optional component that applies an operation to the input after each optimization step. The
    operation performed by this component does not influence the gradient
* `"objectives"`: Optional component that consists of a list of sub-components. Each sub-component tracks an objective
    over the duration of the generation process

You can completely omit optional components from the configuration dictionary if you do not want to use them.

##### Example

```python
from mei.main import MEIMethod


method_fn = "mei.methods.gradient_ascent"
method_config = {
    "initial": {"path": "mei.initial.RandomNormal"},
    "optimizer": {"path": "torch.optim.SGD", "kwargs": {"lr": 0.1}},
    "stopper": {"path": "mei.stoppers.NumIterations", "kwargs": {"num_iterations": 1000}},
    "objectives": [
        {"path": "mei.objectives.EvaluationObjective", "kwargs": {"interval": 10}}
    ],
    "device": "cuda",
}
MEIMethod().add_method(method_fn, method_config, comment="My MEI method")
```

#### 2.4 Specifying  a Seed

Next you have to specify a seed  to make the MEI generation process random but reproducible by inserting a seed into the
MEI seed table.

##### Example

```python
from mei.main import MEISeed


MEISeed().insert1({"mei_seed": 42})
```

#### 2.5 Populating the MEI Table

After configuring everything you can generate MEIs by calling the `populate` method of your `MEI` table. The table will
insert one row for each generated MEI which itself can be found in the `mei` attribute. Additionally each MEI is
associated with a score and an output object which can be found in the `score` and `output` attributes, respectively.

The score should express how well the generation process went but what it exactly represents is up to the used method
function. In the case of the provided function it represents  the final model evaluation.

The output object is an object that is returned by the method function at the end of the generation process.
The included function will return a dictionary that contains the values of the objectives that were tracked during the
generation process.

Note that the `mei` and `output` attributes are stored externally. 

##### Example

```python
MEI().populate()
```

## State

Instances of the `State` class contain information describing a particular state encountered during the optimization
process. This information is used by various components in the framework. The attributes of a state instance are:

* `i_iter`: An integer representing the index of the optimization step this state corresponds to
* `evaluation`: A float representing the evaluation of the model in response to the current input
* `reg_term`: A float representing the current regularization term added to the evaluation before the optimization step
    represented by this state was made. This value will be zero if no transformation is used
* `input_`: A tensor representing the untransformed input to the model. This will be identical to the post-processed
    input from the last step for all steps except the first one
* `transformed_input`: A tensor representing the transformed input to the model. This will be identical to the
    untransformed input if no transformation is used
* `post_processed_input`: A tensor representing the post-processed input. This will be identical to the untransformed
    input if no post-processing is done
* `grad`: A tensor representing the gradient
* `preconditioned_grad` : A tensor representing the preconditioned gradient. This will be identical to the gradient if
    no preconditioning is done
* `stopper_output`: An object returned by the stopper component.

## Components

This section describes each component type in greater detail and how you can implement your own variants.

### Method

This component is the point of entry for the whole optimization process.

It will be called with your NNFabrik-style dataloaders dictionary, your model, your configuration object and an
integer representing the seed. It must return the MEI, a float representing the score the MEI achieved and an output
object. The MEI and the output object must be compatible with PyTorch's `save` function.

##### Example

```python
def method(dataloaders, model, config, seed):
    """Generates a MEI."""
    return mei, score, output
```

After you have implemented your method you can use it by adding it to the MEI method table as described
[here](#23-configuring-the-generation-process).

### Stopper

This component is used to check whether or not to stop the MEI generation process based on the current state of the
generation process.

All custom stoppers must implement the `__call__` method and they should inherit from a abstract base class (ABC) called
`OptimizationStopper`. The stopper will be called after each optimization step with the current state of the
optimization process.  It must return `(True, output)` if the optimization process is to be stopped and
`(False, output)` otherwise. `output` can be any object or `None` but it can not be omitted.

##### Example

Below is the implementation of a custom stopper that stops the MEI generation process once the model's evaluation
reaches a user-specified threshold:

```python
"""Contents of mymodule.py"""

from mei.stoppers import OptimizationStopper


class EvaluationThreshold(OptimizationStopper):
    
    def __init__(self, threshold):
        self.threshold = threshold
        
    def __call__(self, current_state):
        if current_state.evaluation >= self.threshold:
            return True, None
        else:
            return False, None
```

You can then use your custom stopper component by specifying it in the `method_config` like so:

```Python
method_config = {
    "stopper": {"path": "mymodule.EvaluationThreshold", "kwargs": {"threshold": 2.5}},
    ...
}
```

## How to run the tests :test_tube:

Clone this repository and run the following command from within the cloned repository to run all tests:

```bash
docker-compose run pytest
```

## How to contribute :fire:

Pull requests (and issues) are always welcome. This section describes some
preconditions that pull requests need to fulfill.

### Tests

Please make sure your changes pass the tests. Take a look at the [test running
section](#how-to-run-the-tests-test_tube) for instructions on how to run them. Adding tests
for new code is highly recommended.

### Code Style

#### black

This project uses the [black](https://github.com/psf/black) code formatter. You
can check whether your changes comply with its style by running the following
command:

```bash
docker-compose run black
```

Furthermore you can pass a path to the service to have black fix any errors in
the Python modules it finds in the given path.

#### isort

[isort](https://github.com/PyCQA/isort) is used to sort Python imports. You can check the order of imports by running the following command:

```bash
docker-compose run isort
```

The imports can be sorted by passing a path to the service.
