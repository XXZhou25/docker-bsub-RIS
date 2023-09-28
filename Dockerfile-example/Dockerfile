FROM continuumio/anaconda3

SHELL ["/bin/bash", "--login", "-c"]

COPY environment.yml .
RUN conda env create -f environment.yml
RUN conda init bash
RUN conda activate basenji

RUN conda install -c conda-forge cudatoolkit=11.8.0
RUN python3 -m pip install nvidia-cudnn-cu11==8.6.0.163 tensorflow==2.13.0
RUN mkdir -p $CONDA_PREFIX/etc/conda/activate.d
RUN echo 'CUDNN_PATH=$(dirname $(python -c "import nvidia.cudnn;print(nvidia.cudnn.__file__)"))' >> $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh
RUN echo 'export LD_LIBRARY_PATH=$CUDNN_PATH/lib:$CONDA_PREFIX/lib/:$LD_LIBRARY_PATH' >> $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh
RUN . $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh