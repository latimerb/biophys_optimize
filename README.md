# biophys_optimize
Optimization of single-cell biophysically detailed models

Installation:
```bash
$ python setup.py install
```
# Introduction
There are several steps to the process of creating a biophysical model of a neuron. First, you need to have experimental current clamp data which you want your model to match. biophys_optimize was originally designed to be used with the Allen Institute's SDK so if you want to use their experimental data, you can do so right away. If you have your own data, it needs to be in the Neurodata Without Borders (NWB) format. 

# 1) Pre-process
This step preprocesses the experimental data so it is ready for matching with the model.

Run:
```bash
$ python -m biophys_optimize.scripts.run_preprocessing --help
```
to get command line options. There are a handful of JSON files for examples in test_input_files. These will need to be edited to define file paths unique to the user's computer.

## a) Download data using AllenSDK and edit JSON file
Follow the Jupyter Notebook at https://github.com/latimerb/GeneralTutorials/blob/master/AllenSDK/cell_types.ipynb to download electrophysiology and SWC (morphology) data. This will result in two files; a .nwb file containing e-phys data and a .swc file containing morphology data. The Allen Institute's data will have a specimen ID which needs to be added to the test_input_files/test_preprocess_input.json file and you need to update the paths to the data (see below). The sweep IDs are unique for each specimen so you will need to look these up in the .nwb file.

```json
   
"paths": {
    "swc": "/home/AllenSDK/cell_types/specimen_313862022/reconstruction.swc", 
    "storage_directory": "test_storage/", 
    "nwb": "/home/AllenSDK/cell_types/specimen_313862022/ephys.nwb"
}, 
  ```
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

