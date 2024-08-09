# JEDI Development Environment Set-up on Ubuntu 22.04 with GCC Offload Support

## 0. Provision an Azure VM
Follow the Azure documentation with the following modifications:
- When selecting a VM we chose `Standard NC6s v3`
- When selecting an operating system we chose `Ubuntu Server 22.04 LTS - x64 Gen2`
- When selecting a disc size we chose 256 GiB to support building spack-stack and JEDI

References:
- [Quickstart: Create a Linux virtual machine in the Azure portal](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal?tabs=ubuntu)

## 1. Install Nvidia Drivers

Install the Nvidia Drivers as follows:

1. Check for the presence of an Nvidia GPU (should display name of GPU):
   ```
   lspci | grep -i NVIDIA
   ```
1. Install Ubuntu drivers utility:
   ```
   sudo apt update && sudo apt install -y ubuntu-drivers-common
   ```
1. Install Nvidia drivers:
   ```
   sudo ubuntu-drivers install nvidia:535
   ```
1. Add some Nvidia driver packages to a no upgrade list to protect against `apt upgrade` introducing version miss-matches:
   ```
   sudo apt-mark hold libnvidia-common-535 libxnvctrl0
   ```
1. **Reboot** machine.
1. Verify that the driver is working (should report info about GPU):
   ```
   nvidia-smi
   ```

References:
- [Azure N-Series Linux VM Driver Installation](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/n-series-driver-setup)
- [Ubuntu Driver Installation](https://ubuntu.com/server/docs/nvidia-drivers-installation)

## 2. Install CUDA Toolkit

Assuming that you have Nvidia drivers installed, install the CUDA Toolkit as follows:

1. Add Nvidia package repository:
    ```
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2204/x86_64/cuda-keyring_1.1-1_all.deb
    sudo apt install -y ./cuda-keyring_1.1-1_all.deb
    sudo apt update
    ```
1. Install CUDA Toolkit:
   ```
   sudo apt -y install cuda-toolkit-12-2
   ```
1. Set-up paths (you may want to add these to `.profile` or `.bashrc` etc):
    ```
    export PATH=/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    ```

References:
- [CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

## 3. Install GCC-12 with Offload Support

1. Install nvptx-tools using the following script. The script creates a tempory directory in the CWD, where it builds the package from source before installing it (during execution, a key press will be necessary due to the `apt-add-repository` command). **Note**: the reason that we are installing nvptx-tools like this is because the nvptx-tools package for Ubuntu 22.04 is tool old for gcc-12 and using this old package results in compilation issues.
    ```
    #!/bin/bash

    ### Description:
    # This script is for installing `nvptx-tools` version 0.20240423-1 from source on Ubuntu 22.04 which is newer than the system package.

    TMP_DIR=$PWD/tmp

    ### Install dependencies
    sudo apt update
    sudo apt install -y build-essential software-properties-common fakeroot debhelper autotools-dev help2man
    wget -O - https://apt.llvm.org/llvm-snapshot.gpg.key|sudo apt-key add -
    sudo apt-add-repository "deb http://apt.llvm.org/jammy/ llvm-toolchain-jammy-18 main"
    sudo apt update
    sudo apt install -y llvm-18-tools

    ### Create temporary build directory
    mkdir -p $TMP_DIR

    ### Download and prepare source
    cd $TMP_DIR
    wget http://archive.ubuntu.com/ubuntu/pool/universe/n/nvptx-tools/nvptx-tools_0.20240423.orig.tar.xz
    tar -xf nvptx-tools_0.20240423.orig.tar.xz
    wget http://archive.ubuntu.com/ubuntu/pool/universe/n/nvptx-tools/nvptx-tools_0.20240423-1.debian.tar.xz
    tar -xf nvptx-tools_0.20240423-1.debian.tar.xz
    mv debian ./nvptx-tools-0.20240423/

    ### Build package
    cd $TMP_DIR/nvptx-tools-0.20240423/
    dpkg-buildpackage -rfakeroot -b -uc

    ### Install package
    cd $TMP_DIR
    sudo dpkg -i ./nvptx-tools_0.20240423-1_amd64.deb

    ### Destroy temporary build directory
    rm -rf $TMP_DIR
    ```

1. Install gcc-12 with offload supports:
   ```
   sudo apt install -y gcc-12 g++-12 gfortran-12 gcc-12-offload-nvptx
   ```

## 4. Set up Spack-Stack

Set up spack-stack as per its documentation (see [Ubuntu Prerequisites](https://spack-stack.readthedocs.io/en/1.7.0/NewSiteConfigs.html#prerequisites-ubuntu-one-off) and [Create a new environment](https://spack-stack.readthedocs.io/en/1.7.0/NewSiteConfigs.html#newsiteconfigs-linux-createenv)) with the following modifications:
- (optional) When installing the prerequisites, you can skip the installation of the compilers `gcc g++ gfortran` which will install gcc-11 on Ubuntu 22.04.
- When cloning spack-stack, make sure to specify the latest stable release `v1.7.0` to avoid any breakage on the develop branch:
    - clone via `git clone -b release/1.7.0 --recurse-submodules https://github.com/jcsda/spack-stack.git`
    - make sure to follow the setup instructions _for v1.7.0_ as in the links above — check the lower-right corner of the webpage for the documentation version
- When creating the envionment:
    - at step (5): verify that gcc-12 was found
    - at step (7): run `gcc-12 --version` and use the reported version as `YOUR-VERSION`
    - (optional) at step (10): for the jedi-bundle branch `feature/gpu-offload-example` you only need to install the spack metapackage `jedi-base-env`.

Then load the spack-stack environment (you may want to add these to `.profile` or `.bashrc` etc):

```
export SPACK_STACK_DIR="path/to/spack-stack"
module use "${SPACK_STACK_DIR}/envs/azure-skylab-dev/install/modulefiles/Core"
module purge
module load stack-gcc/12.3.0
module load stack-mpich/4.1.2
module load stack-python/3.10.13
module load jedi-base-env
module --no-auto --force unload ecmwf-atlas  # unload atlas with minimal dependency tracking
module load qhull  # reload atlas dependency
```

**Note**: We anticipate that the GPU project will end up using custom branches of atlas. That's why we unload the spack-stack atlas module above, and build atlas within jedi-bundle in the section below.

## 5. Build JEDI

Clone and build the jedi-bundle on branch `feature/gpu-offload-example`; this adds in-bundle atlas, trims the bundle to the repos needed to demonstrate GPU offload in saber, and sets up the GPU offload build environment.

```
mkdir jedi && cd jedi
git clone -b feature/gpu-offload-example https://github.com/JCSDA-internal/jedi-bundle
mkdir build && cd build
ecbuild ../jedi-bundle
make -j6
ctest
```

In this branch, the oops helper `util::multiplyFieldSet` is offloaded to GPU. This can be triggered, for example, in the ctest `saber_test_dirac_stddev_1_1-1`

**Note**: the atlas ctest `atlas_fctest_field_host` is expected to fail because of a bug in the test itself (atlas issue #216)
    


