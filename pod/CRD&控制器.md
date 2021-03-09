除了k8s 内部支持的一些api对象，在k8s 1.7之后，提供了api插件机制：CRD（Custom Resource Definition）

> 允许用户在k8s 中添加一个跟pod、node 类似的、新的 api 资源类型，即：自定义 api 资源

举个例子，添加一个 Network 的 api 资源类型

```yaml
// example-network.yaml  这是一个具体的自定义 api 资源实例 也叫CR
apiVersion: samplecrd.k8s.io/v1
kind: Network
metadata:
  name: example-network
spec:
  cidr: "192.168.0.0/16"
  gateway: "192.168.0.1"
```

```yaml
// network.yaml  这是一个 CRD 文件
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: networks.samplecrd.k8s.io
spec:
  group: samplecrd.k8s.io
  version: v1
  names:
    kind: Network
    plural: networks
  scope: Namespaced
```

通过代码定义 Network 里面具体的参数是什么

```bash
$ tree $GOPATH/src/github.com/<your-name>/k8s-controller-custom-resource
.
├── controller.go
├── crd
│   └── network.yaml
├── example
│   └── example-network.yaml
├── main.go
└── pkg
    └── apis
        └── samplecrd # api 组的名字
            ├── register.go
            └── v1 # 版本
                ├── doc.go
                ├── register.go
                └── types.go # 定义了network 对象的完整描述
```

```go
// pkg/apis/samplecrd 全局变量
package samplecrd

const (
 GroupName = "samplecrd.k8s.io"
 Version   = "v1"
)

//-----------------------------------------------------------------------------------------------------------

// pkg/apis/samplecrd/v1 文档源文件  +<tag_name>[=name] k8s进行代码生成要用嗯 annotation 风格注释
// +k8s:deepcopy-gen=package  请为整个 v1 包里的所有类型定义自动生成 DeepCopy 方法

// +groupName=samplecrd.k8s.io 则定义了这个包对应的 API 组的名字。
package v1

//-----------------------------------------------------------------------------------------------------------

// pkg/apis/samplecrd/v1  定义一个 Network 类型到底有哪些字段（比如，spec 字段里的内容）
package v1
...
// +genclient   请为下面这个 API 资源类型生成对应的 Client 代码
// +genclient:noStatus  这个 API 资源类型定义里，没有 Status 字段。否则，生成的 Client 就会自动带上 UpdateStatus 方法。
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Network describes a Network resource
type Network struct {
 // TypeMeta is the metadata for the resource, like kind and apiversion
 metav1.TypeMeta `json:",inline"`  // API 元数据
 // ObjectMeta contains the metadata for the particular object, including
 // things like...
 //  - name
 //  - namespace
 //  - self link
 //  - labels
 //  - ... etc ...
 metav1.ObjectMeta `json:"metadata,omitempty"`  // 对象元数据
 
 Spec networkspec `json:"spec"` // 自定义部分
}
// networkspec is the spec for a Network resource
type networkspec struct {
 Cidr    string `json:"cidr"`
 Gateway string `json:"gateway"`
}

// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// NetworkList is a list of Network resources
type NetworkList struct {
 metav1.TypeMeta `json:",inline"`
 metav1.ListMeta `json:"metadata"`
 
 Items []Network `json:"items"`
}

//-----------------------------------------------------------------------------------------------------------

// pkg/apis/samplecrd/v1/register.go
package v1
...
// addKnownTypes adds our types to the API scheme by registering
// Network and NetworkList
func addKnownTypes(scheme *runtime.Scheme) error {
 scheme.AddKnownTypes(
  SchemeGroupVersion,
  &Network{},
  &NetworkList{},
 )
 
 // register the type in the scheme
 metav1.AddToGroupVersion(scheme, SchemeGroupVersion)
 return nil
}
```

```bash
# 这个代码生成工具名叫k8s.io/code-generator，使用方法如下所示：

# 代码生成的工作目录，也就是我们的项目路径
$ ROOT_PACKAGE="github.com/resouer/k8s-controller-custom-resource"
# API Group
$ CUSTOM_RESOURCE_NAME="samplecrd"
# API Version
$ CUSTOM_RESOURCE_VERSION="v1"

# 安装k8s.io/code-generator
$ go get -u k8s.io/code-generator/...
$ cd $GOPATH/src/k8s.io/code-generator

# 执行代码自动生成，其中pkg/client是生成目标目录，pkg/apis是类型定义目录
$ ./generate-groups.sh all "$ROOT_PACKAGE/pkg/client" "$ROOT_PACKAGE/pkg/apis" "$CUSTOM_RESOURCE_NAME:$CUSTOM_RESOURCE_VERSION"
```

```bash
$ kubectl apply -f crd/network.yaml  # CRD 
customresourcedefinition.apiextensions.k8s.io/networks.samplecrd.k8s.io created
$ kubectl get crd
NAME                        CREATED AT
networks.samplecrd.k8s.io   2018-09-15T10:57:12Z


$ kubectl apply -f example/example-network.yaml # CR
network.samplecrd.k8s.io/example-network created
$ kubectl get network
NAME              AGE
example-network   8s


$ kubectl describe network example-network
Name:         example-network
Namespace:    default
Labels:       <none>
...API Version:  samplecrd.k8s.io/v1
Kind:         Network
Metadata:
  ...
  Generation:          1
  Resource Version:    468239
  ...
Spec:
  Cidr:     192.168.0.0/16
  Gateway:  192.168.0.1
```



自定义控制器以及 CRD 具体代码查看 [github](https://github.com/eliasyaoyc/k8s-controller-custom-resource?organization=eliasyaoyc&organization=eliasyaoyc)