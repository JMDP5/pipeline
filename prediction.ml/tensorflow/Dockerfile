FROM fluxcapacitor/package-java-openjdk-1.8:master

WORKDIR /root

ENV \
 BAZEL_VERSION=0.4.4 \
 TENSORFLOW_SERVING_VERSION=0.5.1

# TensorFlow Serving Home (not required on PATH)
ENV \
  TENSORFLOW_HOME=/root/serving/tensorflow \
  TENSORFLOW_SERVING_HOME=/root/serving

# Required by TensorFlow Serving
RUN \
  apt-get update && apt-get install -y \
        build-essential \
        curl \
        libcurl3-dev \
        git \
        libfreetype6-dev \
        libpng12-dev \
        libzmq3-dev \
        pkg-config \
        python-dev \
        python-numpy \
        python-pip \
        software-properties-common \
        swig \
        zip \
        zlib1g-dev

RUN \
  pip install grpcio

# Install Python with conda
RUN wget -q https://repo.continuum.io/miniconda/Miniconda3-4.1.11-Linux-x86_64.sh -O /tmp/miniconda.sh  && \
    echo '874dbb0d3c7ec665adf7231bbb575ab2 */tmp/miniconda.sh' | md5sum -c - && \
    bash /tmp/miniconda.sh -f -b -p /opt/conda && \
    /opt/conda/bin/conda install --yes python=3.5 sqlalchemy tornado jinja2 traitlets requests pip && \
    /opt/conda/bin/pip install --upgrade pip && \
    rm /tmp/miniconda.sh

ENV \
  PATH=/opt/conda/bin:$PATH

RUN \
  conda install --yes openblas scikit-learn numpy scipy matplotlib pandas seaborn

RUN \
  echo "build --spawn_strategy=standalone --genrule_strategy=standalone" >>/root/.bazelrc

ENV \
  BAZELRC=/root/.bazelrc

RUN \
  mkdir /root/bazel \
  && cd /root/bazel \
  && curl -fSsL -O https://github.com/bazelbuild/bazel/releases/download/$BAZEL_VERSION/bazel-$BAZEL_VERSION-installer-linux-x86_64.sh \
  && curl -fSsL -o /root/bazel/LICENSE.txt https://raw.githubusercontent.com/bazelbuild/bazel/master/LICENSE.txt \
  && chmod +x bazel-*.sh \
  && ./bazel-$BAZEL_VERSION-installer-linux-x86_64.sh \
  && rm bazel-$BAZEL_VERSION-installer-linux-x86_64.sh

# TensorFlow Serving
RUN \
 cd ~ \
 && git clone -b master --single-branch --recurse-submodules https://github.com/tensorflow/serving.git \
 && cd $TENSORFLOW_SERVING_HOME \
 && git reset --hard be83fe98104f26362dd2819d20b53ffac9b8f592

ENV TF_NEED_CUDA=0
ENV TF_NEED_GCP=0
ENV TF_NEED_JEMALLOC=1
ENV TF_NEED_HDFS=0
ENV TF_NEED_OPENCL=0
ENV TF_ENABLE_XLA=0

RUN \
  cd $TENSORFLOW_HOME \
  && printf "\n\n\n" | ./configure

RUN \
  cd $TENSORFLOW_SERVING_HOME \
  && bazel build -c opt --spawn_strategy=standalone tensorflow_serving/...

ENV \
  PATH=$TENSORFLOW_SERVING_HOME/bazel-bin/tensorflow_serving/model_servers/:$PATH

ENV TF_ENABLE_XLA=1
RUN \
  cd $TENSORFLOW_HOME \
  && bazel build -c opt --spawn_strategy=standalone tensorflow/...

RUN \
  cd $TENSORFLOW_HOME \
  && bazel-bin/tensorflow/tools/pip_package/build_pip_package /tmp/pip

RUN \
  pip --no-cache-dir install --ignore-installed --upgrade /tmp/pip/tensorflow-*.whl

COPY lib/ lib/

RUN \
  cd ~/lib/jni \
  && ln -s ~/lib/jni/libtensorflow_jni-cpu.so libtensorflow_jni.so

RUN \
  mkdir -p /root/logs

ENV \
  LOGS_HOME=/root/logs

COPY src/ src/
COPY build.sbt build.sbt
COPY client/ client/

RUN \
  cd client \
  && /root/apache-maven-3.3.9/bin/mvn clean install \
  && cp target/tensorflow-prediction-client-1.0-SNAPSHOT.jar ../lib

RUN \
  sbt clean package

COPY run run

RUN \
  mkdir -p /root/tensorboard

ENV \
  STORE_HOME=/root/store

RUN \
  ln -s /root/volumes/source.ml/prediction.ml/tensorflow/store

EXPOSE 5678 6006 9000 9001 9002 9040

CMD ["supervise", "."]
