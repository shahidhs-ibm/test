# Building Prow

The instructions specify the steps to build [Prow](https://github.com/kubernetes/test-infra) on Linux on IBM Z for following distributions:

* RHEL (7.8, 7.9, 8.4, 8.6, 8.7, 9.0, 9.1)
* SLES (12 SP5, 15 SP4)
* Ubuntu (18.04, 20.04, 22.04, 22.10)

_**General Notes:**_

* _When following the steps below please use a standard permission user unless otherwise specified._
* _A directory `/<source_root>/` will be referred to in these instructions, this is a temporary writable directory anywhere you'd like to place it._

## Step 1: Build environment set up
Ensure the current user belongs to group `docker`.

Use the below command to add group `docker` if it does not exist:
  ```bash
  sudo groupadd docker
  ```
Use the below command to add current user to group `docker` if it has not been done:
  ```bash
  sudo usermod -aG docker $USER && newgrp docker
  ```

## Step 2: Build using script

If you want to build Prow using manual steps, go to STEP 3.

Use the following commands to build Prow using the build [script](https://github.com/linux-on-ibm-z/scripts/tree/master/Prow). Please make sure you have wget installed.

```bash
wget https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Calico/3.24.5/build_calico.sh

# Build Calico
bash build_calico.sh   [Provide -t option for executing build with tests]
```

If the build and tests complete successfully, go to STEP 7. In case of error, check logs at `<source_root>/logs/` for more details or go to STEP 3 to follow manual build steps.

```bash
export SOURCE_ROOT=/<source_root>/
source $SOURCE_ROOT/setenv.sh    #Source environment file
```

## Step 3: Install the system dependencies and Docker

* RHEL (7.8, 7.9, 8.4, 8.6, 8.7, 9.0, 9.1)

  ```bash
  sudo yum install -y zip tar unzip git vim wget make curl python38-dev gcc gcc-c++ libtool autoconf
  sudo yum remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine podman runc
  sudo yum install -y yum-utils
  sudo yum-config-manager --add-repo https://download.docker.com/linux/rhel/docker-ce.repo
  sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo systemctl start docker
  sudo usermod -aG docker $USER && newgrp docker
  ```

* SLES (12 SP5, 15 SP4)

  ```bash
  sudo zypper refresh
  sudo zypper install -y zip tar unzip git vim wget make curl python3-devel gcc gcc-c++ libtool autoconf
  sudo zypper remove docker docker-client docker-client-latest docker-common docker-latest docker-latest-logrotate docker-logrotate docker-engine runc
  sudo zypper addrepo https://download.docker.com/linux/sles/docker-ce.repo
  sudo zypper install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
  sudo systemctl start docker
  sudo usermod -aG docker $USER && newgrp docker
  ```

* Ubuntu (18.04, 20.04, 22.04, 22.10)

  ```bash
  sudo apt-get update
  sudo apt-get install -y zip tar unzip git vim wget make curl python2.7-dev python3.8-dev gcc g++ python3-distutils libtool libtool-bin autoconf
  sudo apt-get remove docker docker-engine docker.io containerd runc
  sudo apt-get update
  sudo apt-get install ca-certificates curl gnupg lsb-release
  sudo mkdir -m 0755 -p /etc/apt/keyrings
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
  sudo apt-get update
  sudo chmod a+r /etc/apt/keyrings/docker.gpg
  sudo apt-get update
  sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
  sudo systemctl start docker
  sudo usermod -aG docker $USER && newgrp docker
  ```

## Step 4: Install prerequisites

### 4.1) Install Go 1.19

Instructions for building Go can be found [here](https://github.com/linux-on-ibm-z/docs/wiki/Building-Go)
  ```bash
  mkdir $SOURCE_ROOT/go
  export GOPATH=$SOURCE_ROOT/go
  ```

### 4.2) Build and install jq

```bash
cd $SOURCE_ROOT
git clone https://github.com/stedolan/jq.git
cd jq/
git checkout jq-1.5
autoreconf -fi
./configure --disable-valgrind
sudo make -j$(nproc)
sudo make install
```

### 4.3) Build and install Kubectl and Kind

```bash
cd $SOURCE_ROOT
mkdir -p go/src/k8s.io
cd go/src/k8s.io
git clone https://github.com/kubernetes/kubernetes.git
cd kubernetes/
git checkout v1.24.10
sudo ln -s /usr/bin/gcc /usr/bin/s390x-linux-gnu-gcc
make -j$(nproc) all
sudo -E env PATH=$PATH GOPATH=$GOPATH KUBE_RELEASE_RUN_TESTS=N KUBE_GIT_VERSION=$(git --version | awk '{print $3}')  KUBE_BUILD_PLATFORMS=linux/s390x build/release.sh
export PATH=$SOURCE_ROOT/go/src/k8s.io/kubernetes/_output/local/go/bin:$PATH
kubectl version --short
cd ..
git clone https://github.com/kubernetes-sigs/kind.git
cd kind/
git checkout v0.17.0
sed -i 's/@sha256:f52781bc0d7a19fb6c405c2af83abfeb311f130707a0e219175677e366cc45d1//g' pkg/apis/config/defaults/image.go
make build
export PATH=$SOURCE_ROOT/go/src/k8s.io/kind/bin:$PATH
sudo -E env PATH=$PATH GOPATH=$GOPATH KUBE_RELEASE_RUN_TESTS=N KUBE_GIT_VERSION=$(git --version | awk '{print $3}') KUBE_BUILD_PLATFORMS=linux/s390x kind build node-image
docker tag kindest/node:latest kindest/node:v1.25.3
```

## Step 5: Build required components and images

### 5.1) Build Prow from source
  ```bash
  cd $SOURCE_ROOT
  git clone https://github.com/kubernetes/test-infra.git
  git checkout 89ac42333a8e7d3d88eda931740199b2a25252ea
  curl -s https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Prow/master/patch/test-infra-build.patch | patch -p1
  make -C prow build-images
  ```

### 5.2) Run Prow unit tests
  ```bash
  cd $SOURCE_ROOT/test-infra/
  mkdir _bin/jq-1.5
  cp /usr/local/bin/jq _bin/jq-1.5/jq
  cp /usr/local/bin/jq _bin/jq-1.5/jq-linux64
  make test
  ```

## Step 6: Prow deploy on Kind node (Optional)



## Step 7: Prow Integration

Please refer to [Prow Building, Testing, and Deploying](https://docs.prow.k8s.io/docs/getting-started-develop/#building-testing-and-deploying) for more information.

## References

- https://docs.prow.k8s.io/
