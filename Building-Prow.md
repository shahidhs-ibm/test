# Building Prow

The instructions specify the steps to build [Prow](https://github.com/kubernetes/test-infra) on Linux on IBM Z for following distributions:

* RHEL (7.8, 7.9, 8.4, 8.6, 8.7, 9.0, 9.1)
* SLES (12 SP5, 15 SP4)
* Ubuntu (18.04, 20.04, 22.04, 22.10)

_**General Notes:**_

* _When following the steps below please use a standard permission user unless otherwise specified._
* _A directory `/<source_root>/` will be referred to in these instructions, this is a temporary writable directory anywhere you'd like to place it._

## Step 1: Build environment set up
Ensure that [Docker-CE](https://docs.docker.com/engine/install/) is installed.

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
wget https://raw.githubusercontent.com/linux-on-ibm-z/scripts/master/Prow/master/build_prow.sh

# Build Prow
bash build_prow.sh   [Provide -t option for executing build with tests]
```

If the build and tests complete successfully, go to STEP 7. In case of error, check logs at `<source_root>/logs/` for more details or go to STEP 3 to follow manual build steps.

```bash
export SOURCE_ROOT=/<source_root>/
source $SOURCE_ROOT/setenv.sh    #Source environment file
```

## Step 3: Install the system dependencies

* RHEL (7.8, 7.9, 8.4, 8.6, 8.7, 9.0, 9.1)

  ```bash
  sudo yum install -y yum install -y zip tar unzip git vim wget make curl python38-devel gcc gcc-c++ libtool autoconf curl-devel expat-devel gettext-devel openssl-devel zlib-devel perl-CPAN perl-devel
  ```

* SLES (12 SP5, 15 SP4)

  ```bash
  sudo zypper refresh
  sudo zypper install -y zip tar unzip git vim wget make curl python3-devel gcc gcc-c++ libtool autoconf gettext-devel openssl-devel zlib-devel
  ```

* Ubuntu (18.04, 20.04, 22.04, 22.10)

  ```bash
  sudo apt-get update
  sudo apt-get install -y zip tar unzip git vim wget make curl python2.7-dev python3.8-dev gcc g++ python3-distutils libtool libtool-bin autoconf
  ```

## Step 4: Install prerequisites

### 4.1) Install Go 1.19.5

Instructions for building Go can be found [here](https://github.com/linux-on-ibm-z/docs/wiki/Building-Go)
```bash
mkdir $SOURCE_ROOT/go
export GOPATH=$SOURCE_ROOT/go
```

### 4.2) Create 's390x-linux-gnu-gcc' if not present

```bash
sudo ln -s /usr/bin/gcc /usr/bin/s390x-linux-gnu-gcc
```

### 4.3) Build and install jq

```bash
cd $SOURCE_ROOT
git clone https://github.com/stedolan/jq.git
cd jq/
git checkout jq-1.5
autoreconf -fi
./configure --disable-valgrind
sudo make LDFLAGS=-all-static -j$(nproc)
sudo make install
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

### 5.2) Run Prow unit tests (Optional)
  ```bash
  cd $SOURCE_ROOT/test-infra/
  mkdir _bin/jq-1.5
  cp /usr/local/bin/jq _bin/jq-1.5/jq
  cp /usr/local/bin/jq _bin/jq-1.5/jq-linux64
  make test
  ```
  _Note_: If Python tests fails with error `version 'GLIBC_2.XX' not found`, its due to GLIBC version mismatch between the libc.so.6 library on host and libc.so.6 library present in Python container (Image: python:3.9-slim-bullseye). 
  

## Step 6: Run ProwJobs locally as pods in a Kind cluster (Optional)

Please refer to [Running a prowjob locally](https://docs.prow.k8s.io/docs/build-test-update/#running-a-prowjob-locally) for more information.

## Step 7: Prow Integration

Please refer to [Prow Building, Testing, and Deploying](https://docs.prow.k8s.io/docs/getting-started-develop/#building-testing-and-deploying) for more information.

## References

- https://docs.prow.k8s.io/
