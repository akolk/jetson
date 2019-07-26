FROM max-one.local:5001/jetson/ml-base

MAINTAINER Helmut Hoffer von Ankershoffen <helmuthva@googlemail.com>

# Install supervisord
RUN apt-get update && \
    apt-get install -y supervisor && \
    apt-get clean && \
    mkdir -p /var/log/supervisor

# Build bazel build
ENV BAZEL_VERSION=0.24.1
RUN wget --quiet https://github.com/bazelbuild/bazel/releases/download/$BAZEL_VERSION/bazel-$BAZEL_VERSION-dist.zip && \
    mkdir bazel-$BAZEL_VERSION && \
    unzip -q bazel-$BAZEL_VERSION-dist.zip -d bazel-$BAZEL_VERSION && \
    cd bazel-$BAZEL_VERSION && \
    ./compile.sh && \
    cp -f output/bazel /usr/local/bin && \
    cd .. && \
    rm -rf bazel*

# Build TF serving from source using bazel build including Python binding
WORKDIR /tensorflow-serving
ENV BUMP=2
ENV TF_SERVING_VERSION_GIT_BRANCH=master
ENV TF_SERVING_VERSION_GIT_COMMIT=master
# TensorRT version as provided by ml-base
ENV TF_TENSORRT_VERSION=5.1.6
# CUDNN version as provided by ml-base
ENV CUDNN_VERSION=7.5.0
# CUDA as provided by ml-base
ENV TF_NEED_CUDA=1
# CUDA capabilities of Jetson (Nano)
ENV TF_CUDA_COMPUTE_CAPABILITIES=5.3
# Use TensorRT as provided in ml-base
ENV TF_NEED_TENSORRT=1
# Use nvcc as the CUDA compiler
ENV TF_CUDA_CLANG=0
# No nccl
ENV TF_NCCL=0
# No Google Cloud platform support
ENV TF_NEED_GCP=0
# No Hadoop file system support
ENV TF_NEED_HDFS=0
# No Kafka file system support
ENV TF_NEED_KAFKA=0
# No Amazon S3 support
ENV TF_NEED_S3=0
ENV TF_NEED_AWS=0
# No Apache Ignite
ENV TF_IGNITE=0
# Use JE Malloc
ENV TF_NEED_JEMALLOC=1
# No GDR
ENV TF_NEED_GDR=0
# No OpenCL support (this is an NVIDIA board, don't use OpenCL)
ENV TF_NEED_OPENCL=0
# No XLA
ENV TF_ENABLE_XLA=0
# No MKL available for ARM
ENV TF_NEED_MKL=0
ENV TF_DOWNLOAD_MKL=0
# No MPI Support (we use the video card, no need for this)
ENV TF_NEED_MPI=0
# No TF verbs
ENV TF_NEED_VERBS=0
# No nGraph
ENV TF_NEED_NGRAPH=0
# Required for ncc
ENV TMP=/tmp
# Use python of Anaconda as provided in ml-base
ENV BAZEL_PYTHON=/opt/archiconda3/bin/python
COPY /.bazelrc /tmp/tfs.bazelrc
RUN git clone --branch=${TF_SERVING_VERSION_GIT_BRANCH} https://github.com/tensorflow/serving . && \
    git remote add upstream https://github.com/tensorflow/serving.git && \
    if [ "${TF_SERVING_VERSION_GIT_COMMIT}" != "head" ]; then git checkout ${TF_SERVING_VERSION_GIT_COMMIT} ; fi && \
    \
    bazel \
    --bazelrc=/tmp/tfs.bazelrc \
    build \
    --color=yes \
    --curses=yes \
    --jobs="2" \
    --local_resources="1000,1.0,1.0" \
    --verbose_failures \
    --output_filter=DONT_MATCH_ANYTHING \
    --config=cuda \
    --config=nativeopt \
    --copt="-fPIC" \
    --config=jetson \
    tensorflow_serving/model_servers:tensorflow_model_server && \
    cp /tensorflow-serving/bazel-bin/tensorflow_serving/model_servers/tensorflow_model_server /usr/local/bin/tensorflow_model_server && \
    \
    bazel \
    --bazelrc=/tmp/tfs.bazelrc \
    build \
    --color=yes \
    --curses=yes \
    --jobs="2" \
    --local_resources="1000,1.0,1.0" \
    --verbose_failures \
    --output_filter=DONT_MATCH_ANYTHING \
    --config=cuda \
    --config=nativeopt \
    --copt="-fPIC" \
    --config=jetson \
    tensorflow_serving/tools/pip_package:build_pip_package && \
    bazel-bin/tensorflow_serving/tools/pip_package/build_pip_package \
    /tmp/pip && \
    pip --no-cache-dir install \
    /tmp/pip/tensorflow_serving_api_gpu-*.whl && \
    rm -rf /tmp/pip

# Create model directory
ENV MODEL_BASE_PATH=/models
ENV MODEL_NAME=default
RUN mkdir -p ${MODEL_BASE_PATH}

# Install supervisord configuration
COPY /etc/supervisor/conf.d /etc/supervisor/conf.d

# Expose ports
# grpc
EXPOSE 8500
# rest
EXPOSE 8501

# Disable CUDA for TFS - reactivate in childs
ENV CUDA_VISIBLE_DEVICES='0'

CMD ["/usr/bin/supervisord"]


