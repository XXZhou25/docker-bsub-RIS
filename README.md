## Ways to build a docker image
In general, there are two ways to build a custom docker image. 
1. interactive way, by going into the docker image as a root user, then install everything in it. 
   - Install Docker engine https://docs.docker.com/engine/install/
   - register your account on Docker hub
   - Then you will be able to run docker in command line, and do everything
   - You can do this locally(on your own laptop), but cannot do it on RIS Compute. 
2. non-interactive way, by constructing a Dockerfile, to build the docker
   - Can do locally and RIS Compute
   - local（I haven't tried it locally, please see more from Docker documentation）:
     ```
     cd /path/dockerfile 
     docker build -f dockerfile .
     ```
   - RIS Compute LSF(https://docs.ris.wustl.edu/doc/compute/recipes/docker-on-compute.html?highlight=docker_build):
     Assuming I want to build a docker image given a Dockerfile, push it to my docker hub xxzhou25, its name is basenji, its version is 1.0, then the command line for it will be
     ```
     cd /path/Dockerfile 
     bsub -G compute-yeli -q general-interactive -Is -a 'docker_build(xxzhou25/basenji:1.0)' -- --tag xxzhou25/basenji:1.0  .
     ```
     If the file doesn't name Dockerfile, for example named test, then:
     ```
     cd /path/test 
     bsub -G compute-yeli -q general-interactive -Is -a 'docker_build(xxzhou25/basenji:1.0)' -- -f test --tag xxzhou25/basenji:1.0  .
     ```
     RIS recommends building Docker images on a workstation or other computer you have local access to as it makes debugging the build process easier. However, some build processes may require more resources than you have available locally. For these situations, the compute cluster can be used.
     More information: https://docs.ris.wustl.edu/doc/compute/recipes/docker-on-compute.html#docker-on-compute
## Example: building a docker image interactively
This way is what I recommend most, easy to debug, install packages one by one. 
For example, I want to build a docker image including both tensorflow(with cuda, cudnn for GPU support), Anaconda, some packages Basenji project required.
Since Basenji project required tensorflow version 2.8.0 which can run on GPU, it will require correct version of cuda, cudnn, has GPU as a root user(so that cuda can be successfully installed). I don't have GPU locally, and RIS doesn't allow running GPU as a root user, so finally, I choose to use tensorflow 2.8.0 as the base image, and install everything. The whole process is finished on my laptop. 
```
docker pull tensorflow/tensorflow:2.8.0-gpu-jupyter
docker run -itd tensorflow/tensorflow:2.8.0-gpu-jupyter /bin/bash
docker ps (showing 'container id' of docker images that are currently running, assuming 9e22d5b18f62 is its id)
docker exec -it 9e22d5b18f62 /bin/bash
```
After ```docker exec```, we successfully going into the docker as a root user, it usually will show ```root@9e22d5b18f62:/```.
We should be very careful about the path to the tensorflow, especially if we want to build another conda environment, we should add tensorflow's path to it: 
```
dpkg -l
pip list
which python
echo $PATH
```
Then I manually install packages required by basenji: 
```
pip install cython
pip install seaborn
pip install ......
```
If you want to install anaconda
```
apt-get update 
apt-get install wget
wget https://repo.anaconda.com/archive/Anaconda3-2023.07-2-Linux-x86_64.sh
bash Anaconda3-2023.07-2-Linux-x86_64.sh
source .bashrc
conda install pip
conda install ......
pip install ......
```
Again, please be very careful about the path if you create your own environment, like the path to the packages in base image, path to your conda environment, python, pip, since you would want your python/pip can find packages installed in other path. 

After everything installed, exit the docker image by ```exit```. Push this docker image to the docker hub by: 
```
docker commit 9e22d5b18f62 xxzhou25/basenji:1.0
docker tag xxzhou25/basenji:1.0 xxzhou25/basenji:1.0
docker push xxzhou25/basenji:1.0
```
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

export PATH="/opt/conda/condabin:/usr/local/nvidia/bin:/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
python -c "import tensorflow as tf; print(\"Num GPUs Available: \", len(tf.config.list_physical_devices('GPU')))"

cd /storage1/fs1/yeli/Active/xiaoxiao.zhou/projects/basenji/
python setup.py develop --no-deps --user

cd /storage1/fs1/yeli/Active/xiaoxiao.zhou/projects/basenji/tutorials
python ../bin/basenji_train.py models/params_small.json data/heart_l131k -o models/heart
```
