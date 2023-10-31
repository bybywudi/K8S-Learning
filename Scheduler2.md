作为一个容器集群编排与管理项目，Kubernetes 为用户提供的基础设施能力，不仅包括了应用定义和描述的部分，还包括了对应用的资源管理和调度的处理。  
简单的说，调度就是K8S的一系列逻辑，来判断某一个容器应该落在哪个Node上，应该给他多少CPU和内存资源等等。想要解密调度，那么让我们先从被调度的对象Pod的资源模型开始看起。

## 一、资源模型和资源管理

在 Kubernetes 里，Pod 是最小的原子调度单位。这也就意味着，所有跟调度和资源管理相关的属性都应该是属于 Pod 对象的字段。而这其中最重要的部分，就是 Pod 的 CPU 和内存配置

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

图1 Pod 的资源配置

在 Kubernetes 中，像 CPU 这样的资源被称作“可压缩资源”（compressible resources）。它的典型特点是，当可压缩资源不足时，Pod 只会“饥饿”，但不会退出。

而像内存这样的资源，则被称作“不可压缩资源（incompressible resources）。当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。

而由于 Pod 可以由多个 Container 组成，所以 CPU 和内存资源的限额，是要配置在每个 Container 的定义上的。一个Pod的整体资源配置，就是这些Container的配置值的累加和。

### 1.1 limit和request

如图1所示Kubernetes 里 Pod 的 CPU 和内存资源，实际上还要分为 limits 和 requests 两种情况。request是k8s必须要保证的启动资源，limit是将来容器运行时使用的资源上限。

在调度的时候，kube-scheduler 只会按照 requests 的值进行计算。而在真正设置 Cgroups 限制的时候，kubelet 则会按照 limits 的值来进行设置。

Kubernetes 这种对 CPU 和内存资源限额的设计，实际上参考了 Borg 论文中对“动态资源边界”的定义，即：容器化作业在提交时所设置的资源边界，并不一定是调度系统所必须严格遵守的，这是因为在实际场景中，大多数作业使用到的资源其实远小于它所请求的资源限额。

### 1.2 QOS模型

不同的requests和limits设置，会将Pod划分成不同的级别，即QoS模型，当宿主机资源紧张的时候，kubelet 对 Pod 进行 Eviction（即资源回收）时，根据不同的QoS等级来决定回收Pod的先后顺序

QoS分为以下3种：

1.Guarantted

当 Pod 里的每一个 Container 都同时设置了 requests 和 limits，并且 requests 和 limits 值相等的时候，这个 Pod 就属于 Guaranteed 类别，当这个 Pod 创建之后，它的 qosClass 字段就会被 Kubernetes 自动设置为 Guaranteed。需要注意的是，当 Pod 仅设置了 limits 没有设置 requests 的时候，Kubernetes 会自动为它设置与 limits 相同的 requests 值，所以，这也属于 Guaranteed 情况。

2.Burstable

而当 Pod 不满足 Guaranteed 的条件，但至少有一个 Container 设置了 requests。那么这个 Pod 就会被划分到 Burstable 类别。

3.BestEffort

而如果一个 Pod 既没有设置 requests，也没有设置 limits，那么它的 QoS 类别就是 BestEffort。

具体地说，当 Kubernetes 所管理的宿主机上不可压缩资源短缺时，就有可能触发 Eviction。比如，可用内存（memory.available）、可用的宿主机磁盘空间（nodefs.available），以及容器运行时镜像存储空间（imagefs.available）等等达到阈值。当然，这些阈值都是可配置的。当Eviction发生时，首先删除的是BestEffort类别的Pod；其次，是属于 Burstable 类别、并且发生“饥饿”的资源使用量已经超出了 requests 的 Pod；最后，才是 Guaranteed 类别。并且，Kubernetes 会保证只有当 Guaranteed 类别的 Pod 的资源使用量超过了其 limits 的限制，或者宿主机本身正处于 Memory Pressure 状态时，Guaranteed 的 Pod 才可能被选中进行 Eviction 操作。

当然，同一QoS等级的Pod来说，还要再对比Pod的优先级。

既然已经了解了我们要调度的对象Pod, 那么接下来让我具体看看，Kubernetes是如何对他进行调度的。

## 二、默认调度器

默认调度器的主要职责，就是为一个新创建出来的 Pod，寻找一个最合适的节点（Node）。  
而这里“最合适”的含义，包括两层：  
1.从集群所有的节点中，根据调度算法挑选出所有可以运行该 Pod 的节点；  
2.从第一步的结果中，再根据调度算法挑选一个最符合条件的节点作为最终结果。  
所以在具体的调度流程中，默认调度器会首先调用一组叫作 Predicate 的调度算法，来检查每个 Node。然后，再调用一组叫作 Priority 的调度算法，来给上一步得到的结果里的每个 Node 打分。最终的调度结果，就是得分最高的那个 Node。

上述2个步骤对每个pod的调度是串行的，而非并发的。这是因为如果并发的去分配pod，可能一开始认为node是空闲的，但是分配完成之后发现其他pod可能已经在同一时间绑定了该node，造成错误。

### 2.1 调度流程

![](https://github.com/bybywudi/K8S-Learning/blob/main/k8s1.png)

图2. k8s调度流程示意图

#### 2.1.1 循环1

其中，第一个控制循环，我们可以称之为 Informer Path。它的主要目的，是启动一系列 Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。  
比如，当一个待调度 Pod（即：它的 nodeName 字段是空的）被创建出来之后，调度器就会通过 Pod Informer 的 Handler，将这个待调度 Pod 添加进调度队列。  
在默认情况下，Kubernetes 的调度队列是一个 PriorityQueue（优先级队列），并且当某些集群信息发生变化的时候，调度器还会对调度队列里的内容进行一些特殊操作。此外，Kubernetes 的默认调度器还通过缓存以便提高 Predicate 和 Priority 调度算法的执行效率。

#### 2.1.2 循环2

而第二个控制循环，是调度器负责 Pod 调度的主循环，我们可以称之为 Scheduling Path。Scheduling Path 的主要逻辑，就是不断地从调度队列里出队一个 Pod。  
然后，调用 Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。当然，Predicates 算法需要的 Node 信息，都是从 Scheduler Cache 里直接拿到的，这是调度器保证算法执行效率的主要手段之一。  
接下来，调度器就会再调用 Priorities 算法为上述列表里的 Node 打分，分数从 0 到 10。得分最高的 Node，就会作为这次调度的结果。

示例

图为官方给出的调度算法示意图

具体步骤如下：  
1.通过一系列的“predicates”过滤掉不能运行pod的node，比如一个pod需要500M的内存，有些节点剩余内存只有100M了，就会被剔除；  
2.通过一系列的“priority functions”给剩下的node打分，寻找能够运行pod的若干node中最合适的一个node；  
3.得分最高的一个node，也就是被“priority functions”选中的node胜出了，获得了跑对应pod的资格。

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

图3. 调度算法示意图

#### 2.1.3 调度结果的更新

调度算法执行完成后，调度器就需要将 Pod 对象的 nodeName 字段的值，修改为上述 Node 的名字。这个步骤在 Kubernetes 里面被称作 Bind。

但是，为了不在关键调度路径里远程访问 APIServer，Kubernetes 的默认调度器在 Bind 阶段，只会更新 Scheduler Cache 里的 Pod 和 Node 的信息。这种基于“乐观”假设的 API 对象更新方式，在 Kubernetes 里被称作 Assume。Assume 之后，调度器才会创建一个 Goroutine 来异步地向 APIServer 发起更新 Pod 的请求，来真正完成 Bind 操作。如果这次异步的 Bind 过程失败了，其实也没有太大关系，等 Scheduler Cache 同步之后一切就会恢复正常。

### 2.2 调度策略

#### 2.2.1 Predicates预选

**Predicates 在调度过程中的作用，可以理解为 Filter**，即：它按照调度策略，从当前集群的所有节点中，“过滤”出一系列符合条件的节点。这些节点，都是可以运行待调度 Pod 的宿主机。而在 Kubernetes 中，默认的调度策略有如下四种。

1.**GeneralPredicates**

负责的是最基础的调度策略。

比如，PodFitsResources 计算的就是宿主机的 CPU 和内存资源等是否够用。当然，我在前面已经提到过，PodFitsResources 检查的只是 Pod 的 requests 字段。而 PodFitsHost 检查的是，宿主机的名字是否跟 Pod 的 spec.nodeName 一致。PodFitsHostPorts 检查的是，Pod 申请的宿主机端口（spec.nodePort）是不是跟已经被使用的端口有冲突。PodMatchNodeSelector 检查的是，Pod 的 nodeSelector 或者 nodeAffinity 指定的节点，是否与待考察节点匹配，等等。

2.**与 Volume 相关的过滤规则**

这一组过滤规则，负责的是跟容器持久化 Volume 相关的调度策略。

其中，NoDiskConflict 检查的条件，是多个 Pod 声明挂载的持久化 Volume 是否有冲突。比如，AWS EBS 类型的 Volume，是不允许被两个 Pod 同时使用的。而 MaxPDVolumeCountPredicate 检查的条件，则是一个节点上某种类型的持久化 Volume 是不是已经超过了一定数目。等等。

3.**宿主机相关的过滤规则**

主要考察待调度 Pod 是否满足 Node 本身的某些条件。

比如，PodToleratesNodeTaints，负责检查的就是 Node 的“污点”机制（PS，可以简单理解为给Pod和Node提前打上了一些标签）。只有当 Pod 的 Toleration 字段与 Node 的 Taint 字段能够匹配的时候，这个 Pod 才能被调度到该节点上。NodeMemoryPressurePredicate，检查的是当前节点的内存是不是已经不够充足，如果是的话，那么待调度 Pod 就不能被调度到该节点上。等等。

4.**Pod 相关的过滤规则**

这一组规则，跟 GeneralPredicates 大多数是重合的。而比较特殊的，是 PodAffinityPredicate。这个规则的作用，是检查待调度 Pod 与 Node 上的已有 Pod 之间的亲密（affinity）和反亲密（anti-affinity）关系。（PS，与“污点”机制类似，可以简单理解为给Pod与Pod之间的标签是否匹配）

#### 2.2.2 Priorities优选

在 Predicates 阶段完成了节点的“过滤”之后，Priorities 阶段的工作就是为这些节点打分。这里打分的范围是 0-10 分，得分最高的节点就是最后被 Pod 绑定的最佳节点。

最常用到的一个打分规则，是 LeastRequestedPriority，这个算法实际上就是在选择空闲资源（CPU 和 Memory）最多的宿主机。计算方法：

  score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2

还有 BalancedResourceAllocation。它的计算公式如下所示：

  score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10

其中，每种资源的 Fraction 的定义是 ：Pod 请求的资源 / 节点上的可用资源。而 variance 算法的作用，则是计算每两种资源 Fraction 之间的“距离”。而最后选择的，则是资源 Fraction 差距最小的节点。所以说，BalancedResourceAllocation 选择的，其实是调度完成后，所有节点里各种资源分配最均衡的那个节点，从而避免一个节点上 CPU 被大量分配、而 Memory 大量剩余的情况。

此外，还有 NodeAffinityPriority、TaintTolerationPriority 和 InterPodAffinityPriority 这三种 Priority。顾名思义，它们与前面的 PodMatchNodeSelector、PodToleratesNodeTaints 和 PodAffinityPredicate 这三个 Predicate 的含义和计算方法是类似的。但是作为 Priority，一个 Node 满足上述规则的字段数目越多，它的得分就会越高。

以上，就是 Kubernetes 调度器的 Predicates 和 Priorities 里默认调度规则的主要工作原理了。

### 2.3通过代码看看预选和优选

预选的方法是：`func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error)`入参是Pod和NodeList,返回则是适合的NodeList和预选失败的Node的集合。

优选的方法是：`priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)`入参是pod和一系列优选相关的参数，返回是一个优选列表。该返回结果记录了所有Node和Node对应的分值：

  type HostPriority struct {

    // Name of the host

    Host string

    // Score associated with the host

    Score int

  }

  // HostPriorityList declares a []HostPriority type.

  type HostPriorityList []HostPriority

#### 预选和优选的具体步骤

预选函数的定义为

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

该方法的步骤是：

1.遍历已经注册好的预选策略predicates.Ordering()，按顺序执行对应的策略函数  
2.遍历执行每个策略函数，并返回是否合适，预选失败的原因和错误  
3.如果预选函数执行失败，则加入预选失败的数组中，直接返回，后面的预选函数不会再执行  
4.如果该 node 上存在 高优先级 pod 则执行两次预选函数（PS：涉及到优先级和抢占机制，解决的是 Pod 调度失败时该怎么办的问题，本文暂不涉及）

官方给出了一个predicate的顺序的表格，可以看到预选是进行一系列有顺序的对node和pod的判定。

|Position|Predicate|comments (note, justification...)|
|-|-|-|
|1|`CheckNodeConditionPredicate`|we really don’t want to check predicates against unschedulable nodes.|
|2|`PodFitsHost`|we check the pod.spec.nodeName.|
|3|`PodFitsHostPorts`|we check ports asked on the spec.|
|4|`PodMatchNodeSelector`|check node label after narrowing search.|
|5|`PodFitsResources`|this one comes here since it’s not restrictive enough as we do not try to match values but ranges.|
|6|`NoDiskConflict`|Following the resource predicate, we check disk|
|7|`PodToleratesNodeTaints '`|check toleration here, as node might have toleration|
|8|`PodToleratesNodeNoExecuteTaints`|check toleration here, as node might have toleration|
|9|`CheckNodeLabelPresence`|labels are easy to check, so this one goes before|
|10|`checkServiceAffinity`|-|
|11|`MaxPDVolumeCountPredicate`|-|
|12|`VolumeNodePredicate`|-|
|13|`VolumeZonePredicate`|-|
|14|`CheckNodeMemoryPressurePredicate`|doesn’t happen often|
|15|`CheckNodeDiskPressurePredicate`|doesn’t happen often|
|16|`InterPodAffinityMatches`|Most expensive predicate to compute|


以`NoDiskConflict`为例：

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

这个函数遍历pod的Volumes，然后对于pod的每一个Volume，遍历node上的每个pod，看是否和当前podVolume冲突。如果不fit就返回false加原因；如果fit就返回true，预选函数都是这样类似的实现。

优选priorities 调度算法是在 pridicates 算法后执行的，主要功能是对已经过滤出的 nodes 进行打分并选出最佳的一个 node。 默认的调度算法是：

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

|priorities 算法|说明|
|-|-|
|SelectorSpreadPriority|按 service，rs，statefulset 归属计算 Node 上分布最少的同类 Pod数量，数量越少得分越高，默认权重为1|
|InterPodAffinityPriority|pod 亲和性选择策略，默认权重为1|
|LeastRequestedPriority|选择空闲资源（CPU 和 Memory）最多的节点，默认权重为1，其计算方式为：score = (cpu((capacity-sum(requested))10/capacity) + memory((capacity-sum(requested))10/capacity))/2|
|BalancedResourceAllocation|CPU、Memory 以及 Volume 资源分配最均衡的节点，默认权重为1，其计算方式为：score = 10 - variance(cpuFraction,memoryFraction,volumeFraction)*10|
|NodePreferAvoidPodsPriority|判断 node annotation [是否有scheduler.alpha.kubernetes.io/preferAvoidPods](http://xn--scheduler-k28o800n1ec.alpha.kubernetes.io/preferAvoidPods) 标签，类似于 taints 机制，过滤标签中定义类型的 pod，默认权重为10000|
|NodeAffinityPriority|节点亲和性选择策略，默认权重为1|
|TaintTolerationPriority|Pod 是否容忍节点上的 Taint，优先调度到标记了 Taint 的节点，默认权重为1|
|ImageLocalityPriority|待调度 Pod 需要使用的镜像是否存在于该节点，默认权重为1|
|执行 priorities 调度算法的逻辑是在 PrioritizeNodes()函数中，其目的是执行每个 priority 函数为 node 打分，分数为 0-10，其功能主要有：||


PrioritizeNodes() 通过并行运行各个优先级函数来对节点进行打分  
每个优先级函数会给节点打分，打分范围为 0-10 分，0 表示优先级最低的节点，10表示优先级最高的节点  
每个优先级函数有各自的权重  
优先级函数返回的节点分数乘以权重以获得加权分数  
最后计算所有节点的总加权分数
