# tf-conda-basenji
```
docker pull tensorflow/tensorflow:2.8.0-gpu-jupyter
docker run -itd tensorflow/tensorflow:2.8.0-gpu-jupyter /bin/bash
docker ps 
docker exec -it 9e22d5b18f62 /bin/bash
```

Looking for what installed: 
```
dpkg -l
pip list 
echo $PATH
```

Manually install Basenji's dependencies based on environment.yml
