# sample-controller

本库实现了一个简单的controller监控以CustomResourceDefinition(CRD)方式定义的新资源类型Foo.

**提示**: go-get或者包引用请使用`k8s.io/sample-controller`地址

本例演示了如下的基本操作:

* 如何使用CustomResourceDefinition注册一个新的用户资源类型
* 如何create/get/list新资源类型实例
* 如何设置一个新的controller响应新资源的create/update/delete事件

本例使用[k8s.io/code-generator](https://github.com/kubernetes/code-generator)来生成基本的client,informer,lister和deep-copy函数. 你可以通过`./hack/update-codegen.sh`来执行生成.

`update-codegen`将会自动生成如下文件和目录:

* `pkg/apis/samplecontroller/v1alpha1/zz_generated.deepcopy.go`
* `pkg/generated/`

这些文件不应该手动修改,如果你创建你自己的controller基于本例,你不应该复制这些文件而是执行`update-codegen`来重新生成.

## 细节

Sample controller 在很多地方使用了[client-go](https://github.com/kubernetes/client-go/tree/master/tools/cache).更多的细节与机制在[这里](docs/controller-client-go.md)有说明.

## 目标

说明如何构建一个kube-like(与kubernetes内部controller实现相同)的controller.

## 运行

**环境要求**: 因为sample-controller使用`apps/v1`版本deployment所以对Kubernetes集群的版本要求为大于1.9.

```sh
# 假设你已经准备好了kubeconfig,如果在集群内部则可以跳过
go get k8s.io/sample-controller
cd $GOPATH/src/k8s.io/sample-controller
go build -o sample-controller .
./sample-controller -kubeconfig=$HOME/.kube/config

# 创建 CustomResourceDefinition
kubectl create -f artifacts/examples/crd.yaml

# 创建一个Foo资源
kubectl create -f artifacts/examples/example-foo.yaml

# 检查关联的deployment被创建
kubectl get deployments
```

## 使用场景

可以使用CustomResourceDefinition来实现用户自定义的资源类型,它们的行为表现将和kubernetes中的其他资源类型一致,并可以使用`kubectl apply`命令管理.

如下列一些场景:

* 创建/管理外部的数据存储/数据库(例如:CloudSQL/RDS实例)
* 基于Kubernetes的原语进行抽象扩展(例如:基于Service和ReplicationController定义一个etcd集群资源类型)

## 类型定义

每一个用户资源实例都包含一个Spec,它可以通过`struct{}`来定义并被用来进行格式校验.
实际中,Spec可以是随意的key-value数据来说明定义资源的配置与行为.

例如,你实现一个Database的自定义资源,你可能声明如下的DatabaseSpec:

``` go
type DatabaseSpec struct {
	Databases []string `json:"databases"`
	Users     []User   `json:"users"`
	Version   string   `json:"version"`
}

type User struct {
	Name     string `json:"name"`
	Password string `json:"password"`
}
```

## 校验

可以使用[`CustomResourceValidation`](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#validation)功能
使自定义资源支持校验.

在v1.9版本中该功能处于beta阶段并默认开启.

### 例子

在[`crd-validation.yaml`](./artifacts/examples/crd-validation.yaml)中指明了如下校验规则:

`spec.replicas`必须为整型并且最小值为1最大值为10.

上述使用`crd-validation.yaml`创建CRD:

```sh
# 创建CustomResourceDefinition支持校验
kubectl create -f artifacts/examples/crd-validation.yaml
```

## 子资源

在v1.11中默认开启了自定义资源支持`/status`和`/scale`子资源[beta功能](https://kubernetes.io/docs/tasks/access-kubernetes-api/custom-resources/custom-resource-definitions/#subresources).

在v1.10中该功能处于[alpha](https://v1-10.docs.kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#subresources),可以通过为[kube-apiserver](https://kubernetes.io/docs/admin/kube-apiserver)设置`CustomResourceSubresources`参数来开启.

```sh
--feature-gates=CustomResourceSubresources=true
```

### 例子

通过[`crd-status-subresource.yaml`](./artifacts/examples/crd-status-subresource.yaml)创建的CRD支持`/status`子资源.这意味着controller可以使用[`UpdateStatus`](./controller.go#L330)更新资源仅当status符合CRD定义时.

更详细的说明可以参考这里[Kubernetes API conventions](https://git.k8s.io/community/contributors/devel/api-conventions.md#spec-and-status).

上述使用`crd-status-subresource.yaml`创建CRD:

```sh
# 创建CustomResourceDefinition支持status子资源
kubectl create -f artifacts/examples/crd-status-subresource.yaml
```

## 清理

你可以清理掉创建的CustomResourceDefinition通过执行如下命令:

    kubectl delete crd foos.samplecontroller.k8s.io

## 兼容

本库将更新跟随`k8s.io/apimachinery`与`k8s.io/client-go`.
