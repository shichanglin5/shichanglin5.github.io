# StatefulSet Reconcile

## 源码片段

### `StatefulSetController`
控制 sts 的更新，该结构体会包含 `control` 字段，类型是`StatefulSetControlInterface`用来抽象一些 sts 的管理逻辑，目前只有一个实现 `defaultStatefulSetControl`
核心处理逻辑源码：`pkg/controller/statefulset/stateful_set_control.go:542`

### 特性门控
```go
StatefulSetStartOrdinal featuregate.Feature = "StatefulSetStartOrdinal"
StatefulSetAutoDeletePVC featuregate.Feature = "StatefulSetAutoDeletePVC"
MaxUnavailableStatefulSet featuregate.Feature = "MaxUnavailableStatefulSet"
```

#### Pod状态判断
at `pkg/controller/statefulset/stateful_set_utils.go`
```Go
func isRunningAndReady(pod *v1.Pod) bool {
	return pod.Status.Phase == v1.PodRunning && podutil.IsPodReady(pod)
}

func isRunningAndAvailable(pod *v1.Pod, minReadySeconds int32) bool {
	return podutil.IsPodAvailable(pod, minReadySeconds, metav1.Now())
}

// isCreated returns true if pod has been created and is maintained by the API server
func isCreated(pod *v1.Pod) bool {
	return pod.Status.Phase != ""
}

// isPending returns true if pod has a Phase of PodPending
func isPending(pod *v1.Pod) bool {
	return pod.Status.Phase == v1.PodPending
}

// isFailed returns true if pod has a Phase of PodFailed
func isFailed(pod *v1.Pod) bool {
	return pod.Status.Phase == v1.PodFailed
}

// isSucceeded returns true if pod has a Phase of PodSucceeded
func isSucceeded(pod *v1.Pod) bool {
	return pod.Status.Phase == v1.PodSucceeded
}

// isTerminating returns true if pod's DeletionTimestamp has been set
func isTerminating(pod *v1.Pod) bool {
	return pod.DeletionTimestamp != nil
}

// isHealthy returns true if pod is running and ready and has not been terminated
func isHealthy(pod *v1.Pod) bool {
	return isRunningAndReady(pod) && !isTerminating(pod)
}
```
#### 更新策略
```Go
// StatefulSetUpdateStrategyType is a string enumeration type that enumerates
// all possible update strategies for the StatefulSet controller.
// +enum
type StatefulSetUpdateStrategyType string

const (
	// RollingUpdateStatefulSetStrategyType indicates that update will be
	// applied to all Pods in the StatefulSet with respect to the StatefulSet
	// ordering constraints. When a scale operation is performed with this
	// strategy, new Pods will be created from the specification version indicated
	// by the StatefulSet's updateRevision.
	RollingUpdateStatefulSetStrategyType StatefulSetUpdateStrategyType = "RollingUpdate"
	// OnDeleteStatefulSetStrategyType triggers the legacy behavior. Version
	// tracking and ordered rolling restarts are disabled. Pods are recreated
	// from the StatefulSetSpec when they are manually deleted. When a scale
	// operation is performed with this strategy,specification version indicated
	// by the StatefulSet's currentRevision.
	OnDeleteStatefulSetStrategyType StatefulSetUpdateStrategyType = "OnDelete"
)
```
#### sts 更新并发度控制
```Go
// StatefulSetControllerConfiguration contains elements describing StatefulSetController.
type StatefulSetControllerConfiguration struct {
	// concurrentStatefulSetSyncs is the number of statefulset objects that are
	// allowed to sync concurrently. Larger number = more responsive statefulsets,
	// but more CPU (and network) load.
	ConcurrentStatefulSetSyncs int32
}

func startStatefulSetController(ctx context.Context, controllerContext ControllerContext, controllerName string) (controller.Interface, bool, error) {
	go statefulset.NewStatefulSetController(
		ctx,
		controllerContext.InformerFactory.Core().V1().Pods(),
		controllerContext.InformerFactory.Apps().V1().StatefulSets(),
		controllerContext.InformerFactory.Core().V1().PersistentVolumeClaims(),
		controllerContext.InformerFactory.Apps().V1().ControllerRevisions(),
		controllerContext.ClientBuilder.ClientOrDie("statefulset-controller"),
	).Run(ctx, int(controllerContext.ComponentConfig.StatefulSetController.ConcurrentStatefulSetSyncs))
	return nil, true, nil
}

// Run runs the statefulset controller.
func (ssc *StatefulSetController) Run(ctx context.Context, workers int) {
	defer utilruntime.HandleCrash()

	// Start events processing pipeline.
	ssc.eventBroadcaster.StartStructuredLogging(3)
	ssc.eventBroadcaster.StartRecordingToSink(&v1core.EventSinkImpl{Interface: ssc.kubeClient.CoreV1().Events("")})
	defer ssc.eventBroadcaster.Shutdown()

	defer ssc.queue.ShutDown()

	logger := klog.FromContext(ctx)
	logger.Info("Starting stateful set controller")
	defer logger.Info("Shutting down statefulset controller")

	if !cache.WaitForNamedCacheSync("stateful set", ctx.Done(), ssc.podListerSynced, ssc.setListerSynced, ssc.pvcListerSynced, ssc.revListerSynced) {
		return
	}

	for i := 0; i < workers; i++ {
		go wait.UntilWithContext(ctx, ssc.worker, time.Second)
	}

	<-ctx.Done()
}
```

#### Pod更新逻辑
根据 replica 设置，确保 replica 个数的 pod 全部就绪后，再删除一些多余的 pod
- 如果pod状态是Failed或者Succeed，则删除然后创建新的Pod
-
```Go
func (ssc *defaultStatefulSetControl) processReplica(
	ctx context.Context,
	set *apps.StatefulSet,
	currentRevision *apps.ControllerRevision,
	updateRevision *apps.ControllerRevision,
	currentSet *apps.StatefulSet,
	updateSet *apps.StatefulSet,
	monotonic bool,
	replicas []*v1.Pod,
	i int) (bool, error) {
	logger := klog.FromContext(ctx)
	// Delete and recreate pods which finished running.
	//
	// Note that pods with phase Succeeded will also trigger this event. This is
	// because final pod phase of evicted or otherwise forcibly stopped pods
	// (e.g. terminated on node reboot) is determined by the exit code of the
	// container, not by the reason for pod termination. We should restart the pod
	// regardless of the exit code.
	if isFailed(replicas[i]) || isSucceeded(replicas[i]) {
		if isFailed(replicas[i]) {
			ssc.recorder.Eventf(set, v1.EventTypeWarning, "RecreatingFailedPod",
				"StatefulSet %s/%s is recreating failed Pod %s",
				set.Namespace,
				set.Name,
				replicas[i].Name)
		} else {
			ssc.recorder.Eventf(set, v1.EventTypeNormal, "RecreatingTerminatedPod",
				"StatefulSet %s/%s is recreating terminated Pod %s",
				set.Namespace,
				set.Name,
				replicas[i].Name)
		}
		if err := ssc.podControl.DeleteStatefulPod(set, replicas[i]); err != nil {
			return true, err
		}
		replicaOrd := i + getStartOrdinal(set)
		replicas[i] = newVersionedStatefulSetPod(
			currentSet,
			updateSet,
			currentRevision.Name,
			updateRevision.Name,
			replicaOrd)
	}
	// If we find a Pod that has not been created we create the Pod
	if !isCreated(replicas[i]) {
		if utilfeature.DefaultFeatureGate.Enabled(features.StatefulSetAutoDeletePVC) {
			if isStale, err := ssc.podControl.PodClaimIsStale(set, replicas[i]); err != nil {
				return true, err
			} else if isStale {
				// If a pod has a stale PVC, no more work can be done this round.
				return true, err
			}
		}
		if err := ssc.podControl.CreateStatefulPod(ctx, set, replicas[i]); err != nil {
			return true, err
		}
		if monotonic {
			// if the set does not allow bursting, return immediately
			return true, nil
		}
	}

	// If the Pod is in pending state then trigger PVC creation to create missing PVCs
	if isPending(replicas[i]) {
		logger.V(4).Info(
			"StatefulSet is triggering PVC creation for pending Pod",
			"statefulSet", klog.KObj(set), "pod", klog.KObj(replicas[i]))
		if err := ssc.podControl.createMissingPersistentVolumeClaims(ctx, set, replicas[i]); err != nil {
			return true, err
		}
	}

	// If we find a Pod that is currently terminating, we must wait until graceful deletion
	// completes before we continue to make progress.
	if isTerminating(replicas[i]) && monotonic {
		logger.V(4).Info("StatefulSet is waiting for Pod to Terminate",
			"statefulSet", klog.KObj(set), "pod", klog.KObj(replicas[i]))
		return true, nil
	}

	// If we have a Pod that has been created but is not running and ready we can not make progress.
	// We must ensure that all for each Pod, when we create it, all of its predecessors, with respect to its
	// ordinal, are Running and Ready.
	if !isRunningAndReady(replicas[i]) && monotonic {
		logger.V(4).Info("StatefulSet is waiting for Pod to be Running and Ready",
			"statefulSet", klog.KObj(set), "pod", klog.KObj(replicas[i]))
		return true, nil
	}

	// If we have a Pod that has been created but is not available we can not make progress.
	// We must ensure that all for each Pod, when we create it, all of its predecessors, with respect to its
	// ordinal, are Available.
	if !isRunningAndAvailable(replicas[i], set.Spec.MinReadySeconds) && monotonic {
		logger.V(4).Info("StatefulSet is waiting for Pod to be Available",
			"statefulSet", klog.KObj(set), "pod", klog.KObj(replicas[i]))
		return true, nil
	}

	// Enforce the StatefulSet invariants
	retentionMatch := true
	if utilfeature.DefaultFeatureGate.Enabled(features.StatefulSetAutoDeletePVC) {
		var err error
		retentionMatch, err = ssc.podControl.ClaimsMatchRetentionPolicy(ctx, updateSet, replicas[i])
		// An error is expected if the pod is not yet fully updated, and so return is treated as matching.
		if err != nil {
			retentionMatch = true
		}
	}

	if identityMatches(set, replicas[i]) && storageMatches(set, replicas[i]) && retentionMatch {
		return false, nil
	}

	// Make a deep copy so we don't mutate the shared cache
	replica := replicas[i].DeepCopy()
	if err := ssc.podControl.UpdateStatefulPod(ctx, updateSet, replica); err != nil {
		return true, err
	}

	return false, nil
}```