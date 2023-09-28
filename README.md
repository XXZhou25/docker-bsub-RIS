## Building a Custom Docker Image
In general, there are two ways to build a custom docker image.
1. Interactive approach
   by entering the docker image as a root user, then install everything within it. 
   - Install Docker engine, see https://docs.docker.com/engine/install/
   - Once Docker is set up, you can use it via the command line for various tasks.
2. Non-interactive approach
   by writing a Dockerfile, to build the docker.
   - This approach can be performed both locally and on RIS Compute.
   - On local device, after creating your own Dockerfile, run something like:
     ```
     cd /path/to/Dockerfile  
     docker build -f dockerfile .
     ```
     (Note: I haven't tried this locally; please refer to Docker documentation for more accurate details/commands)
   - RIS docker_build LSF(see https://docs.ris.wustl.edu/doc/compute/recipes/docker-on-compute.html?highlight=docker_build):
     Assuming I want to build a docker image from a Dockerfile and push it to my Docker Hub repository xxzhou25, with the name 'basenji', and version '1.0', use the following commands
     ```
     cd /path/to/Dockerfile 
     bsub -G compute-yeli -q general-interactive -Is -a 'docker_build(xxzhou25/basenji:1.0)' -- --tag xxzhou25/basenji:1.0  .
     ```
     If the Dockerfile has a different name (e.g., "test"), you can specify it as follows:
     ```
     cd /path/test 
     bsub -G compute-yeli -q general-interactive -Is -a 'docker_build(xxzhou25/basenji:1.0)' -- -f test --tag xxzhou25/basenji:1.0  .
     ```
     Note:  
     RIS recommends building Docker images on a workstation or other computer you have local access to as it makes debugging the build process easier. However, some build processes may require more resources than you have available locally. For these situations, the compute cluster can be used.  
     See: https://docs.ris.wustl.edu/doc/compute/recipes/docker-on-compute.html#docker-on-compute
## Example: Building a docker image interactively
This is the recommended method for ease of debugging and installing packages individually.  
  
Let's say we want to create a Docker image that includes TensorFlow (with CUDA and cuDNN for GPU support), Anaconda, and various packages required for the Basenji project. Since the Basenji project specifically requires TensorFlow version 2.8.0 with GPU support, we need the correct versions of CUDA and cuDNN, as well as GPU access as a root user (for successful CUDA installation).  
  
However, I don't have GPU access on my local machine, and RIS doesn't permit GPU access as a root user. Therefore, I chose to use TensorFlow 2.8.0 as the base image and manually install everything. The entire process takes place on my laptop.
```
docker pull tensorflow/tensorflow:2.8.0-gpu-jupyter
docker run -itd tensorflow/tensorflow:2.8.0-gpu-jupyter /bin/bash
docker ps (showing 'container id' of docker images that are currently running, assuming 9e22d5b18f62 is its id)
docker exec -it 9e22d5b18f62 /bin/bash
```
After ```docker exec```, we successfully enter the Docker container as a root user, typically indicated by the prompt ```root@CONTAINER_ID:/```(in this case, ```root@9e22d5b18f62:/```).  
Then manually install packages required by basenji: 
```
pip install cython
pip install seaborn
pip install ......
```
(Note: Please be very careful about the path to the tensorflow, especially if we plan to create a conda environment, we should add tensorflow's path to it)  
Some useful commands: 
```
dpkg -l
pip list
which python
echo $PATH
```
If you wish to install Anaconda, you can use these commands:
```
apt-get update 
apt-get install wget
wget https://repo.anaconda.com/archive/Anaconda3-2023.07-2-Linux-x86_64.sh
bash Anaconda3-2023.07-2-Linux-x86_64.sh
source .bashrc
```
**Please be very careful about the installation path of Anaconda. Avoid using "/root/Anaconda" as the path, as it may lead to issues when using this Docker image non-interactively, as you won't have permission to source "/root".**
After everything installed, exit the docker image by ```exit```. Commit your changes and push it to your docker hub by: 
```
docker commit 9e22d5b18f62 xxzhou25/basenji:1.0
docker tag xxzhou25/basenji:1.0 xxzhou25/basenji:1.0
docker push xxzhou25/basenji:1.0
```
This process allows you to create a Docker image with all the required components for your project.
## Example: submit a Bsub job which executing the scripts you want on your docker image, and running on GPU
From RIS Manual, https://docs.ris.wustl.edu/doc/compute/recipes/job-execution-examples.html?highlight=nvidiaa100_sxm4_40gb#gpu-resources
There are three types of GPU in general queue:
- Double Precision
  - TeslaV100_SXM2_32GB
  - NVIDIAA100_SXM4_40GB
- Single Precision
   - NVIDIAA40
Choose whichever GPU you want your job runs on. 
Then, create a bsub job file, eg: train.bsub
```
#!/bin/bash

# JOB HEADERS HERE MEANS THE JOB...
#BSUB -q general
### is submitted to the general queue (-q general)

#BSUB -g /xiaoxiao.z/ood-job-group
### uses the job group /$USER/ood-job-group (-g /$USER/ood-job-group)

#BSUB -R 'gpuhost' 
#BSUB -gpu "num=1:gmodel=NVIDIAA100_SXM4_40GB"

#BSUB -a 'docker(xxzhou25/tf-conda-basenji:1.0)' 
### uses the docker image ubuntu (-a 'docker(ubuntu'))

#BSUB -n 8
### requires 1 processor (-n 1). Use -n 2 for 2 processors.

#BSUB -M 64GB
### will be killed if its memory usage exceeds 2GB (-M 2GB)

#BSUB -W 3600
### will be killed if it runs for longer than 10 minutes (-W 10)

#BSUB -R 'select[mem>32GB && tmp>32GB] rusage[mem=40GB, tmp=40GB] span[hosts=1]'
### will only run on a host with more than 2GB of RAM and 1 GB of local temp disk (-R ‘select[mem>2GB && tmp>1GB]’)
### will consume 2GB of RAM, 1GB of local temp disk (-R ‘rusage[mem=2GB, tmp=1GB]’)
### will span only a single host (-R ‘span[hosts=1]’)

python -c "import tensorflow as tf; print(\"Num GPUs Available: \", len(tf.config.list_physical_devices('GPU')))"

cd /storage1/fs1/yeli/Active/xiaoxiao.zhou/projects/basenji/
python setup.py develop --no-deps --user

cd /storage1/fs1/yeli/Active/xiaoxiao.zhou/projects/basenji/tutorials
python ../bin/basenji_train.py models/params_small.json data/heart_l131k -o models/heart
```
If you want to activate your conda environment, please add those commands in train.bsub:
```
export PATH=/opt/conda/bin:$PATH
. /opt/conda/etc/profile.d/conda.sh
conda activate /opt/conda/envs/basenji
```
Again, please be very careful about the $PATH, since we want our python can find the packages we want to use. 
Sometimes, you could possibly solve it by something like adding path to environment variable, or changing code by adding: 
```
import sys
sys.path.append('')
```
After constructing the .bsub file, submitting the bsub job by
```
bsub < train.bsub
```
You will receive an email once the job is finished. 
It will tell you where the output saved, eg: 
```
Output is larger than limit of 10 KB set by administrator.
Output will be saved at /home/xiaoxiao.z/.lsbatch/1695845435.380425.out.
```
then you can check the output by: 
```
cat /home/xiaoxiao.z/.lsbatch/1695845435.380425.out
```
