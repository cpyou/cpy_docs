开始开发Operator

## 2.4 Kubebuilder 的安装配置

```shell
curl -L -o kubebuilder https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)
# 
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

## 2.5 Demo开始

### 2.5.1创建项目	

```shell
cd ~
mkdir MyOperatorProjects
cd MyOperatorProjects
mkdir application-operator
cd application-operator
# 创建第一个项目
kubebuilder init --domain=py.cn --repo=github.com/cpyou/application-operator --owner=cpy
# Writing kustomize manifests for you to edit...
# Writing scaffold for you to edit...
# Get controller runtime:
# $ go get sigs.k8s.io/controller-runtime@v0.14.4
# Update dependencies:
# $ go mod tidy
# Next: define a resource with:
# $ kubebuilder create api
```

使用git跟踪代码变更

```shell
git init
git add .
git branch -m main
git commit -m 'Init project'
```

### 2.5.2 添加API

```shell
kubebuilder create api --group apps --version v1 --kind Application
```

### 2.5.3 CRD实现

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

### 2.5.4 CRD 部署

```shell
make manifests
# /Users/cpy/MyOperatorProjects/application-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
git status
#未跟踪的文件:
#  （使用 "git add <文件>..." 以包含要提交的内容）
#	config/crd/bases/
#	config/rbac/role.yaml

# 部署CRD
make install
# test -s /Users/cpy/MyOperatorProjects/application-operator/bin/controller-gen && /Users/cpy/MyOperatorProjects/application-operator/bin/controller-gen --version | grep -q v0.11.3 || \
#        GOBIN=/Users/cpy/MyOperatorProjects/application-operator/bin go install sigs.k8s.io/controller-tools/cmd/controller-gen@v0.11.3
# /Users/cpy/MyOperatorProjects/application-operator/bin/controller-gen rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
# /Users/cpy/MyOperatorProjects/application-operator/bin/kustomize build config/crd | kubectl apply -f -
# customresourcedefinition.apiextensions.k8s.io/applications.apps.py.cn created

# 查看CRD
kubectl get crd
# NAME                      CREATED AT
# applications.apps.py.cn   2023-12-24T15:11:38Z
kubectl get application
# No resources found in default namespace.

```

### 2.5.5 CR部署

```yaml
# vim config/samples/apps_v1_application.yaml
apiVersion: apps.py.cn/v1
kind: Application
metadata:
  labels:
    app.kubernetes.io/name: application
    app.kubernetes.io/instance: application-sample
    app.kubernetes.io/part-of: application-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: application-operator
    app: nginx
  name: application-sample
  namespace: default
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80

```

```shell
# 创建application
kubectl apply -f config/samples/apps_v1_application.yaml
# 查看application
kubectl get application
```

### 2.5.6 Controller 实现

```go
# 修改文件 internal/controller/application_controller.go
import (
	"context"
	"fmt"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"time"

	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	appsv1 "github.com/cpyou/application-operator/api/v1"
)

func (r *ApplicationReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	l := log.FromContext(ctx)

	// get the Application
	app := &appsv1.Application{}
	if err := r.Get(ctx, req.NamespacedName, app); err != nil {
		if errors.IsNotFound(err) {
			l.Info("the application is not found")
			return ctrl.Result{}, nil
		}
		l.Error(err, "failed to get the Application")
		return ctrl.Result{RequeueAfter: 1 * time.Minute}, err
	}

	// create pods
	for i := 0; i < int(app.Spec.Replicas); i++ {
		pod := &corev1.Pod{
			ObjectMeta: metav1.ObjectMeta{
				Name:      fmt.Sprintf("%s-%d", app.Name, i),
				Namespace: app.Namespace,
				Labels:    app.Labels,
			},
			Spec: app.Spec.Template.Spec,
		}
		if err := r.Create(ctx, pod); err != nil {
			l.Error(err, "failed to create Pod")
			return ctrl.Result{RequeueAfter: 1 * time.Minute}, err
		}
		l.Info(fmt.Sprintf("the Pod (%s) has created", pod.Name))
	}
	l.Info("all pods has created")
	return ctrl.Result{}, nil
}
```

### 2.5.7 启动 Controller

```shell
# 启动Controller
make run

# 查看结果
kubectl get pod
```

### 2.5.8 部署 Controller

```shell
# 构建镜像
make docker-build IMG=application-operator:v0.0.1

# 推到kind环境
kind load docker-image application-operator:v0.0.1 --name dev

# 部署控制器
make deploy IMG=application-operator:v0.0.1

# 部署出现问题 Google或者查看书籍

# 查看operator程序
kubectl get pod -n application-operator-system
# NAME                                                       READY   STATUS    RESTARTS      AGE
# application-operator-controller-manager-7646564cb5-g8mhk   1/2     Running   4 (16s ago)   4m36s

```

### 2.5.9 资源清理

```shell
# 卸载 Controller
make undeploy

# 卸载 CRD
make uninstall

```

