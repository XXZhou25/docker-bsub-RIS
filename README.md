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
## Example for building a docker image interactively
This way is what I usually do, easy to debug, install packages one by one. 
For example, I want to build a docker image including both tensorflow(with cuda, cudnn for GPU support), Anaconda, some packages Basenji project required.
Since Basenji project required tensorflow version 2.8.0 which can run on GPU, it will require correct version of cuda, cudnn, has GPU as a root user(so that cuda can be successfully installed). I don't have GPU locally, and RIS doesn't allow running GPU as a root user, so finally, I choose to use tensorflow 2.8.0 as the base image, and install everything. The whole process is finished on my laptop. 
```
docker pull tensorflow/tensorflow:2.8.0-gpu-jupyter
docker run -itd tensorflow/tensorflow:2.8.0-gpu-jupyter /bin/bash
docker ps (showing 'container id' of docker images that are currently running, assuming 9e22d5b18f62 is its id)
docker exec -it 9e22d5b18f62 /bin/bash
```
After ```docker exec```, we successfully going into the docker as a root user, it usually will show ```root@9e22d5b18f62:/```.
Looking for what installed: 
```
dpkg -l
pip list 
echo $PATH
```

Manually install Basenji's dependencies based on environment.yml
