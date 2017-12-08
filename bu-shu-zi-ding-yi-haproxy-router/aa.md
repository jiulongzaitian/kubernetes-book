# 部署自定义HAProxy Router

```
苏晓林 xiaol.su@haihangyun.com                    openshift version:3.6
```

## 概览

默认的HAProxy Router已经能够满足绝大部分的用户需求。但是默认的HAProxy Router并不支持所有的HAProxy功能，一些用户可能需要用到这些功能。

有的用户可能需要用到一些application back-ends的新特性，或者修改一些当前操作。router plug-in能通过定制的方式满足所有这些需求。

Router pod使用[golang template](http://golang.org/pkg/text/template/)来创建HAProxy的配置文件。当处理template时，Router需要访问OpenShift Origin的信息，包括Router的部署配置，允许的routes集合和其它一些帮助功能。

当Router pod启动和每次reload时，都会创建一个HAProxy配置文件，然后启动HAProxy。[HAProxy configuration manual](https://cbonte.github.io/haproxy-dconv/configuration-1.5.html) 给出HAProxy的所有特性，以及如何构建一个有效的配置文件。

可以通过将[**configMap**](https://docs.openshift.org/latest/install_config/router/customized_haproxy_router.html#using-configmap-replace-template)添加到template来部署Router pod。通过这种方法部署时，需要修改Router的部署配置，将**configMap**以volume的方式挂载到Router pod中。并且将环境变量`TEMPLATE_FILE`配置为template文件在Router pod中的全路径。

或者，你也可以构建一个自定义的Router镜像，通过它来部署你的Routers。构建Routers的镜像可以不同。通过修改_**haproxy-template.config**_文件，然后重新构建Router的镜像。新的镜像会被上传到集群的镜像仓库，并且Router的部署配置中，**image:**字段更新为新的镜像名称。当集群有更改时，需要重新构建镜像并上传。

无论哪种方式，router pod的部署都是从template文件开始。

## 获得Router的template

HAProxy的template非常巨大且复杂。通常通过修改已有template的方式会更加方便和简单。可以通过以下命令在一个运行router的master节点上获得一个新的_**haproxy-config.template**_。

```
# oc get po
NAME                       READY     STATUS    RESTARTS   AGE
router-2-40fc3             1/1       Running   0          11d
# oc rsh router-2-40fc3 cat haproxy-config.template > haproxy-config.template
# oc rsh router-2-40fc3 cat haproxy.config > haproxy.config
```

或者，你可以登录到运行Router的节点，执行以下命令

```
# docker run --rm --interactive=true --tty --entrypoint=cat \
    openshift/origin-haproxy-router haproxy-config.template
```

将执行结果保存，作为基础配置文件，然后修改得到你的自定义配置，并最终生成配置文件_**haproxy.config**_。

## 修改Router的template

###  Background {#router-template-background}

HAProxy的template是以[golang template](http://golang.org/pkg/text/template/)为基础，能够解析部署时配置的环境变量，下面介绍的所有配置信息以及函数。

HAProxy的配置文件通过template生成，在生成过程中，所有`{{" something "}}`格式的内容将处理解析后写入到配置文件中，其它格式的内容将直接拷贝。

### 模版提供的功能

**define**关键字定义了template处理后生成的配置文件位置

```
{{define "/var/lib/haproxy/conf/haproxy.config"}}pipeline{{end}}
```

表1，template处理函数

| 函数 |  |
| :--- | :--- |
| **`endpointsForAlias(alias ServiceAliasConfig, svc ServiceUnit) []Endpoint`** | 返回可用的endpoints列表. |
| **`env(variable, default string) string`** | 尝试获得pod的环境变量，如果获得失败或为空，返回默认值default. |
| **`matchPattern(pattern, s string) bool`** | 正则匹配，第一个参数为正则表达式，第二个为要匹配的字符串，返回结果为布尔类型. |
| **`isInteger(s string) bool`** | 判断参数变量是否为整型. |
| **`matchValues(s string, allowedValues …​string) bool`** | 匹配字符串s，是否在字符串列表**`allowedValues`**中. |
| **`genSubdomainWildcardRegexp(hostname, path string, exactPath bool) string`** | Generates a regular expression matching the subdomain for hosts \(and paths\) with a wildcard policy. |
| **`generateRouteRegexp(hostname, path string, wildcard bool) string`** | Generates a regular expression matching the route hosts \(and paths\). The first argument is the host name, the second is the path, and the third is a wildcard Boolean. |
| **`genCertificateHostName(hostname string, wildcard bool) string`** | Generates host name to use for serving/matching certificates. First argument is the host name and the second is the wildcard Boolean. |

### Router提供的参数 {#router-info-for-templates}

本节介绍Router可用于OpenShift Origin的参数，这些参数是HAProxy router plug-in提供的，通过`.Fieldname`访问。

表2，Router配置参数

Field

|  | Type | Description |
| :--- | :--- | :--- |
| **`WorkingDir`** | string | The directory that files will be written to, defaults to_**/var/lib/containers/router**_ |
| **`State`** | ``map[string](ServiceAliasConfig)``` | The routes. |
| **`ServiceUnits`** | `map[string]ServiceUnit` | The service lookup. |
| **`DefaultCertificate`** | string | Full path name to the default certificate in pem format. |
| **`PeerEndpoints`** | ```[]Endpoint`` | Peers. |
| **`StatsUser`** | string | User name to expose stats with \(if the template supports it\). |
| **`StatsPassword`** | string | Password to expose stats with \(if the template supports it\). |
| **`StatsPort`** | int | Port to expose stats with \(if the template supports it\). |
| **`BindPorts`** | bool | Whether the router should bind the default ports. |

|  |  |  |
| :--- | :--- | :--- |
| Field | Type | Description |
| **`Name`** | string | The user-specified name of the route. |
| **`Namespace`** | string | The namespace of the route. |
| **`Host`** | string | The host name. For example,`www.example.com`. |
| **`Path`** | string | Optional path. For example,`www.example.com/myservice`where`myservice`is the path. |
| **`TLSTermination`** | `routeapi.TLSTerminationType` | The termination policy for this back-end; drives the mapping files and router configuration. |
| **`Certificates`** | `map[string]Certificate` | Certificates used for securing this back-end. Keyed by the certificate ID. |
| **`Status`** | `ServiceAliasConfigStatus` | Indicates the status of configuration that needs to be persisted. |
| **`PreferPort`** | string | Indicates the port the user wants to expose. If empty, a port will be selected for the service. |
| **`InsecureEdgeTerminationPolicy`** | `routeapi.InsecureEdgeTerminationPolicyType` | Indicates desired behavior for insecure connections to an edge-terminated route:`none`\(or`disable`\),`allow`, or`redirect`. |
| **`RoutingKeyName`** | string | Hash of the route + namespace name used to obscure the cookie ID. |
| **`IsWildcard`** | bool | Indicates this service unit needing wildcard support. |
| **`Annotations`** | `map[string]string` | Annotations attached to this route. |
| **`ServiceUnitNames`** | `map[string]int32` | Collection of services that support this route, keyed by service name and valued on the weight attached to it with respect to other entries in the map. |
| **`ActiveServiceUnits`** | int | Count of the`ServiceUnitNames`with a non-zero weight. |

The`ServiceAliasConfig`is a route for a service. Uniquely identified by host + path. The default template iterates over routes using`{{range $cfgIdx, $cfg := .State }}`. Within such a`{{range}}`block, the template can refer to any field of the current`ServiceAliasConfig`using`$cfg.Field`.

|  |  |  |
| :--- | :--- | :--- |
| Field | Type | Description |
| **`Name`** | string | Name corresponds to a service name + namespace. Uniquely identifies the`ServiceUnit`. |
| **`EndpointTable`** | `[]Endpoint` | Endpoints that back the service. This translates into a final back-end implementation for routers. |

`ServiceUnit`is an encapsulation of a service, the endpoints that back that service, and the routes that point to the service. This is the data that drives the creation of the router configuration files

|  |  |
| :--- | :--- |
| Field | Type |
| **`ID`** | string |
| **`IP`** | string |
| **`Port`** | string |
| **`TargetName`** | string |
| **`PortName`** | string |
| **`IdHash`** | string |
| **`NoHealthCheck`** | bool |

`Endpoint`is an internal representation of a Kubernetes endpoint.

|  |  |  |
| :--- | :--- | :--- |
| Field | Type | Description |
| **`Certificate`** | string | Represents a public/private key pair. It is identified by an ID, which will become the file name. A CA certificate will not have a`PrivateKey`set. |
| **`ServiceAliasConfigStatus`** | string | Indicates that the necessary files for this configuration have been persisted to disk. Valid values: "saved", "". |

|  |  |  |
| :--- | :--- | :--- |
| Field | Type | Description |
| ID | string |  |
| Contents | string | The certificate. |
| PrivateKey | string | The private key. |

|  |  |  |
| :--- | :--- | :--- |
| Field | Type | Description |
| **`TLSTerminationType`** | string | Dictates where the secure communication will stop. |
| **`InsecureEdgeTerminationPolicyType`** | string | Indicates the desired behavior for insecure connections to a route. While each router may make its own decisions on which ports to expose, this is normally port 80. |

`TLSTerminationType`and`InsecureEdgeTerminationPolicyType`dictate where the secure communication will stop.

|  |  |  |
| :--- | :--- | :--- |
| Constant | Value | Meaning |
| `TLSTerminationEdge` | `edge` | Terminate encryption at the edge router. |
| `TLSTerminationPassthrough` | `passthrough` | Terminate encryption at the destination, the destination is responsible for decrypting traffic. |
| `TLSTerminationReencrypt` | `reencrypt` | Terminate encryption at the edge router and re-encrypt it with a new certificate supplied by the destination. |

|  |  |
| :--- | :--- |
| Type | Meaning |
| **`Allow`** | Traffic is sent to the server on the insecure port \(default\). |
| **`Disable`** | No traffic is allowed on the insecure port. |
| **`Redirect`** | Clients are redirected to the secure port. |

None \(`""`\) is the same as`Disable`.  


