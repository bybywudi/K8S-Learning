# Overview
官方对Scheduler的介绍：[scheduler.md](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-scheduling/scheduler.md)  
   
  The Kubernetes scheduler runs as a process alongside the other master components such as the API server. Its interface to the API server is to watch for Pods with an empty PodSpec.NodeName, and for each Pod, it posts a binding indicating where the Pod should be scheduled.   
   
  也就是说Scheduler是一个跑在其他组件边上的独立程序，对接Apiserver寻找PodSpec.NodeName为空的Pod，然后用post的方式发送一个api调用，指定这些pod应该跑在哪个node上。scheduler是相对独立的一个组件，主动访问api server，寻找等待调度的pod，然后通过一系列调度算法寻找哪个node适合跑这个pod，然后将这个pod和node的绑定关系发给api server，从而完成了调度的过程。  
   
# 源码分类
Scheduler的源码可以分为3层：     

[cmd/kube-scheduler/scheduler.go](https://github.com/kubernetes/kubernetes/blob/master/cmd/kube-scheduler/scheduler.go) main函数入口位置，在scheduler过程开始被调用前的一系列初始化工作。  

[pkg/scheduler/scheduler.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/scheduler.go) 调度框架的整体逻辑,其中包括了通过Config创建Scheduler对象的整体逻辑。   

[pkg/scheduler/core/generic_scheduler.go](https://github.com/kubernetes/kubernetes/blob/master/pkg/scheduler/core/generic_scheduler.go) 根据node的状态为node分配合适的pod分配算法,Schedule()函数用于分配Pod。  

# 调度算法示意
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
   
Predicates算法代码：[pkg/scheduler/algorithm/predicates/predicates.go](https://github.com/kubernetes/kubernetes/blob/release-1.13/pkg/scheduler/algorithm/predicates/predicates.go)   
Priority算法代码：[pkg/scheduler/algorithm/priorities](https://github.com/kubernetes/kubernetes/tree/release-1.13/pkg/scheduler/algorithm/priorities).   

# 调度策略配置

默认调度策略是通过defaultPredicates() 和 defaultPriorities()函数定义的.代码在[default.go](https://github.com/kubernetes/kubernetes/blob/release-1.13/pkg/scheduler/algorithmprovider/defaults/defaults.go),可以通过配置文件的方式或者修改.

# 代码分析

## main函数
main函数的代码地址是[cmd/kube-scheduler/scheduler.go](https://github.com/kubernetes/kubernetes/blob/release-1.13/cmd/kube-scheduler/scheduler.go)的第34行。    
```
func main() {
    command := app.NewSchedulerCommand()
    if err := command.Execute(); err != nil {
        fmt.Fprintf(os.Stderr, "%v\n", err)
        os.Exit(1)
    }
}
```
kube-scheduler在运行的时候调用了command.Execute()函数的Run方法，这里也是Scheduler主方法的入口，这个方法的代码如下，具体位置在 pkg/scheduler/scheduler.go:276
```
// Run executes the scheduler based on the given configuration. It only return on error or when stopCh is closed.
func Run(cc schedulerserverconfig.CompletedConfig, stopCh <-chan struct{}) error {
    // Create the scheduler.
    sched, err := scheduler.New(cc.Client,
        cc.InformerFactory.Core().V1().Nodes(),
        stopCh,
        scheduler.WithName(cc.ComponentConfig.SchedulerName))

    // Prepare a reusable runCommand function.
    run := func(ctx context.Context) {
        sched.Run()
        <-ctx.Done()
    }

    ctx, cancel := context.WithCancel(context.TODO()) 
    defer cancel()

    go func() {
        select {
        case <-stopCh:
            cancel()
        case <-ctx.Done():
        }
    }()

    // Leader election is disabled, so runCommand inline until done.
    run(ctx)
    return fmt.Errorf("finished without leader elect")
}
```
这个文件58行有这样的一段注释：
```
// Scheduler watches for new unscheduled pods. It attempts to find
// nodes that they fit on and writes bindings back to the api server.
type Scheduler struct {
    config *factory.Config
}
```
也就是说。Scheduler watch新创建的未被调度的pods，然后尝试寻找合适的node，回写一个绑定关系到api server.   
再看276行的Run方法的逻辑:
```
// Run begins watching and scheduling. It waits for cache to be synced, then starts a goroutine and returns immediately.
func (sched *Scheduler) Run() {
    if !sched.config.WaitForCacheSync() {
        return
    }
    go wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)
}
```
注释中提到这个函数开始watching和scheduling，并启动一个goroutine然后返回，然后开始等待接下来的事件，即`wait.Until()`函数。这里`wait.Until(sched.scheduleOne, 0, sched.config.StopEverything)`也就是说scheduler会一直去访问`sched.scheduleOne`这个函数，直到通道关闭再去访问`sched.config.StopEverything`。   
## pod的调度流程
通过上面的分析，`sched.scheduleOne`这个方法就是pod调度的关键方法，代码在[pkg/scheduler/scheduler.go](https://github.com/kubernetes/kubernetes/blob/release-1.13/pkg/scheduler/scheduler.go)的513行。   
可以查看该方法的注释:`scheduleOne does the entire scheduling workflow for a single pod. It is serialized on the scheduling algorithm’s host fitting.`**也就是说，pod的调度是串行的，而非并发的，一个pod的ScheduleOne方法执行完成后，马上会执行另一个pod的ScheduleOne方法。这是因为如果并发的去分配pod，可能一开始认为node是空闲的，但是分配完成之后发现其他pod可能已经在同一时间绑定了该node，造成错误，但是这个过程的最后一步应该是发送pod的信息到apiserver，而不是等待pod启动，否则效率就太低了。**    
ScheduleOne的主要逻辑如下：
```
func (sched *Scheduler) scheduleOne() {
    pod := sched.config.NextPod()
    suggestedHost, err := sched.schedule(pod)
    if err != nil {
        if fitError, ok := err.(*core.FitError); ok {
            preemptionStartTime := time.Now()
            sched.preempt(pod, fitError)
        }
        return
    }
    assumedPod := pod.DeepCopy()
    allBound, err := sched.assumeVolumes(assumedPod, suggestedHost)
    err = sched.assume(assumedPod, suggestedHost)
    go func() {
        err := sched.bind(assumedPod, &v1.Binding{
            ObjectMeta: metav1.ObjectMeta{Namespace: assumedPod.Namespace, Name: assumedPod.Name, UID: assumedPod.UID},
            Target: v1.ObjectReference{
                Kind: "Node",
                Name: suggestedHost,
            },
        })
    }()
}
```
这个流程图可以表示ScheduleOne的流程，`suggestedHost, err := sched.schedule(pod)`调用了pod schedule的核心方法：
![SchduleOne流程图](https://github.ibm.com/baiyng/Kubernetes-Core-Code-Learning/blob/master/pictures/podSchedule.png)   
其中将pod优选到node的核心算法就是`sched.schedule(pod)`这个方法，进入到该方法：
```
// schedule implements the scheduling algorithm and returns the suggested host.
func (sched *Scheduler) schedule(pod *v1.Pod) (string, error) {
    host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)
    if err != nil {
        pod = pod.DeepCopy()
        sched.config.Error(pod, err)
        sched.config.Recorder.Eventf(pod, v1.EventTypeWarning, "FailedScheduling", "%v", err)
        sched.config.PodConditionUpdater.Update(pod, &v1.PodCondition{
            Type:          v1.PodScheduled,
            Status:        v1.ConditionFalse,
            LastProbeTime: metav1.Now(),
            Reason:        v1.PodReasonUnschedulable,
            Message:       err.Error(),
        })
        return "", err
    }
    return host, err
}
```
核心是这一行：`host, err := sched.config.Algorithm.Schedule(pod, sched.config.NodeLister)`，该方法的参数是pod和NodeList，返回了一个host，作用是传入目标pod和所有可用的Node，来返回一个适合该pod的host。   
进入到[Schedule算法](https://github.com/kubernetes/kubernetes/blob/874f0559d9b358f87959ec0bb7645d9cb3d5f7ba/pkg/scheduler/algorithm/scheduler_interface.go)的78行，可以看到这个方法是一个接口：
```
// ScheduleAlgorithm is an interface implemented by things that know how to schedule pods
// onto machines.
type ScheduleAlgorithm interface {
	Schedule(*v1.Pod, NodeLister) (selectedMachine string, err error)
	// Preempt receives scheduling errors for a pod and tries to create room for
	// the pod by preempting lower priority pods if possible.
	// It returns the node where preemption happened, a list of preempted pods, a
	// list of pods whose nominated node name should be removed, and error if any.
	Preempt(*v1.Pod, NodeLister, error) (selectedNode *v1.Node, preemptedPods []*v1.Pod, cleanupNominatedPods []*v1.Pod, err error)
	// Predicates() returns a pointer to a map of predicate functions. This is
	// exposed for testing.
	Predicates() map[string]FitPredicate
	// Prioritizers returns a slice of priority config. This is exposed for
	// testing.
	Prioritizers() []PriorityConfig
}
```
这个接口有4个方法，实现ScheduleAlgorithm接口的对象意味着知道如何调度pods到nodes上。默认的实现是pkg/scheduler/core/generic_scheduler.go:98 genericScheduler,这个接口有四个方法：   
1.Schedule() //给定pod和nodes，计算出一个适合跑pod的node并返回；   
2.Preempt() //抢占   
3.Predicates() //预选   
4.Prioritizers() //优选   
默认实用的genericScheduler方法如下： 
`func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error)`  
该方法尝试将指定的pod调度到给定的node列表中的一个，如果成功就返回这个node的名字。
## 一般调度过程
调度算法的核心在[pkg/scheduler/core/generic_scheduler.go:139](https://github.com/kubernetes/kubernetes/blob/874f0559d9b358f87959ec0bb7645d9cb3d5f7ba/pkg/scheduler/core/generic_scheduler.go), 将代码过程精简一下：
```
// Schedule tries to schedule the given pod to one of the nodes in the node list.
// If it succeeds, it will return the name of the node.
// If it fails, it will return a FitError error with reasons.
func (g *genericScheduler) Schedule(pod *v1.Pod, nodeLister algorithm.NodeLister) (string, error) {
    nodes, err := nodeLister.List()
    trace.Step("Computing predicates")
    filteredNodes, failedPredicateMap, err := g.findNodesThatFit(pod, nodes)
    trace.Step("Prioritizing")
    priorityList, err := PrioritizeNodes(pod, g.cachedNodeInfoMap, metaPrioritiesInterface, g.prioritizers, filteredNodes, g.extenders)
    trace.Step("Selecting host")
    return g.selectHost(priorityList)
}
```
可见该调度过程有三个主要步骤：   
“Computing predicates”：调用findNodesThatFit()方法；   
“Prioritizing”：调用PrioritizeNodes()方法；   
“Selecting host”：调用selectHost()方法。   

### 预选
预选的方法是：  
`func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error)`  
入参是Pod和NodeList,返回则是适合的NodeList和预选失败的Node的集合。   
   
### 优选
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
   
### Selecting host
`func (g *genericScheduler) selectHost(priorityList schedulerapi.HostPriorityList) (string, error)`   
这一步的入参就是上一步优选出的节点和对应的分数，从而选出最佳的node。

## 具体调度过程
### 预选
保留预选函数`findNodesThatFit()`的主要代码
```
func (g *genericScheduler) findNodesThatFit(pod *v1.Pod, nodes []*v1.Node) ([]*v1.Node, FailedPredicateMap, error) {
    checkNode := func(i int) {
        fits, failedPredicates, err := podFitsOnNode(
            //……
        )
        if fits {
            length := atomic.AddInt32(&filteredLen, 1)
            filtered[length-1] = g.cachedNodeInfoMap[nodeName].Node()
        } 
    }
    workqueue.ParallelizeUntil(ctx, 16, int(allNodes), checkNode)
    if len(filtered) > 0 && len(g.extenders) != 0 {
        for _, extender := range g.extenders {
            // Logic of extenders 
        }
    }
    return filtered, failedPredicateMap, nil
}
```
这里最重要的是调用过程`fits, failedPredicates, err := podFitsOnNode()`，
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

### 优选

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
精简后的代码：参考[pkg/scheduler/core/generic_scheduler.go:624](https://github.com/kubernetes/kubernetes/blob/874f0559d9b358f87959ec0bb7645d9cb3d5f7ba/pkg/scheduler/core/generic_scheduler.go)
```
func PrioritizeNodes(......) (schedulerapi.HostPriorityList, error) {
    // 1.检查是否有自定义配置
    if len(priorityConfigs) == 0 && len(extenders) == 0 {
        result := make(schedulerapi.HostPriorityList, 0, len(nodes))
        for i := range nodes {
            hostPriority, err := EqualPriorityMap(pod, meta, nodeNameToInfo[nodes[i].Name])
            if err != nil {
                return nil, err
            }
            result = append(result, hostPriority)
        }
        return result, nil
    }
    ......
    results := make([]schedulerapi.HostPriorityList, len(priorityConfigs), len(priorityConfigs))
    ......
    // 2.使用 workqueue 启动 16 个 goroutine 并发为 node 打分
    workqueue.ParallelizeUntil(context.TODO(), 16, len(nodes), func(index int) {
        nodeInfo := nodeNameToInfo[nodes[index].Name]
        for i := range priorityConfigs {
            if priorityConfigs[i].Function != nil {
                continue
            }
            var err error
            results[i][index], err = priorityConfigs[i].Map(pod, meta, nodeInfo)
            if err != nil {
                appendError(err)
                results[i][index].Host = nodes[index].Name
            }
        }
    })
    // 3.执行自定义配置
    for i := range priorityConfigs {
        ......
    }
    wg.Wait()
    if len(errs) != 0 {
        return schedulerapi.HostPriorityList{}, errors.NewAggregate(errs)
    }
    // 4.运行 Score plugins
    scoresMap, scoreStatus := framework.RunScorePlugins(pluginContext, pod, nodes)
    if !scoreStatus.IsSuccess() {
        return schedulerapi.HostPriorityList{}, scoreStatus.AsError()
    }
    result := make(schedulerapi.HostPriorityList, 0, len(nodes))
    // 5.为每个 node 汇总分数
    for i := range nodes {
        result = append(result, schedulerapi.HostPriority{Host: nodes[i].Name, Score: 0})
        for j := range priorityConfigs {
            result[i].Score += results[j][i].Score * priorityConfigs[j].Weight
        }
        for j := range scoresMap {
            result[i].Score += scoresMap[j][i].Score
        }
    }
    // 6.执行 extender 
    if len(extenders) != 0 && nodes != nil {
        ......
    }
    ......
    return result, nil
}
```
### 抢占调度

### 亲和性调度

# 总结
kube-scheduler就是一个独立运行的程序，负责和 API Server 交互，获取 PodSpec.NodeName 属性为空的 Pods 列表，然后一个个计算最适合 Pod 运行的 Node，最后将 Pod 绑定到对应的 Node 上。   
总体流程图如下，引用自网络：  
![](https://github.com/bybywudi/K8S-Learning/blob/main/k8s1.png)
