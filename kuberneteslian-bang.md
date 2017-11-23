### 为什么需要联邦
```
                                作者：李昂 邮箱：liang@haihangyun.com
```
---

联邦可以轻松管理多个群集。它通过提供2个主要构件来实现：

* 跨群集同步资源：联邦可以使多个群集中的资源保持同步。例如，可以确保在多个群集中存在相同的deployment资源。
* 跨群集发现：联邦提供了自动配置DNS服务器和对所有群集后端的负载均衡器。例如，您可以确保可以使用全局VIP或DNS记录来访问多个集群的后端。

联邦的其他一些使用方式如下：

* 高可用性：通过把负载放在各集群并自动配置DNS服务器和负载平衡器，联邦会将集群故障的影响降至最低。
* 避免云服务商锁定：通过更轻松地跨群集迁移应用程序，联邦会防止用户被绑架在同一个云服务商。

当你有多个集群时，联邦才会发生作用。而您可能需要多个集群的一些原因是：

* 低延迟：拥有多个区域的集群可以让用户访问离他们最近的集群从而最小化延迟。
* 故障隔离：有多个小型集群而不是单个大型集群进行故障隔离可能会更好（例如：云提供商的不同可用区域中有多个集群）。
* 可伸缩性：对于单个kubernetes集群有可扩展性限制（大多数用户应该不需要这样）。
* 混合云：您可以在不同的云提供商或本地数据中心拥有多个群集。

#### 注意事项

虽然联邦有很多有吸引力的用例，但也有一些注意事项：

* 增加网络带宽和成本：联邦在控制平面监视所有集群，以确保当前状态符合预期。如果集群在不同的云提供商或云提供商的不同地区运行，这可能会导致显著的网络成本。
* 减少跨集群隔离：联合控制平面中的错误可能会影响所有集群。通过将联邦控制平面中的逻辑保持在最低限度，可以缓解这一问题。只要有可能，它大部分都会在控制平面上代表kubernetes集群。联邦在设计和实施也避免在安全方面发生错误，避免了多集群停机。
* 成熟度：联邦这个feature目前相对较新，不太成熟。并不是所有的资源都可用，许多仍然是alpha。[issue38893](https://github.com/kubernetes/kubernetes/issues/38893)列举了已知问题。

#### 混合云能力

Kubernetes集群联盟可以包括运行在不同云提供商（例如Google Cloud，AWS）和本地（例如OpenStack）中的集群。只需在相应的云提供商或本地创建所需的所有群集，然后在联邦API服务器上注册每个群集的API Endpoint和API服务的认证。之后，您的API资源可以跨越不同的集群和云提供商。

### 设置联邦
---
为了能够联合多个集群，首先需要建立一个联邦控制平面。按照[联邦设置手册](https://kubernetes.io/docs/tutorials/federation/set-up-cluster-federation-kubefed/)设置控制平面。

### API 资源
---
一旦设置了控制平面，就可以开始创建联邦API资源。以下指南详细解释了一些资源：

* [Cluster](https://kubernetes.io/docs/tasks/administer-federation/cluster/)
* [ConfigMap](https://kubernetes.io/docs/tasks/administer-federation/configmap/)
* [DaemonSets](https://kubernetes.io/docs/tasks/administer-federation/daemonset/)
* [Deployment](https://kubernetes.io/docs/tasks/administer-federation/deployment/)
* [Events](https://kubernetes.io/docs/tasks/administer-federation/events/)
* [Hpa](https://kubernetes.io/docs/tasks/administer-federation/hpa/)
* [Ingress](https://kubernetes.io/docs/tasks/administer-federation/ingress/)
* [Jobs](https://kubernetes.io/docs/tasks/administer-federation/job/)
* [Namespaces](https://kubernetes.io/docs/tasks/administer-federation/namespaces/)
* [ReplicaSets](https://kubernetes.io/docs/tasks/administer-federation/replicaset/)
* [Secrets](https://kubernetes.io/docs/tasks/administer-federation/secret/)
* [Services](https://kubernetes.io/docs/concepts/cluster-administration/federation-service-discovery/)

### 级联删除
---
Kubernetes 1.6版支持级联删除联邦资源。通过级联删除，当您从联合控制平面中删除资源时，还可以删除所有基础集群中的相应资源。

在使用REST API时，级联删除在默认情况下不会启用。要启用它，请在使用REST API从联合控制层面删除资源时，设置`DeleteOptions.orphanDependents = false`选项。使用kubectl delete可以在默认情况下启用级联删除。你可以通过运行`kubectl delete --cascade = false`来禁用它。

注意：Kubernetes1.5包括对联邦资源子集的级联删除支持。

### 单个集群的范围
---
在诸如Google Compute Engine或Amazon Web Services之类的IaaS提供商中，虚拟机存在于区域或可用区中。我们建议Kubernetes集群中的所有虚拟机应位于相同的可用区，因为：

* 与具有单个全局Kubernetes集群相比，单点故障更少。
* 与跨越可用区域的集群相比，更容易推断单区域群集的可用性属性。
* 当Kubernetes开发人员正在设计系统（例如，对延迟，带宽或相关故障进行假设）时，他们会假设所有机器都在单个数据中心中，或以其他类似方式连接。

每个可用区域有多个群集是可以的，但总体而言，我们认为最好是少一点的理由是：

* 在某些情况下，在一个群集中有更多的节点（更少的资源碎片），可以改进Pod的二进制打包。
* 降低了运营开销（尽管随着操作工具和流程的成熟，优势已经减少）。
* 降低每个集群固定资源成本的成本，例如， apiserver虚拟机（但对于大中型集群整体集群成本的比例很小）。

而多个集群也会有他的道理：

* 严格的安全策略要求将一类工作与另一类工作隔离。
* 可以从测试群集用金丝雀部署到新的集权。

### 选择正确数量的群集

Kubernetes集群数量的选择可能是一个相对静态的选择，只是偶尔重新考虑。相比之下，一个集群中的节点数量和一个service中的pod数量会随着负载和增长而频繁变化。

要选择群集的数量，首先要为在Kubernetes上运行的服务确定哪些区域需要访问，以便为所有最终用户提供足够低的延迟（如果您使用内容分发网络CDN，则CDN托管内容不需要考虑）。法律问题也可能影响到这一点。例如，拥有全球客户群的公司可能会决定在美国，欧盟，美联社和南亚地区拥有集群。我们假设区域的数量为R。

其次，决定同时可以容忍多少个集群不可用。假设最多同时不可用集群数量为U。如果您不确定，那么1是一个不错的选择。

如果在发生集群故障时允许负载均衡器将流量引入到任何区域（用户访问服务可能会发生失败），则至少需要R或U+1群集中的较大者。如果不是（例如，要确保在发生群集故障时所有用户的延迟较低），则需要有R*(U+1)集群（每区域中有U+1集群）。无论如何，请尝试将各个群集放在不同的区域中。

最后，如果您的任何集群需要比Kubernetes集群的最大建议节点数量多，则可能需要更多的集群。Kubernetes v1.3支持最多1000个节点的集群。Kubernetes v1.8支持多达5000个节点的集群。
