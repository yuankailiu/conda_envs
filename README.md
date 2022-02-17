# conda_envs

Setup InSAR data processing codes on Linux / macOS using `conda` environments.

### 1. Install conda

```bash
mkdir -p ~/tools; cd ~/tools

# download, install and setup (mini/ana)conda
# for Linux, use Miniconda3-latest-Linux-x86_64.sh
# for macOS, opt 2: curl https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh -o Miniconda3-latest-MacOSX-x86_64.sh
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh
bash Miniconda3-latest-MacOSX-x86_64.sh -b -p ~/tools/miniconda3
~/tools/miniconda3/bin/conda init bash
```

Close and restart the shell for changes to take effect.

```
conda config --add channels conda-forge
conda install wget git tree mamba --yes
```

### 2. Install ISCE-2, ARIA-tools and MintPy to `insar` environment

Both ISCE-2 and MintPy are now available on the `conda-forge` channel, thus, one could install them by running:

```bash
mamba install -c conda-forge isce2 mintpy
```

The note below installs ISCE-2 from conda or source and ARIA-tools, MintPy, PySolid and PyAPS from source in development mode.

#### a. Download source code

```bash
cd ~/tools
mkdir isce2; cd isce2
mkdir build install src; cd src
git clone https://github.com/isce-framework/isce2.git

cd ~/tools
git clone https://github.com/aria-tools/ARIA-tools.git
git clone https://github.com/insarlab/MintPy.git
git clone https://github.com/insarlab/PySolid.git
git clone https://github.com/insarlab/PyAPS.git
git clone https://github.com/yunjunz/conda_envs.git
```

#### b. Install dependencies to `insar` environment

```bash
# create new environment
conda create --name insar --yes
conda activate insar

# opt 1: install isce-2 with conda (for macOS and Linux)
# set "use_isce_conda=1" in conda_envs/insar/config.rc filef
mamba install -y --file conda_envs/insar/requirements.txt --file MintPy/docs/requirements.txt --file ARIA-tools/requirements.txt isce2

# opt 2: install isce-2 from source (for Linux on kamb only)
# set "use_isce_conda=0" in conda_envs/insar/config.rc file
mamba install -y --file conda_envs/insar/requirements.txt --file MintPy/docs/requirements.txt --file ARIA-tools/requirements.txt --file conda_envs/isce2/requirements.txt

# install MintPy in development mode
# overwrite PySolid and PyAPS installation from conda to the local development mode
python -m pip install -e MintPy
python -m pip install -e PySolid
python -m pip install -e PyAPS

# install dependencies not available from conda
ln -s ${CONDA_PREFIX}/bin/cython ${CONDA_PREFIX}/bin/cython3
python -m pip install scalene   # CPU, GPU and memory profiler
python -m pip install ipynb     # import functions from *.ipynb files
```

#### c. Build and install ISCE-2 from source

For opt 2 (building and installing the development version of ISCE-2 from source) ONLY.

```bash
# load CUDA and compilers module on kamb
# only gcc=7.3.1 works, but not available from conda-forge: https://anaconda.org/conda-forge/gcc_linux-64/files?type=conda
module load cuda/11.2
module load /home/geomod/apps/rhel7/modules/gcc/7.3.1

# generate build system
# before re-run, delete existing contents in build folder
# use GCC-7.3.1 installed by Lijun (the old CUDA-10.1 version does not like GCC-7.5; GCC-7.3.0 also does not work, do not know why)
# more notes on https://github.com/lijun99/isce2-install
cd ~/tools/isce2/build
export GCC_BIN=/net/kraken/home1/geomod/apps/rhel7/gcc7/bin
cmake ~/tools/isce2/src/isce2 -DCMAKE_INSTALL_PREFIX=~/tools/isce2/install -DCMAKE_CUDA_FLAGS="-arch=sm_60" -DCMAKE_PREFIX_PATH=${CONDA_PREFIX} -DCMAKE_BUILD_TYPE=Release -DCMAKE_C_COMPILER=${GCC_BIN}/gcc -DCMAKE_CXX_COMPILER=${GCC_BIN}/g++ -DCMAKE_Fortran_COMPILER=${GCC_BIN}/gfortran

# compile and install
# then under the $ISCE_ROOT/install, there should be `bin` and `packages` folder
make -j 16 # use multiple threads to accelerate
make install
```

#### d. Setup

Create an alias `load_insar` in `~/.bash_profile` file for easy activation, _e.g._:

```bash
alias load_insar='conda activate insar; source ~/tools/conda_envs/insar/config.rc'
```

For opt 2 (building and installing the development version of ISCE-2 from source) ONLY: 

+ set `use_isce_conda=0` in `conda_envs/insar/config.rc` file.

#### e. Test the installation

Run the following to test the installation:

```bash
load_insar               # warm up conda environment
topsApp.py -h            # test ISCE-2
cuDenseOffsets.py -h     # test ISCE-2/PyCuAmpcor (for opt 2 only)
ariaDownload.py -h       # test ARIA-tools
smallbaselineApp.py -h   # test MintPy
```

Run the following for CPU and memory profiler via [`scalene`](https://github.com/emeryberger/scalene) of python script, e.g. `ifgram_inversion.py`:

```bash
scalene --reduced-profile ~/tools/MintPy/mintpy/ifgram_inversion.py ~/data/test/FernandinaSenDT128/mintpy/inputs/ifgramStack.h5 -w no
```

### Useful resources

+ [conda cheatsheet](https://docs.conda.io/projects/conda/en/4.6.0/_downloads/52a95608c49671267e40c689e0bc00ca/conda-cheatsheet.pdf)
