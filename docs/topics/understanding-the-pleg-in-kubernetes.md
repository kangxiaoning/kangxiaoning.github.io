# 理解PLEG工作原理

<show-structure depth="3"/>

**PLEG**的全称是**Pod Lifecycle Event Generator**，顾名思义，是Pod生命周期中产生事件的模块。kubelet (Kubernetes) 中的**PLEG**模块会根据每个匹配的pod事件调整容器运行时状态，并通过应用更改来保持pod缓存的最新状态。

## 1. PLEG启动过程

在`Kubelet.Run()`会启动`pleg`，这里的`pleg`是个interface，无法直接跳转到源码，因此下面通过Debug追踪执行pleg的具体代码。

```Go
func (kl *Kubelet) Run(updates <-chan kubetypes.PodUpdate) {
	if kl.logServer == nil {
		kl.logServer = http.StripPrefix("/logs/", http.FileServer(http.Dir("/var/log/")))
	}
	if kl.kubeClient == nil {
		klog.InfoS("No API server defined - no node status update will be sent")
	}

	// Start the cloud provider sync manager
	if kl.cloudResourceSyncManager != nil {
		go kl.cloudResourceSyncManager.Run(wait.NeverStop)
	}

	if err := kl.initializeModules(); err != nil {
		kl.recorder.Eventf(kl.nodeRef, v1.EventTypeWarning, events.KubeletSetupFailed, err.Error())
		klog.ErrorS(err, "Failed to initialize internal modules")
		os.Exit(1)
	}

	// Start volume manager
	go kl.volumeManager.Run(kl.sourcesReady, wait.NeverStop)

	if kl.kubeClient != nil {
		// Introduce some small jittering to ensure that over time the requests won't start
		// accumulating at approximately the same time from the set of nodes due to priority and
		// fairness effect.
		go wait.JitterUntil(kl.syncNodeStatus, kl.nodeStatusUpdateFrequency, 0.04, true, wait.NeverStop)
		go kl.fastStatusUpdateOnce()

		// start syncing lease
		go kl.nodeLeaseController.Run(wait.NeverStop)
	}
	go wait.Until(kl.updateRuntimeUp, 5*time.Second, wait.NeverStop)

	// Set up iptables util rules
	if kl.makeIPTablesUtilChains {
		kl.initNetworkUtil()
	}

	// Start component sync loops.
	kl.statusManager.Start()

	// Start syncing RuntimeClasses if enabled.
	if kl.runtimeClassManager != nil {
		kl.runtimeClassManager.Start(wait.NeverStop)
	}

	// Start the pod lifecycle event generator.
	kl.pleg.Start()
	kl.syncLoop(updates, kl)
}
```
{collapsible="true" collapsed-title="Kubelet.Run()" default-state="collapsed"}

<procedure>
<img src="kubelet-pleg-start.png" thumbnail="true"/>
</procedure>

Step into到`Start()`函数，可以看到具体执行的是`GenericPLEG.Start()`。

- Debug信息显示`relistPeriod`为1秒，也就是**间隔1秒**执行一次`relist`操作。
<procedure>
<img src="generic-pleg-start.png" thumbnail="true"/>
</procedure>

```Go
func (g *GenericPLEG) Start() {
	go wait.Until(g.relist, g.relistPeriod, wait.NeverStop)
}
```
{collapsible="true" collapsed-title="GenericPLEG.Start()" default-state="collapsed"}

**问题：** 如果`relist()`完成时长大于1秒，会不会导致多个`relist()`同时在运行？

**答：** 不会，从`wait.Until()`的实现可知，同一时刻只有一个`relist()`执行，不管`relist()`花费多长时间，两次执行的间隔都是1秒。

- 注意`sliding`赋值为`true`，表示在`f()`执行完成后计算下次要运行的时间。
```Go
func Until(f func(), period time.Duration, stopCh <-chan struct{}) {
	JitterUntil(f, period, 0.0, true, stopCh)
}
```

```Go
func JitterUntil(f func(), period time.Duration, jitterFactor float64, sliding bool, stopCh <-chan struct{}) {
	BackoffUntil(f, NewJitteredBackoffManager(period, jitterFactor, &clock.RealClock{}), sliding, stopCh)
}
```

- 核心逻辑在`BackoffUntil()`中，每次`f()`执行完成后进入`select`，等待`t.C()`触发下一次执行，而`t.C()`是在函数`f()`执行完成后，再根据`period`计算的时间。
```Go
// BackoffUntil loops until stop channel is closed, run f every duration given by BackoffManager.
//
// If sliding is true, the period is computed after f runs. If it is false then
// period includes the runtime for f.
func BackoffUntil(f func(), backoff BackoffManager, sliding bool, stopCh <-chan struct{}) {
	var t clock.Timer
	for {
		select {
		case <-stopCh:
			return
		default:
		}

		if !sliding {
			t = backoff.Backoff()
		}

		func() {
			defer runtime.HandleCrash()
			f()
		}()

		if sliding {
			t = backoff.Backoff()
		}

		// NOTE: b/c there is no priority selection in golang
		// it is possible for this to race, meaning we could
		// trigger t.C and stopCh, and t.C select falls through.
		// In order to mitigate we re-check stopCh at the beginning
		// of every loop to prevent extra executions of f().
		select {
		case <-stopCh:
			if !t.Stop() {
				<-t.C()
			}
			return
		case <-t.C():
		}
	}
}
```
{collapsible="true" collapsed-title="BackoffUntil()" default-state="collapsed"}

```Go
func (j *jitteredBackoffManagerImpl) Backoff() clock.Timer {
	backoff := j.getNextBackoff()
	if j.backoffTimer == nil {
		j.backoffTimer = j.clock.NewTimer(backoff)
	} else {
		j.backoffTimer.Reset(backoff)
	}
	return j.backoffTimer
}
```
{collapsible="true" collapsed-title="jitteredBackoffManagerImpl.Backoff()" default-state="collapsed"}

```Go
func (j *jitteredBackoffManagerImpl) getNextBackoff() time.Duration {
	jitteredPeriod := j.duration
	if j.jitter > 0.0 {
		jitteredPeriod = Jitter(j.duration, j.jitter)
	}
	return jitteredPeriod
}
```
{collapsible="true" collapsed-title="jitteredBackoffManagerImpl.getNextBackoff()" default-state="collapsed"}

下面进入`relist()`继续追踪。

## 2. relist的作用及实现

`relist()`通过`grpc`远程调用`runtime`查询容器列表，执行`for`循环遍历每个Pod，比较Pod对象的旧版本与当前版本，调用`computeEvents()`计算容器事件并添加到`eventsByPodID`中，最后`for`循环遍历所有事件并进行处理，包括更新Pod缓存，将事件发送至`g.eventChannel`，处理之前更新缓存失败的Pod列表，维护`podsToReinspect`等。

> 每次执行`relist()`都会记录下当前时间，最终会更新到`GenericPLEG.relistTime`字段中。
> 
{style="note"}

由此可见Pod的事件源就是由`relist`产生的，如果这里出现问题，那Pod的变化不能被及时感知，也就得不到及时处理了。


```Go
// relist queries the container runtime for list of pods/containers, compare
// with the internal pods/containers, and generates events accordingly.
func (g *GenericPLEG) relist() {
	klog.V(5).InfoS("GenericPLEG: Relisting")

	if lastRelistTime := g.getRelistTime(); !lastRelistTime.IsZero() {
		metrics.PLEGRelistInterval.Observe(metrics.SinceInSeconds(lastRelistTime))
	}

	timestamp := g.clock.Now()
	defer func() {
		metrics.PLEGRelistDuration.Observe(metrics.SinceInSeconds(timestamp))
	}()

	// Get all the pods.
	podList, err := g.runtime.GetPods(true)
	if err != nil {
		klog.ErrorS(err, "GenericPLEG: Unable to retrieve pods")
		return
	}

	g.updateRelistTime(timestamp)

	pods := kubecontainer.Pods(podList)
	// update running pod and container count
	updateRunningPodAndContainerMetrics(pods)
	g.podRecords.setCurrent(pods)

	// Compare the old and the current pods, and generate events.
	eventsByPodID := map[types.UID][]*PodLifecycleEvent{}
	for pid := range g.podRecords {
		oldPod := g.podRecords.getOld(pid)
		pod := g.podRecords.getCurrent(pid)
		// Get all containers in the old and the new pod.
		allContainers := getContainersFromPods(oldPod, pod)
		for _, container := range allContainers {
			events := computeEvents(oldPod, pod, &container.ID)
			for _, e := range events {
				updateEvents(eventsByPodID, e)
			}
		}
	}

	var needsReinspection map[types.UID]*kubecontainer.Pod
	if g.cacheEnabled() {
		needsReinspection = make(map[types.UID]*kubecontainer.Pod)
	}

	// If there are events associated with a pod, we should update the
	// podCache.
	for pid, events := range eventsByPodID {
		pod := g.podRecords.getCurrent(pid)
		if g.cacheEnabled() {
			// updateCache() will inspect the pod and update the cache. If an
			// error occurs during the inspection, we want PLEG to retry again
			// in the next relist. To achieve this, we do not update the
			// associated podRecord of the pod, so that the change will be
			// detect again in the next relist.
			// TODO: If many pods changed during the same relist period,
			// inspecting the pod and getting the PodStatus to update the cache
			// serially may take a while. We should be aware of this and
			// parallelize if needed.
			if err := g.updateCache(pod, pid); err != nil {
				// Rely on updateCache calling GetPodStatus to log the actual error.
				klog.V(4).ErrorS(err, "PLEG: Ignoring events for pod", "pod", klog.KRef(pod.Namespace, pod.Name))

				// make sure we try to reinspect the pod during the next relisting
				needsReinspection[pid] = pod

				continue
			} else {
				// this pod was in the list to reinspect and we did so because it had events, so remove it
				// from the list (we don't want the reinspection code below to inspect it a second time in
				// this relist execution)
				delete(g.podsToReinspect, pid)
			}
		}
		// Update the internal storage and send out the events.
		g.podRecords.update(pid)

		// Map from containerId to exit code; used as a temporary cache for lookup
		containerExitCode := make(map[string]int)

		for i := range events {
			// Filter out events that are not reliable and no other components use yet.
			if events[i].Type == ContainerChanged {
				continue
			}
			select {
			case g.eventChannel <- events[i]:
			default:
				metrics.PLEGDiscardEvents.Inc()
				klog.ErrorS(nil, "Event channel is full, discard this relist() cycle event")
			}
			// Log exit code of containers when they finished in a particular event
			if events[i].Type == ContainerDied {
				// Fill up containerExitCode map for ContainerDied event when first time appeared
				if len(containerExitCode) == 0 && pod != nil && g.cache != nil {
					// Get updated podStatus
					status, err := g.cache.Get(pod.ID)
					if err == nil {
						for _, containerStatus := range status.ContainerStatuses {
							containerExitCode[containerStatus.ID.ID] = containerStatus.ExitCode
						}
					}
				}
				if containerID, ok := events[i].Data.(string); ok {
					if exitCode, ok := containerExitCode[containerID]; ok && pod != nil {
						klog.V(2).InfoS("Generic (PLEG): container finished", "podID", pod.ID, "containerID", containerID, "exitCode", exitCode)
					}
				}
			}
		}
	}

	if g.cacheEnabled() {
		// reinspect any pods that failed inspection during the previous relist
		if len(g.podsToReinspect) > 0 {
			klog.V(5).InfoS("GenericPLEG: Reinspecting pods that previously failed inspection")
			for pid, pod := range g.podsToReinspect {
				if err := g.updateCache(pod, pid); err != nil {
					// Rely on updateCache calling GetPodStatus to log the actual error.
					klog.V(5).ErrorS(err, "PLEG: pod failed reinspection", "pod", klog.KRef(pod.Namespace, pod.Name))
					needsReinspection[pid] = pod
				}
			}
		}

		// Update the cache timestamp.  This needs to happen *after*
		// all pods have been properly updated in the cache.
		g.cache.UpdateTime(timestamp)
	}

	// make sure we retain the list of pods that need reinspecting the next time relist is called
	g.podsToReinspect = needsReinspection
}
```
{collapsible="true" collapsed-title="GenericPLEG.relist()" default-state="collapsed"}

### 2.1 Debug GetPods()

Debug过程及相关代码参考如下。

<procedure>
<img src="generic-pleg-relist.png" thumbnail="true"/>
</procedure>

在`GetPods()`这一行打断点并Step into，具体实现是`kubeGenericRuntimeManager.GetPods()`。

<procedure>
<img src="kube-generic-runtime-manager-getpods.png" thumbnail="true"/>
</procedure>

```Go
// GetPods returns a list of containers grouped by pods. The boolean parameter
// specifies whether the runtime returns all containers including those already
// exited and dead containers (used for garbage collection).
func (m *kubeGenericRuntimeManager) GetPods(all bool) ([]*kubecontainer.Pod, error) {
	pods := make(map[kubetypes.UID]*kubecontainer.Pod)
	sandboxes, err := m.getKubeletSandboxes(all)
	if err != nil {
		return nil, err
	}
	for i := range sandboxes {
		s := sandboxes[i]
		if s.Metadata == nil {
			klog.V(4).InfoS("Sandbox does not have metadata", "sandbox", s)
			continue
		}
		podUID := kubetypes.UID(s.Metadata.Uid)
		if _, ok := pods[podUID]; !ok {
			pods[podUID] = &kubecontainer.Pod{
				ID:        podUID,
				Name:      s.Metadata.Name,
				Namespace: s.Metadata.Namespace,
			}
		}
		p := pods[podUID]
		converted, err := m.sandboxToKubeContainer(s)
		if err != nil {
			klog.V(4).InfoS("Convert sandbox of pod failed", "runtimeName", m.runtimeName, "sandbox", s, "podUID", podUID, "err", err)
			continue
		}
		p.Sandboxes = append(p.Sandboxes, converted)
	}

	containers, err := m.getKubeletContainers(all)
	if err != nil {
		return nil, err
	}
	for i := range containers {
		c := containers[i]
		if c.Metadata == nil {
			klog.V(4).InfoS("Container does not have metadata", "container", c)
			continue
		}

		labelledInfo := getContainerInfoFromLabels(c.Labels)
		pod, found := pods[labelledInfo.PodUID]
		if !found {
			pod = &kubecontainer.Pod{
				ID:        labelledInfo.PodUID,
				Name:      labelledInfo.PodName,
				Namespace: labelledInfo.PodNamespace,
			}
			pods[labelledInfo.PodUID] = pod
		}

		converted, err := m.toKubeContainer(c)
		if err != nil {
			klog.V(4).InfoS("Convert container of pod failed", "runtimeName", m.runtimeName, "container", c, "podUID", labelledInfo.PodUID, "err", err)
			continue
		}

		pod.Containers = append(pod.Containers, converted)
	}

	// Convert map to list.
	var result []*kubecontainer.Pod
	for _, pod := range pods {
		result = append(result, pod)
	}

	return result, nil
}
```
{collapsible="true" collapsed-title="kubeGenericRuntimeManager.GetPods()" default-state="collapsed"}

代码跳转来到`getKubeletSandboxes()`。

```Go
// getKubeletSandboxes lists all (or just the running) sandboxes managed by kubelet.
func (m *kubeGenericRuntimeManager) getKubeletSandboxes(all bool) ([]*runtimeapi.PodSandbox, error) {
	var filter *runtimeapi.PodSandboxFilter
	if !all {
		readyState := runtimeapi.PodSandboxState_SANDBOX_READY
		filter = &runtimeapi.PodSandboxFilter{
			State: &runtimeapi.PodSandboxStateValue{
				State: readyState,
			},
		}
	}

	resp, err := m.runtimeService.ListPodSandbox(filter)
	if err != nil {
		klog.ErrorS(err, "Failed to list pod sandboxes")
		return nil, err
	}

	return resp, nil
}
```
{collapsible="true" collapsed-title="kubeGenericRuntimeManager.getKubeletSandboxes()" default-state="collapsed"}

打断点Step into到`ListPodSandbox()`的具体实现 - `instrumentedRuntimeService.ListPodSandbox()`。

<procedure>
<img src="get-kubelet-sandboxes.png" thumbnail="true"/>
</procedure>

<procedure>
<img src="instrumented-runtime-service-list-pod-sandbox.png" thumbnail="true"/>
</procedure>

```Go
func (in instrumentedRuntimeService) ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
	const operation = "list_podsandbox"
	defer recordOperation(operation, time.Now())

	out, err := in.service.ListPodSandbox(filter)
	recordError(operation, err)
	return out, err
}
```
{collapsible="true" collapsed-title="instrumentedRuntimeService.ListPodSandbox()" default-state="collapsed"}

打断点Step into到`ListPodSandbox()`的具体实现 - `remoteRuntimeService.ListPodSandbox()`。

<procedure>
<img src="remote-runtime-service-list-pod-sandbox.png" thumbnail="true"/>
</procedure>

```Go
// ListPodSandbox returns a list of PodSandboxes.
func (r *remoteRuntimeService) ListPodSandbox(filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
	klog.V(10).InfoS("[RemoteRuntimeService] ListPodSandbox", "filter", filter, "timeout", r.timeout)
	ctx, cancel := getContextWithTimeout(r.timeout)
	defer cancel()

	if r.useV1API() {
		return r.listPodSandboxV1(ctx, filter)
	}

	return r.listPodSandboxV1alpha2(ctx, filter)
}
```
{collapsible="true" collapsed-title="remoteRuntimeService.ListPodSandbox()" default-state="collapsed"}

打断点Step into到`ListPodSandbox()`的具体实现 - `remoteRuntimeService.ListPodSandboxV1()`。

<procedure>
<img src="remote-runtime-service-list-pod-sandbox-v1.png" thumbnail="true"/>
</procedure>

```Go
func (r *remoteRuntimeService) listPodSandboxV1(ctx context.Context, filter *runtimeapi.PodSandboxFilter) ([]*runtimeapi.PodSandbox, error) {
	resp, err := r.runtimeClient.ListPodSandbox(ctx, &runtimeapi.ListPodSandboxRequest{
		Filter: filter,
	})
	if err != nil {
		klog.ErrorS(err, "ListPodSandbox with filter from runtime service failed", "filter", filter)
		return nil, err
	}

	klog.V(10).InfoS("[RemoteRuntimeService] ListPodSandbox Response", "filter", filter, "items", resp.Items)

	return resp.Items, nil
}
```
{collapsible="true" collapsed-title="remoteRuntimeService.ListPodSandboxV1()" default-state="collapsed"}

打断点Step into到`ListPodSandbox()`的具体实现 - `runtimeServiceClient.ListPodSandbox()`。

<procedure>
<img src="runtime-service-client-list-pod-sandbox.png" thumbnail="true"/>
</procedure>

```Go
func (c *runtimeServiceClient) ListPodSandbox(ctx context.Context, in *ListPodSandboxRequest, opts ...grpc.CallOption) (*ListPodSandboxResponse, error) {
	out := new(ListPodSandboxResponse)
	err := c.cc.Invoke(ctx, "/runtime.v1.RuntimeService/ListPodSandbox", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```
{collapsible="true" collapsed-title="runtimeServiceClient.ListPodSandboxV1()" default-state="collapsed"}

最终来到grpc调用。

<procedure>
<img src="grpc-invoke.png" thumbnail="true"/>
</procedure>

```Go
// Invoke sends the RPC request on the wire and returns after response is
// received.  This is typically called by generated code.
//
// All errors returned by Invoke are compatible with the status package.
func (cc *ClientConn) Invoke(ctx context.Context, method string, args, reply interface{}, opts ...CallOption) error {
	// allow interceptor to see all applicable call options, which means those
	// configured as defaults from dial option as well as per-call options
	opts = combine(cc.dopts.callOptions, opts)

	if cc.dopts.unaryInt != nil {
		return cc.dopts.unaryInt(ctx, method, args, reply, cc, invoke, opts...)
	}
	return invoke(ctx, method, args, reply, cc, opts...)
}
```
{collapsible="true" collapsed-title="ClientConn.Invoke()" default-state="collapsed"}

### 2.2 PLEG的瓶颈分析

前面通过Debug方式了解`relist`部分逻辑，这个过程涉及大量grpc远程调用(I/O密集)，对比新、旧版本计算事件(CPU密集)、更新缓存等操作，整体还是非常消耗主机资源的，参考文章对此做了较完整的总结，如下图所示。

- GetPods()涉及的grpc远程调用
<procedure>
<img src="pleg-getpods.png" thumbnail="true"/>
</procedure>

- UpdateCache()涉及的grpc远程调用
<procedure>
<img src="pleg-updatecache.png" thumbnail="true"/>
</procedure>

## 3. PLEG is not healthy是如何发生的？

从`Kubelet.Run()`中可以看到这个函数还执行了`kl.syncLoop(updates, kl)`，这个函数会持续执行`kl.runtimeState.runtimeErrors()`检查运行时状态。

```Go
// syncLoop is the main loop for processing changes. It watches for changes from
// three channels (file, apiserver, and http) and creates a union of them. For
// any new change seen, will run a sync against desired state and running state. If
// no changes are seen to the configuration, will synchronize the last known desired
// state every sync-frequency seconds. Never returns.
func (kl *Kubelet) syncLoop(updates <-chan kubetypes.PodUpdate, handler SyncHandler) {
	klog.InfoS("Starting kubelet main sync loop")
	// The syncTicker wakes up kubelet to checks if there are any pod workers
	// that need to be sync'd. A one-second period is sufficient because the
	// sync interval is defaulted to 10s.
	syncTicker := time.NewTicker(time.Second)
	defer syncTicker.Stop()
	housekeepingTicker := time.NewTicker(housekeepingPeriod)
	defer housekeepingTicker.Stop()
	plegCh := kl.pleg.Watch()
	const (
		base   = 100 * time.Millisecond
		max    = 5 * time.Second
		factor = 2
	)
	duration := base
	// Responsible for checking limits in resolv.conf
	// The limits do not have anything to do with individual pods
	// Since this is called in syncLoop, we don't need to call it anywhere else
	if kl.dnsConfigurer != nil && kl.dnsConfigurer.ResolverConfig != "" {
		kl.dnsConfigurer.CheckLimitsForResolvConf()
	}

	for {
		if err := kl.runtimeState.runtimeErrors(); err != nil {
			klog.ErrorS(err, "Skipping pod synchronization")
			// exponential backoff
			time.Sleep(duration)
			duration = time.Duration(math.Min(float64(max), factor*float64(duration)))
			continue
		}
		// reset backoff if we have a success
		duration = base

		kl.syncLoopMonitor.Store(kl.clock.Now())
		if !kl.syncLoopIteration(updates, handler, syncTicker.C, housekeepingTicker.C, plegCh) {
			break
		}
		kl.syncLoopMonitor.Store(kl.clock.Now())
	}
}
```
{collapsible="true" collapsed-title="Kubelet.syncloop()" default-state="collapsed"}

- 在`runtimeState.runtimeErrors()`会执行`health check`，如果健康检查失败(`hc.fn()`返回`false`)，则会记录**PLEG is not healthy**。
```Go
func (s *runtimeState) runtimeErrors() error {
	s.RLock()
	defer s.RUnlock()
	errs := []error{}
	if s.lastBaseRuntimeSync.IsZero() {
		errs = append(errs, errors.New("container runtime status check may not have completed yet"))
	} else if !s.lastBaseRuntimeSync.Add(s.baseRuntimeSyncThreshold).After(time.Now()) {
		errs = append(errs, errors.New("container runtime is down"))
	}
	for _, hc := range s.healthChecks {
		if ok, err := hc.fn(); !ok {
			errs = append(errs, fmt.Errorf("%s is not healthy: %v", hc.name, err))
		}
	}
	if s.runtimeError != nil {
		errs = append(errs, s.runtimeError)
	}

	return utilerrors.NewAggregate(errs)
}
```
{collapsible="true" collapsed-title="runtimeState.runtimeErrors()" default-state="collapsed"}

- Debug可定位到`hc.fn()`的具体实现是`GenericPLEG.Healthy()`

<procedure>
<img src="runtime-state-runtime-errors.png" thumbnail="true"/>
</procedure>

<procedure>
<img src="generic-pleg-healthy.png" thumbnail="true"/>
</procedure>

- `GenericPLEG.Healthy()`会判断上次`relist`记录时间到现在是否已经超过3分钟，如果超过3分钟则返回`false`及报错信息。
```Go
// Healthy check if PLEG work properly.
// relistThreshold is the maximum interval between two relist.
func (g *GenericPLEG) Healthy() (bool, error) {
	relistTime := g.getRelistTime()
	if relistTime.IsZero() {
		return false, fmt.Errorf("pleg has yet to be successful")
	}
	// Expose as metric so you can alert on `time()-pleg_last_seen_seconds > nn`
	metrics.PLEGLastSeen.Set(float64(relistTime.Unix()))
	elapsed := g.clock.Since(relistTime)
	if elapsed > relistThreshold {
		return false, fmt.Errorf("pleg was last seen active %v ago; threshold is %v", elapsed, relistThreshold)
	}
	return true, nil
}
```
{collapsible="true" collapsed-title="GenericPLEG.Healthy()" default-state="collapsed"}

综上可知，Kubelet启动后会持续执行健康检查，如果`relist`超时3分钟会导致健康检查失败，进而报错**PLEG is not healthy**。

## 4. PLEG is not healthy和NotReady有什么关系？

在创建kubelet对象时，通过`klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()`将`defaultNodeStatusFuncs()`赋值给了`klet.setNodeStatusFuncs`，后续在上报状态时通过这个函数设置节点状态。
```Go
func NewMainKubelet(kubeCfg *kubeletconfiginternal.KubeletConfiguration,
	kubeDeps *Dependencies,
	crOptions *config.ContainerRuntimeOptions,
	containerRuntime string,
	hostname string,
	hostnameOverridden bool,
	nodeName types.NodeName,
	nodeIPs []net.IP,
	providerID string,
	cloudProvider string,
	certDirectory string,
	rootDirectory string,
	imageCredentialProviderConfigFile string,
	imageCredentialProviderBinDir string,
	registerNode bool,
	registerWithTaints []api.Taint,
	allowedUnsafeSysctls []string,
	experimentalMounterPath string,
	kernelMemcgNotification bool,
	experimentalCheckNodeCapabilitiesBeforeMount bool,
	experimentalNodeAllocatableIgnoreEvictionThreshold bool,
	minimumGCAge metav1.Duration,
	maxPerPodContainerCount int32,
	maxContainerCount int32,
	masterServiceNamespace string,
	registerSchedulable bool,
	keepTerminatedPodVolumes bool,
	nodeLabels map[string]string,
	seccompProfileRoot string,
	nodeStatusMaxImages int32) (*Kubelet, error) {
	if rootDirectory == "" {
		return nil, fmt.Errorf("invalid root directory %q", rootDirectory)
	}
	if kubeCfg.SyncFrequency.Duration <= 0 {
		return nil, fmt.Errorf("invalid sync frequency %d", kubeCfg.SyncFrequency.Duration)
	}

	if kubeCfg.MakeIPTablesUtilChains {
		if kubeCfg.IPTablesMasqueradeBit > 31 || kubeCfg.IPTablesMasqueradeBit < 0 {
			return nil, fmt.Errorf("iptables-masquerade-bit is not valid. Must be within [0, 31]")
		}
		if kubeCfg.IPTablesDropBit > 31 || kubeCfg.IPTablesDropBit < 0 {
			return nil, fmt.Errorf("iptables-drop-bit is not valid. Must be within [0, 31]")
		}
		if kubeCfg.IPTablesDropBit == kubeCfg.IPTablesMasqueradeBit {
			return nil, fmt.Errorf("iptables-masquerade-bit and iptables-drop-bit must be different")
		}
	}

	var nodeHasSynced cache.InformerSynced
	var nodeLister corelisters.NodeLister

	// If kubeClient == nil, we are running in standalone mode (i.e. no API servers)
	// If not nil, we are running as part of a cluster and should sync w/API
	if kubeDeps.KubeClient != nil {
		kubeInformers := informers.NewSharedInformerFactoryWithOptions(kubeDeps.KubeClient, 0, informers.WithTweakListOptions(func(options *metav1.ListOptions) {
			options.FieldSelector = fields.Set{api.ObjectNameField: string(nodeName)}.String()
		}))
		nodeLister = kubeInformers.Core().V1().Nodes().Lister()
		nodeHasSynced = func() bool {
			return kubeInformers.Core().V1().Nodes().Informer().HasSynced()
		}
		kubeInformers.Start(wait.NeverStop)
		klog.Info("Attempting to sync node with API server")
	} else {
		// we don't have a client to sync!
		nodeIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{})
		nodeLister = corelisters.NewNodeLister(nodeIndexer)
		nodeHasSynced = func() bool { return true }
		klog.Info("Kubelet is running in standalone mode, will skip API server sync")
	}

	if kubeDeps.PodConfig == nil {
		var err error
		kubeDeps.PodConfig, err = makePodSourceConfig(kubeCfg, kubeDeps, nodeName, nodeHasSynced)
		if err != nil {
			return nil, err
		}
	}

	containerGCPolicy := kubecontainer.GCPolicy{
		MinAge:             minimumGCAge.Duration,
		MaxPerPodContainer: int(maxPerPodContainerCount),
		MaxContainers:      int(maxContainerCount),
	}

	daemonEndpoints := &v1.NodeDaemonEndpoints{
		KubeletEndpoint: v1.DaemonEndpoint{Port: kubeCfg.Port},
	}

	imageGCPolicy := images.ImageGCPolicy{
		MinAge:               kubeCfg.ImageMinimumGCAge.Duration,
		HighThresholdPercent: int(kubeCfg.ImageGCHighThresholdPercent),
		LowThresholdPercent:  int(kubeCfg.ImageGCLowThresholdPercent),
	}

	enforceNodeAllocatable := kubeCfg.EnforceNodeAllocatable
	if experimentalNodeAllocatableIgnoreEvictionThreshold {
		// Do not provide kubeCfg.EnforceNodeAllocatable to eviction threshold parsing if we are not enforcing Evictions
		enforceNodeAllocatable = []string{}
	}
	thresholds, err := eviction.ParseThresholdConfig(enforceNodeAllocatable, kubeCfg.EvictionHard, kubeCfg.EvictionSoft, kubeCfg.EvictionSoftGracePeriod, kubeCfg.EvictionMinimumReclaim)
	if err != nil {
		return nil, err
	}
	evictionConfig := eviction.Config{
		PressureTransitionPeriod: kubeCfg.EvictionPressureTransitionPeriod.Duration,
		MaxPodGracePeriodSeconds: int64(kubeCfg.EvictionMaxPodGracePeriod),
		Thresholds:               thresholds,
		KernelMemcgNotification:  kernelMemcgNotification,
		PodCgroupRoot:            kubeDeps.ContainerManager.GetPodCgroupRoot(),
	}

	var serviceLister corelisters.ServiceLister
	var serviceHasSynced cache.InformerSynced
	if kubeDeps.KubeClient != nil {
		kubeInformers := informers.NewSharedInformerFactory(kubeDeps.KubeClient, 0)
		serviceLister = kubeInformers.Core().V1().Services().Lister()
		serviceHasSynced = kubeInformers.Core().V1().Services().Informer().HasSynced
		kubeInformers.Start(wait.NeverStop)
	} else {
		serviceIndexer := cache.NewIndexer(cache.MetaNamespaceKeyFunc, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc})
		serviceLister = corelisters.NewServiceLister(serviceIndexer)
		serviceHasSynced = func() bool { return true }
	}

	// construct a node reference used for events
	nodeRef := &v1.ObjectReference{
		Kind:      "Node",
		Name:      string(nodeName),
		UID:       types.UID(nodeName),
		Namespace: "",
	}

	oomWatcher, err := oomwatcher.NewWatcher(kubeDeps.Recorder)
	if err != nil {
		return nil, err
	}

	clusterDNS := make([]net.IP, 0, len(kubeCfg.ClusterDNS))
	for _, ipEntry := range kubeCfg.ClusterDNS {
		ip := net.ParseIP(ipEntry)
		if ip == nil {
			klog.Warningf("Invalid clusterDNS ip '%q'", ipEntry)
		} else {
			clusterDNS = append(clusterDNS, ip)
		}
	}
	httpClient := &http.Client{}

	klet := &Kubelet{
		hostname:                                hostname,
		hostnameOverridden:                      hostnameOverridden,
		nodeName:                                nodeName,
		kubeClient:                              kubeDeps.KubeClient,
		heartbeatClient:                         kubeDeps.HeartbeatClient,
		onRepeatedHeartbeatFailure:              kubeDeps.OnHeartbeatFailure,
		rootDirectory:                           rootDirectory,
		resyncInterval:                          kubeCfg.SyncFrequency.Duration,
		sourcesReady:                            config.NewSourcesReady(kubeDeps.PodConfig.SeenAllSources),
		registerNode:                            registerNode,
		registerWithTaints:                      registerWithTaints,
		registerSchedulable:                     registerSchedulable,
		dnsConfigurer:                           dns.NewConfigurer(kubeDeps.Recorder, nodeRef, nodeIPs, clusterDNS, kubeCfg.ClusterDomain, kubeCfg.ResolverConfig),
		serviceLister:                           serviceLister,
		serviceHasSynced:                        serviceHasSynced,
		nodeLister:                              nodeLister,
		nodeHasSynced:                           nodeHasSynced,
		masterServiceNamespace:                  masterServiceNamespace,
		streamingConnectionIdleTimeout:          kubeCfg.StreamingConnectionIdleTimeout.Duration,
		recorder:                                kubeDeps.Recorder,
		cadvisor:                                kubeDeps.CAdvisorInterface,
		cloud:                                   kubeDeps.Cloud,
		externalCloudProvider:                   cloudprovider.IsExternal(cloudProvider),
		providerID:                              providerID,
		nodeRef:                                 nodeRef,
		nodeLabels:                              nodeLabels,
		nodeStatusUpdateFrequency:               kubeCfg.NodeStatusUpdateFrequency.Duration,
		nodeStatusReportFrequency:               kubeCfg.NodeStatusReportFrequency.Duration,
		os:                                      kubeDeps.OSInterface,
		oomWatcher:                              oomWatcher,
		cgroupsPerQOS:                           kubeCfg.CgroupsPerQOS,
		cgroupRoot:                              kubeCfg.CgroupRoot,
		mounter:                                 kubeDeps.Mounter,
		hostutil:                                kubeDeps.HostUtil,
		subpather:                               kubeDeps.Subpather,
		maxPods:                                 int(kubeCfg.MaxPods),
		podsPerCore:                             int(kubeCfg.PodsPerCore),
		syncLoopMonitor:                         atomic.Value{},
		daemonEndpoints:                         daemonEndpoints,
		containerManager:                        kubeDeps.ContainerManager,
		containerRuntimeName:                    containerRuntime,
		nodeIPs:                                 nodeIPs,
		nodeIPValidator:                         validateNodeIP,
		clock:                                   clock.RealClock{},
		enableControllerAttachDetach:            kubeCfg.EnableControllerAttachDetach,
		makeIPTablesUtilChains:                  kubeCfg.MakeIPTablesUtilChains,
		iptablesMasqueradeBit:                   int(kubeCfg.IPTablesMasqueradeBit),
		iptablesDropBit:                         int(kubeCfg.IPTablesDropBit),
		experimentalHostUserNamespaceDefaulting: utilfeature.DefaultFeatureGate.Enabled(features.ExperimentalHostUserNamespaceDefaultingGate),
		keepTerminatedPodVolumes:                keepTerminatedPodVolumes,
		nodeStatusMaxImages:                     nodeStatusMaxImages,
		lastContainerStartedTime:                newTimeCache(),
	}

	if klet.cloud != nil {
		klet.cloudResourceSyncManager = cloudresource.NewSyncManager(klet.cloud, nodeName, klet.nodeStatusUpdateFrequency)
	}

	var secretManager secret.Manager
	var configMapManager configmap.Manager
	switch kubeCfg.ConfigMapAndSecretChangeDetectionStrategy {
	case kubeletconfiginternal.WatchChangeDetectionStrategy:
		secretManager = secret.NewWatchingSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewWatchingConfigMapManager(kubeDeps.KubeClient)
	case kubeletconfiginternal.TTLCacheChangeDetectionStrategy:
		secretManager = secret.NewCachingSecretManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
		configMapManager = configmap.NewCachingConfigMapManager(
			kubeDeps.KubeClient, manager.GetObjectTTLFromNodeFunc(klet.GetNode))
	case kubeletconfiginternal.GetChangeDetectionStrategy:
		secretManager = secret.NewSimpleSecretManager(kubeDeps.KubeClient)
		configMapManager = configmap.NewSimpleConfigMapManager(kubeDeps.KubeClient)
	default:
		return nil, fmt.Errorf("unknown configmap and secret manager mode: %v", kubeCfg.ConfigMapAndSecretChangeDetectionStrategy)
	}

	klet.secretManager = secretManager
	klet.configMapManager = configMapManager

	if klet.experimentalHostUserNamespaceDefaulting {
		klog.Infof("Experimental host user namespace defaulting is enabled.")
	}

	machineInfo, err := klet.cadvisor.MachineInfo()
	if err != nil {
		return nil, err
	}
	// Avoid collector collects it as a timestamped metric
	// See PR #95210 and #97006 for more details.
	machineInfo.Timestamp = time.Time{}
	klet.setCachedMachineInfo(machineInfo)

	imageBackOff := flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)

	klet.livenessManager = proberesults.NewManager()
	klet.startupManager = proberesults.NewManager()
	klet.podCache = kubecontainer.NewCache()

	// podManager is also responsible for keeping secretManager and configMapManager contents up-to-date.
	mirrorPodClient := kubepod.NewBasicMirrorClient(klet.kubeClient, string(nodeName), nodeLister)
	klet.podManager = kubepod.NewBasicPodManager(mirrorPodClient, secretManager, configMapManager)

	klet.statusManager = status.NewManager(klet.kubeClient, klet.podManager, klet)

	klet.resourceAnalyzer = serverstats.NewResourceAnalyzer(klet, kubeCfg.VolumeStatsAggPeriod.Duration)

	klet.dockerLegacyService = kubeDeps.dockerLegacyService
	klet.runtimeService = kubeDeps.RemoteRuntimeService

	if kubeDeps.KubeClient != nil {
		klet.runtimeClassManager = runtimeclass.NewManager(kubeDeps.KubeClient)
	}

	if containerRuntime == kubetypes.RemoteContainerRuntime && utilfeature.DefaultFeatureGate.Enabled(features.CRIContainerLogRotation) {
		// setup containerLogManager for CRI container runtime
		containerLogManager, err := logs.NewContainerLogManager(
			klet.runtimeService,
			kubeDeps.OSInterface,
			kubeCfg.ContainerLogMaxSize,
			int(kubeCfg.ContainerLogMaxFiles),
		)
		if err != nil {
			return nil, fmt.Errorf("failed to initialize container log manager: %v", err)
		}
		klet.containerLogManager = containerLogManager
	} else {
		klet.containerLogManager = logs.NewStubContainerLogManager()
	}

	runtime, err := kuberuntime.NewKubeGenericRuntimeManager(
		kubecontainer.FilterEventRecorder(kubeDeps.Recorder),
		klet.livenessManager,
		klet.startupManager,
		seccompProfileRoot,
		machineInfo,
		klet,
		kubeDeps.OSInterface,
		klet,
		httpClient,
		imageBackOff,
		kubeCfg.SerializeImagePulls,
		float32(kubeCfg.RegistryPullQPS),
		int(kubeCfg.RegistryBurst),
		imageCredentialProviderConfigFile,
		imageCredentialProviderBinDir,
		kubeCfg.CPUCFSQuota,
		kubeCfg.CPUCFSQuotaPeriod,
		kubeDeps.RemoteRuntimeService,
		kubeDeps.RemoteImageService,
		kubeDeps.ContainerManager.InternalContainerLifecycle(),
		kubeDeps.dockerLegacyService,
		klet.containerLogManager,
		klet.runtimeClassManager,
	)
	if err != nil {
		return nil, err
	}
	klet.containerRuntime = runtime
	klet.streamingRuntime = runtime
	klet.runner = runtime

	runtimeCache, err := kubecontainer.NewRuntimeCache(klet.containerRuntime)
	if err != nil {
		return nil, err
	}
	klet.runtimeCache = runtimeCache

	if kubeDeps.useLegacyCadvisorStats {
		klet.StatsProvider = stats.NewCadvisorStatsProvider(
			klet.cadvisor,
			klet.resourceAnalyzer,
			klet.podManager,
			klet.runtimeCache,
			klet.containerRuntime,
			klet.statusManager)
	} else {
		klet.StatsProvider = stats.NewCRIStatsProvider(
			klet.cadvisor,
			klet.resourceAnalyzer,
			klet.podManager,
			klet.runtimeCache,
			kubeDeps.RemoteRuntimeService,
			kubeDeps.RemoteImageService,
			stats.NewLogMetricsService(),
			kubecontainer.RealOS{})
	}

	klet.pleg = pleg.NewGenericPLEG(klet.containerRuntime, plegChannelCapacity, plegRelistPeriod, klet.podCache, clock.RealClock{})
	klet.runtimeState = newRuntimeState(maxWaitForContainerRuntime)
	klet.runtimeState.addHealthCheck("PLEG", klet.pleg.Healthy)
	if _, err := klet.updatePodCIDR(kubeCfg.PodCIDR); err != nil {
		klog.Errorf("Pod CIDR update failed %v", err)
	}

	// setup containerGC
	containerGC, err := kubecontainer.NewContainerGC(klet.containerRuntime, containerGCPolicy, klet.sourcesReady)
	if err != nil {
		return nil, err
	}
	klet.containerGC = containerGC
	klet.containerDeletor = newPodContainerDeletor(klet.containerRuntime, integer.IntMax(containerGCPolicy.MaxPerPodContainer, minDeadContainerInPod))

	// setup imageManager
	imageManager, err := images.NewImageGCManager(klet.containerRuntime, klet.StatsProvider, kubeDeps.Recorder, nodeRef, imageGCPolicy, crOptions.PodSandboxImage)
	if err != nil {
		return nil, fmt.Errorf("failed to initialize image manager: %v", err)
	}
	klet.imageManager = imageManager

	if kubeCfg.ServerTLSBootstrap && kubeDeps.TLSOptions != nil && utilfeature.DefaultFeatureGate.Enabled(features.RotateKubeletServerCertificate) {
		klet.serverCertificateManager, err = kubeletcertificate.NewKubeletServerCertificateManager(klet.kubeClient, kubeCfg, klet.nodeName, klet.getLastObservedNodeAddresses, certDirectory)
		if err != nil {
			return nil, fmt.Errorf("failed to initialize certificate manager: %v", err)
		}
		kubeDeps.TLSOptions.Config.GetCertificate = func(*tls.ClientHelloInfo) (*tls.Certificate, error) {
			cert := klet.serverCertificateManager.Current()
			if cert == nil {
				return nil, fmt.Errorf("no serving certificate available for the kubelet")
			}
			return cert, nil
		}
	}

	klet.probeManager = prober.NewManager(
		klet.statusManager,
		klet.livenessManager,
		klet.startupManager,
		klet.runner,
		kubeDeps.Recorder)

	tokenManager := token.NewManager(kubeDeps.KubeClient)

	// NewInitializedVolumePluginMgr initializes some storageErrors on the Kubelet runtimeState (in csi_plugin.go init)
	// which affects node ready status. This function must be called before Kubelet is initialized so that the Node
	// ReadyState is accurate with the storage state.
	klet.volumePluginMgr, err =
		NewInitializedVolumePluginMgr(klet, secretManager, configMapManager, tokenManager, kubeDeps.VolumePlugins, kubeDeps.DynamicPluginProber)
	if err != nil {
		return nil, err
	}
	klet.pluginManager = pluginmanager.NewPluginManager(
		klet.getPluginsRegistrationDir(), /* sockDir */
		kubeDeps.Recorder,
	)

	// If the experimentalMounterPathFlag is set, we do not want to
	// check node capabilities since the mount path is not the default
	if len(experimentalMounterPath) != 0 {
		experimentalCheckNodeCapabilitiesBeforeMount = false
		// Replace the nameserver in containerized-mounter's rootfs/etc/resolve.conf with kubelet.ClusterDNS
		// so that service name could be resolved
		klet.dnsConfigurer.SetupDNSinContainerizedMounter(experimentalMounterPath)
	}

	// setup volumeManager
	klet.volumeManager = volumemanager.NewVolumeManager(
		kubeCfg.EnableControllerAttachDetach,
		nodeName,
		klet.podManager,
		klet.statusManager,
		klet.kubeClient,
		klet.volumePluginMgr,
		klet.containerRuntime,
		kubeDeps.Mounter,
		kubeDeps.HostUtil,
		klet.getPodsDir(),
		kubeDeps.Recorder,
		experimentalCheckNodeCapabilitiesBeforeMount,
		keepTerminatedPodVolumes,
		volumepathhandler.NewBlockVolumePathHandler())

	klet.reasonCache = NewReasonCache()
	klet.workQueue = queue.NewBasicWorkQueue(klet.clock)
	klet.podWorkers = newPodWorkers(klet.syncPod, kubeDeps.Recorder, klet.workQueue, klet.resyncInterval, backOffPeriod, klet.podCache)

	klet.backOff = flowcontrol.NewBackOff(backOffPeriod, MaxContainerBackOff)
	klet.podKiller = NewPodKiller(klet)

	etcHostsPathFunc := func(podUID types.UID) string { return getEtcHostsPath(klet.getPodDir(podUID)) }
	// setup eviction manager
	evictionManager, evictionAdmitHandler := eviction.NewManager(klet.resourceAnalyzer, evictionConfig, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.podManager.GetMirrorPodByPod, klet.imageManager, klet.containerGC, kubeDeps.Recorder, nodeRef, klet.clock, etcHostsPathFunc)

	klet.evictionManager = evictionManager
	klet.admitHandlers.AddPodAdmitHandler(evictionAdmitHandler)

	if utilfeature.DefaultFeatureGate.Enabled(features.Sysctls) {
		// Safe, whitelisted sysctls can always be used as unsafe sysctls in the spec.
		// Hence, we concatenate those two lists.
		safeAndUnsafeSysctls := append(sysctlwhitelist.SafeSysctlWhitelist(), allowedUnsafeSysctls...)
		sysctlsWhitelist, err := sysctl.NewWhitelist(safeAndUnsafeSysctls)
		if err != nil {
			return nil, err
		}
		klet.admitHandlers.AddPodAdmitHandler(sysctlsWhitelist)
	}

	// enable active deadline handler
	activeDeadlineHandler, err := newActiveDeadlineHandler(klet.statusManager, kubeDeps.Recorder, klet.clock)
	if err != nil {
		return nil, err
	}
	klet.AddPodSyncLoopHandler(activeDeadlineHandler)
	klet.AddPodSyncHandler(activeDeadlineHandler)

	klet.admitHandlers.AddPodAdmitHandler(klet.containerManager.GetAllocateResourcesPodAdmitHandler())

	criticalPodAdmissionHandler := preemption.NewCriticalPodAdmissionHandler(klet.GetActivePods, killPodNow(klet.podWorkers, kubeDeps.Recorder), kubeDeps.Recorder)
	klet.admitHandlers.AddPodAdmitHandler(lifecycle.NewPredicateAdmitHandler(klet.getNodeAnyWay, criticalPodAdmissionHandler, klet.containerManager.UpdatePluginResources))
	// apply functional Option's
	for _, opt := range kubeDeps.Options {
		opt(klet)
	}

	if sysruntime.GOOS == "linux" {
		// AppArmor is a Linux kernel security module and it does not support other operating systems.
		klet.appArmorValidator = apparmor.NewValidator(containerRuntime)
		klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewAppArmorAdmitHandler(klet.appArmorValidator))
	}
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewNoNewPrivsAdmitHandler(klet.containerRuntime))
	klet.softAdmitHandlers.AddPodAdmitHandler(lifecycle.NewProcMountAdmitHandler(klet.containerRuntime))

	leaseDuration := time.Duration(kubeCfg.NodeLeaseDurationSeconds) * time.Second
	renewInterval := time.Duration(float64(leaseDuration) * nodeLeaseRenewIntervalFraction)
	klet.nodeLeaseController = lease.NewController(
		klet.clock,
		klet.heartbeatClient,
		string(klet.nodeName),
		kubeCfg.NodeLeaseDurationSeconds,
		klet.onRepeatedHeartbeatFailure,
		renewInterval,
		v1.NamespaceNodeLease,
		util.SetNodeOwnerFunc(klet.heartbeatClient, string(klet.nodeName)))

	// setup node shutdown manager
	shutdownManager, shutdownAdmitHandler := nodeshutdown.NewManager(klet.GetActivePods, killPodNow(klet.podWorkers, kubeDeps.Recorder), klet.syncNodeStatus, kubeCfg.ShutdownGracePeriod.Duration, kubeCfg.ShutdownGracePeriodCriticalPods.Duration)

	klet.shutdownManager = shutdownManager
	klet.admitHandlers.AddPodAdmitHandler(shutdownAdmitHandler)

	// Finally, put the most recent version of the config on the Kubelet, so
	// people can see how it was configured.
	klet.kubeletConfiguration = *kubeCfg

	// Generating the status funcs should be the last thing we do,
	// since this relies on the rest of the Kubelet having been constructed.
	klet.setNodeStatusFuncs = klet.defaultNodeStatusFuncs()

	return klet, nil
}
```
{collapsible="true" collapsed-title="NewMainKubelet()" default-state="collapsed"}

`defaultNodeStatusFuncs()`中调用了`ReadyCondition()`，这个函数会检查并设置Node的状态。
```Go
func (kl *Kubelet) defaultNodeStatusFuncs() []func(*v1.Node) error {
	// if cloud is not nil, we expect the cloud resource sync manager to exist
	var nodeAddressesFunc func() ([]v1.NodeAddress, error)
	if kl.cloud != nil {
		nodeAddressesFunc = kl.cloudResourceSyncManager.NodeAddresses
	}
	var validateHostFunc func() error
	if kl.appArmorValidator != nil {
		validateHostFunc = kl.appArmorValidator.ValidateHost
	}
	var setters []func(n *v1.Node) error
	setters = append(setters,
		nodestatus.NodeAddress(kl.nodeIPs, kl.nodeIPValidator, kl.hostname, kl.hostnameOverridden, kl.externalCloudProvider, kl.cloud, nodeAddressesFunc),
		nodestatus.MachineInfo(string(kl.nodeName), kl.maxPods, kl.podsPerCore, kl.GetCachedMachineInfo, kl.containerManager.GetCapacity,
			kl.containerManager.GetDevicePluginResourceCapacity, kl.containerManager.GetNodeAllocatableReservation, kl.recordEvent),
		nodestatus.VersionInfo(kl.cadvisor.VersionInfo, kl.containerRuntime.Type, kl.containerRuntime.Version),
		nodestatus.DaemonEndpoints(kl.daemonEndpoints),
		nodestatus.Images(kl.nodeStatusMaxImages, kl.imageManager.GetImageList),
		nodestatus.GoRuntime(),
	)
	// Volume limits
	setters = append(setters, nodestatus.VolumeLimits(kl.volumePluginMgr.ListVolumePluginWithLimits))

	setters = append(setters,
		nodestatus.MemoryPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderMemoryPressure, kl.recordNodeStatusEvent),
		nodestatus.DiskPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderDiskPressure, kl.recordNodeStatusEvent),
		nodestatus.PIDPressureCondition(kl.clock.Now, kl.evictionManager.IsUnderPIDPressure, kl.recordNodeStatusEvent),
		nodestatus.ReadyCondition(kl.clock.Now, kl.runtimeState.runtimeErrors, kl.runtimeState.networkErrors, kl.runtimeState.storageErrors, validateHostFunc, kl.containerManager.Status, kl.shutdownManager.ShutdownStatus, kl.recordNodeStatusEvent),
		nodestatus.VolumesInUse(kl.volumeManager.ReconcilerStatesHasBeenSynced, kl.volumeManager.GetVolumesInUse),
		// TODO(mtaufen): I decided not to move this setter for now, since all it does is send an event
		// and record state back to the Kubelet runtime object. In the future, I'd like to isolate
		// these side-effects by decoupling the decisions to send events and partial status recording
		// from the Node setters.
		kl.recordNodeSchedulableEvent,
	)
	return setters
}
```
{collapsible="true" collapsed-title="defaultNodeStatusFuncs()" default-state="collapsed"}

在`tryUpdateNodeStatus()`中通过`kl.setNodeStatus(node)`设置节点状态，实现是遍历`klet.setNodeStatusFuncs`并调用每个函数，最终会调用到`ReadyCondition()`。
```Go
func (kl *Kubelet) tryUpdateNodeStatus(tryNumber int) error {
	// In large clusters, GET and PUT operations on Node objects coming
	// from here are the majority of load on apiserver and etcd.
	// To reduce the load on etcd, we are serving GET operations from
	// apiserver cache (the data might be slightly delayed but it doesn't
	// seem to cause more conflict - the delays are pretty small).
	// If it result in a conflict, all retries are served directly from etcd.
	opts := metav1.GetOptions{}
	if tryNumber == 0 {
		util.FromApiserverCache(&opts)
	}
	node, err := kl.heartbeatClient.CoreV1().Nodes().Get(context.TODO(), string(kl.nodeName), opts)
	if err != nil {
		return fmt.Errorf("error getting node %q: %v", kl.nodeName, err)
	}

	originalNode := node.DeepCopy()
	if originalNode == nil {
		return fmt.Errorf("nil %q node object", kl.nodeName)
	}

	podCIDRChanged := false
	if len(node.Spec.PodCIDRs) != 0 {
		// Pod CIDR could have been updated before, so we cannot rely on
		// node.Spec.PodCIDR being non-empty. We also need to know if pod CIDR is
		// actually changed.
		podCIDRs := strings.Join(node.Spec.PodCIDRs, ",")
		if podCIDRChanged, err = kl.updatePodCIDR(podCIDRs); err != nil {
			klog.Errorf(err.Error())
		}
	}

	kl.setNodeStatus(node)

	now := kl.clock.Now()
	if now.Before(kl.lastStatusReportTime.Add(kl.nodeStatusReportFrequency)) {
		if !podCIDRChanged && !nodeStatusHasChanged(&originalNode.Status, &node.Status) {
			// We must mark the volumes as ReportedInUse in volume manager's dsw even
			// if no changes were made to the node status (no volumes were added or removed
			// from the VolumesInUse list).
			//
			// The reason is that on a kubelet restart, the volume manager's dsw is
			// repopulated and the volume ReportedInUse is initialized to false, while the
			// VolumesInUse list from the Node object still contains the state from the
			// previous kubelet instantiation.
			//
			// Once the volumes are added to the dsw, the ReportedInUse field needs to be
			// synced from the VolumesInUse list in the Node.Status.
			//
			// The MarkVolumesAsReportedInUse() call cannot be performed in dsw directly
			// because it does not have access to the Node object.
			// This also cannot be populated on node status manager init because the volume
			// may not have been added to dsw at that time.
			kl.volumeManager.MarkVolumesAsReportedInUse(node.Status.VolumesInUse)
			return nil
		}
	}

	// Patch the current status on the API server
	updatedNode, _, err := nodeutil.PatchNodeStatus(kl.heartbeatClient.CoreV1(), types.NodeName(kl.nodeName), originalNode, node)
	if err != nil {
		return err
	}
	kl.lastStatusReportTime = now
	kl.setLastObservedNodeAddresses(updatedNode.Status.Addresses)
	// If update finishes successfully, mark the volumeInUse as reportedInUse to indicate
	// those volumes are already updated in the node's status
	kl.volumeManager.MarkVolumesAsReportedInUse(updatedNode.Status.VolumesInUse)
	return nil
}
```
{collapsible="true" collapsed-title="tryUpdateNodeStatus()" default-state="collapsed"}

而`ReadyCondition()`会通过`errs := []error{runtimeErrorsFunc(), networkErrorsFunc(), storageErrorsFunc(), nodeShutdownManagerErrorsFunc()}`检查节点状态，其中就包括PLEG异常产生的错误，如果对比发现有PLEG错误就会将节点状态设置为`v1.ConditionFalse`，也就是NotReady。
```go
// ReadyCondition returns a Setter that updates the v1.NodeReady condition on the node.
func ReadyCondition(
	nowFunc func() time.Time, // typically Kubelet.clock.Now
	runtimeErrorsFunc func() error, // typically Kubelet.runtimeState.runtimeErrors
	networkErrorsFunc func() error, // typically Kubelet.runtimeState.networkErrors
	storageErrorsFunc func() error, // typically Kubelet.runtimeState.storageErrors
	appArmorValidateHostFunc func() error, // typically Kubelet.appArmorValidator.ValidateHost, might be nil depending on whether there was an appArmorValidator
	cmStatusFunc func() cm.Status, // typically Kubelet.containerManager.Status
	nodeShutdownManagerErrorsFunc func() error, // typically kubelet.shutdownManager.errors.
	recordEventFunc func(eventType, event string), // typically Kubelet.recordNodeStatusEvent
) Setter {
	return func(node *v1.Node) error {
		// NOTE(aaronlevy): NodeReady condition needs to be the last in the list of node conditions.
		// This is due to an issue with version skewed kubelet and master components.
		// ref: https://github.com/kubernetes/kubernetes/issues/16961
		currentTime := metav1.NewTime(nowFunc())
		newNodeReadyCondition := v1.NodeCondition{
			Type:              v1.NodeReady,
			Status:            v1.ConditionTrue,
			Reason:            "KubeletReady",
			Message:           "kubelet is posting ready status",
			LastHeartbeatTime: currentTime,
		}
		errs := []error{runtimeErrorsFunc(), networkErrorsFunc(), storageErrorsFunc(), nodeShutdownManagerErrorsFunc()}
		requiredCapacities := []v1.ResourceName{v1.ResourceCPU, v1.ResourceMemory, v1.ResourcePods}
		if utilfeature.DefaultFeatureGate.Enabled(features.LocalStorageCapacityIsolation) {
			requiredCapacities = append(requiredCapacities, v1.ResourceEphemeralStorage)
		}
		missingCapacities := []string{}
		for _, resource := range requiredCapacities {
			if _, found := node.Status.Capacity[resource]; !found {
				missingCapacities = append(missingCapacities, string(resource))
			}
		}
		if len(missingCapacities) > 0 {
			errs = append(errs, fmt.Errorf("missing node capacity for resources: %s", strings.Join(missingCapacities, ", ")))
		}
		if aggregatedErr := errors.NewAggregate(errs); aggregatedErr != nil {
			newNodeReadyCondition = v1.NodeCondition{
				Type:              v1.NodeReady,
				Status:            v1.ConditionFalse,
				Reason:            "KubeletNotReady",
				Message:           aggregatedErr.Error(),
				LastHeartbeatTime: currentTime,
			}
		}
		// Append AppArmor status if it's enabled.
		// TODO(tallclair): This is a temporary message until node feature reporting is added.
		if appArmorValidateHostFunc != nil && newNodeReadyCondition.Status == v1.ConditionTrue {
			if err := appArmorValidateHostFunc(); err == nil {
				newNodeReadyCondition.Message = fmt.Sprintf("%s. AppArmor enabled", newNodeReadyCondition.Message)
			}
		}

		// Record any soft requirements that were not met in the container manager.
		status := cmStatusFunc()
		if status.SoftRequirements != nil {
			newNodeReadyCondition.Message = fmt.Sprintf("%s. WARNING: %s", newNodeReadyCondition.Message, status.SoftRequirements.Error())
		}

		readyConditionUpdated := false
		needToRecordEvent := false
		for i := range node.Status.Conditions {
			if node.Status.Conditions[i].Type == v1.NodeReady {
				if node.Status.Conditions[i].Status == newNodeReadyCondition.Status {
					newNodeReadyCondition.LastTransitionTime = node.Status.Conditions[i].LastTransitionTime
				} else {
					newNodeReadyCondition.LastTransitionTime = currentTime
					needToRecordEvent = true
				}
				node.Status.Conditions[i] = newNodeReadyCondition
				readyConditionUpdated = true
				break
			}
		}
		if !readyConditionUpdated {
			newNodeReadyCondition.LastTransitionTime = currentTime
			node.Status.Conditions = append(node.Status.Conditions, newNodeReadyCondition)
		}
		if needToRecordEvent {
			if newNodeReadyCondition.Status == v1.ConditionTrue {
				recordEventFunc(v1.EventTypeNormal, events.NodeReady)
			} else {
				recordEventFunc(v1.EventTypeNormal, events.NodeNotReady)
				klog.Infof("Node became not ready: %+v", newNodeReadyCondition)
			}
		}
		return nil
	}
}
```
{collapsible="true" collapsed-title="ReadyCondition()" default-state="collapsed"}

如下是PLEG相关的两个参数，可见事件多少也会影响PLEG是否健康。
```Go
	// Capacity of the channel for receiving pod lifecycle events. This number
	// is a bit arbitrary and may be adjusted in the future.
	plegChannelCapacity = 1000

	// Generic PLEG relies on relisting for discovering container events.
	// A longer period means that kubelet will take longer to detect container
	// changes and to update pod status. On the other hand, a shorter period
	// will cause more frequent relisting (e.g., container runtime operations),
	// leading to higher cpu usage.
	// Note that even though we set the period to 1s, the relisting itself can
	// take more than 1s to finish if the container runtime responds slowly
	// and/or when there are many container changes in one cycle.
	plegRelistPeriod = time.Second * 1
```

从前面分析可知，当出现**PLEG is not healthy**时，kubelet会主动将节点设置为NotReady并上报给控制面。出现这种情况可能有如下原因。

- 主机负载高，比如CPU、IO瓶颈，导致主机整体变慢
- 大量事件的场景，比如发版会导致大量POD产生事件
- 运行时性能瓶颈，比如响应慢、Pod挂载的NAS无响应
- BUG，比如[Deadlock in PLEG relist() for health check and Kubelet syncLoop()](https://github.com/kubernetes/kubernetes/issues/72482)

社区也在考虑改变PLEG的机制，1.26开始引入，体现在[Switching from Polling to CRI Event-based Updates to Container Status](https://kubernetes.io/docs/tasks/administer-cluster/switch-to-evented-pleg/)。

## 参考

- [Pod Lifecycle Event Generator: Understanding the "PLEG is not healthy" issue in Kubernetes](https://developers.redhat.com/blog/2019/11/13/pod-lifecycle-event-generator-understanding-the-pleg-is-not-healthy-issue-in-kubernetes)
- [Understanding PLEG with source code - Part 1](https://blog.nishipy.com/p/understanding-pleg-with-source-code-part-1/)
- [Understanding PLEG with source code - Part 2](https://blog.nishipy.com/p/understanding-pleg-with-source-code-part-2/)
- [Switching from Polling to CRI Event-based Updates to Container Status](https://kubernetes.io/docs/tasks/administer-cluster/switch-to-evented-pleg/)
