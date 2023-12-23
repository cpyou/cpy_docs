argocd对接LDAP

https://blog.csdn.net/zhuganlai168/article/details/131422269

# 编写ldap-patch-dex.yaml

```yaml
# vi ldap-patch-dex.yaml
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  dex.config: |-
    connectors:
    - type: ldap
      name: ..................
      id: ldap
      config:
        # Ldap server address
        host: 192.168.5.16:389
        insecureNoSSL: true
        insecureSkipVerify: true
        # Variable name stores ldap bindDN in argocd-secret
        bindDN: "$dex.ldap.bindDN"
        # Variable name stores ldap bind password in argocd-secret
        bindPW: "$dex.ldap.bindPW"
        usernamePrompt: .........
        # Ldap user serch attributes
        userSearch:
          baseDN: "ou=people,dc=xxx,dc=com"
          filter: "(objectClass=person)"
          username: uid
          idAttr: uid
          emailAttr: mail
          nameAttr: cn
        # Ldap group serch attributes
        groupSearch:
          baseDN: "ou=argocd,ou=group,dc=xxx,dc=com"
          filter: "(objectClass=groupOfUniqueNames)"
          userAttr: DN
          groupAttr: uniqueMember
          nameAttr: cn
  # 注意：这个是argocd的访问地址，必须配置，否则会导致不会跳转.
  url: https://192.168.80.180:30984

```

2、configMap

```shell
kubectl -n argocd patch configmaps argocd-cm --patch "$(cat ldap-patch-dex.yaml)"
```

3、修改configMap

```
kubectl edit cm argocd-cm -n argocd
```

3、secret

```sh
# bindDN是cn=admin,dc=xxx,dc=com
kubectl -n argocd patch secrets argocd-secret --patch "{\"data\":{\"dex.ldap.bindDN\":\"$(echo cn=admin,dc=deepcoin,dc=info | base64 -w 0)\"}}"

# 密码bindPW是123456
kubectl -n argocd patch secrets argocd-secret --patch "{\"data\":{\"dex.ldap.bindPW\":\"$(echo Deep1@2022 | base64 -w 0)\"}}"

```

- 4、查看日志

```shell
kubectl logs -f -n argocd argocd-dex-server-7689686bd9-4srkc
```

5、删除POD，以重启，让上面的ldap配置生效

```shell
kubectl -n argocd get pod
kubectl delete pod -n argocd argocd-server-7bf49f8b5-ksq6l
kubectl delete pod -n argocd argocd-dex-server-7689686bd9-xwxf8
kubectl -n argocd get pod -w
```

六、argocd的权限
参考其官方网址：https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/

1、直接编辑权限

```
kubectl edit configmaps -n argocd argocd-rbac-cm
```

创建一个devops角色，具备某一部分的权限。



2. 使用kustomization 配合csv生效权限

```
# vim policy.csv
p, role:test-readonly, applications, get, default/*, allow
p, role:test-readonly, projects, get, *, allow
p, role:test-readonly, accounts, get, *, allow
p, role:test-readonly, logs, get, default/*, allow
```



```shell
cat <<EOF >./kustomization.yaml
configMapGenerator:
- name: argocd-rbac-cm
  namespace: argocd
  literals:
  - policy.default=role:none
  - scopes='[groups, name, email]'
  files:
  - policy.csv
generatorOptions:
  disableNameSuffixHash: true
EOF

# 检查内容
kubectl kustomize ./

# 生效内容
kubectl apply -k .
kubectl -n argocd get configmap argocd-rbac-cm -o yaml
kubectl -n argocd edit configmap argocd-rbac-cm
argocd admin settings rbac can puyu.chen@deepcoin.info get applications 'cpy/*' --policy-file policy.csv
```



权限配置示例：

```
# vim 
p, role:org-admin, applications, *, */*, allow
p, role:org-admin, clusters, get, *, allow
p, role:org-admin, repositories, get, *, allow
p, role:org-admin, repositories, create, *, allow
p, role:org-admin, repositories, update, *, allow
p, role:org-admin, repositories, delete, *, allow
p, role:org-admin, projects, get, *, allow
p, role:org-admin, projects, create, *, allow
p, role:org-admin, projects, update, *, allow
p, role:org-admin, projects, delete, *, allow
p, role:org-admin, logs, get, *, allow
p, role:org-admin, exec, create, */*, allow


# 先定义角色及对应的权限项列表
# 角色leader-admin，和默认的角色admin一样
p, role:leader-admin, *, *, *, allow
# 配置应用同步权限
p, role:devops, applications, sync, *, allow
# 配置应用回滚权限
p, role:devops, applications, update, *, allow
# 配置查看应用日志权限
p, role:devops, logs, get, *, allow
# 配置用户可以进入容器终端进行操作的权限  
p, role:devops, exec, create, */*, allow

# 下面是对组进行赋予角色   
# ldap组argocd-admin赋予角色leader-admin
g, argocd-admin, role:leader-admin
# ldap组argocd-users赋予角色devops
g, argocd-users, role:devops
```

