# Descheduler



### kubernetes加入Descheduler

```text
git clone https://github.com/kubernetes-incubator/descheduler.git
cd  descheduler
make image
```



### 创建集群角色

要为卸载程序提供必要的权限以在pod中工作，请创建集群角色：

```text
$ cat << EOF| kubectl create -f -
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: descheduler-cluster-role
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "watch", "list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list", "delete"]
- apiGroups: [""]
  resources: ["pods/eviction"]
  verbs: ["create"]
EOF
```

![](https://img-blog.csdnimg.cn/20201211173357358.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTMxODk4MjQ=,size_16,color_FFFFFF,t_70)



### 创建将用于运行作业的服务帐户：

```text
$ kubectl create sa descheduler-sa -n kube-system
```



### 将群集角色绑定到服务帐户：

```text
$ kubectl create clusterrolebinding descheduler-cluster-role-binding \
    --clusterrole=descheduler-cluster-role \
    --serviceaccount=kube-system:descheduler-sa
```



## RemoveDuplicates策略

该策略发现未充分利用的节点，并且如果可能的话，从其他节点驱逐pod，希望在这些未充分利用的节点上安排被驱逐的pod的重新创建。此策略的参数配置在`nodeResourceUtilizationThresholds`。

节点的利用率低是由可配置的阈值决定的`thresholds`。`thresholds`可以按百分比为cpu，内存和pod数量配置阈值 。如果节点的使用率低于所有（cpu，内存和pod数）的阈值，则该节点被视为未充分利用。目前，pods的请求资源需求被考虑用于计算节点资源利用率。

还有另一个可配置的阈值，`targetThresholds`用于计算可以驱逐pod的潜在节点。任何节点，所述阈值之间，`thresholds`并且`targetThresholds`被视为适当地利用，并且不考虑驱逐。阈值`targetThresholds`也可以按百分比配置为cpu，内存和pod数量。

### 启用策略

```text
[root@k8s-master-test yaml]# cat RemoveDuplicates.yaml
apiVersion: "descheduler/v1alpha1"
kind: "DeschedulerPolicy"
strategies:
  "RemoveDuplicates":
     enabled: True
[root@k8s-master-test yaml]# vim descheduler-cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
      name: descheduler
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: "true"
    spec:
      template:
        spec:
          serviceAccountName: descheduler-sa
          containers:
          - name: descheduler
            image: opsdockerimage/descheduler-descheduler:v0.22.0
            volumeMounts:
            - mountPath: /policy-dir
              name: policy-volume
            command:
            - /bin/descheduler
            - --v=4
            - --max-pods-to-evict-per-node=10
            - --policy-config-file=/policy-dir/RemoveDuplicates.yaml   #此处为<path-to-policy-dir/policy.yaml>文件,容器内路径
          restartPolicy: "OnFailure"
          volumes:
          - name: policy-volume
            configMap:
              name: descheduler-policy-configmap
```

### 创建一个configmap来存储descheduler策略

Descheduler策略在`kube-system`命名空间中创建为ConfigMap，以便可以将其作为卷安装在pod中。

```text
$ kubectl create configmap descheduler-policy-configmap \
     -n kube-system --from-file=<path-to-policy-dir/RemoveDuplicates.yaml>
```

### 创建nginx应用

```text
cat << EOF > nginx-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
      nodeSelector:
        mem: large
EOF
```

```text
[root@k8s-master-test yaml]# kubectl apply -f nginx-deployment.yaml
```

```text
[root@k8s-master-test yaml]# kubectl apply -f descheduler-cronjob.yaml
```

分别为node1，node2打上标签（mem=large)

## LowNodeUtilization策略

该策略发现未充分利用的节点，并且如果可能的话，从其他节点驱逐pod，希望在这些未充分利用的节点上安排被驱逐的pod的重新创建。此策略的参数配置在`nodeResourceUtilizationThresholds`。

节点的利用率低是由可配置的阈值决定的`thresholds`。`thresholds`可以按百分比为cpu，内存和pod数量配置阈值 。如果节点的使用率低于所有（cpu，内存和pod数）的阈值，则该节点被视为未充分利用。目前，pods的请求资源需求被考虑用于计算节点资源利用率。

还有另一个可配置的阈值，`targetThresholds`用于计算可以驱逐pod的潜在节点。任何节点，所述阈值之间，`thresholds`并且`targetThresholds`被视为适当地利用，并且不考虑驱逐。阈值`targetThresholds`也可以按百分比配置为cpu，内存和pod数量。

### 启用策略

```text
[root@k8s-master-test yaml]# cat lownodeutilization.yaml
apiVersion: "descheduler/v1alpha1"
kind: "DeschedulerPolicy"
strategies:
  "LowNodeUtilization":
     enabled: true
     params:
       nodeResourceUtilizationThresholds:
         thresholds:                     #阈值
           "pods": 5
         targetThresholds:             #目标阈值
           "pods": 10
     
[root@k8s-master-test yaml]# vim descheduler-cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
      name: descheduler
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: "true"
    spec:
      template:
        spec:
          serviceAccountName: descheduler-sa
          containers:
          - name: descheduler
            image: opsdockerimage/descheduler-descheduler:v0.22.0
            volumeMounts:
            - mountPath: /policy-dir
              name: policy-volume
            command:
            - /bin/descheduler
            - --v=4
            - --max-pods-to-evict-per-node=10
            - --policy-config-file=/policy-dir/lownodeutilization.yaml   #此处为<path-to-policy-dir/policy.yaml>文件
          restartPolicy: "OnFailure"
          volumes:
          - name: policy-volume
            configMap:
              name: descheduler-policy-configmap
```

### 创建一个configmap来存储descheduler策略

Descheduler策略在`kube-system`命名空间中创建为ConfigMap，以便可以将其作为卷安装在pod中。

```text
$ kubectl create configmap descheduler-policy-configmap \
     -n kube-system --from-file='path-to-policy-dir/lownodeutilization.yaml'
```

### 创建nginx应用

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: descheduler
  labels:
    app: nginx
spec:
  replicas: 10
  nodeSelector:
    max: "true"
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.9.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
      nodeSelector:
        mem: large
```

```text
[root@k8s-master-test yaml]# kubectl apply -f nginx-deployment.yaml
```

```text
[root@k8s-master-test yaml]# kubectl apply -f descheduler-cronjob.yaml
```

### RemovePodsViolatingNodeAffinity策略

此策略可确保从节点中删除违反节点关联的pod。例如，在nodeA上调度了podA，它在调度时满足节点关联性规则`requiredDuringSchedulingIgnoredDuringExecution`，但随着时间的推移，nodeA不再满足该规则，那么如果另一个节点nodeB可用，它满足节点关联性规则，那么podA将被逐出nodeA。

### 启用策略

```
[root@k8s-master-test yaml]# cat removepodsviolatingNodeAffinity.yaml
apiVersion: "descheduler/v1alpha1"
kind: "DeschedulerPolicy"
strategies:
  "RemovePodsViolatingNodeAffinity":
    enabled: true
    params:
      nodeAffinityType:
      - "requiredDuringSchedulingIgnoredDuringExecution"
      nodeFit: false
[root@k8s-master-test yaml]# vim descheduler-cronjob.yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: descheduler
  namespace: kube-system
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    metadata:
      name: descheduler
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: "true"
    spec:
      template:
        spec:
          serviceAccountName: descheduler-sa
          containers:
          - name: descheduler
            image: opsdockerimage/descheduler-descheduler:v0.22.0
            volumeMounts:
            - mountPath: /policy-dir
              name: policy-volume
            command:
            - /bin/descheduler
            - --v=4
            - --max-pods-to-evict-per-node=10
            - --policy-config-file=/policy-dir/removepodsviolatingNodeAffinity.yaml   #此处为<path-to-policy-dir/policy.yaml>文件
          restartPolicy: "OnFailure"
          volumes:
          - name: policy-volume
            configMap:
              name: descheduler-policy-configmap
```

### 创建一个configmap来存储descheduler策略

Descheduler策略在`kube-system`命名空间中创建为ConfigMap，以便可以将其作为卷安装在pod中。

```text
$ kubectl create configmap descheduler-policy-configmap \
     -n kube-system --from-file='path-to-policy-dir/removepodsviolatingNodeAffinity.yaml'
```

### 创建nginx应用

```text
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: descheduler
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 10
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: mem
                operator: In
                values:
                - large
      containers:
      - name: nginx
        image: nginx:1.9.0
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
```

```text
[root@k8s-master-test yaml]# kubectl apply -f nginx-deployment.yaml
[root@k8s-master-test yaml]# kubectl apply -f descheduler-cronjob.yaml
```

