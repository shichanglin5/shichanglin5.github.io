# NUMA 调度问题

## 无锡独享k8s集群

### 调度问题

#### Resources cannot be allocated with Topology locality
POD 一直销毁重建

![vm_operator_event.png](vm_operator_event.png)

![pod_销毁重建.png](pod_销毁重建.png)

```Text
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410301  102192 config.go:398] "Receiving a new pod" pod="vm-cluster-common/vmstorage-common-15"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410404  102192 kubelet.go:2389] "SyncLoop ADD" source="api" pods=["vm-cluster-common/vmstorage-common-15"]
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410533  102192 topology_manager.go:215] "Topology Admit Handler" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" podNamespace="vm-cluster-common" podName="vmstorage-common-15"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410545  102192 config.go:105] "Looking for sources, have seen" sources=["api","file"] seenSources={"api":{},"file":{}}
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410556  102192 scope_container.go:75] "TopologyHints" hints={} pod="vm-cluster-common/vmstorage-common-15" containerName="vmstorage"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410565  102192 config.go:105] "Looking for sources, have seen" sources=["api","file"] seenSources={"api":{},"file":{}}
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410618  102192 policy_static.go:522] "TopologyHints generated" pod="vm-cluster-common/vmstorage-common-15" containerName="vmstorage" cpuHints=[{"NUMANodeAffinity":2,"Preferred":true},{"NUMANodeAffinity":3,"Preferred":false}]
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410626  102192 scope_container.go:75] "TopologyHints" hints={"cpu":[{"NUMANodeAffinity":2,"Preferred":true},{"NUMANodeAffinity":3,"Preferred":false}]} pod="vm-cluster-common/vmstorage-common-15" containerName="vmstorage"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410634  102192 config.go:105] "Looking for sources, have seen" sources=["api","file"] seenSources={"api":{},"file":{}}
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410668  102192 scope_container.go:75] "TopologyHints" hints={"memory":[{"NUMANodeAffinity":1,"Preferred":true}]} pod="vm-cluster-common/vmstorage-common-15" containerName="vmstorage"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410674  102192 policy.go:71] "Hint Provider has no preference for NUMA affinity with any resource"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410683  102192 scope_container.go:83] "ContainerTopologyHint" bestHint={"NUMANodeAffinity":null,"Preferred":false}
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410689  102192 scope_container.go:50] "Best TopologyHint" bestHint={"NUMANodeAffinity":null,"Preferred":false} pod="vm-cluster-common/vmstorage-common-15" containerName="vmstorage"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410713  102192 status_manager.go:687] "updateStatusInternal" version=1 podIsFinished=false pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" containers=""
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.410771  102192 event.go:307] "Event occurred" object="vm-cluster-common/vmstorage-common-15" fieldPath="" kind="Pod" apiVersion="v1" type="Warning" reason="TopologyAffinityError" message="Resources cannot be allocated with Topology locality"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.585952  102192 request.go:629] Waited for 187.343394ms due to client-side throttling, not priority and fairness, request: GET:https://hdszth-tss-k8s-apiserver.17usoft.com:6443/api/v1/namespaces/vm-cluster-common/pods/vmstorage-common-15
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.589024  102192 status_manager.go:863] "Pod was deleted and then recreated, skipping status update" pod="vm-cluster-common/vmstorage-common-15" oldPodUID="5e71a8a1-581b-4be8-84b9-80b507422429" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.589036  102192 state_mem.go:137] "Deleted pod resource allocation and resize state" podUID="5e71a8a1-581b-4be8-84b9-80b507422429"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.589287  102192 status_manager.go:227] "Syncing updated statuses"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.589302  102192 status_manager.go:833] "Sync pod status" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" statusUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" version=1
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.786599  102192 request.go:629] Waited for 197.268893ms due to client-side throttling, not priority and fairness, request: GET:https://hdszth-tss-k8s-apiserver.17usoft.com:6443/api/v1/namespaces/vm-cluster-common/pods/vmstorage-common-15
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.959589  102192 generic.go:224] "GenericPLEG: Relisting"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.960202  102192 kuberuntime_manager.go:436] "Retrieved pods from runtime" all=true
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.986313  102192 request.go:629] Waited for 195.130966ms due to client-side throttling, not priority and fairness, request: PATCH:https://hdszth-tss-k8s-apiserver.17usoft.com:6443/api/v1/namespaces/vm-cluster-common/pods/vmstorage-common-15/status
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.991621  102192 config.go:293] "Setting pods for source" source="api"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.991851  102192 status_manager.go:874] "Patch status for pod" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" patch="{\"metadata\":{\"uid\":\"563dafa2-77a7-4dab-949e-9dbf079a4a70\"},\"status\":{\"conditions\":null,\"message\":\"Pod was rejected: Resources cannot be allocated with Topology locality\",\"phase\":\"Failed\",\"qosClass\":null,\"reason\":\"TopologyAffinityError\",\"startTime\":\"2024-08-15T14:24:53Z\"}}"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.991866  102192 status_manager.go:883] "Status for pod updated successfully" pod="vm-cluster-common/vmstorage-common-15" statusVersion=1 status={"phase":"Failed","message":"Pod was rejected: Resources cannot be allocated with Topology locality","reason":"TopologyAffinityError","startTime":"2024-08-15T14:24:53Z"}
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.991935  102192 kubelet.go:2402] "SyncLoop RECONCILE" source="api" pods=["vm-cluster-common/vmstorage-common-15"]
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.997939  102192 config.go:293] "Setting pods for source" source="api"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998241  102192 kubelet.go:2405] "SyncLoop DELETE" source="api" pods=["vm-cluster-common/vmstorage-common-15"]
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998256  102192 pod_workers.go:770] "Pod is being synced for the first time" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" updateType="update"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998269  102192 pod_workers.go:965] "Notifying pod of pending update" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" workType="terminated"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998285  102192 pod_workers.go:1232] "Processing pod event" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" updateType="terminated"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998302  102192 kubelet.go:2115] "SyncTerminatedPod enter" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998308  102192 kubelet_pods.go:1655] "Generating pod status" podIsTerminal=true pod="vm-cluster-common/vmstorage-common-15"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998327  102192 state_mem.go:106] "Updated pod resize state" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" resizeStatus=""
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998536  102192 kubelet_pods.go:1582] "Pod waiting > 0, pending"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998544  102192 kubelet_pods.go:1668] "Got phase for pod" pod="vm-cluster-common/vmstorage-common-15" oldPhase="Failed" phase="Pending"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998551  102192 kubelet_pods.go:1675] "Status manager phase was terminal, updating phase to match" pod="vm-cluster-common/vmstorage-common-15" phase="Failed"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998576  102192 status_manager.go:687] "updateStatusInternal" version=2 podIsFinished=false pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" containers="(vmstorage state=waiting previous=<none>)"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998586  102192 volume_manager.go:451] "Waiting for volumes to unmount for pod" pod="vm-cluster-common/vmstorage-common-15"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998604  102192 volume_manager.go:480] "All volumes are unmounted for pod" pod="vm-cluster-common/vmstorage-common-15"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998611  102192 kubelet.go:2129] "Pod termination unmounted volumes" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:53 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998635  102192 kubelet.go:2143] "Pod termination cleaned up volume paths" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998648  102192 status_manager.go:227] "Syncing updated statuses"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.998662  102192 status_manager.go:833] "Sync pod status" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" statusUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" version=2
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999324  102192 kubelet.go:2165] "Pod termination removed cgroups" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999336  102192 status_manager.go:490] "TerminatePod calling updateStatusInternal" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999356  102192 status_manager.go:687] "updateStatusInternal" version=3 podIsFinished=true pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" containers="(vmstorage state=terminated=137 previous=<none>)"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999375  102192 kubelet.go:2172] "Pod is terminated and will need no more status updates" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999382  102192 kubelet.go:2174] "SyncTerminatedPod exit" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999389  102192 pod_workers.go:1462] "Pod is complete and the worker can now stop" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999400  102192 pod_workers.go:1308] "Processing pod event done" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" updateType="terminated"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:53.999406  102192 pod_workers.go:953] "Pod worker has stopped" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.001914  102192 config.go:293] "Setting pods for source" source="api"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.002165  102192 kubelet.go:2399] "SyncLoop REMOVE" source="api" pods=["vm-cluster-common/vmstorage-common-15"]
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.002180  102192 config.go:105] "Looking for sources, have seen" sources=["api","file"] seenSources={"api":{},"file":{}}
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.002187  102192 kubelet.go:2228] "Pod has been deleted and must be killed" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.002204  102192 pod_workers.go:842] "Pod is finished processing, no further updates" pod="vm-cluster-common/vmstorage-common-15" podUID="563dafa2-77a7-4dab-949e-9dbf079a4a70" updateType="kill"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.010254  102192 config.go:293] "Setting pods for source" source="api"
Aug 15 22:24:54 wxyd-216-8-20.linux.17usoft.com kubelet[102192]: I0815 22:24:54.010397  102192 config.go:398] "Receiving a new pod" pod="vm-cluster-common/vmstorage-common-15"
```

**手动删除pod触发更新**
修改 vm storage 的 cpu 配置后，删除所有 pod 来解决之前的 pod 占用内存太大，导致死循环问题（pod一直销毁创建）
![WeChatWorkScreenshot_2879b0bc-6063-4189-9dc6-24876a94f893.png](WeChatWorkScreenshot_2879b0bc-6063-4189-9dc6-24876a94f893.png)
![WeChatWorkScreenshot_e49d5e4e-0b17-4d51-bc80-4add4468322c.png](WeChatWorkScreenshot_e49d5e4e-0b17-4d51-bc80-4add4468322c.png)
![WeChatWorkScreenshot_9c46dd12-11c1-4b5b-a407-2296e7274128.png](WeChatWorkScreenshot_9c46dd12-11c1-4b5b-a407-2296e7274128.png)

**查看 NUMA 的节点信息**
从下面信息可以看到 numa 划分为两个 node，分别有 16个cpu和 192GB 内存
```Go
[root@wxyd-216-8-118 ~]# numactl --hardware
available: 2 nodes (0-1)
node 0 cpus: 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15
node 0 size: 192359 MB
node 0 free: 182822 MB
node 1 cpus: 16 17 18 19 20 21 22 23 24 25 26 27 28 29 30 31
node 1 size: 193493 MB
node 1 free: 184412 MB
node distances:
node   0   1 
  0:  10  20 
  1:  20  10 
[root@wxyd-216-8-118 ~]#
```