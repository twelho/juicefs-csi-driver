---
title: 问题排查案例
slug: /troubleshooting-cases
sidebar_position: 7
---

这里收录常见问题的具体排查步骤，你可以直接在本文搜索报错关键字以检索问题。同时，我们也推荐你先掌握[「基础问题排查思路」](./troubleshooting.md#basic-principles)。

## CSI 驱动未安装 / 安装失败

如果 JuiceFS CSI 驱动压根没安装，或者配置错误导致安装失败，那么试图使用 JuiceFS CSI 驱动时，便会有下方报错：

```
driver name csi.juicefs.com not found in the list of registered CSI drivers
```

请回顾[「安装 JuiceFS CSI 驱动」](../getting_started.md)，尤其注意确认 kubelet 根目录正确设置。

## CSI Node pod 异常

如果 CSI Node pod 异常，与 kubelet 通信的 socket 文件不复存在，应用 pod 事件中会看到如下错误日志：

```
/var/lib/kubelet/csi-plugins/csi.juicefs.com/csi.sock: connect: no such file or directory
```

此时需要[检查 CSI Node](./troubleshooting.md#check-csi-node)，确认其异常原因，并排查修复。

## Mount Pod 异常 {#mount-pod-error}

Mount Pod 内运行着 JuiceFS 客户端，出错的可能性多种多样，在这里罗列常见错误，指导排查。

* **Mount Pod 一直卡在 `Pending` 状态，导致应用容器也一并卡死在 `ContainerCreating` 状态**

  此时需要[查看 Mount Pod 事件](./troubleshooting.md#check-mount-pod)，确定症结所在。不过对于 `Pending` 状态，大概率是资源吃紧，导致容器无法创建。

  另外，当节点 kubelet 开启抢占功能，Mount Pod 启动后可能抢占应用资源，导致 Mount Pod 和应用 Pod 均反复创建、销毁，在 Pod 事件中能看到以下信息：

  ```
  Preempted in order to admit critical pod
  ```

  Mount Pod 默认的资源声明是 1 CPU，1GiB 内存，节点资源不足时，便无法启动，或者启动后抢占应用资源。此时需要根据实际情况[调整 Mount Pod 资源声明](../guide/resource-optimization.md#mount-pod-resources)，或者扩容宿主机。

* **Mount Pod 正常退出（exit code 为 0），应用容器卡在 `ContainerCreateError` 状态**

  Mount Pod 是一个常驻进程，如果它退出了（变为 `Completed` 状态），即便退出状态码为 0，也明显属于异常状态。此时应用容器由于挂载点不复存在，会伴随着以下错误事件：

  ```shell {4}
  $ kubectl describe pod juicefs-app
  ...
    Normal   Pulled     8m59s                 kubelet            Successfully pulled image "centos" in 2.8771491s
    Warning  Failed     8m59s                 kubelet            Error: failed to generate container "d51d4373740596659be95e1ca02375bf41cf01d3549dc7944e0bfeaea22cc8de" spec: failed to generate spec: failed to stat "/var/lib/kubelet/pods/dc0e8b63-549b-43e5-8be1-f84b25143fcd/volumes/kubernetes.io~csi/pvc-bc9b54c9-9efb-4cb5-9e1d-7166797d6d6f/mount": stat /var/lib/kubelet/pods/dc0e8b63-549b-43e5-8be1-f84b25143fcd/volumes/kubernetes.io~csi/pvc-bc9b54c9-9efb-4cb5-9e1d-7166797d6d6f/mount: transport endpoint is not connected
  ```

  错误日志里的 `transport endpoint is not connected`，其含义就是创建容器所需的 JuiceFS 挂载点不存在，因此应用容器无法创建。这时需要检查 Mount Pod 的启动命令（以下命令来自[「检查 Mount Pod」](./troubleshooting.md#check-mount-pod)文档）：

  ```shell
  APP_NS=default  # 应用所在的 Kubernetes 命名空间
  APP_POD_NAME=example-app-xxx-xxx

  # 获取 Mount Pod 的名称
  MOUNT_POD_NAME=$(kubectl -n kube-system get po --field-selector spec.nodeName=$(kubectl -n $APP_NS get po $APP_POD_NAME -o jsonpath='{.spec.nodeName}') -l app.kubernetes.io/name=juicefs-mount -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | grep $(kubectl get pv $(kubectl -n $APP_NS get pvc $(kubectl -n $APP_NS get po $APP_POD_NAME -o jsonpath='{..persistentVolumeClaim.claimName}' | awk '{print $1}') -o jsonpath='{.spec.volumeName}') -o jsonpath='{.spec.csi.volumeHandle}'))

  # 获取 Mount Pod 启动命令
  # 形如：["sh","-c","/sbin/mount.juicefs myjfs /jfs/pvc-48a083ec-eec9-45fb-a4fe-0f43e946f4aa -o foreground"]
  kubectl get pod -o jsonpath='{..containers[0].command}' $MOUNT_POD_NAME
  ```

  仔细检查 Mount Pod 启动命令，以上示例中 `-o` 后面所跟的选项即为 JuiceFS 文件系统的挂载参数，如果有多个挂载参数会通过 `,` 连接（如 `-o aaa,bbb`）。如果发现类似 `-o debug foreground` 这样的错误格式（正确格式应该是 `-o debug,foreground`），便会造成 Mount Pod 无法正常启动。此类错误往往是 `mountOptions` 填写错误造成的，请详读[「调整挂载参数」](../guide/pv.md#mount-options)，确保格式正确。

## PVC 异常 {#pvc-error}

* **静态配置中，PV 错误填写了 `storageClassName`，导致初始化异常，PVC 卡在 `Pending` 状态**

  StorageClass 的存在是为了给[「动态配置」](../guide/pv.md#dynamic-provisioning)创建 PV 时提供初始化参数。对于[「静态配置」](../guide/pv.md#static-provisioning)，`storageClassName` 必须填写为空字符串，否则将遭遇类似下方报错：

  ```shell {7}
  $ kubectl describe pvc juicefs-pv
  ...
  Events:
    Type     Reason                Age               From                                                                           Message
    ----     ------                ----              ----                                                                           -------
    Normal   Provisioning          9s (x5 over 22s)  csi.juicefs.com_juicefs-csi-controller-0_872ea36b-0fc7-4b66-bec5-96c7470dc82a  External provisioner is provisioning volume for claim "default/juicefs-pvc"
    Warning  ProvisioningFailed    9s (x5 over 22s)  csi.juicefs.com_juicefs-csi-controller-0_872ea36b-0fc7-4b66-bec5-96c7470dc82a  failed to provision volume with StorageClass "juicefs": claim Selector is not supported
    Normal   ExternalProvisioning  8s (x2 over 23s)  persistentvolume-controller                                                    waiting for a volume to be created, either by external provisioner "csi.juicefs.com" or manually created by system administrator
  ```

* **`volumeHandle` 冲突，导致 PVC 创建失败**

  两个 pod 分别使用各自的 PVC，但引用的 PV 有着相同的 `volumeHandle`，此时 PVC 将伴随着以下错误事件：

  ```shell {6}
  $ kubectl describe pvc jfs-static
  ...
  Events:
    Type     Reason         Age               From                         Message
    ----     ------         ----              ----                         -------
    Warning  FailedBinding  4s (x2 over 16s)  persistentvolume-controller  volume "jfs-static" already bound to a different claim.
  ```

  请检查每个 PVC 对应的 PV，每个 PV 的 `volumeHandle` 必须保证唯一。可以通过以下命令检查 `volumeHandle`：

  ```yaml {12}
  $ kubectl get pv -o yaml juicefs-pv
  apiVersion: v1
  kind: PersistentVolume
  metadata:
    name: juicefs-pv
    ...
  spec:
    ...
    csi:
      driver: csi.juicefs.com
      fsType: juicefs
      volumeHandle: juicefs-volume-abc
      ...
  ```

## 文件系统创建错误（社区版）

如果你选择在 mount pod 中动态地创建文件系统，也就是执行 `juicefs format` 命令，那么当创建失败时，应该会在 CSI Node pod 中看到如下错误：

```
format: ERR illegal address: xxxx
```

这里的 `format`，指的就是 `juicefs format` 命令，以上方的报错，多半是访问元数据引擎出现了问题，请检查你的安全组设置，确保所有 Kubernetes 集群的节点都能访问元数据引擎。

如果使用 Redis 作为元数据引擎，且启用了密码认证，那么可能遇到如下报错：

```
format: NOAUTH Authentication requested.
```

你需要确认元数据引擎 URL 是否正确填写了密码，具体格式请参考[「使用 Redis 作为元数据引擎」](https://juicefs.com/docs/zh/community/databases_for_metadata#redis)。

## 性能问题 {#performance-issue}

相比直接在宿主机上挂载 JuiceFS，CSI 驱动功能更为强大，但也无疑额外增加了复杂度。这里仅介绍一些 CSI 驱动下的特定问题，如果你怀疑所遭遇的性能问题与 CSI 驱动无关，请进一步参考[「社区版」](https://juicefs.com/docs/zh/community/fault_diagnosis_and_analysis)和[「云服务」](https://juicefs.com/docs/zh/cloud/administration/fault_diagnosis_and_analysis)文档学习相关排查方法。

### 读性能差 {#bad-read-performance}

以一个简单的 fio 测试为例，说明在 CSI 驱动下可能面临着怎样的性能问题，以及排查路径。

测试所用的命令，涉及数据集大小为 5 * 500MB = 2.5GB，测得的结果不尽如人意：

```shell
$ fio -directory=. \
  -ioengine=mmap \
  -rw=randread \
  -bs=4k \
  -group_reporting=1 \
  -fallocate=none \
  -time_based=1 \
  -runtime=120 \
  -name=test_file \
  -nrfiles=1 \
  -numjobs=5 \
  -size=500MB
...
  READ: bw=9896KiB/s (10.1MB/s), 9896KiB/s-9896KiB/s (10.1MB/s-10.1MB/s), io=1161MiB (1218MB), run=120167-120167msec
```

遇到性能问题，首先查看[「实时统计数据」](./troubleshooting.md#accesslog-and-stats)。在测试期间，实时监控数据大体如下：

```shell
$ juicefs stats /var/lib/juicefs/volume/pvc-xxx-xxx-xxx-xxx-xxx-xxx
------usage------ ----------fuse--------- ----meta--- -blockcache remotecache ---object--
 cpu   mem   buf | ops   lat   read write| ops   lat | read write| read write| get   put
 302%  287M   24M|  34K 0.07   139M    0 |   0     0 |7100M    0 |   0     0 |   0     0
 469%  287M   29M|  23K 0.10    92M    0 |   0     0 |4513M    0 |   0     0 |   0     0
...  # 后续数据与上方相似
```

JuiceFS 的高性能离不开其缓存设计，因此读性能发生问题时，我们首先关注 `blockcache` 相关指标，也就是磁盘上的数据块缓存文件。注意到上方数据中，`blockcache.read` 一直大于 0，这说明内核没能建立页缓存（Page Cache），所有的读请求都穿透到了位于磁盘的 Block Cache。内核页缓存位于内存，而 Block Cache 位于磁盘，二者的读性能相差极大，读请求持续穿透到磁盘，必定造成较差的性能，因此接下来调查内核缓存为何没能建立。

同样的情况如果发生在宿主机，我们会去看宿主机的内存占用情况，首先确认是否因为内存不足，没有足够空间建立页缓存。在容器中也是类似的，定位到 Mount Pod 对应的 Docker 容器，然后查看其资源占用：

```shell
# $APP_POD_NAME 是应用 pod 名称
$ docker stats $(docker ps | grep $APP_POD_NAME | grep -v "pause" | awk '{print $1}')
CONTAINER ID   NAME          CPU %     MEM USAGE / LIMIT   MEM %     NET I/O   BLOCK I/O   PIDS
90651c348bc6   k8s_POD_xxx   45.1%     1.5GiB / 2GiB       75.00%    0B / 0B   0B / 0B     1
```

注意到内存上限是 2GiB，而 fio 面对的数据集是 2.5G，已经超出了容器内存限制。此时，虽然在 `docker stats` 观察到的内存占用尚未到达 2GiB 天花板，但实际上[页缓存也占用了 cgroup 内存额度](https://www.kernel.org/doc/Documentation/cgroup-v1/memory.txt)，导致内核已经无法建立页缓存，因此调整 [Mount Pod 资源占用](../guide/resource-optimization.md#mount-pod-resources)，增大 Memory Limits，然后重建 PVC、应用 Pod，然后再次运行测试。

:::note
`docker stats` 在 cgroup v1/v2 下有着不同的统计口径，v1 不包含内核页缓存，v2 则包含。本案例在 cgroup v1 下运行，但不影响排查思路与结论。
:::

此处为了方便，我们反方向调参，降低 fio 测试数据集大小，然后测得了理想的结果：

```shell
$ fio -directory=. \
  -ioengine=mmap \
  -rw=randread \
  -bs=4k \
  -group_reporting=1 \
  -fallocate=none \
  -time_based=1 \
  -runtime=120 \
  -name=test_file \
  -nrfiles=1 \
  -numjobs=5 \
  -size=100MB
...
   READ: bw=12.4GiB/s (13.3GB/s), 12.4GiB/s-12.4GiB/s (13.3GB/s-13.3GB/s), io=1492GiB (1602GB), run=120007-120007msec
```

结论：**在容器内使用 JuiceFS，内存上限应大于所访问的数据集大小，否则将无法建立页缓存，损害读性能。**