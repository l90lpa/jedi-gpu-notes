# JEDI Development Environment Set-up on Ubuntu 24.04 with GCC Offload Support

## 0. Provision an Azure VM
Follow the Azure documentation with the following modifications:
- When selecting a VM we chose `Standard NC6s v3`
- When selecting an operating system we chose `Ubuntu Server 24.04 LTS - x64 Gen2`
- When selecting a disc size we chose 256 GiB to support building spack-stack and JEDI

References:
- [Quickstart: Create a Linux virtual machine in the Azure portal](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/quick-create-portal?tabs=ubuntu)

## 1. Upgrade System Packages

1. Upgrade system packages:
   ```
   sudo apt update
   sudo apt upgrade
   ```

## 2. Install Nvidia Drivers

Install the Nvidia Drivers as follows:

1. Check for the presence of an Nvidia GPU (should display name of GPU):
   ```
   lspci | grep -i NVIDIA
   ```
1. Install Ubuntu drivers utility:
   ```
   sudo apt update && sudo apt install -y ubuntu-drivers-common
   ```
1. Install Nvidia drivers (and utilities):
   ```
   sudo ubuntu-drivers install --gpgpu nvidia:570-server
   sudo apt install nvidia-utils-570-server
   ```
1. **Reboot** machine.
1. Verify that the driver is working (should report info about GPU):
   ```
   nvidia-smi
   ```

References:
- [Azure N-Series Linux VM Driver Installation](https://learn.microsoft.com/en-us/azure/virtual-machines/linux/n-series-driver-setup)
- [Ubuntu Driver Installation](https://ubuntu.com/server/docs/nvidia-drivers-installation)

## 3. Install CUDA Toolkit

Assuming that you have Nvidia drivers installed, install the CUDA Toolkit as follows:

1. Add Nvidia package repository:
    ```
    wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
    sudo apt install -y ./cuda-keyring_1.1-1_all.deb
    rm ./cuda-keyring_1.1-1_all.deb
    sudo apt update
    ```
1. Install CUDA Toolkit:
   ```
   sudo apt -y install cuda-toolkit-12-8
   ```
1. Set-up paths (you may want to add these to `.profile` or `.bashrc` etc):
    ```
    export PATH=/usr/local/cuda/bin:$PATH
    export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH
    ```

References:
- [CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)

## 4. Install GCC-13 with Offload Support

1. Install gcc-13 with offload supports:
   ```
   sudo apt install -y gcc-13 g++-13 gfortran-13 gcc-13-offload-nvptx
   ```

## 5. Set up Spack-Stack

Set up spack-stack as per its documentation (see [Ubuntu Prerequisites](https://spack-stack.readthedocs.io/en/1.9.1/NewSiteConfigs.html#prerequisites-ubuntu-one-off) and [Create a new environment](https://spack-stack.readthedocs.io/en/1.9.1/NewSiteConfigs.html#newsiteconfigs-linux-createenv)) with the following modifications:
- When installing the prerequisites:
   - **Do not** perform the `apt upgrade` command.
   - (optional) you can skip the installation of the compilers `gcc g++ gfortran`.
- When cloning spack-stack, make sure to specify the latest stable release `v1.9.1` to avoid any breakage on the develop branch:
    - clone via `git clone -b release/1.9.1 --recurse-submodules https://github.com/jcsda/spack-stack.git`
    - make sure to follow the setup instructions _for v1.9.1_ as in the links above — check the lower-right corner of the webpage for the documentation version
- When creating the envionment:
    - at step (5): verify that gcc-13 was found
    - at step (7): run `gcc-13 --version` and use the reported version as `YOUR-VERSION`

Then load the spack-stack environment (you may want to add these to `.profile` or `.bashrc` etc). Here we have present loading `jedi-base-env` which should be enough for building and testing OOPS, VADER, and SABER. Note, use `module avail` to confirm the versions of the meta-packages (i.e. stack-*) that you have and use them:

```
export SPACK_STACK_DIR="path/to/spack-stack"
module use "${SPACK_STACK_DIR}/envs/<your-environment-name>/install/modulefiles/Core"
module purge
module load stack-gcc/13.3.0
module load stack-mpich/4.2.1
module load stack-python/3.11.7
module load jedi-base-env
```
