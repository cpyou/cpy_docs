开始开发Operator

### 2.4 Kubebuilder 的安装配置

```shell
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/
kubebuilder version
```

https://github.com/kubernetes-sigs/kubebuilder/releases

自己编译

```shell
git clone https://github.com/kubernetes-sigs/kubebuilder.git
cd kubebuilder
git checkout release-3.2.0
make build
make install
```

### 2.5 Demo开始

2.5.1创建项目	

```shell
cd ~
mkdir MyOperatorProjects
cd MyOperatorProjects
mkdir application-operator
cd application-operator
# 创建第一个项目
kubebuilder init --domain=puyu.cn --repo=github.com/cpyou/application-operator --owner=puyu.chen
```

使用git跟踪代码变更

```shell
git init
git add .
git branch -m main
git commit -m 'Init project'
```

2.5.2 添加API

```shell
kubebuilder create api --group apps --version v1 --kind Application
```

2.5.3 CRD实现

vim api/v1/application_types.go

```go
import (
    corev1 "k8s.io/api/core/v1"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// ApplicationSpec defines the desired state of Application
type ApplicationSpec struct {
    // INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
    // Important: Run "make" to regenerate code after modifying this file

    // Foo is an example field of Application. Edit application_types.go to remove/update
    Replicas int32                     `json:"replicas,omitempty"`
    Template corev1.PodTemplateSpec    `json:"template,omitempty"`
}
```

2.5.4 CRD 部署

```shell
make manifests
/Users/cpy/MyOperatorProjects/application-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

