# Build tekton-v0.15.2 on Arm64
This document how to build tekton-v.15.2 on Arm64.

The main reference is:

* https://github.com/tektoncd/pipeline/blob/master/docs/install.md

* https://github.com/jenkins-x-charts/tekton/blob/master/tekton/README.md

# Environment
The build is run natively on Arm64 machines. The server used is:
-Memory : 32 G
-CPU: 32 cores

# prerequisites
* install go  version>=1.13.8

  For arm64, follow this link to download Go 1.13.8 and get an installation instruction:

  https://golang.google.cn/dl/go1.13.8.linux-arm64.tar.gz

* install Pre-commit

  I used pip. Other installation methods exist as well.

  `$ pip install pre-commit`

* install Dep

  https://github.com/golang/dep

* update go proxy and environment

  If your area can not access Google,you should set go proxy

  `$ go env -w GO111MODULE=on`
  `$ go env -w GOPROXY=https://goproxy.io,direct`

* set GOPATH

  `$ export GOPATH=your file`
# Build tekton
```
git clone  -b v0.15.2 https://github.com/tektoncd/pipeline.git
cd pipeline
```
i do not download package lint ,  lint juse check code ,so  i  delete __lint__ in the Makefile 

```
.PHONY: all     
all: fmt lint | $(BIN) ; $(info $(M) building executable…) @ ## Build program binary  
$Q $(GO) build \  
-tags release \
-ldflags '-X $(MODULE)/cmd.Version=$(VERSION) -X $(  MODULE)/cmd.BuildDate=$(DATE)' \   
-o $(BIN)/$(basename $(MODULE)) main.go
```
After the above commands finish, you can check the build result as below.

```
ls cmd
controller  creds-init  entrypoint  git-init  imagedigestexporter  kubeconfigwriter  nop  pullrequest-init  webhook
for i in `realpath cmd/*`; do cd $i; make -f ../../Makefile all; done

ls cmd/*/.bin/github.com/tektoncd
cmd/controller/.bin/github.com/tektoncd:
pipeline
cmd/creds-init/.bin/github.com/tektoncd:
pipeline
cmd/git-init/.bin/github.com/tektoncd:
pipeline
cmd/kubeconfigwriter/.bin/github.com/tektoncd:
pipeline
cmd/nop/.bin/github.com/tektoncd:
pipeline
cmd/pullrequest-init/.bin/github.com/tektoncd:
pipeline
cmd/webhook/.bin/github.com/tektoncd:
pipeline
```


When build entrypoint ,edit the Makefile,add __post_writer.go  runner.go  waiter.go__ 
```
.PHONY: all
all: fmt  | $(BIN) ; $(info $(M) building executable…) @ ## Build program binary
$Q $(GO) build \
-tags release \
-ldflags '-X $(MODULE)/cmd.Version=$(VERSION) -X $(MODULE)/cmd.BuildDate=$(DATE)' \
-o $(BIN)/$(basename $(MODULE)) main.go post_writer.go  runner.go  waiter.go
```
```
$ cd entrypoint 
$ make -f ../../Makefile all
🐱 running gofmt…
🐱 building executable…
ll .bin/github.com/tektoncd/pipeline
-rwxr-xr-x 1 root root 36277501 Sep  7 10:43 .bin/github.com/tektoncd/pipeline*
```
When build  imagedigestexporter,edit the Makefile,add  __digest.go__ 
```
.PHONY: all
all: fmt  | $(BIN) ; $(info $(M) building executable…) @ ## Build program binary
$Q $(GO) build \
-tags release \
-ldflags '-X $(MODULE)/cmd.Version=$(VERSION) -X $(MODULE)/cmd.BuildDate=$(DATE)' \
-o $(BIN)/$(basename $(MODULE)) main.go post_writer.go  runner.go  waiter.go
```
```
$ cd  imagedigestexporter
$ make -f ../../Makefile all
🐱 running gofmt…
🐱 building executable…
ll .bin/github.com/tektoncd/pipeline
-rwxr-xr-x 1 root root 36277501 Sep  7 10:43 .bin/github.com/tektoncd/pipeline*
```

When build  gcs-fetcher ,recovery the Makefile

```
cd pipeline/vendor/github.com/GoogleCloudPlatform/cloud-builders/gcs-fetcher/cmd/gcs-fetcher
make -f pipeline/Makefile all
🐱 running gofmt…
go: not formatting packages in dependency modules
🐱 building executable…
ls .bin/github.com/tektoncd/pipeline
.bin/github.com/tektoncd/pipeline
```

Now ,all binary files are  built.

There are 

controller,  creds-init , entrypoint , gcs-fetcher , git-init,  imagedigestexporter,  kubeconfigwriter , nop , pullrequest-init,  webhook.

next , building container images ,then deploying tekton  with helm

# build images 
the controller Dockerfile
using ubuntu18.04 as base image
```
FROM ubuntu:18.04
RUN  apt-get update &&  apt install openssh-client -y
RUN mkdir /ko-app
COPY ./pipeline  /ko-app/controller
ENTRYPOINT ["/ko-app/controller"]
````

Other dockerfile is similar,just change the  file path and name .
But must copy file in /ko-app.
docker build -t controller:0.15.2-arm64 .

# deploy on cluster

`git clone https://github.com/jenkins-x-charts/tekton `
`cd tekton/tekton`
change the values.yaml,modify that yaml file to use the container image name + tag  for the image you just built on your cluster,such as

```
image:
  upstreamtag: 0.15.2-arm64
  kubeconfigwriter: kubeconfigwriter
  credsinit: creds-init
  gitinit: git-init
  controller: controller
  webhook: webhook
  entrypoint: entrypoint
  pullrequest: pullrequest-init
  imagedigestexporter: imagedigestexporter
  gcsfetcher: gcs-fetcher
```
https://github.com/yyunk/jenkins-x-arm-support/blob/master/yaml/tekton-myvalues.yaml

`helm upgrade tekton tekton --install --values myvalues.yaml`

# Success criteria
is when both helm charts can be installed and the `Deployment` resources create pods which startup, start Running and don't fail / restart for 5 minutes. That means that the pods startup and don't fail basically.
`kubectl get pod -w`
