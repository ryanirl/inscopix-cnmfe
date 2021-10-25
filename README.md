# Inscopix CNMFe
Constrained non-negative matrix factorization algorithm for automated source extraction from microendoscopic data.

### Background
CNMFe is an algorithm used to perform automated source extraction from calcium imaging movies. Specifically, it aims to retrieve the spatial location and temporal dynamics of cells in a movie. The CNMF algorithm was originally developed for two-photon data [Pnevmatikakis, 2016](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC4881387/). The two-photon algorithm was modified with enhanced background subtraction routines to work in the one-photon setting [Zhou, 2018](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC5871355/). The first version of CNMFe, originally developed in MATLAB as [CNMF_E](https://github.com/zhoupc/CNMF_E), was ported to Python in the [CaImAn](https://github.com/flatironinstitute/CaImAn) package, described in [Giovannucci, 2018](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC6342523/). CaImAn CNMFe was ported to C++ for implementation into Inscopix Data Processing Software, improved for performance, and extended to offer greater processing control and transparency.

### Algorithm Overview
CNMFe is an iterative algorithm that solves a non-convex optimization problem, which consists of minimizing the difference between the input movie and the data reconstructed using the model. The process consists of making an initial estimation of the spatial location and temporal activity of cells, which we refer to as the initialization phase, and then refining this estimate over several iterations of processing. The algorithm can be broken down into distinct and reusable processing modules as depicted below. 

The first two steps of Inscopix CNMFe aim to determine how to efficiently process the movie through division of labor and efficient memory management. The following 13 steps (steps 3 through 16) encompass the core functionality of CNMFe and are applied to the movie, or to spatially distinct portions of the movie for parallel processing. Finally, the results from parallel processing are combined and the components scaled based on user-specified parameters.

![CNMFe Modules Overview](img/cnmfe_modules_overview.png?raw=true "CNMFe Modules Overview")

### How to Use Inscopix CNMFe
The simplest and quickest way to use Inscopix CNMFe is to follow the steps outlined below.

**Step 1: Install [Docker](https://docs.docker.com/get-docker/)**

**Step 2: Run Inscopix CNMFe within a docker container**

Place the input movie and the CNMFe parameters stored in a json file in the same folder.
Note that only tiff movies are currently supported.
The docker image will automatically be downloaded the first time you run the command.
```
docker run --rm -ti \
	-v $PWD/test/data:/input \
	-v $PWD/output:/output \
	public.ecr.aws/inscopix/cnmfe /input/movie.tif /input/params.json /output
```

### Description of Individual Parameters
The CNMFe parameters used in this package are listed below along with their respective descriptions.

| Parameter  | Description |
| ----------:|:-------------|
| average_cell_diameter | the average cell diameter of a representative cell in pixels |
| min_pixel_correlation | the minimum correlation of a pixel with its immediate neighbors when searching for new cell centers |
| min_peak_to_noise_ratio | the minimum peak-to-noise ratio of a pixel when searching for new cell centers |
| gaussian_kernel_size | the width in pixels of the Gaussian kernel used for spatial filtering of the movie before cell initialization |
| closing_kernel_size | the size in pixels of the morphological closing kernel used for removing small disconnected components and connecting small cracks within individual cell footprints |
| background_downsampling_factor | the spatial downsampling factor to use when estimating the background activity |
| ring_size_factor | the multiple of the average cell diameter to use for computing the radius of the ring model used for estimating the background activity |
| merge_threshold | the temporal correlation threshold for merging cells that are spatially close |
| number_of_threads | the number of threads to use for processing |
| processing_mode | the processing mode to use to run CNMFe (0: all in memory, 1: sequential patches, 2: parallel patches) <br/><br/><ul><li>All in memory: processes the entire field of view at once.</li><li>Sequential patches: breaks the field of view into overlapping patches and processes them one at a time using the specified number of threads where parallelization is possible.</li><li>Parallel patches:  breaks the field of view into overlapping patches and processes them in parallel using a single thread for each.</li></ul>|
| patch_size | the side length of an individual square patch of the field of view in pixels |
| patch_overlap | the amount of overlap between adjacent patches in pixels |
| output_units | the units of the output temporal traces (0: dF, 1: dF over noise) <br/><br/><ul><li>dF: temporal traces on the same scale of pixel intensity as the original movie. dF is calculated as the average fluorescence activity of all pixels in a cell, scaled so that each spatial footprint has a magnitude of 1.</li><li>dF over noise: temporal traces divided by their respective estimated noise level. This can be interpreted similarly to a z-score, with the added benefit that the noise is a more robust measure of the variance in a temporal trace compared to the standard deviation.</li></ul> |
| output_filetype | the file types into which the output will be saved (0: footprints saved to a tiff file and traces saved to a csv file, 1: output saved to a h5 file under the keys footprints and traces) |

### Building and Contributing to Inscopix CNMFe (for developers)
For those interested in contributing to this package or using it as part of another application, this section provides instructions for compiling Inscopix CNMFe into a static library.
A sample C++ project and unit tests are provided.

#### Expected directory structure
The build instructions assume the dependencies are located in a subdirectory called *lib* as shown below. This structure can be changed by updating the cmake files.
```
isx-cnmfe
 └── lib
      ├── armadillo
      ├── catch
      ├── hdf5
      ├── Intel_MKL
      ├── json
      ├── libtiff
      ├── OpenCV
      ├── Qt
      ├── ThreadPool
      └── pybind11
```

#### Building the static library
Instructions for compiling CNMFe into a static library.
The target platform can be specified using the CMake generator flag (-G).
The specific compilers to use can be specified by setting the CMake flags (CMAKE_C_COMPILER, CMAKE_CXX_COMPILER) in the CMakeLists.txt.
Note that the default Linux build instructions will use gcc-4.8 and g++-4.8.
You can install these specific version using the following command on Ubuntu: `sudo apt install -y g++-4.8 gcc-4.8`

Mac
```
mkdir build
cd build
cmake ..
make
```

Linux
```
mkdir build
cd build
cmake -DCMAKE_C_COMPILER=/usr/bin/gcc-4.8 -DCMAKE_CXX_COMPILER=/usr/bin/g++-4.8 ..
make
```

Windows
```
mkdir build
cd build
cmake -G "Visual Studio 14 2015 Win64" ..
# open build/cnmfe.sln in Visual Studio to build the project
```

#### Example code for using the CNMFe library
An example C++ project using the Inscopix CNMFe library is available in *example.cpp* and 
compiled into an executable when building the app.
The following command will run CNMFe on a small movie recorded in the striatum. 

Mac & Linux
```
./build/runCnmfe test/data/movie.tif test/data/params.json output
```

Windows
```
./build/Release/runCnmfe.exe test/data/movie.tif test/data/params.json output
```

#### Running the unit tests
Once the project is built, unit tests can be run using the following command.

Mac & Linux
```
./build/test/runTests
```

Windows
```
./build/test/Release/runTests.exe
```

#### Building and Using the Inscopix CNMFe Docker Container Locally
To compile the app in a containerized environment:
```
docker build -t inscopix/cnmfe .
```

To run CNMFe within the container:
```
docker run --rm -ti \
	-v $PWD/test/data:/input \
	-v $PWD/output:/output \
    inscopix/cnmfe /input/movie.tif /input/params.json /output
```

#### Using Inscopix CNMFe in Python
##### Step 1: Create a Python virtual environment
```
conda create -n inscopix-cnmfe python=3.9
conda activate inscopix-cnmfe
```

Note that on Windows the environment must be created with the `anaconda` package:
```
conda create -n inscopix-cnmfe python=3.9 anaconda
```

##### Step 2: Build the wheel file
By default the wheel file will be located in the distribution folder (`dist`).
```
python setup.py bdist_wheel
```

##### Step 3: Install the inscopix_cnmfe package using the wheel file

The wheel filename may differ depending on the system and environment used to create it.
```
pip install dist/inscopix_cnmfe-1.0.0-cp39-cp39-macosx_10_12_x86_64.whl
```
You are now ready to use Inscopix CNMFe using Python.

##### Usage in Python
```
import inscopix_cnmfe

inscopix_cnmfe.run_cnmfe(
	input_movie_path='test/data/movie.tif', 
	output_dir_path='output', 
	output_filetype=0,
	average_cell_diameter=7,
	min_corr=0.8,
	min_pnr=10.0,
	gaussian_kernel_size=0,
	closing_kernel_size=0,
	background_downsampling_factor=2,
	ring_size_factor=1.4,
	merge_threshold=0.7,
	num_threads=4,
	processing_mode=2,
	patch_size=80,
	patch_overlap=20,
	trace_output_units=1
)
```

### Dependencies
Below is a list of all the dependencies used in Inscopix CNMFe. 
Note that it may be compatible with other versions of the libraries, but only the ones listed below were tested.
pybind11 is only needed if you intend to build the Python API.

| Package       | Version       |
| ------------- |:-------------:|
| [armadillo](http://arma.sourceforge.net/) | 9.900.1 |
| [Intel MKL](https://software.intel.com/content/www/us/en/develop/tools/oneapi/components/onemkl.html#gs.1020r93) | 2017.2 |
| [OpenCV](https://opencv.org/) | 3.2.0 |
| [hdf5](https://support.hdfgroup.org/HDF5/doc/cpplus_RM/index.html) | 1.10 |
| [libtiff](https://libtiff.gitlab.io/libtiff/) | 4.0.8 |
| [ThreadPool](https://github.com/progschj/ThreadPool) | 1.0 |
| [Qt](https://www.qt.io/) | 5.8.0 |
| [json](https://github.com/nlohmann/json) | 2.0.1 |
| [Catch](https://github.com/catchorg/Catch2) | 1.4.0 |
| [pybind11](https://github.com/pybind/pybind11) | 2.8.0 |

### Tested Systems
Below is a list of systems which Inscopix CNMFe has been built on. Note that it may be compatible with other systems, but only the ones listed below were tested.

| Operating System       | Compiler       |
| ------------- |:-------------:|
| macOS 10.15.7 | Apple Clang 8.0.0 (clang-800.0.38); XCode 8.0 |
| Ubuntu 18.04 LTS | GCC 4.8.5 (Ubuntu 4.8.5-4ubuntu8) |
| Windows 10 | Visual Studio 14 2015 Win64 (MSVC 19.0.24215.1) |

### Troubleshooting

| Operating System | Error   | Solution       |
| ------------- |:------------------:| :-------------:|
| Ubuntu | No rule to make target '/lib64/libz.so' | Create a symlink in /lib64 to libz.so <br> ```sudo ln -s /lib/x86_64-linux-gnu/libz.so.1 /lib64/libz.so``` |

