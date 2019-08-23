# biophys_optimize
Optimization of single-cell biophysically detailed models

Installation:
```bash
$ python setup.py install
```

# 1) Pre-process
This step preprocesses the experimental data so it is ready for matching with the model.

Run:
```bash
$ python -m biophys_optimize.scripts.run_preprocessing --help
```
to get command line options. There are a handful of JSON files for examples in test_input_files. These will need to be edited to define file paths unique to the user's computer.

## a) Download data using AllenSDK and edit JSON file
Follow the Jupyter Notebook at https://github.com/latimerb/GeneralTutorials/blob/master/AllenSDK/cell_types.ipynb to download electrophysiology and SWC data. The specimen ID needs to be added to the test_input_files/test_preprocess_input.json file and you need to update the paths to the data. The sweep IDs are unique for each specimen so you will need to look these up.
      
## b) Run test_preprocessing.py  
```bash
$ python -m biophys_optimize.scripts.run_preprocessing --input_json ./test_input_files/test_preprocess_input.json
```
/biophys_optimize/preprocess.py does the work here. It separates the "cap_checks" (for passive fitting) from the current injection and averages them. It places these in test_storage/upbase.dat (depolarizing current pulse) and test_storage/downbase.dat (hyperpolarizing current pulse). It also calculates some characteristics like the RMP, average spike rate, etc. and places these in /test_storage/preprocess_results.json.


# 2) Passive fitting
This is the stage where the passive parameters (input resistance and time constant) are found from experimental data and used to calculate model parameters (membrane resistance, capacitance, and area).

## a) Edit the passive JSON file
Change all the paths in the /test_input_files/test_passive_input_1.json file just as in step 1a. Other parameters should still be the same. Then run:
```bash
$ python -m biophys_optimize.scripts.run_passive_fitting --input_json ./test_input_files/test_passive_input_1.json
```
The bulk of the code is in /biophys_optimize/neuron_passive_fit.py. Essentially some experimental data is loaded which is the response of the neuronal membrane to a positive and negative current pulse (up_t,up_v,down_t,down_v). Then NEURON's multiple run fitter is used to match a function to this data with parameters ra, cm, rm, a1, and a2. There is a passive_fit_1 and a passive_fit_2 which seem to differ in only one way; the passive_fit_2 function does not use a proxy for Ra while passive_fit_1 uses Ra, cm, and Rm (1/g_leak). 

The parameters from passive_fit_N are saved to passive_fit_N_results.json and that is the end of this step.


# 3) Consolidate passive fitting

This step takes the three different passive fits that were made in step 2 and picks the "best" one. The user must supply passive_fit_1_results.json, passive_fit_2_results.json, and passive_fit_elec_results.json. 

```bash
$ python -m biophys_optimize.scripts.run_consolidate_passive_fitting --input_json ./test_input_files/test_consolidate_input.json
```
The output is placed in test_storage/passive_results.json and will contain the cm and Ra. 

# 4) Optimize

