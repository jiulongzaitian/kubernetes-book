# kubelet CRI

```
作者 张杰  j.zhang8@haihangyun.com
```

每种容器运行时各有所长，许多用户都希望Kubernetes支持更多的运行时,Kubelet与容器运行时通信（或者是CRI插件填充了容器运行时）时，Kubelet就像是客户端，而CRI插件就像对应的服务器。它们之间可以通过Unix 套接字或者gRPC框架进行通信。

![](/assets/import.png)

protocol buffers API包含了两个gRPC服务：ImageService和RuntimeService。ImageService提供了从镜像仓库拉取、查看、和移除镜像的RPC。RuntimeSerivce包含了Pods和容器生命周期管理的RPC，以及跟容器交互的调用\(exec/attach/port-forward\)。一个单块的容器运行时能够管理镜像和容器（例如：Docker和Rkt），并且通过同一个套接字同时提供这两种服务。这个套接字可以在Kubelet里通过标识–container-runtime-endpoint和–image-service-endpoint进行设置。

## 源码分析

kubelet 的CRI 是用proto buffer  和 GRPC 实现的

目录:pkg/kubelet/apis/cri/v1alpha1/runtime

```
api.pb.go  api.proto  BUILD  constants.go
```

其中有一个api.proto 的描述文件，用来生成api.pb.go 代码，具体可以查看[golang grpc 初体验](http://www.jianshu.com/p/774b38306c30)，通过代码追踪，可以知道，转换成pb.go 源码文件的动作在: hack/update-generated-runtime-dockerized.sh

```shell
set -o errexit
set -o nounset
set -o pipefail

KUBE_ROOT=$(dirname "${BASH_SOURCE}")/..
KUBE_REMOTE_RUNTIME_ROOT="${KUBE_ROOT}/pkg/kubelet/apis/cri/v1alpha1/runtime/"
source "${KUBE_ROOT}/hack/lib/init.sh"

kube::golang::setup_env

BINS=(
    vendor/k8s.io/code-generator/cmd/go-to-protobuf/protoc-gen-gogo
)
make -C "${KUBE_ROOT}" WHAT="${BINS[*]}"

if [[ -z "$(which protoc)" || "$(protoc --version)" != "libprotoc 3."* ]]; then
  echo "Generating protobuf requires protoc 3.0.0-beta1 or newer. Please download and"
  echo "install the platform appropriate Protobuf package for your OS: "
  echo
  echo "  https://github.com/google/protobuf/releases"
  echo
  echo "WARNING: Protobuf changes are not being validated"
  exit 1
fi

function cleanup {
    rm -f ${KUBE_REMOTE_RUNTIME_ROOT}/api.pb.go.bak
}

trap cleanup EXIT

gogopath=$(dirname $(kube::util::find-binary "protoc-gen-gogo"))

PATH="${gogopath}:${PATH}" \
  protoc \
  --proto_path="${KUBE_REMOTE_RUNTIME_ROOT}" \
  --proto_path="${KUBE_ROOT}/vendor" \
  --gogo_out=plugins=grpc:${KUBE_REMOTE_RUNTIME_ROOT} ${KUBE_REMOTE_RUNTIME_ROOT}/api.proto

# Update boilerplate for the generated file.
echo "$(cat hack/boilerplate/boilerplate.go.txt ${KUBE_REMOTE_RUNTIME_ROOT}/api.pb.go)" > ${KUBE_REMOTE_RUNTIME_ROOT}/api.pb.go
sed -i".bak" "s/Copyright YEAR/Copyright $(date '+%Y')/g" ${KUBE_REMOTE_RUNTIME_ROOT}/api.pb.go

# Run gofmt to clean up the generated code.
kube::golang::verify_go_version
gofmt -l -s -w ${KUBE_REMOTE_RUNTIME_ROOT}/api.pb.go
```

而 hack/update-generated-runtime-dockerized.sh 是被 hack/update-generated-runtime.sh调用

如果你修改了相关的proto 描述文件，最终需要调用一下这个脚本:hack/update-generated-runtime.sh

# 未完待续。。。



