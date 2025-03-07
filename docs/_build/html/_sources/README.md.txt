# Survival Integration of Multi-omics using Deep-Learning (DeepProg)

This package allows to combine multi-omics data together with survival. Using autoencoders, the pipeline creates new features and identify those linked with survival, using CoxPH regression.
The omic data used in the original study are RNA-Seq, MiR and Methylation. However, this approach can be extended to any combination of omic data.

The current package contains the omic data used in the study and a copy of the model computed. However, it is very easy to recreate a new model from scratch using any combination of omic data.
The omic data and the survival files should be in tsv (Tabular Separated Values) format and examples are provided. The deep-learning framework uses Keras, which is a embedding of Theano / tensorflow/ CNTK.


## Requirements
* Python 2 or 3
* [theano](http://deeplearning.net/software/theano/install.html) (the used version for the manuscript was 0.8.2)
* [tensorflow](https://www.tensorflow.org/) as a more robust alternative to theano
* [cntk](https://github.com/microsoft/CNTK) CNTK is anoter DL library that can present some advantages compared to tensorflow or theano. See [https://docs.microsoft.com/en-us/cognitive-toolkit/](https://docs.microsoft.com/en-us/cognitive-toolkit/)
* R
* the R "survival" package installed.
* numpy, scipy
* scikit-learn (>=0.18)
* rpy2 2.8.6 (for python2 rpy2 can be install with: pip install rpy2==2.8.6, for python3 pip3 install rpy2==2.8.6). It seems that newer version of rpy2 might not work due to a bug (not tested)


```bash
pip install theano --user # Original backend used OR
pip install tensorflow --user # Alternative backend for keras supposely for efficient
pip install keras --user
pip install rpy2==2.8.6 --user

#If you want to use theano or CNTK
nano ~/.keras/keras.json
```

* R installation

```R
install.package("survival")
install.package("glmnet")
source("https://bioconductor.org/biocLite.R")
biocLite("survcomp")
```


### Support for CNTK / tensorflow
* We originally used Keras with theano as backend plateform. However, [Tensorflow](https://www.tensorflow.org/) or [CNTK](https://docs.microsoft.com/en-us/cognitive-toolkit/) are more recent DL framework that can be faster or more stable than theano. Because keras supports these 3 backends, it is possible to use them as alternative to theano. To change backend, please configure the `$HOME/.keras/keras.json` file. (See official instruction [here](https://keras.io/backend/)).

The default configuration file looks like this:

```json
{
    "image_data_format": "channels_last",
    "epsilon": 1e-07,
    "floatx": "float32",
    "backend": "tensorflow"
}
```

## Distributed computation
* It is possible to use the python ray framework [https://github.com/ray-project/ray](https://github.com/ray-project/ray) to control the parallel computation of the multiple models. To use this framework, it is required to install it: `pip install ray --user`
* Alternatively, it is also possible to create the model one by one without the need of the ray framework

## Visualisation module (Experimental)
* To visualise test sets projected into the multi-omic survival space, it is required to install `mpld3` module: `pip install mpld3 --user`
* Note that the pip version of mpld3 installed on my computer presented a [bug](https://github.com/mpld3/mpld3/issues/434): `TypeError: array([1.]) is not JSON serializable `. However, the [newest](https://github.com/mpld3/mpld3) version of the mpld3 available from the github solved this issue. It is therefore recommended to install the newest version to avoid this issue.

## installation (local)

```bash
git clone https://github.com/lanagarmire/SimDeep.git
cd SimDeep
pip install -r requirements.txt --user
```

## Usage
* test if simdeep is functional (all the software are correctly installed):

```bash
  python test/test_dummy_boosting_stacking.py -v # OR
  nosetests test -v # Improved version of python unit testing
  ```

* All the default parameters are defined in the config file: `./simdeep/config.py` but can be passed dynamically. Three types of parameters must be defined:
  * The training dataset (omics + survival input files)
    * In addition, the parameters of the test set, i.e. the omic dataset and the survival file
  * The parameters of the autoencoder (the default parameters works but it might be fine-tuned.
  * The parameters of the classification procedures (default are still good)


## Example datasets and scripts
An omic .tsv file must have this format:

```bash
head mir_dummy.tsv

Samples        dummy_mir_0     dummy_mir_1     dummy_mir_2     dummy_mir_3 ...
sample_test_0  0.469656032287  0.347987447237  0.706633335508  0.440068758445 ...
sample_test_1  0.0453108219657 0.0234642968791 0.593393816691  0.981872970341 ...
sample_test_2  0.908784043793  0.854397550009  0.575879144667  0.553333958713 ...
...

```

a survival file must have this format:

```bash
head survival_dummy.tsv

Samples        days event
sample_test_0  134  1
sample_test_1  291  0
sample_test_2  125  1
sample_test_3  43   0
...

```

As examples, we included two datasets:
* A dummy example dataset in the `example/data/` folder:
```bash
examples
├── data
│   ├── meth_dummy.tsv
│   ├── mir_dummy.tsv
│   ├── rna_dummy.tsv
│   ├── rna_test_dummy.tsv
│   ├── survival_dummy.tsv
│   └── survival_test_dummy.tsv
```

* And a real dataset in the `data` folder. This dataset derives from the TCGA HCC cancer dataset. This dataset needs to be decompressed before processing:

```bash
data
├── meth.tsv.gz
├── mir.tsv.gz
├── rna.tsv.gz
└── survival.tsv

```

## Creating a simple DeepProg model with one autoencoder for each omic
First, we will build a model using the example dataset from `./examples/data/` (These example files are set as default in the config.py file). We will use them to show how to construct a single DeepProg model inferring a autoencoder for each omic

```python

# SimDeep class can be used to build one model with one autoencoder for each omic
from simdeep.simdeep_analysis import SimDeep
from simdeep.extract_data import LoadData

help(SimDeep) # to see all the functions
help(LoadData) # to see all the functions related to loading datasets

# Defining training datasets
from simdeep.config import TRAINING_TSV
from simdeep.config import SURVIVAL_TSV

dataset = LoadData(training_tsv=TRAINING_TSV, survival_tsv=SURVIVAL_TSV)

simDeep = SimDeep(dataset=dataset) # instantiate the model with the dummy example training dataset defined in the config file
simDeep.load_training_dataset() # load the training dataset
simDeep.fit() # fit the model

# Defining test datasets
from simdeep.config import TEST_TSV
from simdeep.config import SURVIVAL_TSV_TEST

simDeep.load_new_test_dataset(TEST_TSV, SURVIVAL_TSV_TEST, fname_key='dummy')

# The test set is a dummy rna expression (generated randomly)
print(simDeep.dataset.test_tsv) # Defined in the config file
# The data type of the test set is also defined to match an existing type
print(simDeep.dataset.data_type) # Defined in the config file
simDeep.predict_labels_on_test_dataset() # Perform the classification analysis and label the set dataset

print(simDeep.test_labels)
print(simDeep.test_labels_proba)

simDeep.save_encoder('dummy_encoder.h5')

```
## Creating a DeepProg model using an ensemble of submodels

Secondly, we will build a more complex DeepProg model constituted of an ensemble of sub-models each originated from a subset of the data. For that purpose, we need to use the `SimDeepBoosting` class:

```python
from simdeep.simdeep_boosting import SimDeepBoosting

help(SimDeepBoosting)

from collections import OrderedDict


path_data = "../examples/data/"
# Example tsv files
tsv_files = OrderedDict([
          ('MIR', 'mir_dummy.tsv'),
          ('METH', 'meth_dummy.tsv'),
          ('RNA', 'rna_dummy.tsv'),
])

# File with survival event
survival_tsv = 'survival_dummy.tsv'

project_name = 'stacked_TestProject'
epochs = 10 # Autoencoder epochs. Other hyperparameters can be fine-tuned. See the example files
seed = 3 # random seed used for reproducibility
nb_it = 5 # This is the number of models to be fitted using only a subset of the training data
nb_threads = 2 # These treads define the number of threads to be used to compute survival function

boosting = SimDeepBoosting(
    nb_threads=nb_threads,
    nb_it=nb_it,
    split_n_fold=3,
    survival_tsv=tsv_files,
    training_tsv=survival_tsv,
    path_data=path_data,
    project_name=project_name,
    path_results=path_data,
    epochs=epochs,
    seed=seed)

# Fit the model
boosting.fit()
# Predict and write the labels
boosting.predict_labels_on_full_dataset()
# Compute internal metrics
boosting.compute_clusters_consistency_for_full_labels()
# COmpute the feature importance
boosting.compute_feature_scores_per_cluster()
# Write the feature importance
boosting.write_feature_score_per_cluster()

boosting.load_new_test_dataset(
    {'RNA': 'rna_dummy.tsv'}, # OMIC file of the test set. It doesnt have to be the same as for training
    'survival_dummy.tsv', # Survival file of the test set
    'TEST_DATA_1', # Name of the test test to be used
)

# Predict the labels on the test dataset
boosting.predict_labels_on_test_dataset()
# Compute C-index
boosting.compute_c_indexes_for_test_dataset()
# See cluster consistency
boosting.compute_clusters_consistency_for_test_labels()

# [EXPERIMENTAL] method to plot the test dataset amongst the class kernel densities
boosting.plot_supervised_kernel_for_test_sets()
```

## Creating a distributed DeepProg model using an ensemble of submodels

We can allow DeepProg to distribute the creation of each submodel on different clusters/nodes/CPUs by using the ray framework.
The configuration of the nodes / clusters, or local CPUs to be used needs to be done when instanciating a new ray object with the ray [API](https://ray.readthedocs.io/en/latest/). It is however quite straightforward to define the number of instances launched on a local machine such as in the example below in which 3 instances are used.

```python
# Instanciate a ray object that will create multiple workers
import ray
ray.init(webui_host='0.0.0.0', num_cpus=3)
# More options can be used (e.g. remote clusters, AWS, memory,...etc...)
# ray can be used locally to maximize the use of CPUs on the local machine
# See ray API: https://ray.readthedocs.io/en/latest/index.html

boosting = SimDeepBoosting(
    ...
    distribute=True, # Additional option to use ray cluster scheduler
    ...
)
...
# Processing
...

# Close clusters and free memory
ray.shutdown()
```

## Example scripts

Example scripts are availables in ./examples/ which will assist you to build a model from scratch with test and real data:

```bash
examples
├── create_autoencoder_from_scratch.py # Construct a simple deeprog model on the dummy example dataset
├── example_with_dummy_data_distributed.py # Process the dummy example dataset using ray
├── example_with_dummy_data.py # Process the dummy example dataset
└── load_3_omics_model.py # Process the example HCC dataset


```


## contact and credentials
* Developer: Olivier Poirion (PhD)
* contact: opoirion@hawaii.edu, o.poirion@gmail.com
