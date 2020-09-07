# build lighthouse-0.0.762 on Arm64 

This document describes how to build lighthouse-0.0.762 on Arm64.The main reference is the build instruction in https://github.com/jenkins-x/lighthouse#building, which is written for and verified on x86-64.

# Environment
The build is run natively on Arm64 machines. The server used is:
-Memory : 32 G
-CPU: 32 cores

# prerequisites
* install go  version>=1.13.8
* install Pre-commit
* install Dep
* update go proxy and environment
* set GOPATH

# Build lighthouse

```shell
$ git clone -b v0.0.0762 https://github.com/jenkins-x/lighthouse.git
$ cd lighthouse
$ make build 
GO111MODULE=on go build -i -ldflags "-X github.com/jenkins-x/lighthouse/pkg/version.Version='0.0.769-dev+a7d56689'" -o bin/webhooks cmd/webhooks/main.go
go: downloading github.com/sirupsen/logrus v1.6.0
go: downloading github.com/jenkins-x/go-scm v1.5.157
go: downloading github.com/prometheus/procfs v0.0.11
go: downloading golang.org/x/sys v0.0.0-20200327173247-9dae0f8f5775
go: downloading github.com/tektoncd/pipeline v0.14.2
go: downloading github.com/mattn/go-zglob v0.0.1
go: downloading github.com/Azure/go-autorest v14.2.0+incompatible
go: extracting github.com/mattn/go-zglob v0.0.1
go: extracting github.com/sirupsen/logrus v1.6.0
go: downloading github.com/gorilla/sessions v1.2.0
......
go: extracting github.com/go-logr/logr v0.1.0
go: extracting gopkg.in/fsnotify.v1 v1.4.7
go: finding sigs.k8s.io/controller-runtime v0.5.0
go: finding github.com/go-logr/logr v0.1.0
go: finding gopkg.in/fsnotify.v1 v1.4.7
GO111MODULE=on go build -i -ldflags "-X github.com/jenkins-x/lighthouse/pkg/version.Version='0.0.769-dev+a7d56689'" -o bin/lighthouse-tekton-controller cmd/tektoncontroller/main.go
GO111MODULE=on go build -i -ldflags "-X github.com/jenkins-x/lighthouse/pkg/version.Version='0.0.769-dev+a7d56689'" -o bin/gc-jobs cmd/gc/main.go
$ ll bin
total 225308
drwxr-xr-x  2 root root     4096 Sep  7 11:13 ./
drwxr-xr-x 12 kong kong     4096 Sep  7 11:12 ../
-rwxr-xr-x  1 root root 50185946 Sep  7 11:12 foghorn*
-rwxr-xr-x  1 root root 38862882 Sep  7 11:13 gc-jobs*
-rwxr-xr-x  1 root root 47533778 Sep  7 11:12 keeper*
-rwxr-xr-x  1 root root 46825787 Sep  7 11:13 lighthouse-tekton-controller*
-rwxr-xr-x  1 root root 47966110 Sep  7 11:12 webhooks*
```

Now ,all binary file has been built.

next , building container images ,then deploying tekton  with helm
# build images 
the controller Dockerfile
using ubuntu18.04 as base image

```
FROM ubuntu:18.04
RUN  apt-get update &&  apt install openssh-client  ca-certificates git -y
COPY foghorn /foghorn
RUN mkdir /jxhome
ENV JX_HOME /jxhome
ENTRYPOINT ["/foghorn"]
```

Other dockerfile is similar,just change the  file path and name .

docker build -t lighthouse-foghorn:0.0.738  .

# deploy on cluster

```
git clone https://github.com/jenkins-x/lighthouse/tree/master/charts/lighthouse

cd lighthouse/charts
```

change the values.yaml,modify that yaml file to use the container image you build.

such as this :

https://github.com/yyunk/jenkins-x-arm-support/blob/master/yaml/lighthouse-myvalues.yaml

Then run:

`helm upgrade lighthouse lighthouse --install --values myvalues.yaml`

# Success criteria
is when both helm charts can be installed and the `Deployment` resources create pods which startup, start Running and don't fail / restart for 5 minutes. That means that the pods startup and don't fail basically.
`kubectl get pod -w`