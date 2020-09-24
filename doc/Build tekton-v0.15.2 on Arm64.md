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

  For arm64, follow this link to download Go 1.13.8 :

  https://golang.google.cn/dl/go1.13.8.linux-arm64.tar.gz

  Download the archive and extract it.

* install Pre-commit

  I used pip. Other installation methods exist as well.

  `$ pip install pre-commit`

* install Dep

  follow this link to install:

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
Golint is a linter for Go source code, golint prints out style mistakes .i did not download  Golint ,  so  i  delete __lint__ in the Makefile.

Because some binary can be built not only by main.go,i change the main.go in Makefile to *.go . So it can build binary  by all  Source code Include in folder.

 Here is an example of what I  changed in the Makefile You can refer to the following link: https://github.com/yyunk/jenkins-x-arm-support/blob/master/doc/pipeline/Makefile#L24

After changing the Makefile, you can enter the  cmd folder,then you can see some folder.

__controller  ,creds-init  ,entrypoint  ,git-init  ,imagedigestexporter , kubeconfigwriter  ,nop , pullrequest-init , webhook.__

Enter each  folder ,and Enter command `$make -f ../../Makefile all` ,the binary  can be built.

Or you can just use shell script 

```for i in `realpath cmd/*`; do cd $i; make -f ../../Makefile all; done```

The binary  file is in  cmd/*/.bin/github.com/tektoncd/pipeline.

When build  gcs-fetcher,you should enter another folder

```shell
cd pipeline/vendor/github.com/GoogleCloudPlatform/cloud-builders/gcs-fetcher/cmd/gcs-fetcher
make -f pipeline/Makefile all
```

Now ,all binary files are  built.

There are 

__controller,  creds-init , entrypoint , gcs-fetcher , git-init,  imagedigestexporter,  kubeconfigwriter , nop , pullrequest-init,  webhook__

next , building container images ,then deploying tekton  with helm.

# build images 
the controller Dockerfile
using ubuntu18.04 as base image

```
FROM ubuntu:18.04
RUN  apt-get update &&  apt install openssh-client git -y
RUN mkdir /ko-app
COPY ./pipeline  /ko-app/controller
ENTRYPOINT ["/ko-app/controller"]
````

Other dockerfile is similar.

Here is an example of what I used,You can refer to the following link: https://github.com/yyunk/jenkins-x-arm-support/tree/master/doc/pipeline


# deploy on cluster

```
git clone https://github.com/jenkins-x-charts/tekton 
cd tekton/tekton
```

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
Here is an example of what I used in my arm64 server.You can refer to the following link:

https://github.com/yyunk/jenkins-x-arm-support/blob/master/yaml/tekton-myvalues.yaml

`helm upgrade tekton tekton --install --values myvalues.yaml`

# Success criteria
When both helm charts can be installed and the `Deployment` resources create pods which startup, start Running and don't fail / restart for 5 minutes. That means that the pods startup and don't fail basically.
You can use command to see it 

`kubectl get pod -w` 

```
tekton-pipelines-controller-68487d8d5c-f6q5g                 1/1     Running            0          1h
tekton-pipelines-webhook-6cff968fb9-6b6rm                    1/1     Running            0         1h

```

