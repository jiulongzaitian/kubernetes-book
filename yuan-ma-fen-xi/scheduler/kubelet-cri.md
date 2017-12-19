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

在api.proto 里描述了两个service，RuntimeService 和ImageService

那这是如何对应kubelet的代码呢 。。。。



## kubelet CRI

pkg/kubelet/kubelet.go  NewMainKubelet（） 方法 里有对grpc server 的初始化

``` golang
	// rktnetes cannot be run with CRI.
	
	if containerRuntime != kubetypes.RktContainerRuntime {
		
		klet.networkPlugin = nil
		// if left at nil, that means it is unneeded
		var legacyLogProvider kuberuntime.LegacyLogProvider

		switch containerRuntime {
		case kubetypes.DockerContainerRuntime:
			// Create and start the CRI shim running as a grpc server.

			streamingConfig := getStreamingConfig(kubeCfg, kubeDeps)
			ds, err := dockershim.NewDockerService(kubeDeps.DockerClientConfig, crOptions.PodSandboxImage, streamingConfig,
				&pluginSettings, runtimeCgroups, kubeCfg.CgroupDriver, crOptions.DockershimRootDirectory,
				crOptions.DockerDisableSharedPID)
			if err != nil {
				return nil, err
			}
			if err := ds.Start(); err != nil {
				return nil, err
			}
			// For now, the CRI shim redirects the streaming requests to the
			// kubelet, which handles the requests using DockerService..
			klet.criHandler = ds

			// The unix socket for kubelet <-> dockershim communication.
			glog.V(5).Infof("RemoteRuntimeEndpoint: %q, RemoteImageEndpoint: %q",
				remoteRuntimeEndpoint,
				remoteImageEndpoint)
			glog.V(2).Infof("Starting the GRPC server for the docker CRI shim.")
			// grpc server

			// 这一块的代码 需要了解 CRI 和docker 关系
			// CRT shim 是一个grpc 的server
			// kubelet 通过grpc 的协议跟它通信
			// 然后shim 再与runc 通信
			server := dockerremote.NewDockerServer(remoteRuntimeEndpoint, ds)
			if err := server.Start(); err != nil {
				return nil, err
			}

			// Create dockerLegacyService when the logging driver is not supported.
			supported, err := ds.IsCRISupportedLogDriver()
			if err != nil {
				return nil, err
			}
			if !supported {
				klet.dockerLegacyService = ds.NewDockerLegacyService()
				legacyLogProvider = dockershim.NewLegacyLogProvider(klet.dockerLegacyService)
			}
		case kubetypes.RemoteContainerRuntime:
			// No-op.
			break
		default:
			return nil, fmt.Errorf("unsupported CRI runtime: %q", containerRuntime)
		}
		runtimeService, imageService, err := getRuntimeAndImageServices(remoteRuntimeEndpoint, remoteImageEndpoint, kubeCfg.RuntimeRequestTimeout)
		if err != nil {
			return nil, err
		}

```

dockerremote.NewDockerServer  就是对Grpc server 的初始化，相信只要玩过PB-Grpc 的都能大体知道初始化的过程

之后会创建 runtimeService 和 imageService， 通过追踪 getRuntimeAndImageServices（） 代码我们可以看到这是grpc client 初始化的过程





# 未完待续。。。



