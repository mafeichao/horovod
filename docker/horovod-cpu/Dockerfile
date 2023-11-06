FROM ubuntu:20.04

#docker build -t hvd-cpu -f docker/horovod-cpu/Dockerfile .
#docker run -it -d --network=host --privileged -v/home/feichao.ma/proj/study:/workspace hvd-cpu bash
#HOROVOD_WITH_PYTORCH=1 HOROVOD_WITH_TENSORFLOW=1 pip install -v -e .

# TensorFlow version is tightly coupled to CUDA and cuDNN so it should be selected carefully
ARG TENSORFLOW_VERSION=2.9.2
ARG PYTORCH_VERSION=1.12.1
ARG PYTORCH_LIGHTNING_VERSION=1.5.9
ARG TORCHVISION_VERSION=0.13.1
ARG MXNET_VERSION=1.9.1

ARG PYSPARK_PACKAGE=pyspark==3.3.0
ARG SPARK_PACKAGE=spark-3.3.0/spark-3.3.0-bin-hadoop2.tgz

ARG PIP_SOURCE=https://pypi.tuna.tsinghua.edu.cn/simple
ARG PIP_HOST=https://pypi.tuna.tsinghua.edu.cn

ARG PYTHON_VERSION=3.8

# to avoid interaction with apt-get
ENV DEBIAN_FRONTEND=noninteractive

# Set default shell to /bin/bash
SHELL ["/bin/bash", "-euo", "pipefail", "-c"]

RUN apt-get update && apt-get install -y --allow-downgrades --allow-change-held-packages --no-install-recommends \
        build-essential \
        cmake \
        g++-7 \
        git \
        curl \
        vim \
        wget \
        ca-certificates \
        libjpeg-dev \
        libpng-dev \
        python${PYTHON_VERSION} \
        python${PYTHON_VERSION}-dev \
        python${PYTHON_VERSION}-distutils \
        librdmacm1 \
        libibverbs1 \
        ibverbs-providers \
        openjdk-8-jdk-headless \
        openssh-client \
        openssh-server \
    && apt-get clean && rm -rf /var/lib/apt/lists/*

# Install Open MPI
#RUN wget --progress=dot:mega -O /tmp/openmpi-4.1.4-bin.tar.gz https://download.open-mpi.org/release/open-mpi/v4.1/openmpi-4.1.4.tar.gz && \
#    cd /tmp && tar -zxf /tmp/openmpi-4.1.4-bin.tar.gz

# Allow OpenSSH to talk to containers without asking for confirmation
RUN mkdir -p /var/run/sshd
RUN cat /etc/ssh/ssh_config | grep -v StrictHostKeyChecking > /etc/ssh/ssh_config.new && \
    echo "    StrictHostKeyChecking no" >> /etc/ssh/ssh_config.new && \
    mv /etc/ssh/ssh_config.new /etc/ssh/ssh_config

RUN ln -s /usr/bin/python${PYTHON_VERSION} /usr/bin/python

RUN curl -O https://bootstrap.pypa.io/get-pip.py && \
    python get-pip.py && \
    rm get-pip.py

# Install PyTorch, TensorFlow, Keras and MXNet
RUN pip install --no-cache-dir torch==${PYTORCH_VERSION} torchvision==${TORCHVISION_VERSION} -i ${PIP_SOURCE} --trusted-host ${PIP_HOST}
RUN pip install --no-cache-dir pytorch_lightning==${PYTORCH_LIGHTNING_VERSION} -i ${PIP_SOURCE} --trusted-host ${PIP_HOST}

RUN pip install --no-cache-dir future typing packaging  -i ${PIP_SOURCE} --trusted-host ${PIP_HOST} 
RUN pip install --no-cache-dir \
    tensorflow-cpu==${TENSORFLOW_VERSION} \
    keras \
    h5py -i ${PIP_SOURCE} --trusted-host ${PIP_HOST}

RUN pip install --no-cache-dir mxnet==${MXNET_VERSION} "numpy<1.24.0" -i ${PIP_SOURCE} --trusted-host ${PIP_HOST}

# Install Spark stand-alone cluster.
#RUN wget --progress=dot:giga "https://www.apache.org/dyn/closer.lua/spark/${SPARK_PACKAGE}?action=download" -O - | tar -xzC /tmp; \
#    archive=$(basename "${SPARK_PACKAGE}") bash -c "mv -v /tmp/\${archive/%.tgz/} /spark"

# Install PySpark.
RUN pip install --no-cache-dir ${PYSPARK_PACKAGE} -i ${PIP_SOURCE} --trusted-host ${PIP_HOST}

# Install Horovod
WORKDIR /horovod
COPY . .
RUN cat spark.tgz | tar -xzC /tmp; \
    archive=$(basename "${SPARK_PACKAGE}") bash -c "mv -v /tmp/\${archive/%.tgz/} /spark"

RUN cat openmpi-4.1.4-bin.tar.gz | tar -xzC /tmp; \
    cd /tmp && mkdir openmpi-4.1.4/build && cd openmpi-4.1.4/build && ../configure --prefix=/usr/local && \
    make -j all && make install && ldconfig && \
    mpirun --version

RUN python setup.py sdist && \
    bash -c "HOROVOD_WITH_TENSORFLOW=1 HOROVOD_WITH_PYTORCH=1 HOROVOD_WITH_MXNET=1 pip install --no-cache-dir -v $(ls /horovod/dist/horovod-*.tar.gz)[spark,ray]" && \
    horovodrun --check-build

# Check all frameworks are working correctly
WORKDIR "/horovod/examples"
RUN python -c "import horovod.tensorflow as hvd; hvd.init()" && \
    python -c "import horovod.torch as hvd; hvd.init()" && \
    python -c "import horovod.mxnet as hvd; hvd.init()"
