记得下周四晚上8点考试
做题记录
2024年11月11日18点08分： 1个小时做了6道题。
2024-12-5：

============CKA考试练习题
执行以下命令，添加kubectl/kubeadm命令补全
kubectl completion bash > /etc/bash_completion.d/kubectl
kubeadm completion bash > /etc/bash_completion.d/kubeadm
source /etc/bash_completion.d/kubectl
source /etc/bash_completion.d/kubeadm

----题目1：节点备份和恢复

命令数量：
难度：4
//ectdctl sanpshot save/restore命令
1，apt install etcd-client -y  //安装etcdctl命令，可能不需要
2，ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
snapshot save /srv/etcd-snapshot.db  
//用ectdctl -h查看命令帮助。注意两点：ETCDCTD_API=3和--endpoints参数
//证书
3，停止使用和写入数据  //关键在于记住/etc/kubternetes/manifest和/var/lib/etcd这两个路径。
mv /etc/kubernetes/manifests /etc/kubernetes/manifests.bak 
 //该路径下存放的是《静态pod的配置文件》。 Kubelet需要这个路径下的文件来创建pod。删除后，会阻止kubelet启动。
//systemctl status kubelet 找到config文件，cat config文件，可以找到这个路径。
mv /var/lib/etcd /var/lib/etcd.bak 
//该路径是etc数据库的文件，删掉文件，在下一步restore时会重新创建。
//这个文件路径需要记住。
4，restore数据库
ETCDCTL_API=3 etcdctl \
--endpoints=https://127.0.0.1:2379 \
--cacert=/etc/kubernetes/pki/etcd/ca.crt \
--cert=/etc/kubernetes/pki/etcd/server.crt \
--key=/etc/kubernetes/pki/etcd/server.key \
--data-dir /var/lib/etcd \
snapshot restore /srv/etcd_exam_backup.db 
//关键在于记住--data-dir这个参数和路径。这个路径下保存的是《etcd数据库文件》。
5，重启kubelet服务
mv /etc/kubernetes/manifests.bak /etc/kubernetes/manifests 
//先恢复静态pod的配置文件，再重启kubelet。不然kubelet会因为找不到配置文件导致重启失败。
systemctl restart kubelet.service
 //重启kubelet
6，验证 
//这个题必须验证，否则容易失败。
kubectl get nodes

----题目2：RBAC授权
命令数量：3
难度：2
You have been asked to create a new ClusterRole for a deployment pipeline and bind it to a specific ServiceAccount scoped to a specific namespace. Task Create a new ClusterRole named deployment-clusterrole, which only allows to create the following resource types:

    Deployment
    StatefulSet
    DaemonSet

Create a new ServiceAccount named cicd-token in the existing namespace app-team1

Bind the new ClusterRole to the new ServiceAccount cicd-token, limited to the namespace app-team1
问题1： rolebinding和clusterrolebinding的区别？
回答：本题目要求的是rolebinding，为什么不是clusterrolebinding？
https://kubernetes.io/zh-cn/docs/reference/access-authn-authz/rbac/#rolebinding-and-clusterrolebinding“
因为RoleBinding 在指定的“名字空间”中执行授权，而 ClusterRoleBinding 在集群范围执行授权。本题目明确说明是在namespace中使用，所以使用rolebinding
”
//需要用到三个kubectl create命令。最后的验证不是必须的。
1）创建clusterrole，给与相关权限。
kubectl create clusterrole NAME --verb create --resource deployment
问题：resource参数中的deployment是单数还是复数，需要区分吗？
回答：不需要区分。
 //用-h命令查看命令举例，不需要记住参数
2）创建serviceaccount，在某个namespace中。
kubectl create serviceaccount NAME -n NAMESAPCE 
//注意添加namespace，在帮助命令里，没有提到namespace
3）创建rolebinding。
//rolebinding是在名字空间有效，clusterrolebinding是在集群有效。所以本题使用rolebinding
kubectl create rolebinding NAME --clusterrole NAME --serviceaccount NAMESPACE:NAME -n NAMESPACE 
//用-h命令查看。注意sa名字中需要添加namespace参数。
//可以用kubectl describe命令检查创建的rolebinding
kubectl -n app-team1 describe rolebindings.rbac.authorization.k8s.io
4) 验证。使用kubectl auth can-i命令，检查权限是否正常。
kubectl -n NAMESPACE auth can-i create NAME --as system:serviceaccount:app-team1:cicd-token 
//用kubectl auth can-i -h命令查看命令举例
//--as的格式：system:serviceaccount:NAMESPACE:SERVICEACCOUNT

-------题目3：节点维护
难度：1颗星。
Set the node named k8s-master as unavailiable and reschedule all the pods running on it
//cordon和drain两个命令。
1，禁用新的调度并观察当前状态
kubectl cordon k8s-master
kubeclt get nodes
2，驱赶到其他节点
kubectl drain k8s-master --delete-emptydir-data --ignore-daemonsets --force
//难点在于记得用命令帮助，查看以ignore和delete开头的两个参数。


----题目4：upgrading Kubeadm clusters
难度：5颗星。
Given an existing kubernetes cluster running version 1.29.1,upgrade all of the Kubernetes control plane and node components on the master node only to version 1.29.2-1.1,Please do not upgrade etcd database.

You are also expected to upgrade kubelete and kubectl on the master node.
解题思路：
--------------在K8S网站搜索"upgrading kubeadm clusters"关键字。
注意：
1，先升级kubeadm，然后升级kubelet和kubectl
2，
1）需要首先install kubeadm，
  1.1 kubectl get nodes //
  1.2 kubectl cordon node
  1.3 kubectl drain node --delete-empty-dir --ignore-daemonsets 
//这两条命令和第3题中的命令相同。
  1.4 apt update && apt-cache policy kubeadm | grep 1.xx.x-x.xx && apt-mark unhold kubeadm 
//找到要升级的版本
  1.5 apt install kubeadm=1.xx.xx-x.xx -y  //先安装kubeadm
  1.6 apt-mark hold kubeadm
  1.7 kubeadm version  
//验证已经安装目标版本
2）然后再upgrade kubeadm
  2.1 kubeadm upgrade plan  //检查是否能安装
  2.2 kubectl upgrade apply v1.xx.xx --etcd-upgrade=false 
//用kubeadm upgrade apply -h查看etcd不升级的参数。
3)升级kubectl和kubelet。
apt-mark unhold kubelet kubectl
apt install kubectl=1.xx.xx-x.xx kubelet=1.xx.x-x.xx -y
apt-mark hold kubelet kubectl
4)重启kubelet服务  //在网站上有这两个命令
systemctl daemon-reload  //重启守护进程
systemctl restart kubelet  //重启kubelet服务
5)uncordon node并验证升级是否成功。正式考试的时候记得uncordon node
kubectl uncordon k8s-master
kubectl get nodes

-----------解题步骤
1，查看当前cluster的版本，kubectl get nodes，//不是必须的
	1.1 查看当前kubeadm版本：kubeadm version
	1.2 查看当前kubelet版本  kubelet --version
	1.3 查看当前kubectl版本 kubectl version
2，禁用调度。kubectl cordon k8s-master //必须执行
		kubectl drain k8s-master --delete-emptydir-data --ignore-daemonsets
3，kubeadm的升级。
	apt update
	apt-mark unhold kubeadm。放开kubeadm的升级限制。
	安装新版本apt install kubeadm=1.29.2-1.1 -y //安装kubeadm指定版本，本步骤的核心命令。
	重新hold版本，避免自动升级。apt-mark unhold kubeadm
	查看kubeadm版本：kubeadm version
3，查询可升级的版本：kubeadm upgrad plan
	在命令的输出结果中，会有一条提示：使用 kubeadm upgrade apply v1.29.2进行升级。
4，kubeadm upgrade apply v1.29.2 --etcd-upgrade=false  //使用upgrade命令升级kubeadm
	注意：在kubeadm upgrade apply命令中，加上--etcd-upgrade=false这个参数，以避免etcd被升级。
	命令结果中会有提示，已经升级clsuter到1.29.2
5，升级kubelet和kubectl
	apt-mark unhold kubelet kubectl //放开kubelet 和kubectl
   	apt install kubectl=1.29.2-1.1 kubelet=1.29.2-1.1 //安装新版本
  	apt-mark hold kubeadm kubelet kubectl //hold当前版本

   	systemctl daemon-reload  //重新加载守护进程，这个命令容易被忘记！！！！
  	systemctl restart kubelet  //重启kubelet
6，放开主节点，并验证cluster节点微码版本。
   	kubectl uncordon k8s-master  //和第二步相对应，放开主节点
   	kubectl get nodes  //验证节点微码版本已经更新

---------Q5 创建networkpolicy
Create a new NetworkPolicy named allow-port-from-namespace that allows Pods in namespace corp to connect to port 80 of other Pods in the internal namespace.

Ensure that the new NetworkPolicy:

    does not allow access to Pods not listening on port 80
    does not allow access from Pods not in namespace corp
============解题思路
//难点在于记住yml文件的格式。尤其是名字空间在yml中的位置
1）networkpoliy限制的是指定的名字空间，metadata.namespace字段指定的名字空间将会被networkpolicy限制。
2）spec.podSelector:{} 表示该名字空间下的所有pod都受到限制
3）policyType是ingress，表示限制进入访问
4）from.namespaceSelector.matchLabels，这个label可以通过kubectl describe namespace来获取，注意修改为用：形式隔开的string。


1，在k8s.io网站中，搜索network policy关键字，复制并修改yml文件
vi networkpolicy.yml
//
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-port-from-namespace
  namespace: internal  //指定名字空间，这个名字空间是被networkpolicy限制的对象。
spec:
  podSelector: {} //改成这样，表示internal名字空间下的所有pod都将受到本网络策略的限制
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: corp  
//允许来自corp名字空间中的流量。使用kubectl describe namespace corp命令查看这个corp的label。
//注意把label中的=改成：
    ports:
    - protocol: TCP
      port: 80  //可以访问

2，kubeclt create -f  //创建networkpolicy
3，kubectl describe -f //查看
4，验证
kubectl get pod -n internal -o wide  //查看internal中的pod的IP地址
curl IP //从node上无法直接访问
kubectl get pod -n corp //查看corp名字空间中的pod名字。
kubectl exec -it corppod -n corp  -- curl 172.16.126.1 
//从corp名字空间中的pod上可以访问

------Q6 创建service
Reconfigure the existing deployment “front-end“ and add a port specification named http exposing port 80/tcp of the existing container nginx
Create a new service named front-end-svc exposing the container port http.
Configure the new service to also expose the individual Pods via a NoedPort on the nodes on which they are scheduled.
//===========本题的重点是2个命令，分别是：
1）kubectl edit
//添加以下字段。可使用kubectl explain deployment.spec.template.spec.containers.ports命令查看具体的字段名称。注意：- containerPort应该与ports对齐。
ports：
- containerPort：80
  name：http
  protocol：tcp
2）kubectl expose命令。
//注意添加--type，--port，--target-port三个参数。
//命令简单技巧：复制kubectl expose -h帮助中命令举例的最后一个，再添加--name和--type两个参数。
3）用curl命令验证可以访问
----问题1：endpoint是什么？：
	回答：endpoint是实现实际服务的端点的集合。使用kubectl get endpoint可以看到结果。
----问题2：几个IP地址的关系？
	回答：clsuter IP：集群中使用的IP。endpoint IP：这是pod的IP地址。node IP: 节点IP。

===========解题步骤：
1，重新配置deployment
kubectl edit deployments.apps front-end 
//编辑deployment后，会自动更新。
//使用kubectl describe deployment查看添加的字段是否生效。
//k8s.io搜索”expose pods to cluster“关键字，在”service连接到应用“章节中找到在yml文件中添加ports的写法。
2，创建服务
kubectl expose deployment front-end --name=front-end-svc --port=80 --target-port=80 --type=NodePort
//--port=80：本服务的端口号。--target-port=80：目标端口，这是容器的端口号，本服务的流量会导入到该容器的这个端口上。
kubectl get service
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
front-end-svc   NodePort    10.104.139.137   <none>        80:32402/TCP   74s
//有两种访问方式。第一种：使用cluster IP+暴露端口号（80）可以访问。第二种：使用node名+节点端口号（32402），也可以访问。
3，验证
curl 10.104.139.137:80 //访问集群IP的80端口，可以成功。
curl k8s-woker2:32402 //访问服务所在的node和端口号，可以成功。
//用pod IP访问是不通的

------------Q7：create ingress
Create a new nginx ingress resource as follows:<------3
    Name: pong<-------1
    Namespace: ing-internal<---------2
    Exposing service 《hi》 on path 《/hi》 using service port 《5678》<-----------4/5/6
Tips:
The availability of service hi can be checked using the following commands,which should retun hi: curl -KL <INTERNAL_IP>/hi
//用“ingress”关键字搜索，复制脚本并修改。
1）编辑yml文件。
vim ingress.yml
//共计有6处需要修改。
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress //修改为pong<--------1
  namespace：ing-internal
 //！！！添加namespace参数<-----2。网页的脚本代码中没有这个参数，需要自己记住添加。
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx-example  //修改为nginx，！！！容易忘记<---------3 
  rules:
  - http:
      paths:
      - path: /testpath  //修改为/hi<----------4
        pathType: Prefix
        backend:
          service:
            name: test  //修改为hi<----------5
            port:
              number: 80  //修改为5678<--------6

2）创建ingress
kubectl create -f ingress.yml 
3）验证
kubectl get ingress -n ing-internal《----记得添加namespace参数
//输出结果中要能看到IP地址，否则就是ingress创建不成功。
curl -kL 192.168.8.5/hi

-------Q8:scale deployment
难度：1颗星
Scale the deployment loadbalancer to 6 pods and record it
//
1，kubectl scale deployment loadbalancer --replicas=6 --record 
//使用kubeclt scale deployment -h命令查看帮助。注意添加--record参数。
2，kubectl get deployments.apps loadbalancer

-------Q9: Assigning pods to nodes
难度：2颗星
Schedule a pod as follows:
    Name: nginx-kusc00401《-----------1
    Image: nginx《-----------2
    Node selector: disk=spinning《------------3
//做题方法： 搜索”assign pod to node“关键字，选择task章节，找到代码段，直接复制后修改。
//记住nodeSelector中disk:spinning的写法，网页中是disktype，需要改
1，vi assignpod.yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-kusc00401<------------1
spec:
  containers:
  - name: nginx
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/nginx<-----------2
    imagePullPolicy: IfNotPresent
  nodeSelector:  //和containers对齐
    disk: spinning <-------------3,！！！！记住这个写法。这个nodeSelector字段在网页的脚本中没有。
2, 创建pod
kubectl create -f assignpod.yml
3, 验证 
kubectl get nodes --show-labels  //找到带有disk=spinning label的节点
kubectl get pods -o wide //确认pod已经被分配到这个节点上。

-----------Q10 Find how many health nodes
难度：1颗星
Check to see how many nodes are ready (not including nodes tainted NoSchedule) and write the number to /opt/kusc00402.txt

1，查看节点描述的Taints字段
kubectl describe node | grep -i taints //查看结果，确定正常节点数量
2， echo NO > /opt/kusc00402.txt

--------Q11 Create multi container in pod
难度：2颗星
Create a pod named kucc1 with a single app container for each of the following images running inside(there may be between 1 and 4 images specified): nginx+redis+memcached+consul.

//搜索"create pod"关键字，复制一段代码。
1，vi multipod.yml
apiVersion: v1
kind: Pod
metadata:
  name: kucc1
spec:
  containers:
  - name: nginx
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/nginx
  - name: redis
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/redis
  - name: memcached
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/memcached
  - name: consul
    image: registry.cn-shanghai.aliyuncs.com/cnlxh/consul
2, kubectl create -f multipod.yml
3, kubectl get pod kucc1

-----------Q12 创建PV
难度：2颗星
Create a persistent volume with name app-config, of capacity 2Gi and access mode ReadWriteMany. the type of volume is hostPath and its location is /srv/app-config
//搜索”create PV task“，找到task章节中生成PV的代码并复制修改
1，vi pv.yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume  //改成app-config
  labels:  //删掉2行
    type: local
spec:
  storageClassName: manual  //删掉
  capacity:
    storage: 10Gi  //改成2Gi
  accessModes:
    - ReadWriteOnce  //改成ReadWriteMany
  hostPath:
    path: "/mnt/data" //改成/srv/app-config
2, kubectl create -f pv.yml
3. kubectl describe -f pv.yml

-----------Q13 创建PVC
创建一个名字为pv-volume的pvc，指定storageClass为csi-hostpath-sc，大小为10Mi 然后创建一个Pod，名字为web-server，镜像为nginx，并且挂载该PVC至/usr/share/nginx/html，挂载的权限为ReadWriteOnce。之后通过kubectl edit或者kubectl path将pvc改成70Mi，并且记录修改记录。

1, vi pvc.yml
// search "pod pvc" get config file.
apiVersion: v1
kind: PersistentVolumeClaim;
metadata:
  name: task-pv-claim //修改为pv-volume <--------------1
spec:
  storageClassName: manual //csi-hostpath-sc <-------------2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi //10Mi <---------------3
2, 创建PVC
kubectl create -f pvc.yml
3, vi pod.yml 
//same webpage as step 1, you can get this config file.
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod //web-server <------------4
spec:
  volumes:
    - name: task-pv-storage  //这个volume name和下面container中mount的volume名字一样即可。
      persistentVolumeClaim:
        claimName: task-pv-claim //pv-volume<----------5
  containers:
    - name: task-pv-container  //这个名字没有要求
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html" 
//  改成/usr/share/nginx/html<-------6
          name: task-pv-storage  //和上面定义的volume的名字一致
4, create pod
kubectl create -f pod.yml
5, make sure storageclass can be extended.
kubectl get storageclass csi-hostpath-sc  				//检查确认其中的ALLOWVOLUMEEXPANSION IS TRUE
kubectl edit storageclass csi-hostpath-sc 				//修改storageclass属性
//change above parameter to true
6, kubectl edit pvc pv.volume --record=true
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 70Mi<----------modify this 

---------------Q14 Monitor pods log
难度：1颗星
监控名为foobar的Pod的日志，并过滤出具有unable-to-access-website 信息的行，然后将写入到 /opt/foobar.txt 
//记住kubectl logs命令
kubectl logs foobar  | grep unable-to-access-website > /opt/foobar.txt

----------------Q15 add sidecar container
Without changing its existing containers, an existing Pod needs to be integrated into Kubernetes's build-in logging architecture(e.g kubectl logs).Adding a streaming sidecar container is a good and common way accomplish this requirement.

Task Add a busybox sidecar container to the existing Pod legacy-app. The new sidecar container has to run the following command:

/bin/sh -c tail -n+1 -f /var/log/legacy-app.log

Use a volume mount named logs to make the file /var/log/legacy-app.log available to the sidecar container.

TIPS

    Don't modify the existing container.
    Don't modify the path of the log file, both containers
    must access it at /var/log/legacy-app.log
//sidecar容器是和main容器一起运行的secondary容器。用于扩展主要容器的功能。
做题步骤：
1，导出容器到yaml文件中
kubectl get pod legacy-app -o yaml > sidecar.yaml
2，编辑yaml文件。vi sidecar.yml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app
....
spec:
  containers:
  - image: registry.cn-shanghai.aliyuncs.com/cnlxh/busybox
    ...
    # 找到第一个容器下面本来就存在的volumeMounts参数，添加下面的logs卷
    volumeMounts:
    - name: logs
      mountPath: /var/log
      ...
    # 新加一个容器，添加name，image，args，volumeMounts等参数。
  - name: busybox
    image: busybox
    args: [/bin/sh, -c, 'tail -n+1 -f /var/log/legacy-app.log']
//这个地方，写command或者args都可以。在线文档中使用的是command。
    volumeMounts:
    - name: logs
      mountPath: /var/log
      ...
  # 创建出这个name为logs的卷。需要注意的是，导出的文件中，本来就有volumes参数，请在它本来的volumes参数下面写logs卷的创建，不然无法创建pod，提示没有卷。
  volumes:
  - name: logs
    emptyDir: {}  
//！！！注意这个volume的类型应该是emptyDir。开始时是空的，pod中所有容器都能读写emptyDir中相同的文件。Pod删除后，emptyDir上的数据也被删除。
3, kubectl delete pod legacy-app  //直接删除pod
4, kubectl create -f sidecar.yml  //重新创建pod
5, kubectl describe pod legacy-app  //检查pod中的容器参数是否与题目要求一致。

---------------Q16 find pod with high CPU use
难度：1颗星
找出具有name=cpu-user标签的Pod，并过滤出使用CPU最高的Pod，然后把它的名字写在/opt/findhighcpu.txt文件里
1，kubectl top pod -l name=cpu-user -A --sort-by cpu
//使用kubectl top pod -h查看命令帮助。主要是-l -A --sort-by这三个参数。

-----------Q17 fixing k8s node state
A kubernetes worker node, named k8s-woker1 is in state NotReady. Investigate why this is the case, and perform any appropriate steps to bring the node to a Ready state,ensuring that any changes are made permanent.

Tips:

1、you can ssh to the failed node using:

ssh k8s-woker1

2、you can assume elevated privileges on the node with the following command：

sudo  -i
1）get node status
kubectl get nodes
2)get node conditions, kubelet stoped is the reason.
kubectl describe node k8s-worker1 //check conditions 
3)ssh to node
ssh k8s-worker1
4)restart docker, cri-docker, kubelet
systemctl start docker cri-docker kubelet
systemctl enable docker cri-docker kubelet



-------------K8S课堂笔记

docker save -o 
----5-25
---上午1：assigning pods to nodes
nodeSelector
requiredDuringSchedulingIgnoredDuringExecution
preferredDuringSchedulingIgnoredDuringExecution
pod只能运行在arm上怎么实现？第一步，node上先打上arm的标签。第二步，nodeSelector，nodeName或亲和性（pod spec.affinity.nodeAffinity）中选择arm类型的node。
直接指定node name的调度方式，如果node关机，会怎么样？
上午2：config map
config map是一种轻量级的POD数据存储和使用方式。
外置配置文件挂载到pod中，实现pod使用不同配置文件启动和服务。
K8S中创建POD时候，使用configMap方式和使用挂载PV的方式引用配置文件有什么区别？分别有什么优点？
prompt: 我想在K8s管理的pod中创建一个mysql容器，mysql的密码使用secret的方式从外部获取，告诉我实现的步骤。
-----下午
configmap和secret
存放小型的配置文件，环境变量。
使用方式：在pod中定义env
CPU, memroy限额：
resouces.requests，resources.limits
resources.quota是namespace的一个属性。
===========01:58 control access for k8s api
role and clusterRole
练习中的“访问控制”单元（除了“网络策略”部分）
03：30以后没有听

----------5-26
上午
control access
kubectl cordon/drain
hpa自动水平伸缩
----下午
sidecar这个题目不好做，可以放弃。
set cuc/cum

---5-18日课程
虚拟机恢复1.29.1快照
namespace没有实现隔离，只是实现了互相看不到。
kubectl -n $namespace ... 写在前面，可以实现自动补齐
pod是K8S中最小的工作单位，pod由容器构成。
11点02分：pod模板
kubectl exec -it  nginx -- /bin/bash
-c 指定进入的容器名称
init容器会首先启动，准备好先决条件，然后正常容器才会启动。

-------------5-19日课程
-----------PDF标题
健康检查
配置存储卷：
配置存储类：

1）服务发现
pod是非永久性资源，IP地址不稳定。通过selector matchLabels： app=mysql的方式，找到pod
service的类型有：ClusterIp类型，NodePort类型
headless类型：访问的是service的名称，而不是IP地址。
kubectl exec -it $podname -- /bin/sh
nslookup headless
真实使用的是ingress
2）下午：
探针，
优雅关闭

-------------
PDF课件内容				回放视频时间点
configure storage volume	02:07	




----在worker上执行：

sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release


sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://mirror.nju.edu.cn/docker-ce/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirror.nju.edu.cn/docker-ce/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

wget https://ghproxy.net/https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.13/cri-dockerd_0.3.13.3-0.ubuntu-focal_amd64.deb
dpkg -i cri-dockerd_0.3.13.3-0.ubuntu-focal_amd64.deb

sed -i 's/ExecStart=.*/ExecStart=\/usr\/bin\/cri-dockerd --container-runtime-endpoint fd:\/\/ --network-plugin=cni --pod-infra-container-image=registry.cn-hangzhou.aliyuncs.com\/google_containers\/pause:3.8/' /lib/systemd/system/cri-docker.service
systemctl daemon-reload
systemctl restart cri-docker.service
systemctl enable cri-docker.service

