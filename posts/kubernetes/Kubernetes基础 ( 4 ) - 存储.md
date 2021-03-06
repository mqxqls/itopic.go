```
{
    "url": "k8s-storage",
    "time": "2020/09/25 12:08",
    "tag": "Kubernetes,容器化",
    "toc": "yes"
}
```

# 一、概述

前一篇中介绍了`Pod`以及`Volumes`，这里接着对`Pod`中依赖的存储资源做介绍。

# 二、ConfigMap

`ConfigMap`提供了一种配置管理方式，将镜像与配置文件解耦，可以通过环境变量、`Volumes`等实现将配置信息注入到容器内部，实现配置独立管理。

## 2.1 资源清单

```
$ kubectl explain cm
KIND:     ConfigMap
VERSION:  v1

DESCRIPTION:
     ConfigMap holds configuration data for pods to consume.

FIELDS:
   apiVersion	<string>
   binaryData	<map[string]string>
     BinaryData contains the binary data. Each key must consist of alphanumeric
     characters, '-', '_' or '.'. BinaryData can contain byte sequences that are
     not in the UTF-8 range. The keys stored in BinaryData must not overlap with
     the ones in the Data field, this is enforced during validation process.
     Using this field will require 1.10+ apiserver and kubelet.

   data	<map[string]string>
     Data contains the configuration data. Each key must consist of alphanumeric
     characters, '-', '_' or '.'. Values with non-UTF-8 byte sequences must use
     the BinaryData field. The keys stored in Data must not overlap with the
     keys in the BinaryData field, this is enforced during validation process.

   kind	<string>
   metadata	<Object>
```

相比前面的资源对象少了`spec`字段，多了`data`和`binaryData`字段，用来存储自定义的配置，类型都是`map[string]string`。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  app_id: "123456"
  log_level: production

```

貌似数字需要打上引号，配置会后通过`kubectl create -f cm.yaml`就可以创建了，然后通过`kubectl describe cm app-config` 可以看到信息。

## 2.2 创建ConfigMap

除了通过资源清单方式创建外，还有一些其他方式：

### 2.2.1 通过文件或目录创建

```
$ ls -lh
total 16
-rw-r--r--@ 1 peng  staff    40B  9 10 14:24 config.ini
-rw-r--r--@ 1 peng  staff    62B  9 10 14:23 database.py

$ cat config.ini
[runtime]
env  = online
host = k8s.local

$ cat database.py
MYSQL_HOST = '127.0.0.1'
MYSQL_USER = 'root'
MYSQL_PORT = 3306

$ kubectl create configmap blog-config --from-file=./
$ kubectl create configmap database --from-file=./database.py

```

`--from-file`指定目录，目录下的所有文件都会在ConfigMap里创建键值对，键名就是文件名，键值就是文件内容。

```
$ kubectl get cm
NAME          DATA   AGE
app-config    2      19m
blog-config   2      5m
database      1      2m18s

$ kubectl describe cm blog-config
Name:         blog-config
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
config.ini:
----
[runtime]
env  = online
host = k8s.local
database.py:
----
MYSQL_HOST = '127.0.0.1'
MYSQL_USER = 'root'
MYSQL_PORT = 3306
Events:  <none>
```

### 2.2.2 通过字面值创建

```
kubectl create configmap network-config --from-literal=GATEWAY=192.168.0.1 --from-literal=DNS1=192.168.0.1
configmap/network-config created
```

## 2.3 使用ConfigMap

### 2.3.1 环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:1.32.0
    command: ["sh", "-c", "sleep 1000"]
    env:
    - name: APPID
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app_id
    - name: DATABASE
      valueFrom:
        configMapKeyRef:
          name: database
          key: database.py
    envFrom:
    - configMapRef:
        name: network-config

```

进容器就可以通过环境变量获取到了。

```
$ kubectl exec -it cm-demo /bin/sh
/ # echo $APPID
123456
/ # echo $DATABASE
MYSQL_HOST = '127.0.0.1' MYSQL_USER = 'root' MYSQL_PORT = 3306
/ # echo $GATEWAY
192.168.0.1
```

### 2.3.2 命令行参数

```
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:1.32.0
    command: ["sh", "-c", "echo $DATABASE"]
    env:
    - name: APPID
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: app_id
    - name: DATABASE
      valueFrom:
        configMapKeyRef:
          name: database
          key: database.py
    envFrom:
    - configMapRef:
        name: network-config

```

`env`的配置和上面一样，可以以参数形式带在启动命名后面。

```
$ kubectl logs cm-demo
MYSQL_HOST = '127.0.0.1' MYSQL_USER = 'root' MYSQL_PORT = 3306
```

### 2.3.3 Volumes挂载

```
apiVersion: v1
kind: Pod
metadata:
  name: cm-demo
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:1.32.0
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: pod-blog-config
      mountPath: /data/
  volumes:
  - name: pod-blog-config
    configMap:
      name: blog-config
```

创建容器之后进入，配置以文件的方式挂载在`/data/`，文件名就是资源清单里的`data.key`，内容就是`data.value`。

```
$ kubectl exec -it cm-demo /bin/sh
/ # ls /data/
config.ini   database.py

/ # cat /data/config.ini
[runtime]
env  = online
host = k8s.local

/ # cat /data/database.py
MYSQL_HOST = '127.0.0.1'
MYSQL_USER = 'root'
MYSQL_PORT = 3306
```

## 2.4 热更新

```
$ kubectl edit cm blog-config
```

延用`Volume`的示例，修改`ConfigMap`之后，过一小会，再次进入容器查看，可以看到上面文件的值就变了，更新`ConfigMap`不会触发`Pod`的滚动更新，应用程序如果有缓存也需要重新读取才能生效。

> Volume挂载的方式可以热更新，但`Env`的方式更新后环境变量不会同步更新。

# 三、Secret

`secret`用来保存小片敏感数据的`k8s`资源，例如密码，`token`，或者秘钥。这类数据当然也可以存放在`Pod`或者镜像中，但是放在`Secret`中是为了更方便的控制如何使用数据，并减少暴露的风险。

## 3.1 资源清单

```
$ kubectl explain Secret
KIND:     Secret
VERSION:  v1

DESCRIPTION:
     Secret holds secret data of a certain type. The total bytes of the values
     in the Data field must be less than MaxSecretSize bytes.

FIELDS:
   apiVersion	<string>
   data	<map[string]string>
     Data contains the secret data. Each key must consist of alphanumeric
     characters, '-', '_' or '.'. The serialized form of the secret data is a
     base64 encoded string, representing the arbitrary (possibly non-string)
     data value here. Described in https://tools.ietf.org/html/rfc4648#section-4
   kind	<string>
   metadata	<Object>
   stringData	<map[string]string>
     stringData allows specifying non-binary secret data in string form. It is
     provided as a write-only convenience method. All keys and values are merged
     into the data field on write, overwriting any existing values. It is never
     output when reading from the API.

   type	<string>
     Used to facilitate programmatic handling of secret data.
```

配置方式和`ConfigMap`类似，`type`有3种类型：

- `Opaque`：base64编码格式的Secret，用来存储密码、秘钥等

- `kubernetes.io/service-account-token`：用来访问Kubernetes API，由Kubernetes自动创建，并且会自动挂载到Pod的`/run/secrets/kubernetes.io/serviceaccount`目录中。
- `kubernetes.io/dockerconfigjson`：用来存储私有docker registry的认证信息

**`kubernetes.io/service-account-token`**：前面安装`kubernetes-dashboard`后的登陆Token就是存储在Secret中，可以看看他的信息。

```
$ kubectl describe secret kubernetes-dashboard-admin-token-kjvvh -n kubernetes-dashboard
Name:         kubernetes-dashboard-admin-token-kjvvh
Namespace:    kubernetes-dashboard
Labels:       <none>
Annotations:  kubernetes.io/service-account.name: kubernetes-dashboard-admin
              kubernetes.io/service-account.uid: e1aaaecd-29f9-4367-be04-ed23f2ced38a

Type:  kubernetes.io/service-account-token

Data
====
ca.crt:     1025 bytes
namespace:  20 bytes
token:      eyJhbGciOiJSUzI1NiIsImtpZCI6Ik1OOWpGMThaYlY2aXdFYldJTU53RHVoa290c21xS3lCM0FsUmZ6Y0ExSXMifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlcm5ldGVzLWRhc2hib2FyZCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbi10b2tlbi1ranZ2aCIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6ImUxYWFhZWNkLTI5ZjktNDM2Ny1iZTA0LWVkMjNmMmNlZDM4YSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDprdWJlcm5ldGVzLWRhc2hib2FyZDprdWJlcm5ldGVzLWRhc2hib2FyZC1hZG1pbiJ9.Akaw3BzGklC_qtaTRGm18zQxSNlDdIizeDOhDS8vh8doAukSVtJZT48PeD3b2g09O_J4q0RUvrF31kTtNIeCvYvbQMDkJYukgDFdRUCTMFCpUCX1BqZE8ZOO1rGFUugTdRn7uv8YMqUINmHctwlW13blwXJBDcbIFDCcT2BRfNGU4YKlifKyNM3SIeqx83J7UspqHdhQxunJ6uXgCXttPZxn1ipQitc-
```

## 3.2 创建Secret

### 3.2.1 Opaque Secret

`Opaque`类型的数据是一个`map`类型，要求`value`是`base64`的编码格式：

```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```

创建之后查看，只看到显示几个字节长度。

```
$ kubectl get secret
NAME                  TYPE                                  DATA   AGE
default-token-zfmn7   kubernetes.io/service-account-token   3      14d
mysecret              Opaque                                2      17s

$ kubectl describe secret mysecret
Name:         mysecret
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
password:  6 bytes
username:  5 bytes
```

### 3.2.2 dockerconfigjson

```
$ kubectl create secret docker-registry myregistry --docker-server=DOCKER_SERVER --docker-username=DOCKER_USER --docker-password=DOCKER_PASSWORD --docker-email=DOCKER_EMAIL
secret "myregistry" created
```

如果要拉私有仓库可以指定`imagePullSecrets`

```
apiVersion: v1
kind: Pod
metadata:
  name: foo
spec:
  containers:
  - name: foo
    image: 192.168.1.100:5000/test:v1
  imagePullSecrets:
  - name: myregistry
```

## 3.3 使用Secret

使用上和`ConfigMap`类似。

### 3.3.1 环境变量

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:1.32.0
    command: ["sh", "-c", "env && sleep 1000"]
    env:
    - name: USERNAME
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: username
    - name: PASSWORD
      valueFrom:
        secretKeyRef:
          name: mysecret
          key: password
```

### 3.3.2 Volume挂载

```
apiVersion: v1
kind: Pod
metadata:
  name: secret-demo
spec:
  restartPolicy: Never
  containers:
  - name: busybox
    image: busybox:1.32.0
    command: ["sh", "-c", "sleep 1000"]
    volumeMounts:
    - name: v-mysecret
      mountPath: /data/
  volumes:
  - name: v-mysecret
    secret:
      secretName: mysecret
```

# 四、Peresistent Volume

## 4.1 关于pv

PersistentVolume (PV) 是外部存储系统中的一块存储空间，由管理员创建和维护。与 Volume 一样，PV 具有持久性，生命周期独立于 Pod。

## 4.2 资源清单

| 参数名                             | 字段类型          | 说明                                                         |
| ---------------------------------- | ----------------- | ------------------------------------------------------------ |
| spec.capacity                      | map[string]string |                                                              |
| spec.capacity.storage              | string            |                                                              |
| spec.accessModes[]                 | []string          | 指定访问模式，支持的模式有：<br />- ReadWriteOnce<br />- ReadOnlyMany<br />- ReadWriteMany |
| spec.persistentVolumeReclaimPolicy | string            | 回收策略<br />- Retain 保留， 手动回收<br />- Recycle 回收，基本删除<br />- Delete<br /> |
| spec.storageClassName              | string            | PV的分类，PVC里可以申请响应的分类                            |

## 4.3 示例

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv003
spec:
  capacity:
    storage: 3Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  storageClassName: host
  hostPath:
    path: /Users/peng/k8s/pv-data/pv003
```

这里创建了3个pv

```
$ kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
pv001   1Gi        RWO            Recycle          Available           host                    2m40s
pv002   2Gi        RWO            Recycle          Available           host                    87s
pv003   3Gi        RWO            Recycle          Available           host                    64s
```



# 五、Persistent Volume Claim

## 5.1 关于PVC

`PersistentVolumeClaim (PVC)` 是对 `PV` 的申请 (Claim)。PVC 通常由普通用户创建和维护。需要为 Pod 分配存储资源时，用户可以创建一个 PVC，指明存储资源的容量大小和访问模式（比如只读）等信息，Kubernetes 会查找并提供满足条件的 PV。

有了 `PersistentVolumeClaim`，用户只需要告诉 `Kubernetes` 需要什么样的存储资源，而不必关心真正的空间从哪里分配，如何访问等底层细节信息。这些 Storage Provider 的底层信息交给管理员来处理，只有管理员才应该关心创建 `PersistentVolume` 的细节信息。

## 5.2 资源清单

描述`pvc`对存储的要求，字段同`pv`。

## 5.3 示例

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
  storageClassName: host
```

创建pvc， spec描述它对存储的要求，对比上面三个pv，pv002 和pv003满足需求，执行之后再看就会发现pv002的状态变成了Bound

```
$ kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM          STORAGECLASS   REASON   AGE
pv001   1Gi        RWO            Recycle          Available                  host                    8m52s
pv002   2Gi        RWO            Recycle          Bound       default/pvc1   host                    7m39s
pv003   3Gi        RWO            Recycle          Available                  host                    7m16s

$ kubectl get pvc
NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc1   Bound    pv002    2Gi        RWO            host           97s
```

下面以Mysql容器关联PVC为例，和Pod关联Volumes类似，将挂载的配置改成了persistentVolumeClaim，创建成功后可以看到mysql数据存储到了pv002对应的目录里，登录容器可以看到mysql创建成功了。

```
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  restartPolicy: Always
  containers:
  - name: mysql
    image: mysql:5.7
    imagePullPolicy: IfNotPresent
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "123456"
    ports:
    - containerPort: 3306
    volumeMounts:
    - name: mysql-pvc-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-pvc-storage
    persistentVolumeClaim: 
      claimName: pvc1
```

## 5.4 PV & PVC

PV与PVC的关系大概是这样子：

![](../../static/uploads/k8s-pvc.png)

- 管理员提前创建好PV池
- 开发人员声明PVC，系统会根据PVC的要求绑定对应的PV
- 最后特定的Pod关联PVC，如果没有可用的PV则Pod创建失败

这里只是演示了Pod引用PVC，PVC和PV关联，PV绑定存储，只是服务架构里最底层的一步，并没有展现出在实际服务部署中优势。后面的章节中将会再着重介绍。

# 六、小结

本章介绍了常用的存储方案，`ConfigMap`，`Secret`，`PV&PVC`，对几种存储方案有个印象，后续使用过程中，在进行说明。至此，跟`Pod`相关的资源清单、`Pod`的用法、`Pod`依赖的周边存储就整理完了，后面将会着重介绍服务部署过程中的三大块，`Controller`、`Service`、`Ingress`。接下来看看如果通过控制器来来控制`Pod`。



---

- [1] [k8s的持久化存储PV&&PVC](https://www.cnblogs.com/benjamin77/p/9944268.html)
- [2] [k8s学习笔记之StorageClass+NFS](https://www.cnblogs.com/panwenbin-logs/p/12196286.html)