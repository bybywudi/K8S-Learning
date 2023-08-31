# 容器调度
作为一个容器集群编排与管理项目，Kubernetes 为用户提供的基础设施能力，不仅包括了应用定义和描述的部分，还包括了对应用的资源管理和调度的处理。   
简单的说，调度就是K8S的一系列逻辑，来判断某一个容器应该落在哪个Node上，应该给他多少CPU和内存资源等等。   
## 资源模型和资源管理
在 Kubernetes 里，Pod 是最小的原子调度单位。这也就意味着，所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段。而这其中最重要的部分，就是 Pod 的 CPU 和内存配置    
```
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: db
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: "password"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
  - name: wp
    image: wordpress
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
在 Kubernetes 中，像 CPU 这样的资源被称作“可压缩资源”（compressible resources）。   
它的典型特点是，当可压缩资源不足时，Pod 只会“饥饿”，但不会退出。而像内存这样的资源，则被称作“不可压缩资源（incompressible resources）。   
当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。而由于 Pod 可以由多个 Container 组成，所以 CPU 和内存资源的限额，是要配置在每个 Container 的定义上的。   
### 配置资源的方式
这样，Pod 整体的资源配置，就由这些 Container 的配置值累加得到。其中，Kubernetes 里为 CPU 设置的单位是“CPU 的个数”。比如，cpu=1 指的就是，这个 Pod 的 CPU 限额是 1 个 CPU。   
当然，具体“1 个 CPU”在宿主机上如何解释，是 1 个 CPU 核心，还是 1 个 vCPU，还是 1 个 CPU 的超线程（Hyperthread），完全取决于宿主机的 CPU 实现方式。   
Kubernetes 只负责保证 Pod 能够使用到“1 个 CPU”的计算能力。   
此外，Kubernetes 允许CPU 限额设置为分数，比如在例子里，CPU limits 的值就是 500m。所谓 500m，指的就是 500 millicpu，也就是 0.5 个 CPU 的意思。   
而对于内存资源来说，它的单位自然就是 bytes。Kubernetes 支持你使用 Ei、Pi、Ti、Gi、Mi、Ki（或者 E、P、T、G、M、K）的方式来作为 bytes 的值。   
比如，在我们的例子里，Memory requests 的值就是 64MiB (2 的 26 次方 bytes) 。这里要注意区分 MiB（mebibyte）和 MB（megabyte）的区别。  
### limit和request
Kubernetes 里 Pod 的 CPU 和内存资源，实际上还要分为 limits 和 requests 两种情况
```
spec.containers[].resources.limits.cpu
spec.containers[].resources.limits.memory
spec.containers[].resources.requests.cpu
spec.containers[].resources.requests.memory
```
当你指定了 requests.cpu=250m 之后，相当于将 Cgroups 的 cpu.shares 的值设置为 (250/1000)*1024。   
而当你没有设置 requests.cpu 的时候，cpu.shares 默认则是 1024。这样，Kubernetes 就通过 cpu.shares 完成了对 CPU 时间的按比例分配。   
而如果你指定了 limits.cpu=500m 之后，则相当于将 Cgroups 的 cpu.cfs_quota_us 的值设置为 (500/1000)*100ms，而 cpu.cfs_period_us 的值始终是 100ms。   
这样，Kubernetes 就为你设置了这个容器只能用到 CPU 的 50%。而对于内存来说，当你指定了 limits.memory=128Mi 之后，相当于将 Cgroups 的 memory.limit_in_bytes 设置为 128 * 1024 * 1024。   
而需要注意的是，在调度的时候，调度器只会使用 requests.memory=64Mi 来进行判断。   
### 简单总结
Borg原则：   
容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。   
在作业被提交后，会主动减小它的资源限额配置，以便容纳更多的作业、提升资源利用率。而当作业资源使用量增加到一定阈值时，Borg 会通过“快速恢复”过程，还原作业原始的资源限额，防止出现异常情况。  
Kubernetes 的 requests+limits 的做法是：用户在提交 Pod 时，可以声明一个相对较小的 requests 值供调度器使用，而 Kubernetes 真正设置给容器 Cgroups 的，则是相对较大的 limits 值。

## QOS模型
### Guarantted
当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 Guaranteed 类别，如下所示：   
```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-ctr
    image: nginx
    resources:
      limits:
        memory: "200Mi"
        cpu: "700m"
      requests:
        memory: "200Mi"
        cpu: "700m"
```
当这个 Pod 创建之后，它的 qosClass 字段就会被 Kubernetes 自动设置为 Guaranteed。    
### Burstable
而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别。比如下面这个例子：
```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-2
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-2-ctr
    image: nginx
    resources:
      limits
        memory: "200Mi"
      requests:
        memory: "100Mi"
```
### BestEffort
而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort。比如下面这个例子：
```
apiVersion: v1
kind: Pod
metadata:
  name: qos-demo-3
  namespace: qos-example
spec:
  containers:
  - name: qos-demo-3-ctr
    image: nginx
```
### 总结
实际上，QoS 划分的主要应用场景，是当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时需要用到的。    
当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction。  
比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等。  
而当 Eviction 发生的时候，kubelet 具体会挑选哪些 Pod 进行删除操作，就需要参考这些 Pod 的 QoS 类别了。   
```
首当其冲的，自然是 BestEffort 类别的 Pod。    
其次，是属于 Burstable 类别、并且发生“饥饿”的资源使用量已经超出了 requests 的 Pod。    
最后，才是 Guaranteed 类别。
```
## 默认调度器
在 Kubernetes 项目中，默认调度器的主要职责，就是为一个新创建出来的 Pod，寻找一个最合适的节点（Node）。    
而这里“最合适”的含义，包括两层：   
1.从集群所有的节点中，根据调度算法挑选出所有可以运行该 Pod 的节点；    
2.从第一步的结果中，再根据调度算法挑选一个最符合条件的节点作为最终结果。   
所以在具体的调度流程中，默认调度器会首先调用一组叫作 Predicate 的调度算法，来检查每个 Node。然后，再调用一组叫作 Priority 的调度算法，来给上一步得到的结果里的每个 Node 打分。最终的调度结果，就是得分最高的那个 Node。
### 调度流程
#### 循环1
其中，第一个控制循环，我们可以称之为 Informer Path。它的主要目的，是启动一系列 Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。   
比如，当一个待调度 Pod（即：它的 nodeName 字段是空的）被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。   
在默认情况下，Kubernetes 的调度队列是一个 PriorityQueue（优先级队列），并且当某些集群信息发生变化的时候，调度器还会对调度队列里的内容进行一些特殊操作。此外，Kubernetes 的默认调度器还通过缓存以便提高 Predicate 和 Priority 调度算法的执行效率。  、
#### 循环2
而第二个控制循环，是调度器负责 Pod 调度的主循环，我们可以称之为 Scheduling Path。Scheduling Path 的主要逻辑，就是不断地从调度队列里出队一个 Pod。   
流程分为预选Predicates和优选Priorities。  
然后，调用 Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。当然，Predicates 算法需要的 Node 信息，都是从 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段之一。   
接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，分数从 0 到 10。得分最高的 Node，就会作为这次调度的结果。   

### 调度算法示意
图为官方给出的调度算法示意图  
```
For given pod:

    +---------------------------------------------+
    |               Schedulable nodes:            |
    |                                             |
    | +--------+    +--------+      +--------+    |
    | | node 1 |    | node 2 |      | node 3 |    |
    | +--------+    +--------+      +--------+    |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Pred. filters: node 3 doesn't have enough resource

    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+
    |             remaining nodes:                |
    |   +--------+                 +--------+     |
    |   | node 1 |                 | node 2 |     |
    |   +--------+                 +--------+     |
    |                                             |
    +-------------------+-------------------------+
                        |
                        |
                        v
    +-------------------+-------------------------+

    Priority function:    node 1: p=2
                          node 2: p=5

    +-------------------+-------------------------+
                        |
                        |
                        v
            select max{node priority} = node 2
```
具体步骤如下：  
1.通过一系列的“predicates”过滤掉不能运行pod的node，比如一个pod需要500M的内存，有些节点剩余内存只有100M了，就会被剔除；  
2.通过一系列的“priority functions”给剩下的node打分，寻找能够运行pod的若干node中最合适的一个node；  
3.得分最高的一个node，也就是被“priority functions”选中的node胜出了，获得了跑对应pod的资格。 
### 补充信息
#### pod的调度流程
pod的调度是串行的，而非并发的，一个pod的ScheduleOne方法执行完成后，马上会执行另一个pod的ScheduleOne方法。这是因为如果并发的去分配pod，可能一开始认为node是空闲的，但是分配完成之后发现其他pod可能已经在同一时间绑定了该node，造成错误，但是这个过程的最后一步应该是发送pod的信息到apiserver，而不是等待pod启动，否则效率就太低了。
#### 通过代码看看预选和优选
##### 预选
预选的方法是：  
`func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error)`  
入参是Pod和NodeList,返回则是适合的NodeList和预选失败的Node的集合。   
##### 优选
优选的方法是：   
`priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)`   
入参是pod和一系列优选相关的参数，返回是一个优选列表。   
该返回结果记录了所有Node和Node对应的分值：
```
type HostPriority struct {
    // Name of the host
    Host string
    // Score associated with the host
    Score int
}
// HostPriorityList declares a []HostPriority type.
type HostPriorityList []HostPriority
```
可以看出，PrioritizeNodes()方法的作用是计算前面的预选过程筛选出来的nodes各自的Score，那么接下来的方法就可以根据分数选出最优的Node。
#### 预选和优选的具体步骤
##### 预选
该函数的定义为   
```
func podFitsOnNode(
    pod *v1.Pod,
    meta algorithm.PredicateMetadata,
    info *schedulercache.NodeInfo,
    predicateFuncs map[string]algorithm.FitPredicate,
    nodeCache *equivalence.NodeCache,
    queue internalqueue.SchedulingQueue,
    alwaysCheckAllPredicates bool,
    equivClass *equivalence.Class,
) (bool, []algorithm.PredicateFailureReason, error) {
    podsAdded := false
    for i := 0; i < 2; i++ {
        metaToUse := meta
        nodeInfoToUse := info
        if i == 0 {
            podsAdded, metaToUse, nodeInfoToUse = addNominatedPods(pod, meta, info, queue)
        } else if !podsAdded || len(failedPredicates) != 0 {
            break
        }
        eCacheAvailable = equivClass != nil && nodeCache != nil && !podsAdded
	// predicates.Ordering()得到的是一个[]string，predicate名字集合
	for predicateID, predicateKey := range predicates.Ordering() {
	    var (
		fit     bool
		reasons []algorithm.PredicateFailureReason
		err     error
	    )
	    if predicate, exist := predicateFuncs[predicateKey]; exist {
		if eCacheAvailable {
		    fit, reasons, err = nodeCache.RunPredicate(predicate, predicateKey, predicateID, pod, metaToUse, nodeInfoToUse, equivClass)
		} else {
		    // 真正调用predicate函数了
		    fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)
		}
		if err != nil {
		    return false, []algorithm.PredicateFailureReason{}, err
		}
		if !fit {
		    // ……
		}
	    }
	}
    }
    return len(failedPredicates) == 0, failedPredicates, nil
}
```
该方法的步骤是：
1.遍历已经注册好的预选策略predicates.Ordering()，按顺序执行对应的策略函数   
2.遍历执行每个策略函数，并返回是否合适，预选失败的原因和错误   
3.如果预选函数执行失败，则加入预选失败的数组中，直接返回，后面的预选函数不会再执行   
4.如果该 node 上存在 高优先级 pod 则执行两次预选函数   
   
对于第一个for循环，翻译官方的注释：
   
出于某些原因考虑我们需要运行两次predicate. 如果node上有更高或者相同优先级的“指定pods”（这里的“指定pods”指的是通过schedule计算后指定要跑在一个node上但是还未真正运行到那个node上的pods），我们将这些pods加入到meta和nodeInfo后执行一次计算过程。
如果这个过程所有的predicates都成功了，我们再假设这些“指定pods”不会跑到node上再运行一次。第二次计算是必须的，因为有一些predicates比如pod亲和性，也许在“指定pods”没有成功跑到node的情况下会不满足。
如果没有“指定pods”或者第一次计算过程失败了，那么第二次计算不会进行。
我们在第一次调度的时候只考虑相等或者更高优先级的pods，因为这些pod是当前pod必须“臣服”的，也就是说不能够从这些pod中抢到资源，这些pod不会被当前pod“抢占”；这样当前pod也就能够安心从低优先级的pod手里抢资源了。
新pod在上述2种情况下都可调度基于一个保守的假设：资源和pod反亲和性等的predicate在“指定pods”被处理为Running时更容易失败；pod亲和性在“指定pods”被处理为Not Running时更加容易失败。
我们不能假设“指定pods”是Running的因为它们当前还没有运行，而且事实上，它们确实有可能最终又被调度到其他node上了。
   
对于第二个for循环，有两个要点。第一个是predicates.Ordering()函数，该函数是根据一个predicate的顺序进行预选，官方给出了一个顺序的表格，可以看到预选是进行一系列有顺序的对node和pod的判定。   

|Position                  | Predicate                        | comments (note, justification...)              |
 ----------------- | ---------------------------- | ------------------
| 1 | `CheckNodeConditionPredicate`  | we really don’t want to check predicates against unschedulable nodes. |
| 2           | `PodFitsHost`            | we check the pod.spec.nodeName. |
| 3           | `PodFitsHostPorts` | we check ports asked on the spec. |
| 4 | `PodMatchNodeSelector`            | check node label after narrowing search. |
| 5           | `PodFitsResources `            | this one comes here since it’s not restrictive enough as we do not try to match values but ranges. |
| 6           | `NoDiskConflict` | Following the resource predicate, we check disk |
| 7 | `PodToleratesNodeTaints '`            | check toleration here, as node might have toleration |
| 8          | `PodToleratesNodeNoExecuteTaints`            | check toleration here, as node might have toleration |
| 9           | `CheckNodeLabelPresence ` | labels are easy to check, so this one goes before |
| 10 | `checkServiceAffinity `            | - |
| 11           | `MaxPDVolumeCountPredicate `            | - |
| 12           | `VolumeNodePredicate ` | - |
| 13 | `VolumeZonePredicate `            | - |
| 14           | `CheckNodeMemoryPressurePredicate`            | doesn’t happen often |
| 15           | `CheckNodeDiskPressurePredicate` | doesn’t happen often |
| 16 | `InterPodAffinityMatches`            | Most expensive predicate to compute |

第二个就是`fit, reasons, err = predicate(pod, metaToUse, nodeInfoToUse)`这个调用，是根据上面给出的顺序调用具体的predicate函数。以`NoDiskConflict`为例：
```
func NoDiskConflict(pod *v1.Pod, meta algorithm.PredicateMetadata, nodeInfo *schedulercache.NodeInfo) (bool, []algorithm.PredicateFailureReason, error) {
    for _, v := range pod.Spec.Volumes {
        for _, ev := range nodeInfo.Pods() {
            if isVolumeConflict(v, ev) {
                return false, []algorithm.PredicateFailureReason{ErrDiskConflict}, nil
            }
        }
    }
    return true, nil, nil
}
```
这个函数遍历pod的Volumes，然后对于pod的每一个Volume，遍历node上的每个pod，看是否和当前podVolume冲突。如果不fit就返回false加原因；如果fit就返回true，预选函数都是这样类似的实现。

##### 优选

priorities 调度算法是在 pridicates 算法后执行的，主要功能是对已经过滤出的 nodes 进行打分并选出最佳的一个 node。
默认的调度算法是：
```
func defaultPriorities() sets.String {
    return sets.NewString(
        priorities.SelectorSpreadPriority,
        priorities.InterPodAffinityPriority,
        priorities.LeastRequestedPriority,
        priorities.BalancedResourceAllocation,
        priorities.NodePreferAvoidPodsPriority,
        priorities.NodeAffinityPriority,
        priorities.TaintTolerationPriority,
        priorities.ImageLocalityPriority,
    )
}

```

|priorities 算法|	说明|
|----------------- | ------------------|
|SelectorSpreadPriority|	按 service，rs，statefulset 归属计算 Node 上分布最少的同类 Pod数量，数量越少得分越高，默认权重为1|
|InterPodAffinityPriority|	pod 亲和性选择策略，默认权重为1|
|LeastRequestedPriority	|选择空闲资源（CPU 和 Memory）最多的节点，默认权重为1，其计算方式为：score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2|
|BalancedResourceAllocation|	CPU、Memory 以及 Volume 资源分配最均衡的节点，默认权重为1，其计算方式为：score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10|
|NodePreferAvoidPodsPriority|	判断 node annotation 是否有scheduler.alpha.kubernetes.io/preferAvoidPods 标签，类似于 taints 机制，过滤标签中定义类型的 pod，默认权重为10000|
|NodeAffinityPriority	|节点亲和性选择策略，默认权重为1|
|TaintTolerationPriority|	Pod 是否容忍节点上的 Taint，优先调度到标记了 Taint 的节点，默认权重为1|
|ImageLocalityPriority	|待调度 Pod 需要使用的镜像是否存在于该节点，默认权重为1|
执行 priorities 调度算法的逻辑是在 PrioritizeNodes()函数中，其目的是执行每个 priority 函数为 node 打分，分数为 0-10，其功能主要有：   
   
PrioritizeNodes() 通过并行运行各个优先级函数来对节点进行打分   
每个优先级函数会给节点打分，打分范围为 0-10 分，0 表示优先级最低的节点，10表示优先级最高的节点   
每个优先级函数有各自的权重   
优先级函数返回的节点分数乘以权重以获得加权分数   
最后计算所有节点的总加权分数  
