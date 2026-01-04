# Compile LAMMPS and DeepMD-kit on HPC

The workflow is divided into four main stages:

1. Compile **LAMMPS (stable_23Jun2022_update4)** via CMake with QTB and SHOCK enabled.
2. Create a compatible **Conda environment with TensorFlow GPU support**.
3. Build **DeepMD-kit 2.2.9 (CUDA variant)** and link it as a LAMMPS plugin.
4. Configure environment variables correctly for runtime on HPC.

The steps are as follows.

<!-- References — Tutorials -->
<div style="background-color:#f9f9f9; padding:14px; border-radius:10px; margin-bottom:18px; border-left:6px solid #DC3C22;">
  <h3 style="margin:0 0 6px 0; font-size:18px; font-weight:600;">References — Tutorials</h3>
  <ol style="padding-left:18px; margin:0; font-size:13px;">
    <li><a href="https://wiki.cheng-group.net/wiki/software_installation/deepmd-kit/deepmd-kit_installation_new/#deepmd-kit_1" style="font-size:13px;">Cheng Group Wiki</a></li>
    <li><a href="https://chiahsinchu.github.io/blog/2023/dp-install/" style="font-size:13px;">DP Install Blog</a></li>
    <li><a href="https://docs.deepmodeling.com/projects/deepmd/en/master/install/install-lammps.html" style="font-size:13px;">DeepMD + LAMMPS Docs</a></li>
  </ol>
</div>

<!-- References — Related -->
<div style="background-color:#f9f9f9; padding:14px; border-radius:10px; margin-bottom:18px; border-left:6px solid #DC3C22;">
  <h3 style="margin:0 0 6px 0; font-size:18px; font-weight:600;">References — Related</h3>
  <ol style="padding-left:18px; margin:0; font-size:13px;">
    <li><a href="https://docs.lammps.org/fix_msst.html" style="font-size:13px;">LAMMPS MSST Fix</a></li>
    <li><a href="https://docs.lammps.org/fix_qbmsst.html" style="font-size:13px;">LAMMPS QB-MSST Fix</a></li>
    <li><a href="https://github.com/sheyua/LAMMPS-QTB/tree/master" style="font-size:13px;">LAMMPS-QTB Repo</a></li>
  </ol>
</div>

---

## 1. Create and Activate Conda Environment

```bash
conda create -n dp-qtb python=3.10.13 "tensorflow=2.10.*=gpu*"  
conda activate dp-qtb

pip install --upgrade pip
pip install tensorflow scikit-build ninja
pip install protobuf==3.19
```

---

## 2. Create Activation / Deactivation Scripts

Generate empty files:

```bash
mkdir -p $CONDA_PREFIX/etc/conda/activate.d
mkdir -p $CONDA_PREFIX/etc/conda/deactivate.d
touch $CONDA_PREFIX/etc/conda/activate.d/activate.sh
touch $CONDA_PREFIX/etc/conda/deactivate.d/deactivate.sh
```

Edit `activate.sh`:

```bash
vi $CONDA_PREFIX/etc/conda/activate.d/activate.sh
```

Then, copy the script below into activate.sh. Please verify that the installed GCC and CUDA versions are compatible for successful compilation.

```bash
export TMP_LD_LIBRARY_PATH=$LD_LIBRARY_PATH                              
export TMP_PATH=$PATH

module load cmake/3.27.9-gcc-9.3.0-hnclq4h
module load cuda/11.3
module load gcc/9.3.0
module load mpi/2021.5.0

export CC=`which gcc`
export CXX=`which g++`
export FC=`which gfortran`
export DP_VARIANT=cuda

export deepmd_source_dir=/data/qyou/software/deepmd-kit/deepmd-kit_2.2.9
export tensorflow_root=$deepmd_source_dir/build/tensorflow_root
export deepmd_root=$deepmd_source_dir/build/deepmd_root
export LAMMPS_PLUGIN_PATH=$deepmd_root/lib/deepmd_lmp
export LAMMPS_PREFIX=/data/user/software/lammps-stable_23Jun2022_update4
export LD_LIBRARY_PATH=$tensorflow_root/lib:$deepmd_root/lib:/data/user/conda/env/dp-qtb/lib:$LD_LIBRARY_PATH
export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$CONDA_PREFIX/lib/python3.10/site-packages/tensorflow:$deepmd_root/lib:$LD_LIBRARY_PATH
export PATH=$CONDA_PREFIX/bin:$LAMMPS_PREFIX/build:$PATH
```

Edit `deactivate.sh`:
```bash
vi $CONDA_PREFIX/etc/conda/deactivate.d/deactivate.sh
```

```bash
module unload cmake/3.27.9-gcc-9.3.0-hnclq4h
module unload cuda/11.3
module unload gcc/9.3.0
module unload mpi/2021.5.0

unset deepmd_source_dir tensorflow_root deepmd_root LAMMPS_PLUGIN_PATH DP_VARIANT

export LD_LIBRARY_PATH=$TMP_LD_LIBRARY_PATH
unset TMP_LD_LIBRARY_PATH

export PATH=$TMP_PATH
unset TMP_PATH
```

Exit the environment:

```bash
conda deactivate
```
---

## 3. Download LAMMPS and DeepMD-kit

```bash
wget https://github.com/lammps/lammps/archive/refs/tags/stable_23Jun2022_update4.tar.gz
tar -xvzf stable_23Jun2022_update4.tar.gz
```

```bash
git clone --recursive https://github.com/deepmodeling/deepmd-kit.git deepmd-kit_2.2.9 -b v2.2.9
```

---

## 4. Load Required Modules (LAMMPS Build)

```bash
module load cmake/3.27.9-gcc-9.3.0-hnclq4h
module load cuda/11.3
module load gcc/9.3.0
module load mpi/2021.5.0  # ⚠ Must match DeepMD MPI version!

export CC=$(which gcc)
export CXX=$(which g++)
export FC=$(which gfortran)
```

---

## 5. Configure and Install LAMMPS via CMake

```bash
cd /data/qyou/software/lammps-stable_23Jun2022_update4
mkdir -p build && cd build
```

Run the CMake command below to configure the compilers, enable MPI/OMP, and build LAMMPS with required packages.

```bash
cmake ../cmake -DCMAKE_C_COMPILER=gcc -DCMAKE_CXX_COMPILER=g++ \
      -DCMAKE_Fortran_COMPILER=gfortran \
      -D BUILD_MPI=yes -D BUILD_OMP=yes -D LAMMPS_MACHINE=intel_cpu_intelmpi \
      -D CMAKE_INSTALL_PREFIX=/data/qyou/software/lammps-stable_23Jun2022_update4 \
      -D CMAKE_INSTALL_LIBDIR=lib \
      -D CMAKE_INSTALL_FULL_LIBDIR=/data/qyou/software/lammps-stable_23Jun2022_update4/lib \
      -C ../cmake/presets/most.cmake -C ../cmake/presets/nolib.cmake \
      -D PKG_PLUGIN=yes -D PKG_MOLECULE=yes -D PKG_RIGID=yes \
      -D PKG_KSPACE=yes -D PKG_EXTRA-DUMP=yes \
      -D PKG_QTB=yes -D PKG_SHOCK=yes \
      -D BUILD_SHARED_LIBS=yes

make -j32
make install
```

✔ LAMMPS executable generated: `lmp_intel_cpu_intelmpi`

---

## 6. Build DeepMD-kit Core Library (on GPU Machine)

```bash
ssh 191
conda activate dp-qtb
```

Prepare directories:

```bash
mkdir -p $tensorflow_root $deepmd_root
```

Install source in editable mode:

```bash
cd $deepmd_source_dir
pip install -e .
```

Compile DeepMD:

```bash
cd $deepmd_source_dir/source/build
rm -rf *

cmake -DUSE_CUDA_TOOLKIT=ON \
      -DENABLE_TENSORFLOW=TRUE \
      -DUSE_TF_PYTHON_LIBS=TRUE \
      -DENABLE_PYTORCH=TRUE \
      -DCAFFE2_USE_CUDNN=TRUE \
      -DCMAKE_INSTALL_PREFIX=$deepmd_root \
      -DLAMMPS_SOURCE_ROOT=$LAMMPS_PREFIX ..

make -j20
make install
```

✔ Plugin generated under: `$deepmd_root/lib/deepmd_lmp/`

---

## 7. Create a Stub `tensorflow_root` to Help CMake Locate TF Libraries

```bash
mkdir -p $tensorflow_root/lib && cd $tensorflow_root

ln -s /data/qyou/conda/env/dp-qtb/lib/python3.10/site-packages/tensorflow/include .

cd lib
ln -s /data/qyou/conda/env/dp-qtb/lib/python3.10/site-packages/tensorflow/python/_pywrap_tensorflow_internal.so libtensorflow_cc.so
ln -s /data/qyou/conda/env/dp-qtb/lib/python3.10/site-packages/tensorflow/libtensorflow_framework.so.2 .
ln -s libtensorflow_framework.so.2 libtensorflow_framework.so
```

---

## 8. Test LAMMPS + DeepMD Plugin

Run:

```bash
lmp_intel_cpu_intelmpi -h
```

Example LAMMPS input snippet:

```
variable        deepmd_root     string /data/qyou/software/deepmd-kit/deepmd-kit_2.2.9/build/deepmd_root
plugin load     ${deepmd_root}/lib/libdeepmd_lmp.so
pair_style      deepmd ${path_of_mlp}
pair_coeff      * *   O  H
```
---

## 9. Slurm script

```bash
#!/bin/bash
#SBATCH -N 1
#SBATCH --ntasks-per-node=4
#SBATCH -J iceIh
#SBATCH --partition=gpu1,gpu2,gpu3 
#SBATCH --gres=gpu:1

# add modulefiles
module load miniconda/3
module load mpi/2021.5.0
source activate dp-qtb
export TF_INTER_OP_PARALLELISM_THREADS=3
export TF_INTRA_OP_PARALLELISM_THREADS=2
export OMP_NUM_THREADS=2
lmp_intel_cpu_intelmpi -i lmp-qbmsst.inp                             
```
