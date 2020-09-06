
# Automated Delineation of Grounding Lines with a Convolutional Neural Network
## By Yara Mohajerani
---

This repository contains the pipeline used for the automatic delineation of glacier grounding lines from interferograms.

The step-by-step procedure is described below.

### 1. Pre-process geocoded data (geotiff files) into numpy arrays to be used in the training of the neural network:

`python geocoded_preprocess.py --DIR=<data directory> --N_TEST=<# of tiles set aside for testing>`

In addition, there are extra optional flags `AUGMENT` which provides additional augmentation by flipping each tile horizontally, verticall, and both horizontally and vertically, as well as `DILATE` which increases the width of the labelled grounding lines for traning. These options are set to `False` by detault. But by just listing them (i.e. `--AUGMENT` and `DILATE`) you can activate these additional operations.

### 2. Training (A) or Testing (B) neural network

**A.** Train the neural network:
This step is done in a Jupyter Notebook so that it can be run on Google Colab with GPU access for faster computation. The full notebook with executable cells and documentation is `GL_delineation_geocoded_complex_customLoss.ipynb`.

**or**

**B.** Run pre-trained neural network. No need to run this if you have already run 2A.
If you want to start from a pre-trained network, you can just use the `test_model.py` script. 

`python test_model.py`

For now you can change the setting inside the script. This script produces output for both the train and test data. 

### 3. Optional Post-Processing Notebook for exploration and outputting multi-plot PNGs of the pipeline
This is again in a Jupyter Notebook. Note that the next step does not depend on the output of this step so it is optional. The notebook with executable cells and documentation is `GL_postprocessing_geocoded_complex.ipynb`.

### 4. Post-Processing: Stitching tiles together
Here we stitch the 512x512 tiles back together before postprocessing.

`python stitch_tile.py --DIR=<subdirectory with outputs> --NX=<width (512)> --NY=<height (512)> --KERNEL=<guassian width>`

The Guassian kernel refers to the variance of the Gaussian averaging kernel, givn by `kernel_weight = np.exp(-(gxx**2+gyy**2)/sigma_kernel)` where `gxx` and `gyy` are the grids.

If you want to use a uniform averaging kernel instead of Gaussian kernel to average the overlapping tiles, `--noFLAG` to the command line arguments. However, it is preferable to use Gaussian averaging so that the center of each tile counts more than the edges, in order to avoid edge effects.

### 5. Post-Processing: Vectorizing Results and Converting to Shapefiles
Use the combined tiles from the previous step to convert the raster output of the neural network to vectorized LineStrings and save as Shapefiles.

`python polygonize.py.py --DIR=<subdirectory with outputs> --FILTER=<minimum line threshold in meters> --OUT_BASE=<base directory for slurm outputs> --IN_BASE=<base directory for input (parent directory of "--DIR" --noMASK

The `FILTER` input refers to the minimum threshold used to clean up the output. Every line segment shorter than this threshold is disregarded. In addition, note that you can use the `--noMASK` in commandline arguments to not output training vs test masks. If not specified, masks will also be outputted. This is only useful if the input scenes are a combinatino of training and testing tiles (which is the case for the original training and testing data on the Getz Ice Shelf).

### 6. Post-Processing: Drawing centerlines
Use the slurm files produced by `polygonize.py` in the previous step to processes centerlines in parallel. Note this is currently designed for the partitions on UC Irvine's [GreenPlanet](https://ps.uci.edu/greenplanet/Partitions-new-names), but can be adjusted accordingly for other users. This is done by calling the `run_centerline.py` script.

`python run_centerline.py <input_file1> <input_file2>`

Where you can list an unlimited number of input files to be run in serial. But the Slurm script runs these in parallel to significantly cut down on processing time.

Note that currently `polygonize.py` is set up such that the centerline is done in parallel for each line segment of each DInSAR scene. If you want to combine the slurm jobs such that each scene is run as one job, run the following script:

`python combine_centerline_slurms.py <input_file_list>`

where the input is the list of all files produced by `polygonize.py` (e.g. `total_job_list_8.0km.sh`). Make sure you provide the full path. This program will output a similar output with the suffix `combined` (e.g. `total_job_list_8.0km_combined.sh`) which runs each scene as a single job to reduce the total number of submitted jobs. Note that the output of running 

`sh total_job_list_8.0km_combined.sh`

is still individual shapefiles for each line segment. Note that only the lines that are not classified as noise are run through the centerline routine.

### 7. Combining centerlines
To combine the individual centerlines produced in the previous step, run

`python combine_shapefiles.py --DIR=<complete path to shapefile directory> --FILTER=<minimum line threshold in meters in exisiting files>

Note that `--DIR` is the full path to the directory where the shapefiles to be combined are. `--FILTER` is the same filter size previously (in meters) that is the suffix of the files to be combined.

### 8. Error Analysis
Now we can use the vectorized output from the previous step to assess the uncertainty. 

`python mean_difference.py --DIR=<subdirectory with outputs> --FILTER=<minimum line threshold in meters>`

The arguments are the same as the vectorization step above. In addition, since the previous step categorizes and labels each geometric object, we can do the error analysis with or without pinning points. The default settings do NOT include pinning points. If you want to include them, add the `--PINNING` flag to the commandline arguments.
