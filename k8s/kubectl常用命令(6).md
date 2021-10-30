## 1、查看集群状态

```bash
kubectl version --short=true 查看客户端及服务端程序版本信息
kubectl cluster-info 查看集群信息
```

## **2、创建资源对象**

```text
kubectl run name --image=(镜像名) --replicas=(备份数) --port=(容器要暴露的端口) --labels=(设定自定义标签)
kubectl create -f **.yaml  陈述式对象配置管理方式
kubectl apply -f **.yaml  声明式对象配置管理方式（也适用于更新等）
```

## 3、查看资源对象

```text
kubectl get namespace 查看命名空间
kubectl get pods,services -o wide (-o 输出格式 wide表示plain-text)
kubectl get pod -l "key=value,key=value" -n kube-system (-l 标签选择器(多个的话是与逻辑)，-n 指定命名空间，不指定默认default)
kubectl get pod -l "key1 in (val1,val2),!key2" -L key (-l 基于集合的标签选择器, -L查询结果显示标签) 注意：为了避免和shell解释器解析!,必须要为此类表达式使用单引号
kubectl get pod -w(-w 监视资源变动信息)
```

## 4、打印容器中日志信息

```text
kubectl logs name -f -c container_name -n kube-system (-f 持续监控，-c如果pod中只有一个容器不用加)
```

## 5、在容器中执行命令

```text
kubectl exec name -c container_name -n kube-system -- 具体命令
kubectl exec -it pod_name /bin/sh 进入容器的交互式shell
```

## 6、删除资源对象

```text
kubectl delete [pods/services/deployments/...] name 删除指定资源对象
kubectl delete [pods/services/deployments/...] -l key=value -n kube-system  删除kube-system下指定标签的资源对象
kubectl delete [pods/services/deployments/...] --all -n kube-system 删除kube-system下所有资源对象
kubectl delete [pods/services/deployments/...] source_name --force --grace-period=0 -n kube-system 强制删除Terminating的资源对象
kubectl delete -f xx.yaml
kubectl apply -f xx.yaml --prune -l <labels>(一般不用这种方式删除)
kubectl delete rs rs_name --cascade=fale(默认删除控制器会同时删除其管控的所有Pod对象，加上cascade=false就只删除rs)
```

## 7、更新资源对象

```text
kubectl replace -f xx.yaml --force(--force 如果需要基于此前的配置文件进行替换，需要加上force)
```

## 8、将服务暴露出去(创建Service)

```text
kubectl expose deployments/deployment_name --type="NodePort" --port=(要暴露的容器端口) --name=(Service对象名字)
```

## 9、扩容和缩容

```text
kubectl scale deployment/deployment_name --replicas=N
kubectl scale deployment/deployment_name --replicas=N --current-replicas=M 只有当前副本数等于M时才会执行扩容或者缩容
```

## 10、查看API版本

```text
kubectl api-versions
```

## 11、在本地主机上为API Server启动一个代理网关

```text
kubectl proxy --port=8080
之后就可以通过curl来对此套字节发起访问请求
curl localhost:8080/api/v1/namespaces/ | jq .items[].metadata.name (jq可以对json进行过滤)
```

## 12、当定义资源配置文件时，不知道怎么定义的时候，可以查看某类型资源的配置字段解释

```text
kubectl explain pods/deployments/...(二级对象可用类似于pods.spec这种方式查看)
```

## 13、查看某资源对象的配置文件

```text
kubectl get source_type source_name -o yaml --export(--export表示省略由系统生成的信息) 后面加 > file.yaml就可以快速生成一个配置文件了
```

## 14、标签管理相关命令

```text
kubectl label pods/pod_name key=value 添加标签,如果是修改的话需要后面添加--overwrite
kubectl label nodes node_name key=value 给工作节点添加标签，后续可以使用nodeSelector来指定pod被调度到指定的工作节点上运行
```

## 15、注解管理相关命令

```text
kubectl annotate pods pod_name key=value
```

## 16、patch修改Deployment控制器进行控制器升级

```text
kubectl patch deployment deployment-demo -p '{"spec": {"minReadySeconds": 5}}'(-p 以补丁形式更新补丁形式默认是json)
kubectl set image deployments deployment-demo myapp=ikubernetes/myapp:v2 修改depolyment中的镜像文件
kubectl rollout status deployment deployment-demo 打印滚动更新过程中的状态信息
kubectl get deployments deployment-demo --watch 监控deployment的更新过程
kubectl kubectl rollout pause deployments deployment-demo 暂停更新
kubectl rollout resume deployments deployment-demo 继续更新
kubectl rollout history deployments deployment-demo 查看历史版本(能查到具体的历史需要在apply的时候加上--record参数)
kubectl rollout undo deployments deployment-demo --to-revision=2 回滚到指定版本，不加--to-version则回滚到上一个版本
```

