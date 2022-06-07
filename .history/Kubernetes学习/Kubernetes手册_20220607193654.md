#  1.  Kubernetes 入门

##  1.  Kubernetes 生产环境

###  1.  Kubernetes 容器运行时
容器运行时
-----

你需要在集群内每个节点上安装一个 容器运行时 以使 Pod 可以运行在上面。本文概述了所涉及的内容并描述了与节点设置相关的任务。

Kubernetes 1.24 要求你使用符合容器运行时接口 (CRI)的运行时。

有关详细信息，请参阅该文的 CRI 版本支持。 本页简要介绍在 Kubernetes 中几个常见的容器运行时的用法。

*   ​`containerd` ​
*   ​`CRI-O` ​
*   ​`Docker Engine` ​
*   ​`Mirantis Container Runtime`​

> Note:  
> 提示：v1.24 之前的 Kubernetes 版本包括与 Docker Engine 的直接集成，使用名为 dockershim 的组件。 这种特殊的直接整合不再是 Kubernetes 的一部分 （这次删除被作为 v1.20 发行版本的一部分宣布）。   
> 如果你正在运行 v1.24 以外的 Kubernetes 版本，检查该版本的文档。

安装和配置先决条件
---------

以下步骤将通用设置应用于 Linux 上的 Kubernetes 节点。

如果你确定不需要某个特定设置，则可以跳过它。

#### 转发 IPv4 并让 iptables 看到桥接流量

通过运行 ​`lsmod | grep br_netfilter`​ 来验证 ​`br_netfilter`​ 模块是否已加载。

若要显式加载此模块，请运行 ​`sudo modprobe br_netfilter`​。

为了让 Linux 节点的 iptables 能够正确查看桥接流量，请确认 ​`sysctl` ​配置中的 ​`net.bridge.bridge-nf-call-iptables`​ 设置为 1。 例如：

`cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf overlay br_netfilter EOF  sudo modprobe overlay sudo modprobe br_netfilter  # 设置所需的 sysctl 参数，参数在重新启动后保持不变 cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf net.bridge.bridge-nf-call-iptables  = 1 net.bridge.bridge-nf-call-ip6tables = 1 net.ipv4.ip_forward                 = 1 EOF  # 应用 sysctl 参数而不重新启动 sudo sysctl --system`

Cgroup 驱动程序 
------------

在 Linux 上，控制组（CGroup）用于限制分配给进程的资源。

当某个 Linux 系统发行版使用 [systemd](https://www.freedesktop.org/wiki/Software/systemd/) 作为其初始化系统时，初始化进程会生成并使用一个 root 控制组（​`cgroup`​），并充当 cgroup 管理器。 Systemd 与 cgroup 集成紧密，并将为每个 systemd 单元分配一个 cgroup。 你也可以配置容器运行时和 kubelet 使用 ​`cgroupfs`​。 连同 systemd 一起使用 ​`cgroupfs` ​意味着将有两个不同的 cgroup 管理器。

单个 cgroup 管理器将简化分配资源的视图，并且默认情况下将对可用资源和使用 中的资源具有更一致的视图。 当有两个管理器共存于一个系统中时，最终将对这些资源产生两种视图。 在此领域人们已经报告过一些案例，某些节点配置让 kubelet 和 docker 使用 ​`cgroupfs`​，而节点上运行的其余进程则使用 systemd; 这类节点在资源压力下 会变得不稳定。

更改设置，令容器运行时和 kubelet 使用 ​`systemd` ​作为 cgroup 驱动，以此使系统更为稳定。 对于 Docker, 设置 ​`native.cgroupdriver=systemd`​ 选项。

注意：更改已加入集群的节点的 cgroup 驱动是一项敏感的操作。 如果 kubelet 已经使用某 cgroup 驱动的语义创建了 pod，更改运行时以使用 别的 cgroup 驱动，当为现有 Pods 重新创建 PodSandbox 时会产生错误。 重启 kubelet 也可能无法解决此类问题。 如果你有切实可行的自动化方案，使用其他已更新配置的节点来替换该节点， 或者使用自动化方案来重新安装。

#### Cgroup v2

Cgroup v2 是 cgroup Linux API 的下一个版本。与 cgroup v1 不同的是， Cgroup v2 只有一个层次结构，而不是每个控制器有一个不同的层次结构。

新版本对 cgroup v1 进行了多项改进，其中一些改进是：

*   更简洁、更易于使用的 API
*   可将安全子树委派给容器
*   更新的功能，如压力失速信息（Pressure Stall Information）

尽管内核支持混合配置，即其中一些控制器由 cgroup v1 管理，另一些由 cgroup v2 管理， Kubernetes 仅支持使用同一 cgroup 版本来管理所有控制器。

如果 systemd 默认不使用 cgroup v2，你可以通过在内核命令行中添加 ​`systemd.unified_cgroup_hierarchy=1`​ 来配置系统去使用它。

`# 此示例适用于使用 DNF 包管理器的 Linux 操作系统 # 你的系统可能使用不同的方法来设置 Linux 内核使用的命令行。 sudo dnf install -y grubby && \   sudo grubby \   --update-kernel=ALL \   --args="systemd.unified_cgroup_hierarchy=1"`

如果更改内核的命令行，则必须重新启动节点才能使更改生效。

切换到 cgroup v2 时，用户体验不应有任何明显差异， 除非用户直接在节点上或在容器内访问 cgroup 文件系统。 为了使用它，CRI 运行时也必须支持 cgroup v2。

#### 将 kubeadm 托管的集群迁移到 systemd 驱动

如果你希望将现有的由 kubeadm 管理的集群迁移到 systemd cgroup 驱动程序， 请按照配置 cgroup 驱动程序操作。

CRI 版本支持
--------

你的容器运行时必须至少支持容器运行时接口的 v1alpha2。

Kubernetes 1.24 默认使用 v1 的 CRI API。如果容器运行时不支持 v1 API， 则 kubelet 会回退到使用（已弃用的）v1alpha2 API。

容器运行时 
------

#### containerd

本节概述了使用 containerd 作为 CRI 运行时的必要步骤。

使用以下命令在系统上安装 Containerd：

按照[开始使用 containerd](https://github.com/containerd/containerd/blob/main/docs/getting-started.md) 的说明进行操作。 创建有效的配置文件 ​`config.toml`​ 后返回此步骤。

*   Linux

你可以在路径 ​`/etc/containerd/config.toml`​ 下找到此文件。

*   Windows

你可以在路径 \`​`C:\Program Files\containerd\config.toml`​\` 下找到此文件。

在 Linux 上，containerd 的默认 CRI 套接字是 ​`/run/containerd/containerd.sock`​。 在 Windows 上，默认 CRI 端点是 ​`npipe://./pipe/containerd-containerd`​。

#### 配置 systemd cgroup 驱动程序

结合 ​`runc` ​使用 ​`systemd` ​cgroup 驱动，在 ​`/etc/containerd/config.toml`​ 中设置

`[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]   ...   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]     SystemdCgroup = true`

如果你应用此更改，请确保重新启动 containerd：

`sudo systemctl restart containerd`

当使用 kubeadm 时，请手动配置 kubelet 的 cgroup 驱动.

#### CRI-O

本节包含安装 CRI-O 作为容器运行时的必要步骤。

要安装 CRI-O，请按照 [CRI-O 安装说明](https://github.com/cri-o/cri-o/blob/main/install.md target=)执行操作。

#### cgroup 驱动程序

CRI-O 默认使用 systemd cgroup 驱动程序，这对你来说可能工作得很好。要切换到 ​`cgroupfs` ​cgroup 驱动程序， 请编辑 ​`/etc/crio/crio.conf`​ 或在 ​`/etc/crio/crio.conf.d/02-cgroup-manager.conf`​ 中放置一个插入式配置 ，例如：

`[crio.runtime] conmon_cgroup = "pod" cgroup_manager = "cgroupfs"`

你还应该注意到 ​`conmon_cgroup` ​被更改，当使用 CRI-O 和 ​`cgroupfs` ​时，必须将其设置为值 ​`pod`​。 通常需要保持 kubelet 的 cgroup 驱动配置（通常通过 kubeadm 完成）和 CRI-O 同步。

对于 CRI-O，CRI 套接字默认为 ​`/var/run/crio/crio.sock`​。

#### Docker Engine

> Note: 以下操作假设你使用 [cri-dockerd](https://github.com/Mirantis/cri-dockerd) 适配器来将 Docker Engine 与 Kubernetes 集成。

1.  在你的每个节点上，遵循[安装 Docker 引擎](https://docs.docker.com/engine/install/ target=)指南为你的 Linux 发行版安装 Docker。
2.  按照源代码仓库中的说明安装 [cri-dockerd](https://github.com/Mirantis/cri-dockerd)。

对于 ​`cri-dockerd`​，默认情况下，CRI 套接字是 ​`/run/cri-dockerd.sock`​。

#### Mirantis 容器运行时

[Mirantis Container Runtime](https://docs.mirantis.com/mcr/20.10/overview.html) (MCR) 是一种商用容器运行时，以前称为 Docker 企业版。 你可以使用 MCR 中包含的开源 ​[`cri-dockerd`​](https://github.com/Mirantis/cri-dockerd) 组件将 Mirantis Container Runtime 与 Kubernetes 一起使用。

要了解有关如何安装 Mirantis Container Runtime 的更多信息，请访问 [MCR 部署指南](https://docs.mirantis.com/mcr/20.10/install.html)。

检查名为 ​`cri-docker.socket`​ 的 systemd 单元以找出 CRI 套接字的路径。

##  2.  Kubernetes 使用部署工具安装Kubernetes

###  1.  Kubernetes 安装kubeadm
在开始之前
-----

*   一台兼容的 Linux 主机。Kubernetes 项目为基于 Debian 和 Red Hat 的 Linux 发行版以及一些不提供包管理器的发行版提供通用的指令
*   每台机器 2 GB 或更多的 RAM （如果少于这个数字将会影响你应用的运行内存)
*   2 CPU 核或更多
*   集群中的所有机器的网络彼此均能相互连接(公网和内网都可以)
*   节点之中不可以有重复的主机名、MAC 地址或 product\_uuid
*   开启机器上的某些端口
*   禁用交换分区。为了保证 kubelet 正常工作，你 必须 禁用交换分区

确保每个节点上 MAC 地址和 product\_uuid 的唯一性
----------------------------------

*   你可以使用命令 ​`ip link`​ 或 ​`ifconfig -a`​ 来获取网络接口的 MAC 地址
*   可以使用 ​`sudo cat /sys/class/dmi/id/product_uuid`​ 命令对 product\_uuid 校验

一般来讲，硬件设备会拥有唯一的地址，但是有些虚拟机的地址可能会重复。 Kubernetes 使用这些值来唯一确定集群中的节点。 如果这些值在每个节点上不唯一，可能会导致安装 [失败](https://github.com/kubernetes/kubeadm/issues/31)。

检查网络适配器
-------

如果你有一个以上的网络适配器，同时你的 Kubernetes 组件通过默认路由不可达，我们建议你预先添加 IP 路由规则，这样 Kubernetes 集群就可以通过对应的适配器完成连接。

允许 iptables 检查桥接流量
------------------

确保 ​`br_netfilter`​ 模块被加载。这一操作可以通过运行 ​`lsmod | grep br_netfilter`​ 来完成。若要显式加载该模块，可执行 ​`sudo modprobe br_netfilter`​。

为了让你的 Linux 节点上的 iptables 能够正确地查看桥接流量，你需要确保在你的 ​`sysctl` ​配置中将 ​`net.bridge.bridge-nf-call-iptables`​ 设置为 1。例如：

`cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf br_netfilter EOF  cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf net.bridge.bridge-nf-call-ip6tables = 1 net.bridge.bridge-nf-call-iptables = 1 EOF sudo sysctl --system`

检查所需端口
------

启用必要的端口后才能使 Kubernetes 的各组件相互通信。可以使用 netcat 之类的工具来检查端口是否启用，例如：

`nc 127.0.0.1 6443`

你使用的 Pod 网络插件 (详见后续章节) 也可能需要开启某些特定端口。由于各个 Pod 网络插件的功能都有所不同， 请参阅他们各自文档中对端口的要求。

安装容器运行时
-------

为了在 Pod 中运行容器，Kubernetes 使用 容器运行时（Container Runtime）。

默认情况下，Kubernetes 使用 容器运行时接口（Container Runtime Interface，CRI） 来与你所选择的容器运行时交互。

如果你不指定运行时，kubeadm 会自动尝试通过扫描已知的端点列表来检测已安装的容器运行时。

如果检测到有多个或者没有容器运行时，kubeadm 将抛出一个错误并要求你指定一个想要使用的运行时。

###  2.  Kubernetes 对kubeadm进行故障排查
对 kubeadm 进行故障排查
----------------

与任何程序一样，你可能会在安装或者运行 kubeadm 时遇到错误。 本文列举了一些常见的故障场景，并提供可帮助你理解和解决这些问题的步骤。

如果你的问题未在下面列出，请执行以下步骤：

*   如果你认为问题是 kubeadm 的错误：

*   转到 [github.com/kubernetes/kubeadm](https://github.com/kubernetes/kubeadm/issues) 并搜索存在的问题。
*   如果没有问题，请 [打开](https://github.com/kubernetes/kubeadm/issues/new) 并遵循问题模板。

*   如果你对 kubeadm 的工作方式有疑问，可以在 [Slack](https://slack.k8s.io/) 上的 ​`#kubeadm`​ 频道提问， 或者在 [StackOverflow](https://stackoverflow.com/questions/tagged/kubernetes) 上提问。 请加入相关标签，例如 ​`#kubernetes`​ 和 ​`#kubeadm`​，这样其他人可以帮助你。

由于缺少 RBAC，无法将 v1.18 Node 加入 v1.17 集群
------------------------------------

自从 v1.18 后，如果集群中已存在同名 Node，kubeadm 将禁止 Node 加入集群。 这需要为 bootstrap-token 用户添加 RBAC 才能 GET Node 对象。

但这会导致一个问题，v1.18 的 ​`kubeadm join`​ 无法加入由 kubeadm v1.17 创建的集群。

要解决此问题，你有两种选择：

使用 kubeadm v1.18 在控制平面节点上执行 ​`kubeadm init phase bootstrap-token`​。 请注意，这也会启用 bootstrap-token 的其余权限。

或者，也可以使用 ​`kubectl apply -f ...`​ 手动应用以下 RBAC：

`apiVersion: rbac.authorization.k8s.io/v1 kind: ClusterRole metadata:   name: kubeadm:get-nodes rules: - apiGroups:   - ""   resources:   - nodes   verbs:   - get --- apiVersion: rbac.authorization.k8s.io/v1 kind: ClusterRoleBinding metadata:   name: kubeadm:get-nodes roleRef:   apiGroup: rbac.authorization.k8s.io   kind: ClusterRole   name: kubeadm:get-nodes subjects: - apiGroup: rbac.authorization.k8s.io   kind: Group   name: system:bootstrappers:kubeadm:default-node-token`

在安装过程中没有找到 ebtables 或者其他类似的可执行文件
--------------------------------

如果在运行 ​`kubeadm init`​ 命令时，遇到以下的警告

`[preflight] WARNING: ebtables not found in system path [preflight] WARNING: ethtool not found in system path`

那么或许在你的节点上缺失 ​`ebtables`​、​`ethtool` ​或者类似的可执行文件。 你可以使用以下命令安装它们：

*   对于 Ubuntu/Debian 用户，运行 ​`apt install ebtables ethtool`​ 命令。
*   对于 CentOS/Fedora 用户，运行 ​`yum install ebtables ethtool`​ 命令。

在安装过程中，kubeadm 一直等待控制平面就绪
-------------------------

如果你注意到 ​`kubeadm init`​ 在打印以下行后挂起：

`[apiclient] Created API client, waiting for the control plane to become ready`

这可能是由许多问题引起的。最常见的是：

*   网络连接问题。在继续之前，请检查你的计算机是否具有全部联通的网络连接。
*   容器运行时的 cgroup 驱动不同于 kubelet 使用的 cgroup 驱动。
*   控制平面上的 Docker 容器持续进入崩溃状态或（因其他原因）挂起。你可以运行 docker ps 命令来检查以及 docker logs 命令来检视每个容器的运行日志。 

当删除托管容器时 kubeadm 阻塞 
--------------------

如果容器运行时停止并且未删除 Kubernetes 所管理的容器，可能发生以下情况：

`sudo kubeadm reset`

`[preflight] Running pre-flight checks [reset] Stopping the kubelet service [reset] Unmounting mounted directories in "/var/lib/kubelet" [reset] Removing kubernetes-managed containers (block)`

一个可行的解决方案是重新启动 Docker 服务，然后重新运行 ​`kubeadm reset`​： 你也可以使用 ​`crictl` ​来调试容器运行时的状态。

Pods 处于 RunContainerError、CrashLoopBackOff 或者 Error 状态
------------------------------------------------------

在 ​`kubeadm init`​ 命令运行后，系统中不应该有 pods 处于这类状态。

*   在 ​`kubeadm init`​ 命令执行完后，如果有 pods 处于这些状态之一，请在 kubeadm 仓库提起一个 issue。​`coredns` ​(或者 ​`kube-dns`​) 应该处于 ​`Pending` ​状态， 直到你部署了网络插件为止。
*   如果在部署完网络插件之后，有 Pods 处于 ​`RunContainerError`​、​`CrashLoopBackOff` ​或 ​`Error` ​状态之一，并且 ​`coredns` ​（或者 ​`kube-dns`​）仍处于 ​`Pending` ​状态， 那很可能是你安装的网络插件由于某种原因无法工作。你或许需要授予它更多的 RBAC 特权或使用较新的版本。请在 Pod Network 提供商的问题跟踪器中提交问题， 然后在此处分类问题。
*   如果你安装的 Docker 版本早于 1.12.1，请在使用 ​`systemd` ​来启动 ​`dockerd` ​和重启 ​`docker` ​时， 删除 ​`MountFlags=slave`​ 选项。 你可以在 ​`/usr/lib/systemd/system/docker.service`​ 中看到 MountFlags。 MountFlags 可能会干扰 Kubernetes 挂载的卷，并使 Pods 处于 ​`CrashLoopBackOff` ​状态。 当 Kubernetes 不能找到 ​`var/run/secrets/kubernetes.io/serviceaccount`​ 文件时会发生错误。

coredns 停滞在 Pending 状态
----------------------

这一行为是 预期之中 的，因为系统就是这么设计的。 kubeadm 的网络供应商是中立的，因此管理员应该选择 安装 pod 的网络插件。 你必须完成 Pod 的网络配置，然后才能完全部署 CoreDNS。 在网络被配置好之前，DNS 组件会一直处于 ​`Pending` ​状态。

HostPort 服务无法工作
---------------

此 ​`HostPort` ​和 ​`HostIP` ​功能是否可用取决于你的 Pod 网络配置。请联系 Pod 网络插件的作者， 以确认 ​`HostPort` ​和 ​`HostIP` ​功能是否可用。

已验证 Calico、Canal 和 Flannel CNI 驱动程序支持 HostPort。

有关更多信息，请参考 [CNI portmap 文档](https://github.com/containernetworking/plugins/blob/main/plugins/meta/portmap/README.md).

如果你的网络提供商不支持 portmap CNI 插件，你或许需要使用 NodePort 服务的功能 或者使用 ​`HostNetwork=true`​。

无法通过其服务 IP 访问 Pod
-----------------

*   许多网络附加组件尚未启用 hairpin 模式 该模式允许 Pod 通过其服务 IP 进行访问。这是与 [CNI](https://github.com/containernetworking/cni/issues/476) 有关的问题。 请与网络附加组件提供商联系，以获取他们所提供的 hairpin 模式的最新状态。
*   如果你正在使用 VirtualBox (直接使用或者通过 Vagrant 使用)，你需要 确保 ​`hostname -i`​ 返回一个可路由的 IP 地址。默认情况下，第一个接口连接不能路由的仅主机网络。 解决方法是修改 ​`/etc/hosts`​，请参考示例 [Vagrantfile](https://github.com/errordeveloper/kubernetes-ansible-vagrant/blob/22dd39dfc06111235620e6c4404a96ae146f26fd/Vagrantfile target=)。

TLS 证书错误
--------

以下错误指出证书可能不匹配。

`# kubectl get pods Unable to connect to the server: x509: certificate signed by unknown authority (possibly because of "crypto/rsa: verification error" while trying to verify candidate authority certificate "kubernetes")`

*   验证 ​`$HOME/.kube/config`​ 文件是否包含有效证书，并 在必要时重新生成证书。在 kubeconfig 文件中的证书是 base64 编码的。 该 ​`base64 --decode`​ 命令可以用来解码证书，​`openssl x509 -text -noout`​ 命令 可以用于查看证书信息。
*   使用如下方法取消设置 ​`KUBECONFIG` ​环境变量的值：

`unset KUBECONFIG`

或者将其设置为默认的 ​`KUBECONFIG` ​位置：

`export KUBECONFIG=/etc/kubernetes/admin.conf`

*   另一个方法是覆盖 ​`kubeconfig` ​的现有用户 "管理员"：

`mv  $HOME/.kube $HOME/.kube.bak mkdir $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config`

Kubelet 客户端证书轮换失败 
------------------

默认情况下，kubeadm 使用 ​`/etc/kubernetes/kubelet.conf`​ 中指定的 ​`/var/lib/kubelet/pki/kubelet-client-current.pem`​ 符号链接 来配置 kubelet 自动轮换客户端证书。如果此轮换过程失败，你可能会在 kube-apiserver 日志中看到 诸如 ​`x509: certificate has expired or is not yet valid`​ 之类的错误。要解决此问题，你必须执行以下步骤：

1.  从故障节点备份和删除 ​`/etc/kubernetes/kubelet.conf`​ 和 ​`/var/lib/kubelet/pki/kubelet-client*`​。
2.  在集群中具有 ​`/etc/kubernetes/pki/ca.key`​ 的、正常工作的控制平面节点上 执行 ​`kubeadm kubeconfig user --org system:nodes --client-name system:node:$NODE > kubelet.conf`​。 ​`$NODE`​ 必须设置为集群中现有故障节点的名称。 手动修改生成的 ​`kubelet.conf`​ 以调整集群名称和服务器端点， 或传递 ​`kubeconfig user --config`​（此命令接受 ​`InitConfiguration`​）。 如果你的集群没有 ​`ca.key`​，你必须在外部对 ​`kubelet.conf`​ 中的嵌入式证书进行签名。
3.  将得到的 ​`kubelet.conf`​ 文件复制到故障节点上，作为 ​`/etc/kubernetes/kubelet.conf`​。
4.  在故障节点上重启 kubelet（​`systemctl restart kubelet`​），等待 ​`/var/lib/kubelet/pki/kubelet-client-current.pem`​ 重新创建。
5.  手动编辑 ​`kubelet.conf`​ 指向轮换的 kubelet 客户端证书，方法是将 ​`client-certificate-data`​ 和 ​`client-key-data`​ 替换为：

`client-certificate: /var/lib/kubelet/pki/kubelet-client-current.pem client-key: /var/lib/kubelet/pki/kubelet-client-current.pem`

7.  重新启动 kubelet。
8.  确保节点状况变为 ​`Ready`​。

在 Vagrant 中使用 flannel 作为 pod 网络时的默认 NIC
---------------------------------------

以下错误可能表明 Pod 网络中出现问题：

`Error from server (NotFound): the server could not find the requested resource`

*   如果你正在 Vagrant 中使用 flannel 作为 pod 网络，则必须指定 flannel 的默认接口名称。

Vagrant 通常为所有 VM 分配两个接口。第一个为所有主机分配了 IP 地址 ​`10.0.2.15`​，用于获得 NATed 的外部流量。

这可能会导致 flannel 出现问题，它默认为主机上的第一个接口。这导致所有主机认为它们具有 相同的公共 IP 地址。为防止这种情况，传递 ​`--iface eth1`​ 标志给 flannel 以便选择第二个接口。

容器使用的非公共 IP
-----------

在某些情况下 ​`kubectl logs`​ 和 ​`kubectl run`​ 命令或许会返回以下错误，即便除此之外集群一切功能正常：

`Error from server: Get https://10.19.0.41:10250/containerLogs/default/mysql-ddc65b868-glc5m/mysql: dial tcp 10.19.0.41:10250: getsockopt: no route to host`

*   这或许是由于 Kubernetes 使用的 IP 无法与看似相同的子网上的其他 IP 进行通信的缘故， 可能是由机器提供商的政策所导致的。
*   DigitalOcean 既分配一个共有 IP 给 ​`eth0`​，也分配一个私有 IP 在内部用作其浮动 IP 功能的锚点， 然而 ​`kubelet` ​将选择后者作为节点的 ​`InternalIP` ​而不是公共 IP

使用 ​`ip addr show`​ 命令代替 ​`ifconfig` ​命令去检查这种情况，因为 ​`ifconfig` ​命令 不会显示有问题的别名 IP 地址。或者指定的 DigitalOcean 的 API 端口允许从 droplet 中 查询 anchor IP：

`curl http://169.254.169.254/metadata/v1/interfaces/public/0/anchor_ipv4/address`

解决方法是通知 ​`kubelet` ​使用哪个 ​`--node-ip`​。当使用 DigitalOcean 时，可以是公网IP（分配给 ​`eth0` ​的）， 或者是私网IP（分配给 ​`eth1` ​的）。私网 IP 是可选的。 kubadm ​`NodeRegistrationOptions` ​结构 的 ​`KubeletExtraArgs` ​部分被用来处理这种情况。

然后重启 ​`kubelet`​：

`systemctl daemon-reload systemctl restart kubelet`

coredns pods 有 CrashLoopBackOff 或者 Error 状态
-------------------------------------------

如果有些节点运行的是旧版本的 Docker，同时启用了 SELinux，你或许会遇到 ​`coredns` ​pods 无法启动的情况。 要解决此问题，你可以尝试以下选项之一：

*   升级到 Docker 的较新版本
*   [禁用 SELinux](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/security-enhanced_linux/sect-security-enhanced_linux-enabling_and_disabling_selinux-disabling_selinux)
*   修改 ​`coredns` ​部署以设置 ​`allowPrivilegeEscalation` ​为 ​`true`​：

`kubectl -n kube-system get deployment coredns -o yaml | \   sed 's/allowPrivilegeEscalation: false/allowPrivilegeEscalation: true/g' | \   kubectl apply -f -`

CoreDNS 处于 ​`CrashLoopBackOff` ​时的另一个原因是当 Kubernetes 中部署的 CoreDNS Pod 检测 到环路时。[有许多解决方法](https://github.com/coredns/coredns/tree/master/plugin/loop target=) 可以避免在每次 CoreDNS 监测到循环并退出时，Kubernetes 尝试重启 CoreDNS Pod 的情况。

> Warning: 禁用 SELinux 或设置 ​`allowPrivilegeEscalation` ​为 ​`true` ​可能会损害集群的安全性。

etcd pods 持续重启
--------------

如果你遇到以下错误：

`rpc error: code = 2 desc = oci runtime error: exec failed: container_linux.go:247: starting container process caused "process_linux.go:110: decoding init error from pipe caused \"read parent: connection reset by peer\""`

如果你使用 Docker 1.13.1.84 运行 CentOS 7 就会出现这种问题。 此版本的 Docker 会阻止 kubelet 在 etcd 容器中执行。

为解决此问题，请选择以下选项之一：

*   回滚到早期版本的 Docker，例如 1.13.1-75

`yum downgrade docker-1.13.1-75.git8633870.el7.centos.x86_64 docker-client-1.13.1-75.git8633870.el7.centos.x86_64 docker-common-1.13.1-75.git8633870.el7.centos.x86_64`

*   安装较新的推荐版本之一，例如 18.06:

`sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo yum install docker-ce-18.06.1.ce-3.el7.x86_64`

无法将以逗号分隔的值列表传递给 --component-extra-args 标志内的参数
---------------------------------------------

​`kubeadm init`​ 标志例如 ​`--component-extra-args`​ 允许你将自定义参数传递给像 kube-apiserver 这样的控制平面组件。然而，由于解析 (​`mapStringString`​) 的基础类型值，此机制将受到限制。

如果你决定传递一个支持多个逗号分隔值（例如 ​`--apiserver-extra-args "enable-admission-plugins=LimitRanger,NamespaceExists"`​）参数， 将出现 ​`flag: malformed pair, expect string=string`​ 错误。 发生这种问题是因为参数列表 ​`--apiserver-extra-args`​ 预期的是 ​`key=value`​ 形式， 而这里的 ​`NamespacesExists`​ 被误认为是缺少取值的键名。

一种解决方法是尝试分离 ​`key=value`​ 对，像这样： ​`--apiserver-extra-args "enable-admission-plugins=LimitRanger,enable-admission-plugins=NamespaceExists"`​ 但这将导致键 ​`enable-admission-plugins`​ 仅有值 ​`NamespaceExists`​。

已知的解决方法是使用 kubeadm 配置文件。

在节点被云控制管理器初始化之前，kube-proxy 就被调度了
--------------------------------

在云环境场景中，可能出现在云控制管理器完成节点地址初始化之前，kube-proxy 就被调度到新节点了。 这会导致 kube-proxy 无法正确获取节点的 IP 地址，并对管理负载平衡器的代理功能产生连锁反应。

在 kube-proxy Pod 中可以看到以下错误：

`server.go:610] Failed to retrieve node IP: host IP unknown; known addresses: [] proxier.go:340] invalid nodeIP, initializing kube-proxy with 127.0.0.1 as nodeIP`

一种已知的解决方案是修补 kube-proxy DaemonSet，以允许在控制平面节点上调度它， 而不管它们的条件如何，将其与其他节点保持隔离，直到它们的初始保护条件消除：

`kubectl -n kube-system patch ds kube-proxy -p='{ "spec": { "template": { "spec": { "tolerations": [ { "key": "CriticalAddonsOnly", "operator": "Exists" }, { "effect": "NoSchedule", "key": "node-role.kubernetes.io/master" }, { "effect": "NoSchedule", "key": "node-role.kubernetes.io/control-plane" } ] } } } }'`

此问题的跟踪[在这里](https://github.com/kubernetes/kubeadm/issues/1027)。

节点上的 /usr 被以只读方式挂载
------------------

在类似 Fedora CoreOS 或者 Flatcar Container Linux 这类 Linux 发行版本中， 目录 ​`/usr`​ 是以只读文件系统的形式挂载的。 在支持 [FlexVolume](https://github.com/kubernetes/community/blob/ab55d85/contributors/devel/sig-storage/flexvolume.md)时， 类似 kubelet 和 kube-controller-manager 这类 Kubernetes 组件使用默认路径 ​`/usr/libexec/kubernetes/kubelet-plugins/volume/exec/`​， 而 FlexVolume 的目录 必须是可写入的，该功能特性才能正常工作。 （注意：FlexVolume 在 Kubernetes v1.23 版本中已被弃用）

为了解决这个问题，你可以使用 kubeadm 的配置文件 来配置 FlexVolume 的目录。

在（使用 ​`kubeadm init`​ 创建的）主控制节点上，使用 ​`--config`​ 参数传入如下文件：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: InitConfiguration nodeRegistration:   kubeletExtraArgs:     volume-plugin-dir: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/" --- apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration controllerManager:   extraArgs:     flex-volume-plugin-dir: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/"`

在加入到集群中的节点上，使用下面的文件：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: JoinConfiguration nodeRegistration:   kubeletExtraArgs:     volume-plugin-dir: "/opt/libexec/kubernetes/kubelet-plugins/volume/exec/"`

或者，你要可以更改 ​`/etc/fstab`​ 使得 ​`/usr`​ 目录能够以可写入的方式挂载，不过 请注意这样做本质上是在更改 Linux 发行版的某种设计原则。

kubeadm upgrade plan 输出错误信息 context deadline exceeded
-----------------------------------------------------

在使用 ​`kubeadm` ​来升级某运行外部 etcd 的 Kubernetes 集群时可能显示这一错误信息。 这并不是一个非常严重的一个缺陷，之所以出现此错误信息，原因是老的 kubeadm 版本会对外部 etcd 集群执行版本检查。你可以继续执行 ​`kubeadm upgrade apply ...`​。

这一问题已经在 1.19 版本中得到修复。

kubeadm reset 会卸载 /var/lib/kubelet
----------------------------------

如果已经挂载了 ​`/var/lib/kubelet`​ 目录，执行 ​`kubeadm reset`​ 操作的时候 会将其卸载。

要解决这一问题，可以在执行了 ​`kubeadm reset`​ 操作之后重新挂载 ​`/var/lib/kubelet`​ 目录。

这是一个在 1.15 中引入的故障，已经在 1.20 版本中修复。

无法在 kubeadm 集群中安全地使用 metrics-server 
------------------------------------

在 kubeadm 集群中可以通过为 [metrics-server](https://github.com/kubernetes-sigs/metrics-server) 设置 ​`--kubelet-insecure-tls`​ 来以不安全的形式使用该服务。 建议不要在生产环境集群中这样使用。

如果你需要在 metrics-server 和 kubelet 之间使用 TLS，会有一个问题， kubeadm 为 kubelet 部署的是自签名的服务证书。这可能会导致 metrics-server 端报告下面的错误信息：

`x509: certificate signed by unknown authority x509: certificate is valid for IP-foo not IP-bar`

另请参阅 [How to run the metrics-server securely](https://github.com/kubernetes-sigs/metrics-server/blob/master/FAQ.md target=)。

###  3.  Kubernetes 使用kubeadm创建集群
使用 kubeadm 创建集群
---------------

使用 ​`kubeadm`​，你能创建一个符合最佳实践的最小化 Kubernetes 集群。 事实上，你可以使用 ​`kubeadm` ​配置一个通过 Kubernetes 一致性测试的集群。 ​`kubeadm` ​还支持其他集群生命周期功能， 例如启动引导令牌和集群升级。

kubeadm 工具很棒，如果你需要：

*   一个尝试 Kubernetes 的简单方法。
*   一个现有用户可以自动设置集群并测试其应用程序的途径。
*   其他具有更大范围的生态系统和/或安装工具中的构建模块。

你可以在各种机器上安装和使用 ​`kubeadm`​：笔记本电脑， 一组云服务器，Raspberry Pi 等。无论是部署到云还是本地， 你都可以将 ​`kubeadm` ​集成到预配置系统中，例如 Ansible 或 Terraform。

在开始之前
-----

要遵循本指南，你需要：

*   一台或多台运行兼容 deb/rpm 的 Linux 操作系统的计算机；例如：Ubuntu 或 CentOS。
*   每台机器 2 GB 以上的内存，内存不足时应用会受限制。
*   用作控制平面节点的计算机上至少有2个 CPU。
*   集群中所有计算机之间具有完全的网络连接。你可以使用公共网络或专用网络。

你还需要使用可以在新集群中部署特定 Kubernetes 版本对应的 ​`kubeadm`​。

Kubernetes 版本及版本偏差策略适用于 ​`kubeadm` ​以及整个 Kubernetes。 查阅该策略以了解支持哪些版本的 Kubernetes 和 ​`kubeadm`​。 该页面是为 Kubernetes v1.24 编写的。

​`kubeadm` ​工具的整体功能状态为一般可用性（GA）。一些子功能仍在积极开发中。 随着工具的发展，创建集群的实现可能会略有变化，但总体实现应相当稳定。

> Note: 根据定义，在 ​`kubeadm alpha`​ 下的所有命令均在 alpha 级别上受支持。

目标
--

*   安装单个控制平面的 Kubernetes 集群
*   在集群上安装 Pod 网络，以便你的 Pod 可以相互连通

操作指南
----

#### 主机准备

在所有主机上安装 容器运行时 和 kubeadm。

> Note:  
> 如果你已经安装了kubeadm，执行 ​`apt-get update && apt-get upgrade`​ 或 ​`yum update`​ 以获取 kubeadm 的最新版本。  
> 升级时，kubelet 每隔几秒钟重新启动一次， 在 crashloop 状态中等待 kubeadm 发布指令。crashloop 状态是正常现象。 初始化控制平面后，kubelet 将正常运行。

#### 准备所需的容器镜像 

这个步骤是可选的，只适用于你希望 ​`kubeadm init`​ 和 ​`kubeadm join`​ 不去下载存放在 ​`k8s.gcr.io`​ 上的默认的容器镜像的情况。

当你在离线的节点上创建一个集群的时候，Kubeadm 有一些命令可以帮助你预拉取所需的镜像。

Kubeadm 允许你给所需要的镜像指定一个自定义的镜像仓库。

#### 初始化控制平面节点

控制平面节点是运行控制平面组件的机器， 包括 etcd （集群数据库） 和 API Server （命令行工具 kubectl 与之通信）。

1.  （推荐）如果计划将单个控制平面 kubeadm 集群升级成高可用， 你应该指定 ​`--control-plane-endpoint`​ 为所有控制平面节点设置共享端点。 端点可以是负载均衡器的 DNS 名称或 IP 地址。
2.  选择一个 Pod 网络插件，并验证是否需要为 ​`kubeadm init`​ 传递参数。 根据你选择的第三方网络插件，你可能需要设置 ​`--pod-network-cidr`​ 的值。 
3.  （可选）​`kubeadm` ​试图通过使用已知的端点列表来检测容器运行时。 使用不同的容器运行时或在预配置的节点上安装了多个容器运行时，请为 ​`kubeadm init`​ 指定 ​`--cri-socket`​ 参数。
4.  （可选）除非另有说明，否则 ​`kubeadm` ​使用与默认网关关联的网络接口来设置此控制平面节点 API server 的广播地址。 要使用其他网络接口，请为 ​`kubeadm init`​ 设置 ​`--apiserver-advertise-address=<ip-address>`​ 参数。 要部署使用 IPv6 地址的 Kubernetes 集群， 必须指定一个 IPv6 地址，例如 ​`--apiserver-advertise-address=fd00::101`​

要初始化控制平面节点，请运行：

`kubeadm init <args>`

#### 关于 apiserver-advertise-address 和 ControlPlaneEndpoint 的注意事项 

​`--apiserver-advertise-address`​ 可用于为控制平面节点的 API server 设置广播地址， ​`--control-plane-endpoint`​ 可用于为所有控制平面节点设置共享端点。

​`--control-plane-endpoint`​ 允许 IP 地址和可以映射到 IP 地址的 DNS 名称。 请与你的网络管理员联系，以评估有关此类映射的可能解决方案。

这是一个示例映射：

`192.168.0.102 cluster-endpoint`

其中 ​`192.168.0.102`​ 是此节点的 IP 地址，​`cluster-endpoint`​ 是映射到该 IP 的自定义 DNS 名称。 这将允许你将 ​`--control-plane-endpoint=cluster-endpoint`​ 传递给 ​`kubeadm init`​，并将相同的 DNS 名称传递给 ​`kubeadm join`​。 稍后你可以修改 ​`cluster-endpoint`​ 以指向高可用性方案中的负载均衡器的地址。

kubeadm 不支持将没有 ​`--control-plane-endpoint`​ 参数的单个控制平面集群转换为高可用性集群。

#### 更多信息

要再次运行 ​`kubeadm init`​，你必须首先卸载集群。

如果将具有不同架构的节点加入集群， 请确保已部署的 DaemonSet 对这种体系结构具有容器镜像支持。

​`kubeadm init`​ 首先运行一系列预检查以确保机器 准备运行 Kubernetes。这些预检查会显示警告并在错误时退出。然后 ​`kubeadm init`​ 下载并安装集群控制平面组件。这可能会需要几分钟。 完成之后你应该看到：

`Your Kubernetes control-plane has initialized successfully!  To start using your cluster, you need to run the following as a regular user:    mkdir -p $HOME/.kube   sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config   sudo chown $(id -u):$(id -g) $HOME/.kube/config  You should now deploy a Pod network to the cluster. Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:   /docs/concepts/cluster-administration/addons/  You can now join any number of machines by running the following on each node as root:    kubeadm join <control-plane-host>:<control-plane-port> --token <token> --discovery-token-ca-cert-hash sha256:<hash>`

要使非 root 用户可以运行 kubectl，请运行以下命令， 它们也是 ​`kubeadm init`​ 输出的一部分：

`mkdir -p $HOME/.kube sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config sudo chown $(id -u):$(id -g) $HOME/.kube/config`

或者，如果你是 ​`root` ​用户，则可以运行：

`export KUBECONFIG=/etc/kubernetes/admin.conf`

> Warning:  
> kubeadm 对 ​`admin.conf`​ 中的证书进行签名时，将其配置为 ​`Subject: O = system:masters, CN = kubernetes-admin`​。 ​`system:masters`​ 是一个例外的、超级用户组，可以绕过鉴权层（例如 RBAC）。 不要将 ​`admin.conf`​ 文件与任何人共享，应该使用 ​`kubeadm kubeconfig user`​ 命令为其他用户生成 kubeconfig 文件，完成对他们的定制授权。

记录 ​`kubeadm init`​ 输出的 ​`kubeadm join`​ 命令。 你需要此命令将节点加入集群。

令牌用于控制平面节点和加入节点之间的相互身份验证。 这里包含的令牌是密钥。确保它的安全， 因为拥有此令牌的任何人都可以将经过身份验证的节点添加到你的集群中。 可以使用 ​`kubeadm token`​ 命令列出，创建和删除这些令牌。

#### 安装 Pod 网络附加组件 

> Caution:  
> 本节包含有关网络设置和部署顺序的重要信息。 在继续之前，请仔细阅读所有建议。  
> 你必须部署一个基于 Pod 网络插件的 容器网络接口 (CNI)，以便你的 Pod 可以相互通信。 在安装网络之前，集群 DNS (CoreDNS) 将不会启动。  
> 
> *   注意你的 Pod 网络不得与任何主机网络重叠： 如果有重叠，你很可能会遇到问题。 （如果你发现网络插件的首选 Pod 网络与某些主机网络之间存在冲突， 则应考虑使用一个合适的 CIDR 块来代替， 然后在执行 ​`kubeadm init`​ 时使用 ​`--pod-network-cidr`​ 参数并在你的网络插件的 YAML 中替换它）。
> *   默认情况下，​`kubeadm` ​将集群设置为使用和强制使用 RBAC（基于角色的访问控制）。 确保你的 Pod 网络插件支持 RBAC，以及用于部署它的 manifests 也是如此。
> *   如果要为集群使用 IPv6（双协议栈或仅单协议栈 IPv6 网络）， 请确保你的 Pod 网络插件支持 IPv6。 IPv6 支持已在 CNI [v0.6.0](https://github.com/containernetworking/cni/releases/tag/v0.6.0) 版本中添加。

> Note: kubeadm 应该是与 CNI 无关的，对 CNI 驱动进行验证目前不在我们的端到端测试范畴之内。 如果你发现与 CNI 插件相关的问题，应在其各自的问题跟踪器中记录而不是在 kubeadm 或 kubernetes 问题跟踪器中记录。

一些外部项目为 Kubernetes 提供使用 CNI 的 Pod 网络，其中一些还支持网络策略。

你可以使用以下命令在控制平面节点或具有 kubeconfig 凭据的节点上安装 Pod 网络附加组件：

`kubectl apply -f <add-on.yaml>`

每个集群只能安装一个 Pod 网络。

安装 Pod 网络后，你可以通过在 ​`kubectl get pods --all-namespaces`​ 输出中检查 CoreDNS Pod 是否 ​`Running` ​来确认其是否正常运行。 一旦 CoreDNS Pod 启用并运行，你就可以继续加入节点。

如果你的网络无法正常工作或 CoreDNS 不在“运行中”状态，请查看 ​`kubeadm` ​的 故障排除指南。

#### 托管节点标签 

默认情况下，kubeadm 启用 NodeRestriction 准入控制器来限制 kubelets 在节点注册时可以应用哪些标签。准入控制器文档描述 kubelet ​`--node-labels`​ 选项允许使用哪些标签。 其中 ​`node-role.kubernetes.io/control-plane`​ 标签就是这样一个受限制的标签， kubeadm 在节点创建后使用特权客户端手动应用此标签。 你可以使用一个有特权的 kubeconfig，比如由 kubeadm 管理的 ​`/etc/kubernetes/admin.conf`​， 通过执行 ​`kubectl label`​ 来手动完成操作。

#### 控制平面节点隔离

默认情况下，出于安全原因，你的集群不会在控制平面节点上调度 Pod。 如果你希望能够在控制平面节点上调度 Pod，例如单机 Kubernetes 集群，请运行:

`kubectl taint nodes --all node-role.kubernetes.io/control-plane- node-role.kubernetes.io/master-`

输出看起来像：

`node "test-01" untainted`

这将从任何拥有 ​`node-role.kubernetes.io/control-plane`​ 和 ​`node-role.kubernetes.io/master`​ 污点的节点上移除该污点。

包括控制平面节点，这意味着调度程序将能够在任何地方调度 Pods。

> Note: ​`node-role.kubernetes.io/master`​ 污点已被废弃，kubeadm 将在 1.25 版本中停止使用它。

#### 加入节点

节点是你的工作负载（容器和 Pod 等）运行的地方。要将新节点添加到集群，请对每台计算机执行以下操作：

*   SSH 到机器
*   成为 root （例如 ​`sudo su -`​）
*   运行 ​`kubeadm init`​ 输出的命令，例如：

`kubeadm join --token <token> <control-plane-host>:<control-plane-port> --discovery-token-ca-cert-hash sha256:<hash>`

如果没有令牌，可以通过在控制平面节点上运行以下命令来获取令牌：

`kubeadm token list`

输出类似于以下内容：

`TOKEN                    TTL  EXPIRES              USAGES           DESCRIPTION            EXTRA GROUPS 8ewj1p.9r9hcjoqgajrj4gi  23h  2018-06-12T02:51:28Z authentication,  The default bootstrap  system:                                                    signing          token generated by     bootstrappers:                                                                     'kubeadm init'.        kubeadm:                                                                                            default-node-token`

默认情况下，令牌会在 24 小时后过期。如果要在当前令牌过期后将节点加入集群， 则可以通过在控制平面节点上运行以下命令来创建新令牌：

`kubeadm token create`

输出类似于以下内容：

`5didvk.d09sbcov8ph2amjw`

如果你没有 ​`--discovery-token-ca-cert-hash`​ 的值，则可以通过在控制平面节点上执行以下命令链来获取它：

`openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | \    openssl dgst -sha256 -hex | sed 's/^.* //'`

输出类似于以下内容：

`8cb2de97839780a412b93877f8507ad6c94f73add17d5d7058e91741c9d5ec78`

> Note: 要为 ​`<control-plane-host>:<control-plane-port>`​ 指定 IPv6 元组，必须将 IPv6 地址括在方括号中，例如：​`[fd00::101]:2073`​

输出应类似于：

`[preflight] Running pre-flight checks  ... (log output of join workflow) ...  Node join complete: * Certificate signing request sent to control-plane and response   received. * Kubelet informed of new secure connection details.  Run 'kubectl get nodes' on control-plane to see this machine join.`

几秒钟后，当你在控制平面节点上执行 ​`kubectl get nodes`​，你会注意到该节点出现在输出中。

> Note: 由于集群节点通常是按顺序初始化的，CoreDNS Pods 很可能都运行在第一个控制面节点上。 为了提供更高的可用性，请在加入至少一个新节点后 使用 ​`kubectl -n kube-system rollout restart deployment coredns`​ 命令，重新平衡 CoreDNS Pods。

#### （可选）从控制平面节点以外的计算机控制集群

为了使 kubectl 在其他计算机（例如笔记本电脑）上与你的集群通信， 你需要将管理员 kubeconfig 文件从控制平面节点复制到工作站，如下所示：

`scp root@<control-plane-host>:/etc/kubernetes/admin.conf . kubectl --kubeconfig ./admin.conf get nodes`

> Note:  
> 上面的示例假定为 root 用户启用了 SSH 访问。如果不是这种情况， 你可以使用 ​`scp`​ 将 ​`admin.conf`​ 文件复制给其他允许访问的用户。  
> admin.conf 文件为用户提供了对集群的超级用户特权。 该文件应谨慎使用。对于普通用户，建议生成一个你为其授予特权的唯一证书。 你可以使用 ​`kubeadm alpha kubeconfig user --client-name <CN>`​ 命令执行此操作。 该命令会将 KubeConfig 文件打印到 STDOUT，你应该将其保存到文件并分发给用户。 之后，使用 ​`kubectl create (cluster)rolebinding`​ 授予特权。

#### （可选）将 API 服务器代理到本地主机

如果要从集群外部连接到 API 服务器，则可以使用 ​`kubectl proxy`​：

`scp root@<control-plane-host>:/etc/kubernetes/admin.conf . kubectl --kubeconfig ./admin.conf proxy`

你现在可以在本地访问 API 服务器 ​`http://localhost:8001/api/v1`​。

清理
--

如果你在集群中使用了一次性服务器进行测试，则可以关闭这些服务器，而无需进一步清理。你可以使用 ​`kubectl config delete-cluster`​ 删除对集群的本地引用。

但是，如果要更干净地取消配置集群， 则应首先[清空节点](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands target=)并确保该节点为空， 然后取消配置该节点。

#### 删除节点

使用适当的凭证与控制平面节点通信，运行：

`kubectl drain <node name> --delete-emptydir-data --force --ignore-daemonsets`

在删除节点之前，请重置 ​`kubeadm` ​安装的状态：

`kubeadm reset`

重置过程不会重置或清除 iptables 规则或 IPVS 表。如果你希望重置 iptables，则必须手动进行：

`iptables -F && iptables -t nat -F && iptables -t mangle -F && iptables -X`

如果要重置 IPVS 表，则必须运行以下命令：

`ipvsadm -C`

现在删除节点：

`kubectl delete node <node name>`

如果你想重新开始，只需运行 ​`kubeadm init`​ 或 ​`kubeadm join`​ 并加上适当的参数。

#### 清理控制平面

你可以在控制平面主机上使用 ​`kubeadm reset`​ 来触发尽力而为的清理。

版本偏差策略
------

虽然 kubeadm 允许所管理的组件有一定程度的版本偏差， 但是建议你将 kubeadm 的版本与控制平面组件、kube-proxy 和 kubelet 的版本相匹配。

#### kubeadm 中的 Kubernetes 版本偏差

kubeadm 可以与 Kubernetes 组件一起使用，这些组件的版本与 kubeadm 相同，或者比它大一个版本。 Kubernetes 版本可以通过使用 ​`--kubeadm init`​ 的 ​`--kubernetes-version`​ 标志或使用 ​`--config`​ 时的 ​`ClusterConfiguration.kubernetesVersion`​ 字段指定给 kubeadm。 这个选项将控制 kube-apiserver、kube-controller-manager、kube-scheduler 和 kube-proxy 的版本。

例如：

*   kubeadm 的版本为 1.24。
*   ​`kubernetesVersion` ​必须为 1.24 或者 1.23。

#### kubeadm 中 kubelet 的版本偏差

与 Kubernetes 版本类似，kubeadm 可以使用与 kubeadm 相同版本的 kubelet， 或者比 kubeadm 老一个版本的 kubelet。

例如：

*   kubeadm 的版本为 1.24
*   主机上的 kubelet 版本必须为 1.24 或者 1.23

#### kubeadm 支持的 kubeadm 的版本偏差

kubeadm 命令在现有节点或由 kubeadm 管理的整个集群上的操作有一定限制。

如果新的节点加入到集群中，用于 ​`kubeadm join`​ 的 kubeadm 二进制文件必须与用 ​`kubeadm init`​ 创建集群或用 ​`kubeadm upgrade`​ 升级同一节点时所用的 kubeadm 版本一致。 类似的规则适用于除了 ​`kubeadm upgrade`​ 以外的其他 kubeadm 命令。

​`kubeadm join`​ 的例子：

*   使用 ​`kubeadm init`​ 创建集群时使用版本为 1.24 的 kubeadm。
*   加入的节点必须使用版本为 1.24 的 kubeadm 二进制文件。

对于正在升级的节点，所使用的的 kubeadm 必须与管理该节点的 kubeadm 具有相同的 MINOR 版本或比后者新一个 MINOR 版本。

​`kubeadm upgrade`​ 的例子:

*   用于创建或升级节点的 kubeadm 版本为 1.23。
*   用于升级节点的 kubeadm 版本必须为 1.23 或 1.24。

局限性
---

#### 集群弹性

此处创建的集群具有单个控制平面节点，运行单个 etcd 数据库。 这意味着如果控制平面节点发生故障，你的集群可能会丢失数据并且可能需要从头开始重新创建。

解决方法：

*   定期[备份 etcd](https://etcd.io/)。 kubeadm 配置的 etcd 数据目录位于控制平面节点上的 ​`/var/lib/etcd`​ 中。
*   使用多个控制平面节点。你可以阅读 可选的高可用性拓扑选择集群拓扑提供的 高可用性。

#### 平台兼容性

kubeadm deb/rpm 软件包和二进制文件是为 amd64、arm (32-bit)、arm64、ppc64le 和 s390x 构建的遵循[多平台提案](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/multi-platform.md)。

从 v1.12 开始还支持用于控制平面和附加组件的多平台容器镜像。

只有一些网络提供商为所有平台提供解决方案。请查阅上方的网络提供商清单或每个提供商的文档以确定提供商是否支持你选择的平台。

> Note:要为<control-plane-host>:<control-plane-port>指定IPv6元组，必须将IPv6地址括在方括号中，例如：\[fd00::101\]:2073

###  4.  Kubernetes 使用kubeadm API定制组件
使用 kubeadm API 定制组件
-------------------

本页面介绍了如何自定义 kubeadm 部署的组件。 你可以使用 ​`ClusterConfiguration` ​结构中定义的参数，或者在每个节点上应用补丁来定制控制平面组件。 你可以使用 ​`KubeletConfiguration` ​和 ​`KubeProxyConfiguration` ​结构分别定制 kubelet 和 kube-proxy 组件。

所有这些选项都可以通过 kubeadm 配置 API 实现。

> Note:  
> kubeadm 目前不支持对 CoreDNS 部署进行定制。 你必须手动更新 ​`kube-system/coredns`​ ConfigMap 并在更新后重新创建 CoreDNS Pods。 或者，你可以跳过默认的 CoreDNS 部署并部署你自己的 CoreDNS 变种。

使用 ClusterConfiguration 中的标志自定义控制平面 
------------------------------------

kubeadm ​`ClusterConfiguration` ​对象为用户提供了一种方法， 用以覆盖传递给控制平面组件（如 APIServer、ControllerManager、Scheduler 和 Etcd）的默认参数。 各组件配置使用如下字段定义：

*   ​`apiServer` ​
*   ​`controllerManager` ​
*   ​`scheduler` ​
*   ​`etcd`​

这些结构包含一个通用的 ​`extraArgs` ​字段，该字段由 ​`key: value`​ 组成。 要覆盖控制平面组件的参数：

1.  将适当的字段 ​`extraArgs` ​添加到配置中。
2.  向字段 ​`extraArgs` ​添加要覆盖的参数值。
3.  用 ​`--config <YOUR CONFIG YAML>`​ 运行 ​`kubeadm init`​。

> Note:  
> 你可以通过运行 ​`kubeadm config print init-defaults`​ 并将输出保存到你所选的文件中， 以默认值形式生成 ​`ClusterConfiguration` ​对象。

> Note:  
> ​`ClusterConfiguration` ​对象目前在 kubeadm 集群中是全局的。 这意味着你添加的任何标志都将应用于同一组件在不同节点上的所有实例。 要在不同节点上为每个组件应用单独的配置，你可以使用补丁。

> Note:  
> 当前不支持重复的参数（keys）或多次传递相同的参数 ​`--foo`​。 要解决此问题，你必须使用补丁。

#### APIServer 参数 

使用示例：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration kubernetesVersion: v1.16.0 apiServer:   extraArgs:     anonymous-auth: "false"     enable-admission-plugins: AlwaysPullImages,DefaultStorageClass     audit-log-path: /home/johndoe/audit.log`

#### ControllerManager 参数 

使用示例：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration kubernetesVersion: v1.16.0 controllerManager:   extraArgs:     cluster-signing-key-file: /home/johndoe/keys/ca.key     deployment-controller-sync-period: "50"`

Scheduler 参数 
-------------

使用示例：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration kubernetesVersion: v1.16.0 scheduler:   extraArgs:     config: /etc/kubernetes/scheduler-config.yaml   extraVolumes:     - name: schedulerconfig       hostPath: /home/johndoe/schedconfig.yaml       mountPath: /etc/kubernetes/scheduler-config.yaml       readOnly: true       pathType: "File"`

#### Etcd 参数 

使用示例：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration etcd:   local:     extraArgs:       election-timeout: 1000`

使用补丁定制控制平面 
-----------

FEATURE STATE: Kubernetes v1.22 \[beta\]

Kubeadm 允许将包含补丁文件的目录传递给各个节点上的 ​`InitConfiguration` ​和 ​`JoinConfiguration`​。 这些补丁可被用作控制平面组件清单写入磁盘之前的最后一个自定义步骤。

可以使用 ​`--config <你的 YAML 格式控制文件>`​ 将配置文件传递给 ​`kubeadm init`​：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: InitConfiguration patches:   directory: /home/user/somedir`

> Note:  
> 对于 ​`kubeadm init`​，你可以传递一个包含 ​`ClusterConfiguration` ​和 ​`InitConfiguration` ​的文件，以 ​`---`​ 分隔。

你可以使用 ​`--config <你的 YAML 格式配置文件>`​ 将配置文件传递给 ​`kubeadm join`​：

`apiVersion: kubeadm.k8s.io/v1beta3 kind: JoinConfiguration patches:   directory: /home/user/somedir`

补丁目录必须包含名为 ​`target[suffix][+patchtype].extension`​ 的文件。 例如，​`kube-apiserver0+merge.yaml`​ 或只是 ​`etcd.json`​。

*   ​`target` ​可以是 ​`kube-apiserver`​、​`kube-controller-manager`​、​`kube-scheduler`​ 和 ​`etcd` ​之一。
*   ​`patchtype` ​可以是 ​`strategy`​、​`merge` ​或 ​`json` ​之一，并且这些必须匹配 kubectl 支持 的补丁格式。 默认补丁类型是 ​`strategic` ​的。
*   ​`extension` ​必须是 ​`json` ​或 ​`yaml`​。
*   ​`suffix` ​是一个可选字符串，可用于确定首先按字母数字应用哪些补丁。

> Note:  
> 如果你使用 ​`kubeadm upgrade`​ 升级 kubeadm 节点，你必须再次提供相同的补丁，以便在升级后保留自定义配置。 为此，你可以使用 ​`--patches`​ 参数，该参数必须指向同一目录。 ​`kubeadm upgrade`​ 目前不支持用于相同目的的 API 结构配置。

自定义 kubelet 
------------

要自定义 kubelet，你可以在同一配置文件中的 ​`ClusterConfiguration` ​或 ​`InitConfiguration` ​之外添加一个 ​`KubeletConfiguration`​，用 ​`---`​ 分隔。 然后可以将此文件传递给 ​`kubeadm init`​。

> Note:  
> kubeadm 将相同的 ​`KubeletConfiguration` ​配置应用于集群中的所有节点。 要应用节点特定设置，你可以使用 ​`kubelet` ​参数进行覆盖，方法是将它们传递到 ​`InitConfiguration` ​和 ​`JoinConfiguration` ​支持的 ​`nodeRegistration.kubeletExtraArgs`​ 字段中。

自定义 kube-proxy 
---------------

要自定义 kube-proxy，你可以在 ​`ClusterConfiguration` ​或 ​`InitConfiguration` ​之外添加一个 由 ​`---`​ 分隔的 ​`KubeProxyConfiguration`​， 传递给 ​`kubeadm init`​。

> Note:  
> kubeadm 将 kube-proxy 部署为 DaemonSet， 这意味着 ​`KubeProxyConfiguration` ​将应用于集群中的所有 kube-proxy 实例。

###  5.  Kubernetes 高可用拓扑选项
高可用拓扑选项
-------

本页面介绍了配置高可用（HA）Kubernetes 集群拓扑的两个选项。

你可以设置 HA 集群：

*   使用堆叠（stacked）控制平面节点，其中 etcd 节点与控制平面节点共存
*   使用外部 etcd 节点，其中 etcd 在与控制平面不同的节点上运行

在设置 HA 集群之前，你应该仔细考虑每种拓扑的优缺点。

> Note:  
> kubeadm 静态引导 etcd 集群。 阅读 etcd [集群指南](https://github.com/etcd-io/etcd/blob/release-3.4/Documentation/op-guide/clustering.md target=)以获得更多详细信息。

堆叠（Stacked）etcd 拓扑 
-------------------

堆叠（Stacked）HA 集群是一种这样的[拓扑](https://en.wikipedia.org/wiki/Network_topology)， 其中 etcd 分布式数据存储集群堆叠在 kubeadm 管理的控制平面节点上，作为控制平面的一个组件运行。

每个控制平面节点运行 ​`kube-apiserver`​、​`kube-scheduler`​ 和 ​`kube-controller-manager`​ 实例。

​`kube-apiserver`​ 使用负载均衡器暴露给工作节点。

每个控制平面节点创建一个本地 etcd 成员（member），这个 etcd 成员只与该节点的 ​`kube-apiserver`​ 通信。 这同样适用于本地 ​`kube-controller-manager`​ 和 ​`kube-scheduler`​ 实例。

这种拓扑将控制平面和 etcd 成员耦合在同一节点上。相对使用外部 etcd 集群， 设置起来更简单，而且更易于副本管理。

然而，堆叠集群存在耦合失败的风险。如果一个节点发生故障，则 etcd 成员和控制平面实例都将丢失， 并且冗余会受到影响。你可以通过添加更多控制平面节点来降低此风险。

因此，你应该为 HA 集群运行至少三个堆叠的控制平面节点。

这是 kubeadm 中的默认拓扑。当使用 ​`kubeadm init`​ 和 ​`kubeadm join --control-plane`​ 时， 在控制平面节点上会自动创建本地 etcd 成员。

![](https://atts.w3cschool.cn/attachments/image/20220531/1653965800244371.svg)  

外部 etcd 拓扑 
-----------

具有外部 etcd 的 HA 集群是一种这样的[拓扑](https://zh.wikipedia.org/wiki/%E7%BD%91%E7%BB%9C%E6%8B%93%E6%89%91)， 其中 etcd 分布式数据存储集群在独立于控制平面节点的其他节点上运行。

就像堆叠的 etcd 拓扑一样，外部 etcd 拓扑中的每个控制平面节点都运行 ​`kube-apiserver`​，​`kube-scheduler`​ 和 ​`kube-controller-manager`​ 实例。 同样，​`kube-apiserver`​ 使用负载均衡器暴露给工作节点。但是 etcd 成员在不同的主机上运行， 每个 etcd 主机与每个控制平面节点的 ​`kube-apiserver`​ 通信。

这种拓扑结构解耦了控制平面和 etcd 成员。因此它提供了一种 HA 设置， 其中失去控制平面实例或者 etcd 成员的影响较小，并且不会像堆叠的 HA 拓扑那样影响集群冗余。

但此拓扑需要两倍于堆叠 HA 拓扑的主机数量。

具有此拓扑的 HA 集群至少需要三个用于控制平面节点的主机和三个用于 etcd 节点的主机。

![](https://atts.w3cschool.cn/attachments/image/20220531/1653965875455303.svg)

###  6.  Kubernetes 利用kubeadm创建高可用集群
利用 kubeadm 创建高可用集群
------------------

本文讲述了使用 kubeadm 设置一个高可用的 Kubernetes 集群的两种不同方式：

*   使用具有堆叠的控制平面节点。这种方法所需基础设施较少。etcd 成员和控制平面节点位于同一位置。
*   使用外部集群。这种方法所需基础设施较多。控制平面的节点和 etcd 成员是分开的。

如果你在安装 HA 集群时遇到问题，请在 kubeadm [问题跟踪](https://github.com/kubernetes/kubeadm/issues/new)里向我们提供反馈。

> Caution: 这篇文档没有讲述在云提供商上运行集群的问题。在云环境中，此处记录的方法不适用于类型为 LoadBalancer 的服务对象，或者具有动态的 PersistentVolumes。

在开始之前
-----

根据集群控制平面所选择的拓扑结构不同，准备工作也有所差异：

*   堆叠（Stacked） etcd 拓扑

需要准备：

*   配置满足 kubeadm 的最低要求 的三台机器作为控制面节点。奇数台控制平面节点有利于机器故障或者网络分区时进行重新选主。

*   机器已经安装好容器运行时，并正常运行

*   配置满足 kubeadm 的最低要求 的三台机器作为工作节点

*   机器已经安装好容器运行时，并正常运行

*   在集群中，确保所有计算机之间存在全网络连接（公网或私网）
*   在所有机器上具有 sudo 权限

*   可以使用其他工具；本教程以 ​`sudo` ​举例

*   从某台设备通过 SSH 访问系统中所有节点的能力
*   所有机器上已经安装 ​`kubeadm` ​和 ​`kubelet`​

*   外部 etcd 拓扑

需要准备：

*   配置满足 kubeadm 的最低要求 的三台机器作为控制面节点。奇数台控制平面节点有利于机器故障或者网络分区时进行重新选主。

*   机器已经安装好容器运行时，并正常运行

*   配置满足 kubeadm 的最低要求 的三台机器作为工作节点

*   机器已经安装好容器运行时，并正常运行

*   在集群中，确保所有计算机之间存在全网络连接（公网或私网）
*   在所有机器上具有 sudo 权限

*   可以使用其他工具；本教程以 ​`sudo` ​举例

*   从某台设备通过 SSH 访问系统中所有节点的能力
*   所有机器上已经安装 ​`kubeadm` ​和 ​`kubelet` ​

还需要准备：

*   给 etcd 集群使用的另外三台及以上机器。为了分布式一致性算法达到更好的投票效果，集群必须由奇数个节点组成。

*   机器上已经安装 ​`kubeadm` ​和 ​`kubelet`​。
*   机器上同样需要安装好容器运行时，并能正常运行。

#### 容器镜像

每台主机需要能够从 Kubernetes 容器镜像仓库（ ​`k8s.gcr.io`​ ）读取和拉取镜像。 想要在无法拉取 Kubernetes 仓库镜像的机器上部署高可用集群也是可行的。通过其他的手段保证主机上已经有对应的容器镜像即可。

#### 命令行 

一旦集群创建成功，需要在 PC 上安装 kubectl 用于管理 Kubernetes。为了方便故障排查，也可以在每个控制平面节点上安装 ​`kubectl`​。

这两种方法的第一步
---------

#### 为 kube-apiserver 创建负载均衡器

> Note: 使用负载均衡器需要许多配置。你的集群搭建可能需要不同的配置。 下面的例子只是其中的一方面配置。

1.  创建一个名为 kube-apiserver 的负载均衡器解析 DNS。

*   在云环境中，应该将控制平面节点放置在 TCP 转发负载平衡后面。 该负载均衡器将流量分配给目标列表中所有运行状况良好的控制平面节点。 API 服务器的健康检查是在 kube-apiserver 的监听端口（默认值 ​`:6443`​） 上进行的一个 TCP 检查。
*   不建议在云环境中直接使用 IP 地址。
*   负载均衡器必须能够在 API 服务器端口上与所有控制平面节点通信。 它还必须允许其监听端口的入站流量。
*   确保负载均衡器的地址始终匹配 kubeadm 的 ​`ControlPlaneEndpoint` ​地址。
*   阅读[软件负载平衡选项指南](https://github.com/kubernetes/kubeadm/blob/main/docs/ha-considerations.md target=) 以获取更多详细信息。

3.  添加第一个控制平面节点到负载均衡器并测试连接：

`nc -v LOAD_BALANCER_IP PORT`

由于 apiserver 尚未运行，预期会出现一个连接拒绝错误。 然而超时意味着负载均衡器不能和控制平面节点通信。 如果发生超时，请重新配置负载均衡器与控制平面节点进行通信。

6.  将其余控制平面节点添加到负载均衡器目标组。

使用堆控制平面和 etcd 节点
----------------

#### 控制平面节点的第一步

1.  初始化控制平面：

`sudo kubeadm init --control-plane-endpoint "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" --upload-certs`

*   你可以使用 ​`--kubernetes-version`​ 标志来设置要使用的 Kubernetes 版本。 建议将 kubeadm、kebelet、kubectl 和 Kubernetes 的版本匹配。
*   这个 ​`--control-plane-endpoint`​ 标志应该被设置成负载均衡器的地址或 DNS 和端口。
*   这个 ​`--upload-certs`​ 标志用来将在所有控制平面实例之间的共享证书上传到集群。

> Note: 标志 ​`kubeadm init`​、​`--config`​ 和 ​`--certificate-key`​ 不能混合使用， 因此如果你要使用 kubeadm 配置，你必须在相应的配置结构 （位于 ​`InitConfiguration` ​和 ​`JoinConfiguration: controlPlane`​）添加 ​`certificateKey` ​字段。

> Note: 一些 CNI 网络插件如 Calico 需要 CIDR 例如 ​`192.168.0.0/16`​ 和一些像 Weave 没有。通过传递 ​`--pod-network-cidr`​ 标志添加 pod CIDR，或者你可以使用 kubeadm 配置文件，在 ​`ClusterConfiguration` ​的 ​`networking` ​对象下设置 ​`podSubnet` ​字段。

*   输出类似于：

`... You can now join any number of control-plane node by running the following command on each as a root: kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07  Please note that the certificate-key gives access to cluster sensitive data, keep it secret! As a safeguard, uploaded-certs will be deleted in two hours; If necessary, you can use kubeadm init phase upload-certs to reload certs afterward.  Then you can join any number of worker nodes by running the following on each as root:   kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866`

*   将此输出复制到文本文件。 稍后你将需要它来将控制平面节点和工作节点加入集群。
*   当使用 ​`--upload-certs`​ 调用 ​`kubeadm init`​ 时，主控制平面的证书被加密并上传到 ​`kubeadm-certs`​ Secret 中。
*   要重新上传证书并生成新的解密密钥，请在已加入集群节点的控制平面上使用以下命令：

`sudo kubeadm init phase upload-certs --upload-certs`

*   你还可以在 ​`init` ​期间指定自定义的 ​`--certificate-key`​，以后可以由 ​`join` ​使用。 要生成这样的密钥，可以使用以下命令：

`kubeadm certs certificate-key`

> Note: ​`kubeadm-certs`​ Secret 和解密密钥会在两个小时后失效。

> Caution: 正如命令输出中所述，证书密钥可访问集群敏感数据。请妥善保管！

11.  应用你所选择的 CNI 插件：安装 CNI 驱动。如果适用，请确保配置与 kubeadm 配置文件中指定的 Pod CIDR 相对应。

> Note: 在进行下一步之前，必须选择并部署合适的网络插件。 否则集群不会正常运行。

13.  输入以下内容，并查看控制平面组件的 Pods 启动：

`kubectl get pod -n kube-system -w`

#### 其余控制平面节点的步骤 

> Note: 从 kubeadm 1.15 版本开始，你可以并行加入多个控制平面节点。 在此版本之前，你必须在第一个节点初始化后才能依序的增加新的控制平面节点。

对于每个其他控制平面节点，你应该：

*   执行先前由第一个节点上的 ​`kubeadm init`​ 输出提供给你的 join 命令。 它看起来应该像这样：

`sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866 --control-plane --certificate-key f8902e114ef118304e561c3ecd4d0b543adc226b7a07f675f56564185ffe0c07`

*   这个 ​`--control-plane`​ 标志通知 ​`kubeadm join`​ 创建一个新的控制平面。
*   ​`--certificate-key ...`​ 将导致从集群中的 ​`kubeadm-certs`​ Secret 下载控制平面证书并使用给定的密钥进行解密。

外部 etcd 节点
----------

使用外部 etcd 节点设置集群类似于用于堆叠 etcd 的过程， 不同之处在于你应该首先设置 etcd，并在 kubeadm 配置文件中传递 etcd 信息。

#### 设置 ectd 集群

1.  按照下文 去设置 etcd 集群。
2.  根据本章-手动证书分发 的描述配置 SSH。
3.  将以下文件从集群中的任何 etcd 节点复制到第一个控制平面节点：

`export CONTROL_PLANE="ubuntu@10.0.0.7" scp /etc/kubernetes/pki/etcd/ca.crt "${CONTROL_PLANE}": scp /etc/kubernetes/pki/apiserver-etcd-client.crt "${CONTROL_PLANE}": scp /etc/kubernetes/pki/apiserver-etcd-client.key "${CONTROL_PLANE}":`

*   用第一台控制平面机的 ​`user@host`​ 替换 ​`CONTROL_PLANE` ​的值。

#### 设置第一个控制平面节点 

*   用以下内容创建一个名为 ​`kubeadm-config.yaml`​ 的文件：

`--- apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration kubernetesVersion: stable controlPlaneEndpoint: "LOAD_BALANCER_DNS:LOAD_BALANCER_PORT" # change this (see below) etcd:   external:     endpoints:       - https://ETCD_0_IP:2379 # change ETCD_0_IP appropriately       - https://ETCD_1_IP:2379 # change ETCD_1_IP appropriately       - https://ETCD_2_IP:2379 # change ETCD_2_IP appropriately     caFile: /etc/kubernetes/pki/etcd/ca.crt     certFile: /etc/kubernetes/pki/apiserver-etcd-client.crt     keyFile: /etc/kubernetes/pki/apiserver-etcd-client.key`

> Note: 这里的堆叠（stacked）etcd 和外部 etcd 之前的区别在于设置外部 etcd 需要一个 ​`etcd` ​的 ​`external` ​对象下带有 etcd 端点的配置文件。 如果是内部 etcd，是自动管理的。

*   在你的集群中，将配置模板中的以下变量替换为适当值：

*   ​`LOAD_BALANCER_DNS`​
*   ​`LOAD_BALANCER_PORT`​
*   ​`ETCD_0_IP`​
*   ​`ETCD_1_IP`​
*   ​`ETCD_2_IP`​

以下的步骤与设置内置 etcd 的集群是相似的：

1.  在节点上运行 ​`sudo kubeadm init --config kubeadm-config.yaml --upload-certs`​ 命令。
2.  记下输出的 join 命令，这些命令将在以后使用。
3.  应用你选择的 CNI 插件。

> Note: 在进行下一步之前，必须选择并部署合适的网络插件。 否则集群不会正常运行。

#### 其他控制平面节点的步骤

步骤与设置内置 etcd 相同：

*   确保第一个控制平面节点已完全初始化。
*   使用保存到文本文件的 join 命令将每个控制平面节点连接在一起。 建议一次加入一个控制平面节点。
*   不要忘记默认情况下，​`--certificate-key`​ 中的解密秘钥会在两个小时后过期。

列举控制平面之后的常见任务
-------------

#### 安装工作节点

你可以使用之前存储的 ​`kubeadm init`​ 命令的输出将工作节点加入集群中：

`sudo kubeadm join 192.168.0.200:6443 --token 9vr73a.a8uxyaju799qwdjv --discovery-token-ca-cert-hash sha256:7c2e69131a36ae2a042a339b33381c6d0d43887e2de83720eff5359e26aec866`

手动证书分发 
-------

如果你选择不将 ​`kubeadm init`​ 与 ​`--upload-certs`​ 命令一起使用， 则意味着你将必须手动将证书从主控制平面节点复制到 将要加入的控制平面节点上。

有许多方法可以实现这种操作。在下面的例子中我们使用 ​`ssh` ​和 ​`scp`​：

如果要在单独的一台计算机控制所有节点，则需要 SSH。

1.  在你的主设备上启用 ssh-agent，要求该设备能访问系统中的所有其他节点：

`eval $(ssh-agent)`

3.  将 SSH 身份添加到会话中：

`ssh-add ~/.ssh/path_to_private_key`

5.  检查节点间的 SSH 以确保连接是正常运行的

*   SSH 到任何节点时，请确保添加 ​`-A`​ 标志：

`ssh -A 10.0.0.7`

*   当在任何节点上使用 sudo 时，请确保保持环境变量设置，以便 SSH 转发能够正常工作：

`sudo -E -s`

7.  在所有节点上配置 SSH 之后，你应该在运行过 ​`kubeadm init`​ 命令的第一个 控制平面节点上运行以下脚本。 该脚本会将证书从第一个控制平面节点复制到另一个控制平面节点：

在以下示例中，用其他控制平面节点的 IP 地址替换 ​`CONTROL_PLANE_IPS`​。

`USER=ubuntu # 可定制 CONTROL_PLANE_IPS="10.0.0.7 10.0.0.8" for host in ${CONTROL_PLANE_IPS}; do     scp /etc/kubernetes/pki/ca.crt "${USER}"@$host:     scp /etc/kubernetes/pki/ca.key "${USER}"@$host:     scp /etc/kubernetes/pki/sa.key "${USER}"@$host:     scp /etc/kubernetes/pki/sa.pub "${USER}"@$host:     scp /etc/kubernetes/pki/front-proxy-ca.crt "${USER}"@$host:     scp /etc/kubernetes/pki/front-proxy-ca.key "${USER}"@$host:     scp /etc/kubernetes/pki/etcd/ca.crt "${USER}"@$host:etcd-ca.crt     scp /etc/kubernetes/pki/etcd/ca.key "${USER}"@$host:etcd-ca.key done`

> Caution: 只需要复制上面列表中的证书。kubeadm 将负责生成其余证书以及加入控制平面实例所需的 SAN。 如果你错误地复制了所有证书，由于缺少所需的 SAN，创建其他节点可能会失败。

11.  然后，在每个即将加入集群的控制平面节点上，你必须先运行以下脚本，然后 再运行 ​`kubeadm join`​。 该脚本会将先前复制的证书从主目录移动到 ​`/etc/kubernetes/pki`​：

`USER=ubuntu # 可定制 mkdir -p /etc/kubernetes/pki/etcd mv /home/${USER}/ca.crt /etc/kubernetes/pki/ mv /home/${USER}/ca.key /etc/kubernetes/pki/ mv /home/${USER}/sa.pub /etc/kubernetes/pki/ mv /home/${USER}/sa.key /etc/kubernetes/pki/ mv /home/${USER}/front-proxy-ca.crt /etc/kubernetes/pki/ mv /home/${USER}/front-proxy-ca.key /etc/kubernetes/pki/ mv /home/${USER}/etcd-ca.crt /etc/kubernetes/pki/etcd/ca.crt mv /home/${USER}/etcd-ca.key /etc/kubernetes/pki/etcd/ca.key`

###  7.  Kubernetes 使用kubeadm创建一个高可用etcd集群
使用 kubeadm 创建一个高可用 etcd 集群 
---------------------------

> Note:  
> 在本指南中，使用 kubeadm 作为外部 etcd 节点管理工具，请注意 kubeadm 不计划支持此类节点的证书更换或升级。 对于长期规划是使用 [etcdadm](https://github.com/kubernetes-sigs/etcdadm) 增强工具来管理这些方面。

默认情况下，kubeadm 在每个控制平面节点上运行一个本地 etcd 实例。也可以使用外部的 etcd 集群，并在不同的主机上提供 etcd 实例。

这个任务将指导你创建一个由三个成员组成的高可用外部 etcd 集群，该集群在创建过程中可被 kubeadm 使用。

在开始之前
-----

*   三个可以通过 2379 和 2380 端口相互通信的主机。本文档使用这些作为默认端口。不过，它们可以通过 kubeadm 的配置文件进行自定义。
*   每个主机必须安装 systemd 和 bash 兼容的 shell。
*   每台主机必须安装有容器运行时、kubelet 和 kubeadm。
*   一些可以用来在主机间复制文件的基础设施。例如 ssh 和 scp 就可以满足需求。

建立集群
----

一般来说，是在一个节点上生成所有证书并且只分发这些必要的文件到其它节点上。

> Note:  
> kubeadm 包含生成下述证书所需的所有必要的密码学工具；在这个例子中，不需要其他加密工具。

> Note: 下面的例子使用 IPv4 地址，但是你也可以使用 IPv6 地址配置 kubeadm、kubelet 和 etcd。一些 Kubernetes 选项支持双协议栈，但是 etcd 不支持。 

####  将 kubelet 配置为 etcd 的服务管理器。

> Note: 你必须在要运行 etcd 的所有主机上执行此操作。

由于 etcd 是首先创建的，因此你必须通过创建具有更高优先级的新文件来覆盖 kubeadm 提供的 kubelet 单元文件。

`cat << EOF > /etc/systemd/system/kubelet.service.d/20-etcd-service-manager.conf [Service] ExecStart= # 将下面的 "systemd" 替换为你的容器运行时所使用的 cgroup 驱动。 # kubelet 的默认值为 "cgroupfs"。 # 如果需要的话，将 "--container-runtime-endpoint " 的值替换为一个不同的容器运行时。 ExecStart=/usr/bin/kubelet --address=127.0.0.1 --pod-manifest-path=/etc/kubernetes/manifests --cgroup-driver=systemd Restart=always EOF  systemctl daemon-reload systemctl restart kubelet`

检查 kubelet 的状态以确保其处于运行状态：

`systemctl status kubelet`

####  为 kubeadm 创建配置文件。

使用以下脚本为每个将要运行 etcd 成员的主机生成一个 kubeadm 配置文件。

`# 使用你的主机 IP 替换 HOST0、HOST1 和 HOST2 的 IP 地址 export HOST0=10.0.0.6 export HOST1=10.0.0.7 export HOST2=10.0.0.8   # 使用你的主机名更新 NAME0, NAME1 和 NAME2  export NAME0="infra0"  export NAME1="infra1"  export NAME2="infra2"  # 创建临时目录来存储将被分发到其它主机上的文件 mkdir -p /tmp/${HOST0}/ /tmp/${HOST1}/ /tmp/${HOST2}/   HOSTS=(${HOST0} ${HOST1} ${HOST2})  NAMES=(${NAME0} ${NAME1} ${NAME2})   for i in "${!HOSTS[@]}"; do  HOST=${HOSTS[$i]}  NAME=${NAMES[$i]}  cat << EOF > /tmp/${HOST}/kubeadmcfg.yaml  ---  apiVersion: "kubeadm.k8s.io/v1beta3"  kind: InitConfiguration  nodeRegistration:      name: ${NAME}  localAPIEndpoint:      advertiseAddress: ${HOST}  ---  apiVersion: "kubeadm.k8s.io/v1beta3"  kind: ClusterConfiguration  etcd:      local:          serverCertSANs:          - "${HOST}"          peerCertSANs:          - "${HOST}"          extraArgs:              initial-cluster: ${NAMES[0]}=https://${HOSTS[0]}:2380,${NAMES[1]}=https://${HOSTS[1]}:2380,${NAMES[2]}=https://${HOSTS[2]}:2380              initial-cluster-state: new              name: ${NAME}              listen-peer-urls: https://${HOST}:2380              listen-client-urls: https://${HOST}:2379              advertise-client-urls: https://${HOST}:2379              initial-advertise-peer-urls: https://${HOST}:2380  EOF  done`

####  生成证书颁发机构

如果你已经拥有 CA，那么唯一的操作是复制 CA 的 ​`crt` ​和 ​`key` ​文件到 ​`etc/kubernetes/pki/etcd/ca.crt`​ 和 ​`/etc/kubernetes/pki/etcd/ca.key`​。 复制完这些文件后继续下一步，“为每个成员创建证书”。

如果你还没有 CA，则在 ​`$HOST0`​（你为 kubeadm 生成配置文件的位置）上运行此命令。

`kubeadm init phase certs etcd-ca`

这一操作创建如下两个文件

*   ​`/etc/kubernetes/pki/etcd/ca.crt` ​
*   ​`/etc/kubernetes/pki/etcd/ca.key`​

####  为每个成员创建证书

`kubeadm init phase certs etcd-server --config=/tmp/${HOST2}/kubeadmcfg.yaml kubeadm init phase certs etcd-peer --config=/tmp/${HOST2}/kubeadmcfg.yaml kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST2}/kubeadmcfg.yaml kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST2}/kubeadmcfg.yaml cp -R /etc/kubernetes/pki /tmp/${HOST2}/ # 清理不可重复使用的证书 find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete  kubeadm init phase certs etcd-server --config=/tmp/${HOST1}/kubeadmcfg.yaml kubeadm init phase certs etcd-peer --config=/tmp/${HOST1}/kubeadmcfg.yaml kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST1}/kubeadmcfg.yaml kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST1}/kubeadmcfg.yaml cp -R /etc/kubernetes/pki /tmp/${HOST1}/ find /etc/kubernetes/pki -not -name ca.crt -not -name ca.key -type f -delete  kubeadm init phase certs etcd-server --config=/tmp/${HOST0}/kubeadmcfg.yaml kubeadm init phase certs etcd-peer --config=/tmp/${HOST0}/kubeadmcfg.yaml kubeadm init phase certs etcd-healthcheck-client --config=/tmp/${HOST0}/kubeadmcfg.yaml kubeadm init phase certs apiserver-etcd-client --config=/tmp/${HOST0}/kubeadmcfg.yaml # 不需要移动 certs 因为它们是给 HOST0 使用的  # 清理不应从此主机复制的证书 find /tmp/${HOST2} -name ca.key -type f -delete find /tmp/${HOST1} -name ca.key -type f -delete`

####  复制证书和 kubeadm 配置

证书已生成，现在必须将它们移动到对应的主机。

`USER=ubuntu HOST=${HOST1} scp -r /tmp/${HOST}/* ${USER}@${HOST}: ssh ${USER}@${HOST} USER@HOST $ sudo -Es root@HOST $ chown -R root:root pki root@HOST $ mv pki /etc/kubernetes/`

####  确保已经所有预期的文件都存在

​`$HOST0`​ 所需文件的完整列表如下：

`/tmp/${HOST0} └── kubeadmcfg.yaml --- /etc/kubernetes/pki ├── apiserver-etcd-client.crt ├── apiserver-etcd-client.key └── etcd     ├── ca.crt     ├── ca.key     ├── healthcheck-client.crt     ├── healthcheck-client.key     ├── peer.crt     ├── peer.key     ├── server.crt     └── server.key`

在 ​`$HOST1`​ 上：

`$HOME └── kubeadmcfg.yaml --- /etc/kubernetes/pki ├── apiserver-etcd-client.crt ├── apiserver-etcd-client.key └── etcd     ├── ca.crt     ├── healthcheck-client.crt     ├── healthcheck-client.key     ├── peer.crt     ├── peer.key     ├── server.crt     └── server.key`

在 ​`$HOST2`​ 上：

`$HOME └── kubeadmcfg.yaml --- /etc/kubernetes/pki ├── apiserver-etcd-client.crt ├── apiserver-etcd-client.key └── etcd     ├── ca.crt     ├── healthcheck-client.crt     ├── healthcheck-client.key     ├── peer.crt     ├── peer.key     ├── server.crt     └── server.key`

####  创建静态 Pod 清单

既然证书和配置已经就绪，是时候去创建清单了。 在每台主机上运行 ​`kubeadm` ​命令来生成 etcd 使用的静态清单。

 `root@HOST0 $ kubeadm init phase etcd local --config=/tmp/${HOST0}/kubeadmcfg.yaml  root@HOST1 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml  root@HOST2 $ kubeadm init phase etcd local --config=$HOME/kubeadmcfg.yaml`

####  可选：检查集群运行状况

`docker run --rm -it \ --net host \ -v /etc/kubernetes:/etc/kubernetes k8s.gcr.io/etcd:${ETCD_TAG} etcdctl \ --cert /etc/kubernetes/pki/etcd/peer.crt \ --key /etc/kubernetes/pki/etcd/peer.key \ --cacert /etc/kubernetes/pki/etcd/ca.crt \ --endpoints https://${HOST0}:2379 endpoint health --cluster ... https://[HOST0 IP]:2379 is healthy: successfully committed proposal: took = 16.283339ms https://[HOST1 IP]:2379 is healthy: successfully committed proposal: took = 19.44402ms https://[HOST2 IP]:2379 is healthy: successfully committed proposal: took = 35.926451ms`

*   将 ​`${ETCD_TAG}`​ 设置为你的 etcd 镜像的版本标签，例如 ​`3.4.3-0`​。 要查看 kubeadm 使用的 etcd 镜像和标签，请执行 ​`kubeadm config images list --kubernetes-version ${K8S_VERSION}`​， 例如，其中的 ​`${K8S_VERSION}`​ 可以是 ​`v1.17.0`​。
*   将 ​`${HOST0}`​ 设置为要测试的主机的 IP 地址。

###  8.  Kubernetes 使用kubeadm配置集群中的每个kubelet
使用 kubeadm 配置集群中的每个 kubelet
---------------------------

> Note: Dockershim 自 1.24 版起已从 Kubernetes 项目中删除。

FEATURE STATE: Kubernetes v1.11 \[stable\]

kubeadm CLI 工具的生命周期与 kubelet 解耦；kubelet 是一个守护程序，在 Kubernetes 集群中的每个节点上运行。 当 Kubernetes 初始化或升级时，kubeadm CLI 工具由用户执行，而 kubelet 始终在后台运行。

由于kubelet是守护程序，因此需要通过某种初始化系统或服务管理器进行维护。 当使用 DEB 或 RPM 安装 kubelet 时，配置系统去管理 kubelet。 你可以改用其他服务管理器，但需要手动地配置。

集群中涉及的所有 kubelet 的一些配置细节都必须相同， 而其他配置方面则需要基于每个 kubelet 进行设置，以适应给定机器的不同特性（例如操作系统、存储和网络）。 你可以手动地管理 kubelet 的配置，但是 kubeadm 现在提供一种 ​`KubeletConfiguration` ​API 类型 用于集中管理 kubelet 的配置。

Kubelet 配置模式 
-------------

以下各节讲述了通过使用 kubeadm 简化 kubelet 配置模式，而不是在每个节点上手动地管理 kubelet 配置。

#### 将集群级配置传播到每个 kubelet 中 

你可以通过 ​`kubeadm init`​ 和 ​`kubeadm join`​ 命令为 kubelet 提供默认值。 有趣的示例包括使用其他容器运行时或通过服务器设置不同的默认子网。

如果你想使用子网 ​`10.96.0.0/12`​ 作为服务的默认网段，你可以给 kubeadm 传递 ​`--service-cidr`​ 参数：

`kubeadm init --service-cidr 10.96.0.0/12`

现在，可以从该子网分配服务的虚拟 IP。 你还需要通过 kubelet 使用 ​`--cluster-dns`​ 标志设置 DNS 地址。 在集群中的每个管理器和节点上的 kubelet 的设置需要相同。 kubelet 提供了一个版本化的结构化 API 对象，该对象可以配置 kubelet 中的大多数参数，并将此配置推送到集群中正在运行的每个 kubelet 上。 此对象被称为 ​`KubeletConfiguration`​。 ​`KubeletConfiguration` ​允许用户指定标志，例如用骆峰值代表集群的 DNS IP 地址，如下所示：

`apiVersion: kubelet.config.k8s.io/v1beta1 kind: KubeletConfiguration clusterDNS: - 10.96.0.10`

#### 提供特定于某实例的配置细节 

由于硬件、操作系统、网络或者其他主机特定参数的差异。某些主机需要特定的 kubelet 配置。 以下列表提供了一些示例。

*   由 kubelet 配置标志 ​`--resolv-conf`​ 指定的 DNS 解析文件的路径在操作系统之间可能有所不同， 它取决于你是否使用 ​`systemd-resolved`​。 如果此路径错误，则在其 kubelet 配置错误的节点上 DNS 解析也将失败。
*   除非你使用云驱动，否则默认情况下 Node API 对象的 ​`.metadata.name`​ 会被设置为计算机的主机名。 如果你需要指定一个与机器的主机名不同的节点名称，你可以使用 ​`--hostname-override`​ 标志覆盖默认值。
*   当前，kubelet 无法自动检测容器运行时使用的 cgroup 驱动程序， 但是值 ​`--cgroup-driver`​ 必须与容器运行时使用的 cgroup 驱动程序匹配，以确保 kubelet 的健康运行状况。
*   要指定容器运行时，你必须用 ​`--container-runtime-endpoint=<path>`​ 标志来指定端点。

你可以在服务管理器（例如 systemd）中设定某个 kubelet 的配置来指定这些参数。

使用 kubeadm 配置 kubelet 
----------------------

如果自定义的 ​`KubeletConfiguration` ​API 对象使用像 ​`kubeadm ... --config some-config-file.yaml`​ 这样的配置文件进行传递，则可以配置 kubeadm 启动的 kubelet。

通过调用 ​`kubeadm config print init-defaults --component-configs KubeletConfiguration`​， 你可以看到此结构中的所有默认值。

#### 使用 kubeadm init 时的工作流程 

当调用 ​`kubeadm init`​ 时，kubelet 的配置会被写入磁盘 ​`/var/lib/kubelet/config.yaml`​， 并上传到集群 ​`kube-system`​ 命名空间的 ​`kubelet-config`​ ConfigMap。 kubelet 配置信息也被写入 ​`/etc/kubernetes/kubelet.conf`​，其中包含集群内所有 kubelet 的基线配置。 此配置文件指向允许 kubelet 与 API 服务器通信的客户端证书。 这解决了将集群级配置传播到每个 kubelet 的需求。

针对为特定实例提供配置细节的第二种模式， kubeadm 的解决方法是将环境文件写入 ​`/var/lib/kubelet/kubeadm-flags.env`​，其中包含了一个标志列表， 当 kubelet 启动时，该标志列表会传递给 kubelet 标志在文件中的显示方式如下：

`KUBELET_KUBEADM_ARGS="--flag1=value1 --flag2=value2 ..."`

除了启动 kubelet 时所使用的标志外，该文件还包含动态参数，例如 cgroup 驱动程序以及是否使用其他容器运行时套接字（​`--cri-socket`​）。

将这两个文件编组到磁盘后，如果使用 systemd，则 kubeadm 尝试运行以下两个命令：

`systemctl daemon-reload && systemctl restart kubelet`

如果重新加载和重新启动成功，则正常的 ​`kubeadm init`​ 工作流程将继续。

#### 使用 kubeadm join 时的工作流程 

当运行 ​`kubeadm join`​ 时，kubeadm 使用 Bootstrap Token 证书执行 TLS 引导，该引导会获取一份证书， 该证书需要下载 ​`kubelet-config`​ ConfigMap 并把它写入 ​`/var/lib/kubelet/config.yaml`​ 中。 动态环境文件的生成方式恰好与 ​`kubeadm init`​ 完全相同。

接下来，​`kubeadm` ​运行以下两个命令将新配置加载到 kubelet 中：

`systemctl daemon-reload && systemctl restart kubelet`

在 kubelet 加载新配置后，kubeadm 将写入 ​`/etc/kubernetes/bootstrap-kubelet.conf`​ KubeConfig 文件中， 该文件包含 CA 证书和引导程序令牌。 kubelet 使用这些证书执行 TLS 引导程序并获取唯一的凭据，该凭据被存储在 ​`/etc/kubernetes/kubelet.conf`​ 中。

当 ​`/etc/kubernetes/kubelet.conf`​ 文件被写入后，kubelet 就完成了 TLS 引导过程。 Kubeadm 在完成 TLS 引导过程后将删除 ​`/etc/kubernetes/bootstrap-kubelet.conf`​ 文件。

kubelet 的 systemd drop-in 文件 
-----------------------------

​`kubeadm` ​中附带了有关系统如何运行 kubelet 的 systemd 配置文件。 请注意 kubeadm CLI 命令不会修改此文件。

通过 ​`kubeadm` ​[DEB 包](https://github.com/kubernetes/release/blob/master/cmd/kubepkg/templates/latest/deb/kubeadm/10-kubeadm.conf) 或者 [RPM 包](https://github.com/kubernetes/release/blob/master/cmd/kubepkg/templates/latest/rpm/kubeadm/10-kubeadm.conf) 安装的配置文件被写入 ​`/etc/systemd/system/kubelet.service.d/10-kubeadm.conf`​ 并由 systemd 使用。 它对原来的 [RPM 版本 ​`kubelet.service`](https://github.com/kubernetes/release/blob/master/cmd/kubepkg/templates/latest/rpm/kubelet/kubelet.service)​ 或者 [DEB 版本 ​`kubelet.service`](https://github.com/kubernetes/release/blob/master/cmd/kubepkg/templates/latest/deb/kubelet/lib/systemd/system/kubelet.service)​ 作了增强：

`[Service] Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf" Environment="KUBELET_CONFIG_ARGS=--config=/var/lib/kubelet/config.yaml" # 这是 "kubeadm init" 和 "kubeadm join" 运行时生成的文件，动态地填充 KUBELET_KUBEADM_ARGS 变量 EnvironmentFile=-/var/lib/kubelet/kubeadm-flags.env # 这是一个文件，用户在不得已下可以将其用作替代 kubelet args。 # 用户最好使用 .NodeRegistration.KubeletExtraArgs 对象在配置文件中替代。 # KUBELET_EXTRA_ARGS 应该从此文件中获取。 EnvironmentFile=-/etc/default/kubelet ExecStart= ExecStart=/usr/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_KUBEADM_ARGS $KUBELET_EXTRA_ARGS`

此文件指定由 kubeadm 为 kubelet 管理的所有文件的默认位置。

*   用于 TLS 引导程序的 KubeConfig 文件为 ​`/etc/kubernetes/bootstrap-kubelet.conf`​， 但仅当 ​`/etc/kubernetes/kubelet.conf`​ 不存在时才能使用。
*   具有唯一 kubelet 标识的 KubeConfig 文件为 ​`/etc/kubernetes/kubelet.conf`​。
*   包含 kubelet 的组件配置的文件为 ​`/var/lib/kubelet/config.yaml`​。
*   包含的动态环境的文件 ​`KUBELET_KUBEADM_ARGS`​ 是来源于 ​`/var/lib/kubelet/kubeadm-flags.env`​。
*   包含用户指定标志替代的文件 ​`KUBELET_EXTRA_ARGS` ​是来源于 ​`/etc/default/kubelet`​（对于 DEB），或者 ​`/etc/sysconfig/kubelet`​（对于 RPM）。 ​`KUBELET_EXTRA_ARGS` ​在标志链中排在最后，并且在设置冲突时具有最高优先级。

Kubernetes 可执行文件和软件包内容 
-----------------------

Kubernetes 版本对应的 DEB 和 RPM 软件包是：

软件包名称

描述

`kubeadm`

给 kubelet 安装 `/usr/bin/kubeadm` CLI 工具和 kubelet 的 systemd drop-in 文件。

`kubelet`

安装 `/usr/bin/kubelet` 可执行文件。

`kubectl`

安装 `/usr/bin/kubectl` 可执行文件。

`cri-tools`

从 [cri-tools git 仓库](https://github.com/kubernetes-sigs/cri-tools)中安装 `/usr/bin/crictl` 可执行文件。

`kubernetes-cni`

从 [plugins git 仓库](https://github.com/containernetworking/plugins)中安装 `/opt/cni/bin` 可执行文件。

###  9.  Kubernetes 使用kubeadm支持双协议栈
使用 kubeadm 支持双协议栈
-----------------

FEATURE STATE: Kubernetes v1.23 \[stable\]

你的集群包含双协议栈组网支持， 这意味着集群网络允许你在两种地址族间任选其一。在集群中，控制面可以为同一个 Pod 或者 Service 同时赋予 IPv4 和 IPv6 地址。

在开始之前
-----

你需要已经遵从安装 kubeadm 中所给的步骤安装了 kubeadm 工具。

针对你要作为节点使用的每台服务器， 确保其允许 IPv6 转发。在 Linux 节点上，你可以通过以 root 用户在每台服务器上运行 ​`sysctl -w net.ipv6.conf.all.forwarding=1`​ 来完成设置。

你需要一个可以使用的 IPv4 和 IPv6 地址范围。集群操作人员通常为 IPv4 使用 私有地址范围。对于 IPv6，集群操作人员通常会基于分配给该操作人员的地址范围， 从 ​`2000::/3`​ 中选择一个全局的单播地址块。你不需要将集群的 IP 地址范围路由 到公众互联网。

> Note:  
> 如果你在使用 ​`kubeadm upgrade`​ 命令升级现有的集群，​`kubeadm` ​不允许更改 Pod 的 IP 地址范围（“集群 CIDR”），也不允许更改集群的服务地址范围（“Service CIDR”）。

#### 创建双协议栈集群 

要使用 ​`kubeadm init`​ 创建一个双协议栈集群，你可以传递与下面的例子类似的命令行参数：

`# 这里的地址范围仅作示例使用 kubeadm init --pod-network-cidr=10.244.0.0/16,2001:db8:42:0::/56 --service-cidr=10.96.0.0/16,2001:db8:42:1::/112`

为了更便于理解，参看下面的名为 ​`kubeadm-config.yaml`​ 的 kubeadm 配置文件， 该文件用于双协议栈控制面的主控制节点。

`--- apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration networking:   podSubnet: 10.244.0.0/16,2001:db8:42:0::/56   serviceSubnet: 10.96.0.0/16,2001:db8:42:1::/112 --- apiVersion: kubeadm.k8s.io/v1beta3 kind: InitConfiguration localAPIEndpoint:   advertiseAddress: "10.100.0.1"   bindPort: 6443 nodeRegistration:   kubeletExtraArgs:     node-ip: 10.100.0.2,fd00:1:2:3::2`

InitConfiguration 中的 ​`advertiseAddress` ​给出 API 服务器将公告自身要监听的 IP 地址。​`advertiseAddress` ​的取值与 ​`kubeadm init`​ 的标志 ​`--apiserver-advertise-address`​ 的取值相同。

运行 kubeadm 来实例化双协议栈控制面节点：

`kubeadm init --config=kubeadm-config.yaml`

kube-controller-manager 标志 ​`--node-cidr-mask-size-ipv4|--node-cidr-mask-size-ipv6`​ 是使用默认值来设置的。

> Note:  
> 标志 ​`--apiserver-advertise-address`​ 不支持双协议栈。

#### 向双协议栈集群添加节点 

在添加节点之前，请确保该节点具有 IPv6 可路由的网络接口并且启用了 IPv6 转发。

下面的名为 ​`kubeadm-config.yaml`​ 的 kubeadm 配置文件 示例用于向集群中添加工作节点。

`apiVersion: kubeadm.k8s.io/v1beta3 kind: JoinConfiguration discovery:   bootstrapToken:     apiServerEndpoint: 10.100.0.1:6443     token: "clvldh.vjjwg16ucnhp94qr"     caCertHashes:     - "sha256:a4863cde706cfc580a439f842cc65d5ef112b7b2be31628513a9881cf0d9fe0e"     # 请更改上面的认证信息，使之与你的集群中实际使用的令牌和 CA 证书匹配 nodeRegistration:   kubeletExtraArgs:     node-ip: 10.100.0.3,fd00:1:2:3::3`

下面的名为 ​`kubeadm-config.yaml`​ 的 kubeadm 配置文件 示例用于向集群中添加另一个控制面节点。

`apiVersion: kubeadm.k8s.io/v1beta3 kind: JoinConfiguration controlPlane:   localAPIEndpoint:     advertiseAddress: "10.100.0.2"     bindPort: 6443 discovery:   bootstrapToken:     apiServerEndpoint: 10.100.0.1:6443     token: "clvldh.vjjwg16ucnhp94qr"     caCertHashes:     - "sha256:a4863cde706cfc580a439f842cc65d5ef112b7b2be31628513a9881cf0d9fe0e"     # 请更改上面的认证信息，使之与你的集群中实际使用的令牌和 CA 证书匹配 nodeRegistration:   kubeletExtraArgs:     node-ip: 10.100.0.4,fd00:1:2:3::4`

JoinConfiguration.controlPlane 中的 ​`advertiseAddress` ​设定 API 服务器将公告自身要监听的 IP 地址。​`advertiseAddress` ​的取值与 ​`kubeadm join`​ 的标志 ​`--apiserver-advertise-address`​ 的取值相同。

`kubeadm join --config=kubeadm-config.yaml`

#### 创建单协议栈集群 

> Note:  
> 双协议栈支持并不意味着你需要使用双协议栈来寻址。 你可以部署一个启用了双协议栈联网特性的单协议栈集群。

为了更便于理解，参看下面的名为 ​`kubeadm-config.yaml`​ 的 kubeadm 配置文件示例， 该文件用于单协议栈控制面节点。

`apiVersion: kubeadm.k8s.io/v1beta3 kind: ClusterConfiguration networking:   podSubnet: 10.244.0.0/16   serviceSubnet: 10.96.0.0/16`

###  10.  Kubernetes 使用Kops安装Kubernetes
使用 Kops 安装 Kubernetes
---------------------

本篇快速入门介绍了如何在 AWS 上轻松安装 Kubernetes 集群。 本篇使用了一个名为 ​`[kops](https://github.com/kubernetes/kops)`​ 的工具。

kops 是一个自动化的制备系统：

*   全自动安装流程
*   使用 DNS 识别集群
*   自我修复：一切都在自动扩缩组中运行
*   支持多种操作系统（如 Debian、Ubuntu 16.04、CentOS、RHEL、Amazon Linux 和 CoreOS） - 参考 [images.md](https://github.com/kubernetes/kops/blob/master/docs/operations/images.md)
*   支持高可用
*   可以直接提供或者生成 terraform 清单 - 参考 [terraform.md](https://github.com/kubernetes/kops/blob/master/docs/terraform.md)

在开始之前
-----

*   你必须安装 kubectl。
*   你必须安装 [kops](https://github.com/kubernetes/kops target=) 到 64 位的（AMD64 和 Intel 64）设备架构上。
*   你必须拥有一个 [AWS 账户](https://docs.aws.amazon.com/polly/latest/dg/setting-up.html)， 生成 [IAM 秘钥](https://docs.aws.amazon.com/general/latest/gr/aws-sec-cred-types.html target=) 并 [配置](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html target=) 该秘钥。IAM 用户需要[足够的权限许可](https://github.com/kubernetes/kops/blob/master/docs/getting_started/aws.md target=)。

创建集群
----

#### (1/5) 安装 kops

##### 安装 

从[下载页面](https://github.com/kubernetes/kops/releases)下载 kops （从源代码构建也很方便）：

*   macOS

使用下面的命令下载最新发布版本：

`curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-darwin-amd64`

要下载特定版本，使用特定的 kops 版本替换下面命令中的部分：

`$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)`

例如，要下载 kops v1.20.0，输入：

`curl -LO https://github.com/kubernetes/kops/releases/download/v1.20.0/kops-darwin-amd64`

令 kops 二进制文件可执行：

`chmod +x kops-darwin-amd64`

将 kops 二进制文件移到你的 PATH 下：

`sudo mv kops-darwin-amd64 /usr/local/bin/kops`

你也可以使用 [Homebrew](https://brew.sh/) 安装 kops：

`brew update && brew install kops`

*   Linux

使用命令下载最新发布版本：

`curl -LO https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64`

要下载 kops 的特定版本，用特定的 kops 版本替换下面命令中的部分：

`$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)`

例如，要下载 kops v1.20 版本，输入：

`curl -LO https://github.com/kubernetes/kops/releases/download/v1.20.0/kops-linux-amd64`

令 kops 二进制文件可执行：

`chmod +x kops-linux-amd64`

将 kops 二进制文件移到 PATH 下：

`sudo mv kops-linux-amd64 /usr/local/bin/kops`

你也可以使用 [Homebrew](https://docs.brew.sh/Homebrew-on-Linux) 来安装 kops：

`brew update && brew install kops`

#### (2/5) 为你的集群创建一个 route53 域名

kops 在集群内部和外部都使用 DNS 进行发现操作，这样你可以从客户端访问 kubernetes API 服务器。

kops 对集群名称有明显的要求：它应该是有效的 DNS 名称。这样一来，你就不会再使集群混乱， 可以与同事明确共享集群，并且无需依赖记住 IP 地址即可访问集群。

你可以，或许应该使用子域名来划分集群。作为示例，我们将使用域名 ​`useast1.dev.example.com`​。 这样，API 服务器端点域名将为 ​`api.useast1.dev.example.com`​。

Route53 托管区域可以服务子域名。你的托管区域可能是 ​`useast1.dev.example.com`​，还有 ​`dev.example.com`​ 甚至 ​`example.com`​。 kops 可以与以上任何一种配合使用，因此通常你出于组织原因选择不同的托管区域。 例如，允许你在 ​`dev.example.com`​ 下创建记录，但不能在 ​`example.com`​ 下创建记录。

假设你使用 ​`dev.example.com`​ 作为托管区域。你可以使用 [正常流程](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/CreatingNewSubdomain.html) 或者使用诸如 ​`aws route53 create-hosted-zone --name dev.example.com --caller-reference 1`​ 之类的命令来创建该托管区域。

然后，你必须在父域名中设置你的 DNS 记录，以便该域名中的记录可以被解析。 在这里，你将在 ​`example.com`​ 中为 dev 创建 DNS 记录。 如果它是根域名，则可以在域名注册机构配置 DNS 记录。 例如，你需要在购买 ​`example.com`​ 的地方配置 ​`example.com`​。

检查你的 route53 域已经被正确设置（这是导致问题的最常见原因！）。 如果你安装了 dig 工具，则可以通过运行以下步骤再次检查集群是否配置正确：

`dig DNS dev.example.com`

你应该看到 Route53 分配了你的托管区域的 4 条 DNS 记录。

#### (3/5) 创建一个 S3 存储桶来存储集群状态

kops 使你即使在安装后也可以管理集群。为此，它必须跟踪已创建的集群及其配置、所使用的密钥等。 此信息存储在 S3 存储桶中。S3 权限用于控制对存储桶的访问。

多个集群可以使用同一 S3 存储桶，并且你可以在管理同一集群的同事之间共享一个 S3 存储桶 - 这比传递 kubecfg 文件容易得多。 但是有权访问 S3 存储桶的任何人都将拥有对所有集群的管理访问权限， 因此你不想在运营团队之外共享它。

因此，通常每个运维团队都有一个 S3 存储桶（而且名称通常对应于上面托管区域的名称！）

在我们的示例中，我们选择 ​`dev.example.com`​ 作为托管区域，因此我们选择 ​`clusters.dev.example.com`​ 作为 S3 存储桶名称。

*   导出 ​`AWS_PROFILE`​ 文件（如果你需要选择一个配置文件用来使 AWS CLI 正常工作）
*   使用 ​`aws s3 mb s3://clusters.dev.example.com`​ 创建 S3 存储桶
*   你可以进行 ​`export KOPS_STATE_STORE=s3://clusters.dev.example.com`​ 操作， 然后 kops 将默认使用此位置。 我们建议将其放入你的 bash profile 文件或类似文件中。

#### (4/5) 建立你的集群配置

运行 ​`kops create cluster`​ 以创建你的集群配置：

​`kops create cluster --zones=us-east-1c useast1.dev.example.com` ​

kops 将为你的集群创建配置。请注意，它\_仅\_创建配置，实际上并没有创建云资源 - 你将在下一步中使用 kops update cluster 进行配置。 这使你有机会查看配置或进行更改。

它打印出可用于进一步探索的命令：

*   使用以下命令列出集群：​`kops get cluster` ​
*   使用以下命令编辑该集群：​`kops edit cluster useast1.dev.example.com` ​
*   使用以下命令编辑你的节点实例组：​`kops edit ig --name = useast1.dev.example.com nodes` ​
*   使用以下命令编辑你的主实例组：​`kops edit ig --name = useast1.dev.example.com master-us-east-1c`​

如果这是你第一次使用 kops，请花几分钟尝试一下！ 实例组是一组实例，将被注册为 kubernetes 节点。 在 AWS 上，这是通过 auto-scaling-groups 实现的。你可以有多个实例组。 例如，如果你想要的是混合实例和按需实例的节点，或者 GPU 和非 GPU 实例。

#### (5/5) 在 AWS 中创建集群

运行 "kops update cluster" 以在 AWS 中创建集群：

`kops update cluster useast1.dev.example.com --yes`

这需要几秒钟的时间才能运行，但实际上集群可能需要几分钟才能准备就绪。 每当更改集群配置时，都会使用 ​`kops update cluster`​ 工具。 它将对配置进行的更改应用于你的集群 - 根据需要重新配置 AWS 或者 kubernetes。

例如，在你运行 ​`kops edit ig nodes`​ 之后，然后运行 ​`kops update cluster --yes`​ 应用你的配置，有时你还必须运行 ​`kops rolling-update cluster`​ 立即回滚更新配置。

如果没有 ​`--yes` ​参数，​`kops update cluster`​ 操作将向你显示其操作的预览效果。这对于生产集群很方便！

清理
--

*   删除集群：​`kops delete cluster useast1.dev.example.com --yes`​

###  11.  Kubernetes 使用Kubespray安装Kubernetes
使用 Kubespray 安装 Kubernetes
--------------------------

此快速入门有助于使用 [Kubespray](https://github.com/kubernetes-sigs/kubespray) 安装在 GCE、Azure、OpenStack、AWS、vSphere、Packet（裸机）、Oracle Cloud Infrastructure（实验性）或 Baremetal 上托管的 Kubernetes 集群。

Kubespray 是一个由 [Ansible](https://docs.ansible.com/) playbooks、 [清单（inventory）](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/ansible.md)、 制备工具和通用 OS/Kubernetes 集群配置管理任务的领域知识组成的。 Kubespray 提供：

*   高可用性集群
*   可组合属性
*   支持大多数流行的 Linux 发行版

*   Ubuntu 16.04、18.04、20.04
*   CentOS / RHEL / Oracle Linux 7、8
*   Debian Buster、Jessie、Stretch、Wheezy
*   Fedora 31、32
*   Fedora CoreOS
*   openSUSE Leap 15
*   Kinvolk 的 Flatcar Container Linux

*   持续集成测试

要选择最适合你的用例的工具，请阅读 ​`kubeadm` ​和 ​`kops` ​之间的 [这份比较](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/comparisons.md)

创建集群
----

#### （1/5）满足下层设施要求

按以下[要求](https://github.com/kubernetes-sigs/kubespray target=)来配置服务器：

*   在将运行 Ansible 命令的计算机上安装 Ansible v2.9 和 python-netaddr
*   运行 Ansible Playbook 需要 Jinja 2.11（或更高版本）
*   目标服务器必须有权访问 Internet 才能拉取 Docker 镜像。否则， 需要其他配置（[请参见离线环境](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/offline-environment.md)）
*   目标服务器配置为允许 IPv4 转发
*   你的 SSH 密钥必须复制到部署集群的所有服务器中
*   防火墙不是由 kubespray 管理的。你需要根据需求设置适当的规则策略。为了避免部署过程中出现问题，可以禁用防火墙。
*   如果从非 root 用户帐户运行 kubespray，则应在目标服务器中配置正确的特权升级方法 并指定 ​`ansible_become` ​标志或命令参数 ​`--become`​ 或 ​`-b`​

Kubespray 提供以下实用程序来帮助你设置环境：

*   为以下云驱动提供的 [Terraform](https://www.terraform.io/) 脚本：
*   [AWS](https://github.com/kubernetes-sigs/kubespray/tree/master/contrib/terraform/aws)
*   [OpenStack](http://sitebeskuethree/contrigetbernform/contribeskubernform/contribeskupernform/https/sitebesku/master/)
*   Packet

#### （2/5）编写清单文件

设置服务器后，请创建一个 [Ansible 的清单文件](https://docs.ansible.com/ansible/latest/network/getting_started/first_inventory.html)。 你可以手动执行此操作，也可以通过动态清单脚本执行此操作。有关更多信息，请参阅 “[建立你自己的清单](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md target=)”。

#### （3/5）规划集群部署

Kubespray 能够自定义部署的许多方面：

*   选择部署模式： kubeadm 或非 kubeadm
*   CNI（网络）插件
*   DNS 配置
*   控制平面的选择：本机/可执行文件或容器化
*   组件版本
*   Calico 路由反射器
*   组件运行时选项

*   Docker
*   containerd
*   CRI-O

*   证书生成方式

可以修改[变量文件](https://docs.ansible.com/ansible/latest/user_guide/playbooks_variables.html) 以进行 Kubespray 定制。 如果你刚刚开始使用 Kubespray，请考虑使用 Kubespray 默认设置来部署你的集群 并探索 Kubernetes 。

#### （4/5）部署集群

接下来，部署你的集群：

使用 [ansible-playbook](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md target=) 进行集群部署。

`ansible-playbook -i your/inventory/inventory.ini cluster.yml -b -v \   --private-key=~/.ssh/private_key`

大型部署（超过 100 个节点）可能需要 [特定的调整](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/large-deployments.md)， 以获得最佳效果。

#### （5/5）验证部署

Kubespray 提供了一种使用 [Netchecker](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/netcheck.md) 验证 Pod 间连接和 DNS 解析的方法。 Netchecker 确保 netchecker-agents Pods 可以解析 DNS 请求， 并在默认命名空间内对每个请求执行 ping 操作。 这些 Pod 模仿其他工作负载类似的行为，并用作集群运行状况指示器。

集群操作
----

Kubespray 提供了其他 Playbooks 来管理集群： scale 和 upgrade。

#### 扩展集群

你可以通过运行 scale playbook 向集群中添加工作节点。有关更多信息， 请参见 “[添加节点](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md target=)”。 你可以通过运行 remove-node playbook 来从集群中删除工作节点。有关更多信息， 请参见 “[删除节点](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/getting-started.md target=)”。

#### 升级集群

你可以通过运行 upgrade-cluster Playbook 来升级集群。有关更多信息，请参见 “[升级](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/upgrades.md)”。

清理
--

你可以通过 [reset](https://github.com/kubernetes-sigs/kubespray/blob/master/reset.yml) Playbook 重置节点并清除所有与 Kubespray 一起安装的组件。

> Caution: 运行 reset playbook 时，请确保不要意外地将生产集群作为目标！  

反馈
--

*   Slack 频道：[#kubespray](https://kubernetes.slack.com/?redir=%2Fmessages%2Fkubespray%2F) （你可以在[此处](https://slack.k8s.io/)获得邀请）
*   [GitHub 问题](https://github.com/kubernetes-sigs/kubespray/issues)

##  12.  Kubernetes 对Windows的支持
Kubernetes 对 Windows 的支持
------------------------

在很多组织中，其服务和应用的很大比例是 Windows 应用。 [Windows 容器](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/)提供了一种对进程和包依赖关系 进行封装的现代方式，这使得用户更容易采用 DevOps 实践，令 Windows 应用同样遵从 云原生模式。 Kubernetes 已经成为事实上的标准容器编排器，Kubernetes 1.14 发行版本中包含了将 Windows 容器调度到 Kubernetes 集群中 Windows 节点上的生产级支持，从而使得巨大 的 Windows 应用生态圈能够充分利用 Kubernetes 的能力。 对于同时投入基于 Windows 应用和 Linux 应用的组织而言，他们不必寻找不同的编排系统 来管理其工作负载，其跨部署的运维效率得以大幅提升，而不必关心所用操作系统。

kubernetes 中的 Windows 容器 
-------------------------

若要在 Kubernetes 中启用对 Windows 容器的编排，可以在现有的 Linux 集群中 包含 Windows 节点。在 Kubernetes 上调度 Pods 中的 Windows 容器与调用基于 Linux 的容器类似。

为了运行 Windows 容器，你的 Kubernetes 集群必须包含多个操作系统，控制面 节点运行 Linux，工作节点则可以根据负载需要运行 Windows 或 Linux。 Windows Server 2019 是唯一被支持的 Windows 操作系统，在 Windows 上启用 [Kubernetes 节点](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md target=) 支持（包括 kubelet, [容器运行时](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd)、 以及 kube-proxy）。关于 Windows 发行版渠道的详细讨论，可参见  [Microsoft 文档](https://docs.microsoft.com/en-us/windows-server/get-started/servicing-channels-comparison)。

> Note: Kubernetes 控制面，包括主控组件， 继续在 Linux 上运行。 目前没有支持完全是 Windows 节点的 Kubernetes 集群的计划。

> Note: 在本文中，当我们讨论 Windows 容器时，我们所指的是具有进程隔离能力的 Windows 容器。具有 [Hyper-V 隔离能力](https://docs.microsoft.com/en-us/virtualization/windowscontainers/manage-containers/hyperv-container) 的 Windows 容器计划在将来发行版本中推出。

支持的功能与局限性 
----------

### 支持的功能 

#### Windows 操作系统版本支持 

参考下面的表格，了解 Kubernetes 中支持的 Windows 操作系统。 同一个异构的 Kubernetes 集群中可以同时包含 Windows 和 Linux 工作节点。 Windows 容器仅能调度到 Windows 节点，Linux 容器则只能调度到 Linux 节点。

Kubernetes 版本

Windows Server LTSC 版本

Windows Server SAC 版本

_Kubernetes v1.20_

Windows Server 2019

Windows Server ver 1909, Windows Server ver 2004

_Kubernetes v1.21_

Windows Server 2019

Windows Server ver 2004, Windows Server ver 20H2

_Kubernetes v1.22_

Windows Server 2019

Windows Server ver 2004, Windows Server ver 20H2

关于不同的 Windows Server 版本的服务渠道，包括其支持模式等相关信息可以在 [Windows Server servicing channels](https://docs.microsoft.com/en-us/windows-server/get-started/servicing-channels-comparison) 找到。

我们并不指望所有 Windows 客户都为其应用频繁地更新操作系统。 对应用的更新是向集群中引入新代码的根本原因。 对于想要更新运行于 Kubernetes 之上的容器中操作系统的客户，我们会在添加对新 操作系统版本的支持时提供指南和分步的操作指令。 该指南会包含与集群节点一起来升级用户应用的建议升级步骤。 Windows 节点遵从 Kubernetes 版本偏差策略（节点到控制面的 版本控制），与 Linux 节点的现行策略相同。

Windows Server 主机操作系统会受 [Windows Server](https://www.microsoft.com/en-us/windows-server/pricing) 授权策略控制。Windows 容器镜像则遵从 [Windows 容器的补充授权条款](https://docs.microsoft.com/en-us/virtualization/windowscontainers/images-eula) 约定。

带进程隔离的 Windows 容器受一些严格的兼容性规则约束， [其中宿主 OS 版本必须与容器基准镜像的 OS 版本相同](https://docs.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/version-compatibility?tabs=windows-server-2022%2Cwindows-11-21H2)。 一旦我们在 Kubernetes 中支持带 Hyper-V 隔离的 Windows 容器， 这一约束和兼容性规则也会发生改变。

#### Pause 镜像 

Kubernetes 维护着一个多体系结构镜像，其中包括对 Windows 的支持。 对于 Kubernetes v1.22，推荐的 pause 镜像是 ​`k8s.gcr.io/pause:3.5`​。 [源代码](https://github.com/kubernetes/kubernetes/tree/master/build/pause)可在 GitHub 上找到。

Microsoft 维护了一个支持 Linux 和 Windows amd64 的多体系结构镜像： ​`mcr.microsoft.com/oss/kubernetes/pause:3.5`​。 此镜像与 Kubernetes 维护的镜像是从同一来源构建，但所有 Windows 二进制文件 均由 Microsoft [签名](https://docs.microsoft.com/en-us/windows-hardware/drivers/install/authenticode)。 当生产环境需要被签名的二进制文件时，建议使用 Microsoft 维护的镜像。

#### 计算 

从 API 和 kubectl 的角度，Windows 容器的表现在很大程度上与基于 Linux 的容器 是相同的。不过也有一些与关键功能相关的差别值得注意，这些差别列举于 局限性小节中。

关键性的 Kubernetes 元素在 Windows 下与其在 Linux 下工作方式相同。我们在本节中 讨论一些关键性的负载支撑组件及其在 Windows 中的映射。

*   Pods

Pod 是 Kubernetes 中最基本的构造模块，是 Kubernetes 对象模型中你可以创建或部署的 最小、最简单元。你不可以在同一 Pod 中部署 Windows 和 Linux 容器。 Pod 中的所有容器都会被调度到同一节点（Node），而每个节点代表的是一种特定的平台 和体系结构。Windows 容器支持 Pod 的以下能力、属性和事件：

*   在带进程隔离和卷共享支持的 Pod 中运行一个或多个容器
*   Pod 状态字段
*   就绪态（Readiness）和活跃性（Liveness）探针
*   postStart 和 preStop 容器生命周期事件
*   ConfigMap、Secrets：用作环境变量或卷
*   emptyDir 卷
*   从宿主系统挂载命名管道
*   资源限制

*   控制器（Controllers）

Kubernetes 控制器处理 Pod 的期望状态。Windows 容器支持以下负载控制器：

*   ReplicaSet
*   ReplicationController
*   Deployment
*   StatefulSet
*   DaemonSet
*   Job
*   CronJob

*   服务（Services）

Kubernetes Service 是一种抽象对象，用来定义 Pod 的一个逻辑集合及用来访问这些 Pod 的策略。Service 有时也称作微服务（Micro-service）。你可以使用服务来实现 跨操作系统的连接。在 Windows 系统中，服务可以使用下面的类型、属性和能力：

*   Service 环境变量
*   NodePort
*   ClusterIP
*   LoadBalancer
*   ExternalName
*   无头（Headless）服务

Pods、控制器和服务是在 Kubernetes 上管理 Windows 负载的关键元素。 不过，在一个动态的云原生环境中，这些元素本身还不足以用来正确管理 Windows 负载的生命周期。我们为此添加了如下功能特性：

*   Pod 和容器的度量（Metrics）
*   对水平 Pod 自动扩展的支持
*   对 kubectl exec 命令的支持
*   资源配额
*   调度器抢占

#### 容器运行时 

##### Docker EE

FEATURE STATE: Kubernetes v1.14 \[stable\]

Docker EE-basic 19.03+ 是建议所有 Windows Server 版本采用的容器运行时。 该容器运行时能够与 kubelet 中的 dockershim 代码协同工作。

##### CRI-ContainerD

FEATURE STATE: Kubernetes v1.20 \[stable\]

ContainerD 1.4.0+ 也可作为 Windows Kubernetes 节点上的容器运行时。

#### 持久性存储 

使用 Kubernetes 卷，对数据持久性和 Pod 卷 共享有需求的复杂应用也可以部署到 Kubernetes 上。 管理与特定存储后端或协议相关的持久卷时，相关的操作包括：对卷的配备（Provisioning）、 去配（De-provisioning）和调整大小，将卷挂接到 Kubernetes 节点或从节点上解除挂接， 将卷挂载到需要持久数据的 Pod 中的某容器或从容器上卸载。 负责实现为特定存储后端或协议实现卷管理动作的代码以 Kubernetes 卷 插件的形式发布。 Windows 支持以下大类的 Kubernetes 卷插件：

##### 树内卷插件 

与树内卷插件（In-Tree Volume Plugin）相关的代码都作为核心 Kubernetes 代码基 的一部分发布。树内卷插件的部署不需要安装额外的脚本，也不需要额外部署独立的 容器化插件组件。这些插件可以处理：对应存储后端上存储卷的配备、去配和尺寸更改， 将卷挂接到 Kubernetes 或从其上解挂，以及将卷挂载到 Pod 中各个容器上或从其上 卸载。以下树内插件支持 Windows 节点：

*   ​`awsElasticBlockStore` ​
*   ​`azureDisk` ​
*   ​`azureFile` ​
*   ​`gcePersistentDisk` ​
*   ​`vsphereVolume`​

##### FlexVolume 插件 

与 FlexVolume 插件相关的代码是作为 树外（Out-of-tree）脚本或可执行文件来发布的，因此需要在宿主系统上直接部署。 FlexVolume 插件处理将卷挂接到 Kubernetes 节点或从其上解挂、将卷挂载到 Pod 中 各个容器上或从其上卸载等操作。对于与 FlexVolume 插件相关联的持久卷的配备和 去配操作，可以通过外部的配置程序来处理。这类配置程序通常与 FlexVolume 插件 相分离。下面的 FlexVolume  [插件](https://github.com/Microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows) 可以以 PowerShell 脚本的形式部署到宿主系统上，支持 Windows 节点：

*   [SMB](https://github.com/microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows/plugins/microsoft.com~smb.cmd)
*   [iSCSI](https://github.com/microsoft/K8s-Storage-Plugins/tree/master/flexvolume/windows/plugins/microsoft.com~iscsi.cmd)

##### CSI 插件 

FEATURE STATE: Kubernetes v1.22 \[stable\]

与 CSI 插件相关联的代码作为 树外脚本和可执行文件来发布且通常发布为容器镜像形式，并使用 DaemonSet 和 StatefulSet 这类标准的 Kubernetes 构造体来部署。 CSI 插件处理 Kubernetes 中的很多卷管理操作：对卷的配备、去配和调整大小， 将卷挂接到 Kubernetes 节点或从节点上解除挂接，将卷挂载到需要持久数据的 Pod 中的某容器或从容器上卸载，使用快照和克隆来备份或恢复持久数据。

来支持；csi-proxy 是一个社区管理的、独立的可执行文件，需要预安装在每个 Windows 节点之上。请参考你要部署的 CSI 插件的部署指南以进一步了解其细节。

CSI 插件与执行本地存储操作的 CSI 节点插件通信。 在 Windows 节点上，CSI 节点插件通常调用处理本地存储操作的 csi-proxy 公开的 API, [csi-proxy](https://github.com/kubernetes-csi/csi-proxy) 由社区管理。

有关安装的更多详细信息，请参阅你要部署的 Windows CSI 插件的环境部署指南。 你也可以参考以下[安装步骤](https://github.com/kubernetes-csi/csi-proxy target=) 。

#### 联网 

Windows 容器的联网是通过 CNI 插件 来暴露出来的。Windows 容器的联网行为与虚拟机的联网行为类似。 每个容器有一块虚拟的网络适配器（vNIC）连接到 Hyper-V 的虚拟交换机（vSwitch）。 宿主的联网服务（Host Networking Service，HNS）和宿主计算服务（Host Compute Service，HCS）协同工作，创建容器并将容器的虚拟网卡连接到网络上。 HCS 负责管理容器，HNS 则负责管理网络资源，例如：

*   虚拟网络（包括创建 vSwitch）
*   端点（Endpoint）/ vNIC
*   名字空间（Namespace）
*   策略（报文封装、负载均衡规则、访问控制列表、网络地址转译规则等等）

支持的服务规约类型如下：

*   NodePort
*   ClusterIP
*   LoadBalancer
*   ExternalName

##### 网络模式 

Windows 支持五种不同的网络驱动/模式：二层桥接（L2bridge）、二层隧道（L2tunnel）、 覆盖网络（Overlay）、透明网络（Transparent）和网络地址转译（NAT）。 在一个包含 Windows 和 Linux 工作节点的异构集群中，你需要选择一种对 Windows 和 Linux 兼容的联网方案。下面是 Windows 上支持的一些树外插件及何时使用某种 CNI 插件的建议：

网络驱动

描述

容器报文更改

网络插件

网络插件特点

L2bridge

容器挂接到外部 vSwitch 上。容器挂接到下层网络之上，但由于容器的 MAC 地址在入站和出站时被重写，物理网络不需要这些地址。

MAC 地址被重写为宿主系统的 MAC 地址，IP 地址也可能依据 HNS OutboundNAT 策略重写为宿主的 IP 地址。

[win-bridge](https://github.com/containernetworking/plugins/tree/master/plugins/main/windows/win-bridge)、 [Azure-CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md)、 Flannel 宿主网关（host-gateway）使用 win-bridge

win-bridge 使用二层桥接（L2bridge）网络模式，将容器连接到下层宿主系统上， 从而提供最佳性能。需要用户定义的路由（User-Defined Routes，UDR）才能 实现节点间的连接。

L2Tunnel

这是二层桥接的一种特殊情形，但仅被用于 Azure 上。 所有报文都被发送到虚拟化环境中的宿主机上并根据 SDN 策略进行处理。

MAC 地址被改写，IP 地址在下层网络上可见。

[Azure-CNI](https://github.com/Azure/azure-container-networking/blob/master/docs/cni.md)

Azure-CNI 使得容器能够与 Azure vNET 集成，并允许容器利用 \[Azure 虚拟网络\](https://azure.microsoft.com/en-us/services/virtual-network/) 所提供的功能特性集合。例如，可以安全地连接到 Azure 服务上或者使用 Azure NSG。 你可以参考 \[azure-cni\](https://docs.microsoft.com/en-us/azure/aks/concepts-network#azure-cni-advanced-networking) 所提供的一些示例。

覆盖网络（Kubernetes 中为 Windows 提供的覆盖网络支持处于 \*alpha\* 阶段）

每个容器会获得一个连接到外部 vSwitch 的虚拟网卡（vNIC）。 每个覆盖网络都有自己的、通过定制 IP 前缀来定义的 IP 子网。 覆盖网络驱动使用 VxLAN 封装。

封装于外层包头内。

[Win-overlay](https://github.com/containernetworking/plugins/tree/master/plugins/main/windows/win-overlay)、 Flannel VXLAN（使用 win-overlay）

当（比如出于安全原因）期望虚拟容器网络与下层宿主网络隔离时， 应该使用 win-overlay。如果你的数据中心可用 IP 地址受限， 覆盖网络允许你在不同的网络中复用 IP 地址（每个覆盖网络有不同的 VNID 标签）。 这一选项要求在 Windows Server 2009 上安装 \[KB4489899\](https://support.microsoft.com/help/4489899) 补丁。

透明网络（\[ovn-kubernetes\](https://github.com/openvswitch/ovn-kubernetes) 的特殊用例）

需要一个外部 vSwitch。容器挂接到某外部 vSwitch 上，该 vSwitch 通过逻辑网络（逻辑交换机和路由器）允许 Pod 间通信。

报文或者通过 \[GENEVE\](https://datatracker.ietf.org/doc/draft-gross-geneve/) 来封装， 或者通过 \[STT\](https://datatracker.ietf.org/doc/draft-davie-stt/) 隧道来封装， 以便能够到达不在同一宿主系统上的每个 Pod。  
报文通过 OVN 网络控制器所提供的隧道元数据信息来判定是转发还是丢弃。  
北-南向通信通过 NAT 网络地址转译来实现。

[ovn-kubernetes](https://github.com/openvswitch/ovn-kubernetes)

\[通过 Ansible 来部署\](https://github.com/openvswitch/ovn-kubernetes/tree/master/contrib)。 所发布的 ACL 可以通过 Kubernetes 策略来应用实施。支持 IPAM 。 负载均衡能力不依赖 kube-proxy。 网络地址转译（NAT）也不需要 iptables 或 netsh。

NAT（**未在 Kubernetes 中使用**）

容器获得一个连接到某内部 vSwitch 的 vNIC 接口。 DNS/DHCP 服务通过名为 \[WinNAT\](https://blogs.technet.microsoft.com/virtualization/2016/05/25/windows-nat-winnat-capabilities-and-limitations/) 的内部组件来提供。

MAC 地址和 IP 地址都被重写为宿主系统的 MAC 地址和 IP 地址。

[nat](https://github.com/Microsoft/windows-container-networking/tree/master/plugins/nat)

列在此表中仅出于完整性考虑

如前所述，[Flannel](https://github.com/flannel-io/flannel) CNI meta 插件 在 Windows 上也是 被支持 的，方法是通过 [VXLAN 网络后端](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md target=) （alpha 阶段 ：委托给 win-overlay）和 [主机-网关（host-gateway）网络后端](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md target=) （稳定版本；委托给 win-bridge 实现）。 此插件支持将操作委托给所引用的 CNI 插件（win-overlay、win-bridge）之一， 从而能够与 Windows 上的 Flannel 守护进程（Flanneld）一同工作，自动为节点 分配子网租期，创建 HNS 网络。 该插件读入其自身的配置文件（cni.conf），并将其与 FlannelD 所生成的 subnet.env 文件中的环境变量整合，之后将其操作委托给所引用的 CNI 插件之一以完成网络发现， 并将包含节点所被分配的子网信息的正确配置发送给 IPAM 插件（例如 host-local）。

对于节点、Pod 和服务对象，可针对 TCP/UDP 流量支持以下网络数据流：

*   Pod -> Pod （IP 寻址）
*   Pod -> Pod （名字寻址）
*   Pod -> 服务（集群 IP）
*   Pod -> 服务（部分限定域名，仅适用于名称中不包含“.”的情形）
*   Pod -> 服务（全限定域名）
*   Pod -> 集群外部（IP 寻址）
*   Pod -> 集群外部（DNS 寻址）
*   节点 -> Pod
*   Pod -> 节点

##### IP 地址管理（IPAM） 

Windows 上支持以下 IPAM 选项：

*   [host-local](https://github.com/containernetworking/plugins/tree/main/plugins/ipam/host-local)
*   HNS IPAM (Inbox 平台 IPAM，未指定 IPAM 时的默认设置）
*   [Azure-vnet-ipam](https://github.com/Azure/azure-container-networking/blob/master/docs/ipam.md)（仅适用于 azure-cni ）

##### 负载均衡与服务 

在 Windows 系统上，你可以使用以下配置来设定服务和负载均衡行为：

功能特性

描述

所支持的 Kubernetes 版本

所支持的 Windows OS 版本

如何启用

会话亲和性

确保来自特定客户的连接每次都被交给同一 Pod。

v1.20+

\[Windows Server vNext Insider Preview Build 19551\](https://blogs.windows.com/windowsexperience/2020/01/28/announcing-windows-server-vnext-insider-preview-build-19551/) 或更高版本

将 `service.spec.sessionAffinitys` 设置为 "ClientIP"

直接服务器返回（DSR）

这是一种负载均衡模式，IP 地址的修正和负载均衡地址转译（LBNAT） 直接在容器的 vSwitch 端口上处理；服务流量到达时，其源端 IP 地址 设置为来源 Pod 的 IP。

v1.20+

Windows Server 2019

为 kube-proxy 设置标志：\`--feature-gates="WinDSR=true" --enable-dsr=true\`

保留目标地址

对服务流量略过 DNAT 步骤，这样就可以在到达后端 Pod 的报文中保留目标服务的 虚拟 IP 地址。还要禁止节点之间的转发。

v1.20+

Windows Server 1903 或更高版本

在服务注解中设置 \`"preserve-destination": "true"\` 并启用 kube-proxy 中的 DSR 标志。

IPv4/IPv6 双栈网络

在集群内外同时支持原生的 IPv4-到-IPv4 和 IPv6-到-IPv6 通信。

v1.19+

Windows Server 2004 或更高版本

参见 \[IPv4/IPv6 双栈网络\](#ipv4ipv6-dual-stack)

保留客户端 IP

确保入站流量的源 IP 地址被保留。同样要禁止节点之间的转发。

v1.20+

Windows Server 2019 或更高版本

将 `service.spec.externalTrafficPolicy` 设置为 "Local"， 并在 kube-proxy 上启用 DSR。

#### IPv4/IPv6 双栈支持 

你可以通过使用 ​`IPv6DualStack` ​特性门控 来为 ​`l2bridge` ​网络启用 IPv4/IPv6 双栈联网支持。

对 Windows 而言，在 Kubernetes 中使用 IPv6 需要 Windows Server 2004 （内核版本 10.0.19041.610）或更高版本。

目前 Windows 上的覆盖网络（VXLAN）还不支持双协议栈联网。

### 局限性 

在 Kubernetes 架构和节点阵列中仅支持将 Windows 作为工作节点使用。 这意味着 Kubernetes 集群必须总是包含 Linux 主控节点，零个或者多个 Linux 工作节点以及零个或者多个 Windows 工作节点。

#### 资源处理

Linux 上使用 Linux 控制组（CGroups）作为 Pod 的边界，以实现资源控制。 容器都创建于这一边界之内，从而实现网络、进程和文件系统的隔离。 控制组 CGroups API 可用来收集 CPU、I/O 和内存的统计信息。 与此相比，Windows 为每个容器创建一个带有系统名字空间过滤设置的 Job 对象， 以容纳容器中的所有进程并提供其与宿主系统间的逻辑隔离。 没有现成的名字空间过滤设置是无法运行 Windows 容器的。 这也意味着，系统特权无法在宿主环境中评估，因而 Windows 上也就不存在特权容器。 归咎于独立存在的安全账号管理器（Security Account Manager，SAM），容器也不能 获得宿主系统上的任何身份标识。

#### 资源预留 

##### 内存预留 

Windows 不像 Linux 一样有一个内存耗尽（Out-of-memory）进程杀手（Process Killer）机制。Windows 总是将用户态的内存分配视为虚拟请求，页面文件（Pagefile） 是必需的。这一差异的直接结果是 Windows 不会像 Linux 那样出现内存耗尽的状况， 系统会将进程内存页面写入磁盘而不会因内存耗尽而终止进程。 当内存被过量使用且所有物理内存都被用光时，系统的换页行为会导致性能下降。

使用 kubelet 参数 ​`--kubelet-reserve`​ 与/或 ​`-system-reserve`​ 可以统计 节点上的内存用量（各容器之外），进而可能将内存用量限制在一个合理的范围，。 这样做会减少节点可分配内存。

在你部署工作负载时，对容器使用资源限制（必须仅设置 limits 或者让 limits 等于 requests 值）。这也会从 NodeAllocatable 中耗掉部分内存量，从而避免在节点 负荷已满时调度器继续向节点添加 Pods。

避免过量分配的最佳实践是为 kubelet 配置至少 2 GB 的系统预留内存，以供 Windows、Docker 和 Kubernetes 进程使用。

##### CPU 预留 

为了统计 Windows、Docker 和其他 Kubernetes 宿主进程的 CPU 用量，建议 预留一定比例的 CPU，以便对事件作出相应。此值需要根据 Windows 节点上 CPU 核的个数来调整，要确定此百分比值，用户需要为其所有节点确定 Pod 密度的上线，并监控系统服务的 CPU 用量，从而选择一个符合其负载需求的值。

使用 kubelet 参数 ​`--kubelet-reserve`​ 与/或 ​`-system-reserve`​ 可以统计 节点上的 CPU 用量（各容器之外），进而可能将 CPU 用量限制在一个合理的范围，。 这样做会减少节点可分配 CPU。

##### 功能特性限制 

*   终止宽限期（Termination Grace Period）：未实现
*   单文件映射：将用 CRI-ContainerD 来实现
*   终止消息（Termination message）：将用 CRI-ContainerD 来实现
*   特权容器：Windows 容器当前不支持
*   巨页（Huge Pages）：Windows 容器当前不支持
*   现有的节点问题探测器（Node Problem Detector）仅适用于 Linux，且要求使用特权容器。 一般而言，我们不设想此探测器能用于 Windows 节点，因为 Windows 不支持特权容器。
*   并非支持共享名字空间的所有功能特性（参见 API 节以了解详细信息）

#### 与 Linux 相比参数行为的差别

以下 kubelet 参数的行为在 Windows 节点上有些不同，描述如下：

*   ​`--kubelet-reserve`​、​`--system-reserve`​ 和 ​`--eviction-hard`​ 标志 会更新节点可分配资源量
*   未实现通过使用 ​`--enforce-node-allocable`​ 来完成的 Pod 驱逐
*   未实现通过使用 ​`--eviction-hard`​ 和 ​`--eviction-soft`​ 来完成的 Pod 驱逐
*   ​`MemoryPressure` ​状况未实现
*   ​`kubelet` ​不会采取措施来执行基于 OOM 的驱逐动作
*   Windows 节点上运行的 kubelet 没有内存约束。 ​`--kubelet-reserve`​ 和 ​`--system-reserve`​ 不会为 kubelet 或宿主系统上运行 的进程设限。这意味着 kubelet 或宿主系统上的进程可能导致内存资源紧张， 而这一情况既不受节点可分配量影响，也不会被调度器感知。
*   在 Windows 节点上存在一个额外的参数用来设置 kubelet 进程的优先级，称作 ​`--windows-priorityclass`​。此参数允许 kubelet 进程获得与 Windows 宿主上 其他进程相比更多的 CPU 时间片。 关于可用参数值及其含义的进一步信息可参考 [Windows Priority Classes](https://docs.microsoft.com/en-us/windows/win32/procthread/scheduling-priorities target=)。 为了让 kubelet 总能够获得足够的 CPU 周期，建议将此参数设置为 ​`ABOVE_NORMAL_PRIORITY_CLASS` ​或更高。

#### 存储 

Windows 上包含一个分层的文件系统来挂载容器的分层，并会基于 NTFS 来创建一个 拷贝文件系统。容器中的所有文件路径都仅在该容器的上下文内完成解析。

*   Docker 卷挂载仅可针对容器中的目录进行，不可针对独立的文件。 这一限制不适用于 CRI-containerD。
*   卷挂载无法将文件或目录投射回宿主文件系统。
*   不支持只读文件系统，因为 Windows 注册表和 SAM 数据库总是需要写访问权限。 不过，Windows 支持只读的卷。
*   不支持卷的用户掩码和访问许可，因为宿主与容器之间并不共享 SAM，二者之间不存在 映射关系。所有访问许可都是在容器上下文中解析的。

因此，Windows 节点上不支持以下存储功能特性：

*   卷的子路径挂载；只能在 Windows 容器上挂载整个卷。
*   为 Secret 执行子路径挂载；
*   宿主挂载投射；
*   默认访问模式 defaultMode（因为该特性依赖 UID/GID）；
*   只读的根文件系统；映射的卷仍然支持 ​`readOnly`​；
*   块设备映射；
*   将内存作为存储介质；
*   类似 UUID/GUID、每用户不同的 Linux 文件系统访问许可等文件系统特性；
*   基于 NFS 的存储和卷支持；
*   扩充已挂载卷（resizefs）。

#### 联网 

Windows 容器联网与 Linux 联网有着非常重要的差别。 [Microsoft documentation for Windows Container Networking](https://docs.microsoft.com/en-us/virtualization/windowscontainers/container-networking/architecture) 中包含额外的细节和背景信息。

Windows 宿主联网服务和虚拟交换机实现了名字空间隔离，可以根据需要为 Pod 或容器 创建虚拟的网络接口（NICs）。不过，很多类似 DNS、路由、度量值之类的配置数据都 保存在 Windows 注册表数据库中而不是像 Linux 一样保存在 ​`/etc/...`​ 文件中。 Windows 为容器提供的注册表与宿主系统的注册表是分离的，因此类似于将 /etc/resolv.conf 文件从宿主系统映射到容器中的做法不会产生与 Linux 系统相同的效果。 这些信息必须在容器内部使用 Windows API 来配置。 因此，CNI 实现需要调用 HNS，而不是依赖文件映射来将网络细节传递到 Pod 或容器中。

Windows 节点不支持以下联网功能：

*   Windows Pod 不能使用宿主网络模式；
*   从节点本地访问 NodePort 会失败（但从其他节点或外部客户端可访问）
*   Windows Server 的未来版本中会支持从节点访问服务的 VIP；
*   每个服务最多支持 64 个后端 Pod 或独立的目标 IP 地址；
*   kube-proxy 的覆盖网络支持是 Beta 特性。此外，它要求在 Windows Server 2019 上安装 [KB4482887](https://support.microsoft.com/en-us/topic/march-1-2019-kb4482887-os-build-17763-348-f7a9f207-0627-1fb9-cca7-29bb7b26027f) 补丁；
*   非 DSR（保留目标地址）模式下的本地流量策略；
*   连接到覆盖网络的 Windows 容器不支持使用 IPv6 协议栈通信。 要使得这一网络驱动支持 IPv6 地址需要在 Windows 平台上开展大量的工作， 还需要在 Kubernetes 侧修改 kubelet、kube-proxy 以及 CNI 插件。
*   通过 win-overlay、win-bridge 和 Azure-CNI 插件使用 ICMP 协议向集群外通信。 尤其是，Windows 数据面 （[VFP](https://www.microsoft.com/en-us/research/project/azure-virtual-filtering-platform/)） 不支持转换 ICMP 报文。这意味着：

*   指向同一网络内目标地址的 ICMP 报文（例如 Pod 之间的 ping 通信）是可以工作的， 没有局限性；
*   TCP/UDP 报文可以正常工作，没有局限性；
*   指向远程网络的 ICMP 报文（例如，从 Pod 中 ping 外部互联网的通信）无法被转换， 因此也无法被路由回到其源点；
*   由于 TCP/UDP 包仍可被转换，用户可以将 ​`ping <目标>`​ 操作替换为 ​`curl <目标>`​ 以便能够调试与外部世界的网络连接。

Kubernetes v1.15 中添加了以下功能特性：

*   ​`kubectl port-forward` ​

##### CNI 插件 

*   Windows 参考网络插件 win-bridge 和 win-overlay 当前未实现 [CNI spec](https://github.com/containernetworking/cni/blob/master/SPEC.md) v0.4.0， 原因是缺少检查（CHECK）用的实现。
    
*   Windows 上的 Flannel VXLAN CNI 有以下局限性：

1.  其设计上不支持从节点到 Pod 的连接。 只有在 Flannel v0.12.0 或更高版本后才有可能访问本地 Pods。
2.  我们被限制只能使用 VNI 4096 和 UDP 端口 4789。 VNI 的限制正在被解决，会在将来的版本中消失（开源的 Flannel 更改）。 参见官方的 [Flannel VXLAN](https://github.com/flannel-io/flannel/blob/master/Documentation/backends.md target=) 后端文档以了解关于这些参数的详细信息。

##### DNS

*   不支持 DNS 的 ClusterFirstWithHostNet 配置。Windows 将所有包含 “.” 的名字 视为全限定域名（FQDN），因而不会对其执行部分限定域名（PQDN）解析。
*   在 Linux 上，你可以有一个 DNS 后缀列表供解析部分限定域名时使用。 在 Windows 上，我们只有一个 DNS 后缀，即与该 Pod 名字空间相关联的 DNS 后缀（例如 ​`mydns.svc.cluster.local`​）。 Windows 可以解析全限定域名、或者恰好可用该后缀来解析的服务名称。 例如，在 default 名字空间中生成的 Pod 会获得 DNS 后缀 ​`default.svc.cluster.local`​。在 Windows Pod 中，你可以解析 ​`kubernetes.default.svc.cluster.local`​ 和 ​`kubernetes`​，但无法解析二者 之间的形式，如 ​`kubernetes.default`​ 或 ​`kubernetes.default.svc`​。
*   在 Windows 上，可以使用的 DNS 解析程序有很多。由于这些解析程序彼此之间 会有轻微的行为差别，建议使用 ​`Resolve-DNSName`​ 工具来完成名字查询解析。

##### IPv6

Windows 上的 Kubernetes 不支持单协议栈的“只用 IPv6”联网选项。 不过，系统支持在 IPv4/IPv6 双协议栈的 Pod 和节点上运行单协议家族的服务。

##### 会话亲和性 

不支持使用 ​`service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`​ 来为 Windows 服务设置最大会话粘滞时间。

##### 安全性 

Secret 以明文形式写入节点的卷中（而不是像 Linux 那样写入内存或 tmpfs 中）。 这意味着客户必须做以下两件事：

1.  使用文件访问控制列表来保护 Secret 文件所在的位置
2.  使用 [BitLocker](https://docs.microsoft.com/en-us/windows/security/information-protection/bitlocker/bitlocker-how-to-deploy-on-windows-server) 来执行卷层面的加密

用户可以为 Windows Pods 或 Container 设置 ​`RunAsUserName` ​以便以非节点默认用户来执行容器中的进程。这大致等价于设置 ​`RunAsUser`​。

不支持特定于 Linux 的 Pod 安全上下文特权，例如 SELinux、AppArmor、Seccomp、 权能字（POSIX 权能字）等等。

此外，如前所述，Windows 不支持特权容器。

#### API

对 Windows 而言，大多数 Kubernetes API 的工作方式没有变化。 一些不易察觉的差别通常体现在 OS 和容器运行时上的不同。 在某些场合，负载 API （如 Pod 或 Container）的某些属性在设计时假定其 在 Linux 上实现，因此会无法在 Windows 上运行。

在较高层面，不同的 OS 概念有：

*   身份标识 - Linux 使用证书类型来表示用户 ID（UID）和组 ID（GID）。用户和组名 没有特定标准，它们是 ​`/etc/groups`​ 或 ​`/etc/passwd`​ 中的别名表项，会映射回 UID+GID。Windows 使用一个更大的二进制安全标识符（SID），保存在 Windows 安全访问管理器（Security Access Manager，SAM）数据库中。此数据库并不在宿主系统 与容器间，或者任意两个容器之间共享。
*   文件许可 - Windows 使用基于 SID 的访问控制列表，而不是基于 UID+GID 的访问权限位掩码。
*   文件路径 - Windows 上的习惯是使用 ​`\`​ 而非 ​`/`​。Go 语言的 IO 库同时接受这两种文件路径分隔符。不过，当你在指定要在容器内解析的路径或命令行时， 可能需要使用 ​`\`​。
*   信号（Signal） - Windows 交互式应用以不同方式来处理终止事件，并可实现以下方式之一或组合：

*   UI 线程处理包含 ​`WM_CLOSE` ​在内的良定的消息
*   控制台应用使用控制处理程序来处理 Ctrl-C 或 Ctrl-Break
*   服务会注册服务控制处理程序，接受 ​`SERVICE_CONTROL_STOP` ​控制代码

退出代码遵从相同的习惯，0 表示成功，非 0 值表示失败。 特定的错误代码在 Windows 和 Linux 上可能会不同。不过，从 Kubernetes 组件 （kubelet、kube-proxy）所返回的退出代码是没有变化的。

*   ​`v1.Container.ResourceRequirements.limits.cpu`​ 和 ​`v1.Container.ResourceRequirements.limits.memory`​ - Windows 不对 CPU 分配设置硬性的限制。与之相反，Windows 使用一个份额（share）系统。 基于毫核（millicores）的现有字段值会被缩放为相对的份额值，供 Windows 调度器使用。

*   Windows 容器运行时中没有实现巨页支持，因此相关特性不可用。 巨页支持需要判定用户的特权 而这一特性无法在容器级别配置。

*   ​`v1.Container.ResourceRequirements.requests.cpu`​ 和 ​`v1.Container.ResourceRequirements.requests.memory`​ - 请求 值会从节点可分配资源中扣除，从而可用来避免节点上的资源过量分配。 但是，它们无法用来在一个已经过量分配的节点上提供资源保障。 如果操作员希望彻底避免过量分配，作为最佳实践，他们就需要为所有容器设置资源请求值。
*   ​`v1.Container.SecurityContext.allowPrivilegeEscalation`​ - 在 Windows 上无法实现，对应的权能无一可在 Windows 上生效。
*   ​`v1.Container.SecurityContext.Capabilities`​ - Windows 上未实现 POSIX 权能机制
*   ​`v1.Container.SecurityContext.privileged`​ - Windows 不支持特权容器
*   ​`v1.Container.SecurityContext.procMount`​ - Windows 不包含 ​`/proc`​ 文件系统
*   ​`v1.Container.SecurityContext.readOnlyRootFilesystem`​ - 在 Windows 上无法实现， 要在容器内使用注册表或运行系统进程就必需写访问权限。
*   ​`v1.Container.SecurityContext.runAsGroup`​ - 在 Windows 上无法实现，没有 GID 支持
*   ​`v1.Container.SecurityContext.runAsNonRoot`​ - Windows 上没有 root 用户。 与之最接近的等价用户是 ContainerAdministrator，而该身份标识在节点上并不存在。
*   ​`v1.Container.SecurityContext.runAsUser`​ - 在 Windows 上无法实现， 因为没有作为整数支持的 GID。
*   ​`v1.Container.SecurityContext.seLinuxOptions`​ - 在 Windows 上无法实现， 因为没有 SELinux
*   ​`V1.Container.terminationMessagePath`​ - 因为 Windows 不支持单个文件的映射，这一功能 在 Windows 上也受限。默认值 ​`/dev/termination-log`​ 在 Windows 上也无法使用因为 对应路径在 Windows 上不存在。

##### V1.Pod

*   ​`v1.Pod.hostIPC`​、​`v1.Pod.hostPID`​ - Windows 不支持共享宿主系统的名字空间
*   ​`v1.Pod.hostNetwork`​ - Windows 操作系统不支持共享宿主网络
*   ​`v1.Pod.dnsPolicy`​ - 不支持 ​`ClusterFirstWithHostNet`​，因为 Windows 不支持宿主网络
*   ​`v1.Pod.podSecurityContext`​ - 参见下面的 ​`v1.PodSecurityContext`​
*   ​`v1.Pod.shareProcessNamespace`​ - 此为 Beta 特性且依赖于 Windows 上未实现 的 Linux 名字空间。 Windows 无法共享进程名字空间或者容器的根文件系统。只能共享网络。
*   ​`v1.Pod.terminationGracePeriodSeconds`​ - 这一特性未在 Windows 版本的 Docker 中完全实现。 参见[问题报告](https://github.com/moby/moby/issues/25982)。 目前实现的行为是向 ​`ENTRYPOINT` ​进程发送 ​`CTRL_SHUTDOWN_EVENT` ​事件，之后 Windows 默认 等待 5 秒钟，并最终使用正常的 Windows 关机行为关闭所有进程。 这里的 5 秒钟默认值实际上保存在  [容器内](https://github.com/moby/moby/issues/25982 target=) 的 Windows 注册表中，因此可以在构造容器时重载。
*   ​`v1.Pod.volumeDevices`​ - 此为 Beta 特性且未在 Windows 上实现。Windows 无法挂接 原生的块设备到 Pod 中。
*   ​`v1.Pod.volumes`​ - ​`emptyDir`​、​`secret`​、​`configMap` ​和 ​`hostPath` ​都可正常工作且在 TestGrid 中测试。

*   ​`v1.emptyDir.volumeSource`​ - Windows 上节点的默认介质是磁盘。 不支持将内存作为介质，因为 Windows 不支持内置的 RAM 磁盘。

*   ​`v1.VolumeMount.mountPropagation`​ - Windows 上不支持挂载传播。

##### V1.PodSecurityContext

PodSecurityContext 的所有选项在 Windows 上都无法工作。这些选项列在下面仅供参考。

*   ​`v1.PodSecurityContext.seLinuxOptions`​ - Windows 上无 SELinux
*   ​`v1.PodSecurityContext.runAsUser`​ - 提供 UID；Windows 不支持
*   ​`v1.PodSecurityContext.runAsGroup`​ - 提供 GID；Windows 不支持
*   ​`v1.PodSecurityContext.runAsNonRoot`​ - Windows 上没有 root 用户 最接近的等价账号是 ​`ContainerAdministrator`​，而该身份标识在节点上不存在
*   ​`v1.PodSecurityContext.supplementalGroups`​ - 提供 GID；Windows 不支持
*   ​`v1.PodSecurityContext.sysctls`​ - 这些是 Linux sysctl 接口的一部分；Windows 上 没有等价机制。

#### 操作系统版本限制 

Windows 有着严格的兼容性规则，宿主 OS 的版本必须与容器基准镜像 OS 的版本匹配。 目前仅支持容器操作系统为 Windows Server 2019 的 Windows 容器。 对于容器的 Hyper-V 隔离、允许一定程度上的 Windows 容器镜像版本向后兼容性等等， 都是将来版本计划的一部分。

获取帮助和故障排查 
----------

Kubernetes 中日志是故障排查的一个重要元素。确保你在尝试从其他贡献者那里获得 故障排查帮助时提供日志信息。你可以按照 SIG-Windows [贡献指南和收集日志](https://github.com/kubernetes/community/blob/master/sig-windows/CONTRIBUTING.md target=) 所给的指令来操作。

*   我怎样知道 ​`start.ps1`​ 是否已成功完成？

你应该能看到节点上运行的 kubelet、kube-proxy 和（如果你选择 Flannel 作为联网方案）flanneld 宿主代理进程，它们的运行日志显示在不同的 PowerShell 窗口中。此外，你的 Windows 节点应该在你的 Kubernetes 集群 列举为 "Ready" 节点。

*   我可以将 Kubernetes 节点进程配置为服务运行在后台么？

kubelet 和 kube-proxy 都已经被配置为以本地 Windows 服务运行， 并且在出现失效事件（例如进程意外结束）时通过自动重启服务来提供一定的弹性。 你有两种办法将这些节点组件配置为服务。

*   以本地 Windows 服务的形式

Kubelet 和 kube-proxy 可以用 ​`sc.exe`​ 以本地 Windows 服务的形式运行：

`# 用两个单独的命令为 kubelet 和 kube-proxy 创建服务 sc.exe create <组件名称> binPath="<可执行文件路径> -service <其它参数>"  # 请注意如果参数中包含空格，必须使用转义 sc.exe create kubelet binPath= "C:\kubelet.exe --service --hostname-override 'minion' <其它参数>"  # 启动服务 Start-Service kubelet Start-Service kube-proxy  # 停止服务 Stop-Service kubelet (-Force) Stop-Service kube-proxy (-Force)  # 查询服务状态 Get-Service kubelet Get-Service kube-proxy`

*   使用 nssm.exe

你也总是可以使用替代的服务管理器，例如[nssm.exe](https://nssm.cc/)，来为你在后台运行 这些进程（​`flanneld`​、​`kubelet` ​和 ​`kube-proxy`​）。你可以使用这一 [示例脚本](https://github.com/microsoft/SDN/blob/master/Kubernetes/flannel/register-svc.ps1)， 利用 ​`nssm.exe`​ 将 ​`kubelet`​、​`kube-proxy`​ 和 ​`flanneld.exe`​ 注册为要在后台运行的 Windows 服务。

`register-svc.ps1 -NetworkMode <网络模式> -ManagementIP <Windows 节点 IP> -ClusterCIDR <集群子网> -KubeDnsServiceIP <kube-dns 服务 IP> -LogDir <日志目录>`

这里的参数解释如下：

*   ​`NetworkMode`​：网络模式 l2bridge（flannel host-gw，也是默认值）或 overlay（flannel vxlan）选做网络方案
*   ​`ManagementIP`​：分配给 Windows 节点的 IP 地址。你可以使用 ipconfig 得到此值
*   ​`ClusterCIDR`​：集群子网范围（默认值为 10.244.0.0/16）
*   ​`KubeDnsServiceIP`​：Kubernetes DNS 服务 IP（默认值为 10.96.0.10）
*   ​`LogDir`​：kubelet 和 kube-proxy 的日志会被重定向到这一目录中的对应输出文件， 默认值为 ​`C:\k`​。

若以上所引用的脚本不适合，你可以使用下面的例子手动配置 ​`nssm.exe`​。

注册 flanneld.exe：

`nssm install flanneld C:\flannel\flanneld.exe nssm set flanneld AppParameters --kubeconfig-file=c:\k\config --iface=<ManagementIP> --ip-masq=1 --kube-subnet-mgr=1 nssm set flanneld AppEnvironmentExtra NODE_NAME=<hostname> nssm set flanneld AppDirectory C:\flannel nssm start flanneld`

注册 kubelet.exe：

`nssm install kubelet C:\k\kubelet.exe nssm set kubelet AppParameters --hostname-override=<hostname> --v=6 --pod-infra-container-image=k8s.gcr.io/pause:3.5 --resolv-conf="" --allow-privileged=true --enable-debugging-handlers --cluster-dns=<DNS-service-IP> --cluster-domain=cluster.local --kubeconfig=c:\k\config --hairpin-mode=promiscuous-bridge --image-pull-progress-deadline=20m --cgroups-per-qos=false  --log-dir=<log directory> --logtostderr=false --enforce-node-allocatable="" --network-plugin=cni --cni-bin-dir=c:\k\cni --cni-conf-dir=c:\k\cni\config nssm set kubelet AppDirectory C:\k nssm start kubelet`

注册 kube-proxy.exe（二层网桥模式和主机网关模式）

`nssm install kube-proxy C:\k\kube-proxy.exe nssm set kube-proxy AppDirectory c:\k nssm set kube-proxy AppParameters --v=4 --proxy-mode=kernelspace --hostname-override=<hostname>--kubeconfig=c:\k\config --enable-dsr=false --log-dir=<log directory> --logtostderr=false nssm.exe set kube-proxy AppEnvironmentExtra KUBE_NETWORK=cbr0 nssm set kube-proxy DependOnService kubelet nssm start kube-proxy`

注册 kube-proxy.exe（覆盖网络模式或 VxLAN 模式）

`nssm install kube-proxy C:\k\kube-proxy.exe nssm set kube-proxy AppDirectory c:\k nssm set kube-proxy AppParameters --v=4 --proxy-mode=kernelspace --feature-gates="WinOverlay=true" --hostname-override=<hostname> --kubeconfig=c:\k\config --network-name=vxlan0 --source-vip=<source-vip> --enable-dsr=false --log-dir=<log directory> --logtostderr=false nssm set kube-proxy DependOnService kubelet nssm start kube-proxy`

作为初始的故障排查操作，你可以使用在 [nssm.exe](https://nssm.cc/) 中使用下面的标志 以便将标准输出和标准错误输出重定向到一个输出文件：

`nssm set <服务名称> AppStdout C:\k\mysvc.log nssm set <服务名称> AppStderr C:\k\mysvc.log`

要了解更多的细节，可参见官方的 [nssm 用法](https://nssm.cc/usage)文档。

*   我的 Windows Pods 无法连接网络

如果你在使用虚拟机，请确保 VM 网络适配器均已开启 MAC 侦听（Spoofing）。

*   我的 Windows Pods 无法 ping 外部资源

Windows Pods 目前没有为 ICMP 协议提供出站规则。不过 TCP/UDP 是支持的。 尝试与集群外资源连接时，可以将 ​`ping <IP>`​ 命令替换为对应的 ​`curl <IP>`​ 命令。

如果你还遇到问题，很可能你在 [cni.conf](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/l2bridge/cni/config/cni.conf) 中的网络配置值得额外的注意。你总是可以编辑这一静态文件。 配置的更新会应用到所有新创建的 Kubernetes 资源上。

Kubernetes 网络的需求之一是集群内部无需网络地址转译（NAT）即可实现通信。 为了符合这一要求，对所有我们不希望出站时发生 NAT 的通信都存在一个 [ExceptionList](https://github.com/Microsoft/SDN/blob/master/Kubernetes/flannel/l2bridge/cni/config/cni.conf target=)。 然而这也意味着你需要将你要查询的外部 IP 从 ExceptionList 中移除。 只有这时，从你的 Windows Pod 发起的网络请求才会被正确地通过 SNAT 转换以接收到 来自外部世界的响应。 就此而言，你在 ​`cni.conf`​ 中的 ​`ExceptionList` ​应该看起来像这样：

`"ExceptionList": [     "10.244.0.0/16",  # 集群子网     "10.96.0.0/12",   # 服务子网     "10.127.130.0/24" # 管理（主机）子网 ]`

*   我的 Windows 节点无法访问 NodePort 服务

从节点自身发起的本地 NodePort 请求会失败。这是一个已知的局限。 NodePort 服务的访问从其他节点或者外部客户端都可正常进行。

*   容器的 vNICs 和 HNS 端点被删除了

这一问题可能因为 ​`hostname-override`​ 参数未能传递给 kube-proxy 而导致。解决这一问题时，用户需要按如下方式将主机名传递给 kube-proxy：

`C:\k\kube-proxy.exe --hostname-override=$(hostname)`

*   使用 Flannel 时，我的节点在重新加入集群时遇到问题

无论何时，当一个之前被删除的节点被重新添加到集群时，flannelD 都会将为节点分配 一个新的 Pod 子网。 用户需要将将下面路径中的老的 Pod 子网配置文件删除：

`Remove-Item C:\k\SourceVip.json Remove-Item C:\k\SourceVipRequest.json`

*   在启动了 ​`start.ps1`​ 之后，flanneld 一直停滞在 "Waiting for the Network to be created" 状态

关于这一[问题](https://github.com/flannel-io/flannel/issues/1066)有很多的报告； 最可能的一种原因是关于何时设置 Flannel 网络的管理 IP 的时间问题。 一种解决办法是重新启动 ​`start.ps1`​ 或者按如下方式手动重启之：

`[Environment]::SetEnvironmentVariable("NODE_NAME", "<Windows 工作节点主机名>") C:\flannel\flanneld.exe --kubeconfig-file=c:\k\config --iface=<Windows 工作节点 IP> --ip-masq=1 --kube-subnet-mgr=1`

*   我的 Windows Pods 无法启动，因为缺少 ​`/run/flannel/subnet.env`​ 文件

这表明 Flannel 网络未能正确启动。你可以尝试重启 flanneld.exe 或者将文件手动地 从 Kubernetes 主控节点的 ​`/run/flannel/subnet.env`​ 路径复制到 Windows 工作 节点的 ​`C:\run\flannel\subnet.env`​ 路径，并将 ​`FLANNEL_SUBNET` ​行改为一个 不同的数值。例如，如果期望节点子网为 ​`10.244.4.1/24`​：

`FLANNEL_NETWORK=10.244.0.0/16 FLANNEL_SUBNET=10.244.4.1/24 FLANNEL_MTU=1500 FLANNEL_IPMASQ=true`

*   我的 Windows 节点无法使用服务 IP 访问我的服务

这是 Windows 上当前网络协议栈的一个已知的限制。 Windows Pods 能够访问服务 IP。

*   启动 kubelet 时找不到网络适配器

Windows 网络堆栈需要一个虚拟的适配器，这样 Kubernetes 网络才能工作。 如果下面的命令（在管理员 Shell 中）没有任何返回结果，证明虚拟网络创建 （kubelet 正常工作的必要前提之一）失败了：

`Get-HnsNetwork | ? Name -ieq "cbr0" Get-NetAdapter | ? Name -Like "vEthernet (Ethernet*"`

当宿主系统的网络适配器名称不是 "Ethernet" 时，通常值得更改 ​`start.ps1`​ 脚本中的 [InterfaceName](https://github.com/microsoft/SDN/blob/master/Kubernetes/flannel/start.ps1 target=) 参数来重试。否则可以查验 ​`start-kubelet.ps1`​ 的输出，看看是否在虚拟网络创建 过程中报告了其他错误。

*   我的 Pods 停滞在 "Container Creating" 状态或者反复重启

检查你的 pause 镜像是与你的 OS 版本兼容的。 如果你安装的是更新版本的 Windows，比如说 某个 Insider 构造版本，你需要相应地调整要使用的镜像。 请参照 Microsoft 的 [Docker 仓库](https://hub.docker.com/u/microsoft/) 了解镜像。不管怎样，pause 镜像的 Dockerfile 和示例服务都期望镜像的标签 为 ​`:latest`​。

*   ​`kubectl port-forward`​ 失败，错误信息为 "unable to do port forwarding: wincat not found"

此功能是在 Kubernetes v1.15 中实现的，pause 基础设施容器 ​`mcr.microsoft.com/oss/kubernetes/pause:3.4.1`​ 中包含了 wincat.exe。 请确保你使用的是这些版本或者更新版本。 如果你想要自行构造你自己的 pause 基础设施容器，要确保其中包含了 [wincat](https://github.com/kubernetes-sigs/sig-windows-tools/tree/master/cmd/wincat)

Windows 的端口转发支持需要在 pause 基础设施容器 中提供 wincat.exe。 确保你使用的是与你的 Windows 操作系统版本兼容的受支持镜像。 如果你想构建自己的 pause 基础架构容器，请确保包含 [wincat](https://github.com/kubernetes/kubernetes/tree/master/build/pause/windows/wincat)。

*   我的 Kubernetes 安装失败，因为我的 Windows Server 节点在防火墙后面

如果你处于防火墙之后，那么必须定义如下 PowerShell 环境变量：

`[Environment]::SetEnvironmentVariable("HTTP_PROXY", "http://proxy.example.com:80/", [EnvironmentVariableTarget]::Machine) [Environment]::SetEnvironmentVariable("HTTPS_PROXY", "http://proxy.example.com:443/", [EnvironmentVariableTarget]::Machine)`

*   ​`pause` ​容器是什么？

在一个 Kubernetes Pod 中，一个基础设施容器，或称 "pause" 容器，会被首先创建出来， 用以托管容器端点。属于同一 Pod 的容器，包括基础设施容器和工作容器，会共享相同的 网络名字空间和端点（相同的 IP 和端口空间）。我们需要 pause 容器来工作容器崩溃或 重启的状况，以确保不会丢失任何网络配置。

### 进一步探究 

如果以上步骤未能解决你遇到的问题，你可以通过以下方式获得在 Kubernetes 中的 Windows 节点上运行 Windows 容器的帮助：

*   StackOverflow [Windows Server Container](https://stackoverflow.com/questions/tagged/windows-server-container) 主题
*   Kubernetes 官方论坛 [discuss.kubernetes.io](https://discuss.kubernetes.io/)
*   Kubernetes Slack [#SIG-Windows 频道](https://kubernetes.slack.com/?redir=%2Fmessages%2Fsig-windows)

报告问题和功能需求 
----------

如果你遇到看起来像是软件缺陷的问题，或者你想要提起某种功能需求，请使用 [GitHub 问题跟踪系统](https://github.com/kubernetes/kubernetes/issues)。 你可以在 [GitHub](https://github.com/kubernetes/kubernetes/issues/new/choose) 上发起 Issue 并将其指派给 SIG-Windows。你应该首先搜索 Issue 列表，看看是否 该 Issue 以前曾经被报告过，以评论形式将你在该 Issue 上的体验追加进去，并附上 额外的日志信息。SIG-Windows Slack 频道也是一个获得初步支持的好渠道，可以在 生成新的 Ticket 之前对一些想法进行故障分析。

在登记软件缺陷时，请给出如何重现该问题的详细信息，例如：

*   Kubernetes 版本：kubectl 版本
*   环境细节：云平台、OS 版本、网络选型和配置情况以及 Docker 版本
*   重现该问题的详细步骤
*   [相关的日志](https://github.com/kubernetes/community/blob/master/sig-windows/CONTRIBUTING.md#gathering-logs)
*   通过为该 Issue 添加 ​`/sig windows`​ 评论为其添加 ​`sig/windows`​ 标签， 进而引起 SIG-Windows 成员的注意。

##  13.  Kubernetes Windows容器的调度指南
Kubernetes 中 Windows 容器的调度指南  

Windows 应用程序构成了许多组织中运行的服务和应用程序的很大一部分。 本指南将引导你完成在 Kubernetes 中配置和部署 Windows 容器的步骤。

目标
--

*   配置一个示例 deployment 以在 Windows 节点上运行 Windows 容器
*   （可选）使用组托管服务帐户（GMSA）为你的 Pod 配置 Active Directory 身份

在开始之前
-----

*   创建一个 Kubernetes 集群，其中包括一个控制平面和 运行 Windows 服务器的工作节点
*   重要的是要注意，对于 Linux 和 Windows 容器，在 Kubernetes 上创建和部署服务和工作负载的行为几乎相同。 与集群接口的 kubectl 命令相同。 提供以下部分中的示例只是为了快速启动 Windows 容器的使用体验。

入门：部署 Windows 容器
----------------

要在 Kubernetes 上部署 Windows 容器，你必须首先创建一个示例应用程序。 下面的示例 YAML 文件创建了一个简单的 Web 服务器应用程序。 创建一个名为 ​`win-webserver.yaml`​ 的服务规约，其内容如下：

`apiVersion: v1 kind: Service metadata:   name: win-webserver   labels:     app: win-webserver spec:   ports:     # the port that this service should serve on     - port: 80       targetPort: 80   selector:     app: win-webserver   type: NodePort --- apiVersion: apps/v1 kind: Deployment metadata:   labels:     app: win-webserver   name: win-webserver spec:   replicas: 2   selector:     matchLabels:       app: win-webserver   template:     metadata:       labels:         app: win-webserver       name: win-webserver     spec:      containers:       - name: windowswebserver         image: mcr.microsoft.com/windows/servercore:ltsc2019         command:         - powershell.exe         - -command         - "<#code used from https://gist.github.com/19WAS85/5424431#> ; $listener = New-Object System.Net.HttpListener ; $listener.Prefixes.Add('http://*:80/') ; $listener.Start() ; $callerCounts = @{} ; Write-Host('Listening at http://*:80/') ; while ($listener.IsListening) { ;$context = $listener.GetContext() ;$requestUrl = $context.Request.Url ;$clientIP = $context.Request.RemoteEndPoint.Address ;$response = $context.Response ;Write-Host '' ;Write-Host('> {0}' -f $requestUrl) ;  ;$count = 1 ;$k=$callerCounts.Get_Item($clientIP) ;if ($k -ne $null) { $count += $k } ;$callerCounts.Set_Item($clientIP, $count) ;$ip=(Get-NetAdapter | Get-NetIpAddress); $header='<html><body><H1>Windows Container Web Server</H1>' ;$callerCountsString='' ;$callerCounts.Keys | % { $callerCountsString+='<p>IP {0} callerCount {1} ' -f $ip[1].IPAddress,$callerCounts.Item($_) } ;$footer='</body></html>' ;$content='{0}{1}{2}' -f $header,$callerCountsString,$footer ;Write-Output $content ;$buffer = [System.Text.Encoding]::UTF8.GetBytes($content) ;$response.ContentLength64 = $buffer.Length ;$response.OutputStream.Write($buffer, 0, $buffer.Length) ;$response.Close() ;$responseStatus = $response.StatusCode ;Write-Host('< {0}' -f $responseStatus)  } ; "      nodeSelector:       kubernetes.io/os: windows`

> Note: 端口映射也是支持的，但为简单起见，在此示例中容器端口 80 直接暴露给服务。

1.  检查所有节点是否健康：

`kubectl get nodes`

3.  部署服务并观察 pod 更新：

`kubectl apply -f win-webserver.yaml kubectl get pods -o wide -w`

正确部署服务后，两个 Pod 都标记为 “Ready”。要退出 watch 命令，请按 Ctrl + C。

6.  检查部署是否成功。验证：

*   Windows 节点上每个 Pod 有两个容器，使用 ​`docker ps`​
*   Linux 控制平面节点列出两个 Pod，使用 ​`kubectl get pods`​
*   跨网络的节点到 Pod 通信，从 Linux 控制平面节点 ​`curl` ​你的 pod IPs 的端口 80，以检查 Web 服务器响应
*   Pod 到 Pod 的通信，使用 docker exec 或 kubectl exec 在 Pod 之间 （以及跨主机，如果你有多个 Windows 节点）进行 ping 操作
*   服务到 Pod 的通信，从 Linux 控制平面节点和各个 Pod 中 ​`curl` ​虚拟服务 IP （在 ​`kubectl get services`​ 下可见）
*   服务发现，使用 Kubernetes ​`curl` ​服务名称 默认 DNS 后缀
*   入站连接，从 Linux 控制平面节点或集群外部的计算机 ​`curl` ​NodePort
*   出站连接，使用 kubectl exec 从 Pod 内部 curl 外部 IP

> Note: 由于当前平台对 Windows 网络堆栈的限制，Windows 容器主机无法访问在其上调度的服务的 IP。只有 Windows pods 才能访问服务 IP。

可观测性 
-----

### 抓取来自工作负载的日志

日志是可观测性的重要一环；使用日志用户可以获得对负载运行状况的洞察， 因而日志是故障排查的一个重要手法。 因为 Windows 容器中的 Windows 容器和负载与 Linux 容器的行为不同， 用户很难收集日志，因此运行状态的可见性很受限。 例如，Windows 工作负载通常被配置为将日志输出到 Windows 事件跟踪 （Event Tracing for Windows，ETW），或者将日志条目推送到应用的事件日志中。  [LogMonitor](https://github.com/microsoft/windows-container-tools/tree/master/LogMonitor) 是 Microsoft 提供的一个开源工具，是监视 Windows 容器中所配置的日志源 的推荐方式。 LogMonitor 支持监视时间日志、ETW 提供者模块以及自定义的应用日志， 并使用管道的方式将其输出到标准输出（stdout），以便 ​`kubectl logs <pod>`​ 这类命令能够读取这些数据。

请遵照 LogMonitor GitHub 页面上的指令，将其可执行文件和配置文件复制到 你的所有容器中，并为其添加必要的入口点（Entrypoint），以便 LogMonitor 能够将你的日志输出推送到标准输出（stdout）。

使用可配置的容器用户名
-----------

从 Kubernetes v1.16 开始，可以为 Windows 容器配置与其镜像默认值不同的用户名 来运行其入口点和进程。 此能力的实现方式和 Linux 容器有些不同。

使用组托管服务帐户管理工作负载身份
-----------------

从 Kubernetes v1.14 开始，可以将 Windows 容器工作负载配置为使用组托管服务帐户（GMSA）。 组托管服务帐户是 Active Directory 帐户的一种特定类型，它提供自动密码管理， 简化的服务主体名称（SPN）管理以及将管理委派给跨多台服务器的其他管理员的功能。 配置了 GMSA 的容器可以访问外部 Active Directory 域资源，同时携带通过 GMSA 配置的身份。

污点和容忍度
------

目前，用户需要将 Linux 和 Windows 工作负载运行在各自特定的操作系统的节点上， 因而需要结合使用污点和节点选择算符。这可能仅给 Windows 用户造成不便。 推荐的方法概述如下，其主要目标之一是该方法不应破坏与现有 Linux 工作负载的兼容性。

如果 ​`IdentifyPodOS` ​特性门控是启用的， 你可以（并且应该）为 Pod 设置 ​`.spec.os.name`​ 以表明该 Pod 中的容器所针对的操作系统。对于运行 Linux 容器的 Pod，设置 ​`.spec.os.name`​ 为 ​`linux`​。对于运行 Windows 容器的 Pod，设置 ​`.spec.os.name`​ 为 ​`Windows`​。

> Note: 从 1.24 开始，​`IdentifyPodOS` ​功能处于 Beta 阶段，默认启用。

在将 Pod 分配给节点时，调度程序不使用 ​`.spec.os.name`​ 的值。你应该使用正常的 Kubernetes 机制将 Pod 分配给节点， 确保集群的控制平面将 Pod 放置到适合运行的操作系统。 ​`.spec.os.name`​ 值对 Windows Pod 的调度没有影响，因此仍然需要污点、容忍度以及节点选择器， 以确保 Windows Pod 调度至合适的 Windows 节点。

### 确保特定操作系统的工作负载落在适当的容器主机上

用户可以使用污点和容忍度确保 Windows 容器可以调度在适当的主机上。目前所有 Kubernetes 节点都具有以下默认标签：

*   kubernetes.io/os = \[windows|linux\]
*   kubernetes.io/arch = \[amd64|arm64|...\]

如果 Pod 规范未指定诸如 ​`"kubernetes.io/os": windows`​ 之类的 nodeSelector，则该 Pod 可能会被调度到任何主机（Windows 或 Linux）上。 这是有问题的，因为 Windows 容器只能在 Windows 上运行，而 Linux 容器只能在 Linux 上运行。 最佳实践是使用 nodeSelector。

但是，我们了解到，在许多情况下，用户都有既存的大量的 Linux 容器部署，以及一个现成的配置生态系统， 例如社区 Helm charts，以及程序化 Pod 生成案例，例如 Operators。 在这些情况下，你可能会不愿意更改配置添加 nodeSelector。替代方法是使用污点。 由于 kubelet 可以在注册期间设置污点，因此可以轻松修改它，使其仅在 Windows 上运行时自动添加污点。

例如：​`--register-with-taints='os=windows:NoSchedule'` ​

向所有 Windows 节点添加污点后，Kubernetes 将不会在它们上调度任何负载（包括现有的 Linux Pod）。 为了使某 Windows Pod 调度到 Windows 节点上，该 Pod 需要 nodeSelector 和合适的匹配的容忍度设置来选择 Windows。

`nodeSelector:     kubernetes.io/os: windows     node.kubernetes.io/windows-build: '10.0.17763' tolerations:     - key: "os"       operator: "Equal"       value: "windows"       effect: "NoSchedule"`

### 处理同一集群中的多个 Windows 版本

每个 Pod 使用的 Windows Server 版本必须与该节点的 Windows Server 版本相匹配。 如果要在同一集群中使用多个 Windows Server 版本，则应该设置其他节点标签和 nodeSelector。

Kubernetes 1.17 自动添加了一个新标签 ​`node.kubernetes.io/windows-build`​ 来简化此操作。 如果你运行的是旧版本，则建议手动将此标签添加到 Windows 节点。

此标签反映了需要兼容的 Windows 主要、次要和内部版本号。以下是当前每个 Windows Server 版本使用的值。

产品名称

内部编号

Windows Server 2019

10.0.17763

Windows Server version 1809

10.0.17763

Windows Server version 1903

10.0.18362

### 使用 RuntimeClass 简化

RuntimeClass 可用于 简化使用污点和容忍度的过程。 集群管理员可以创建 ​`RuntimeClass` ​对象，用于封装这些污点和容忍度。

1.  将此文件保存到 ​`runtimeClasses.yml`​ 文件。 它包括适用于 Windows 操作系统、体系结构和版本的 ​`nodeSelector`​。

`apiVersion: node.k8s.io/v1 kind: RuntimeClass metadata:   name: windows-2019 handler: 'docker' scheduling:   nodeSelector:     kubernetes.io/os: 'windows'     kubernetes.io/arch: 'amd64'     node.kubernetes.io/windows-build: '10.0.17763'   tolerations:   - effect: NoSchedule     key: os     operator: Equal     value: "windows"`

3.  集群管理员执行 ​`kubectl create -f runtimeClasses.yml`​ 操作
4.  根据需要向 Pod 规约中添加 ​`runtimeClassName: windows-2019`​，例如：

`apiVersion: apps/v1 kind: Deployment metadata:   name: iis-2019   labels:     app: iis-2019 spec:   replicas: 1   template:     metadata:       name: iis-2019       labels:         app: iis-2019     spec:       runtimeClassName: windows-2019       containers:       - name: iis         image: mcr.microsoft.com/windows/servercore/iis:windowsservercore-ltsc2019         resources:           limits:             cpu: 1             memory: 800Mi           requests:             cpu: .1             memory: 300Mi         ports:           - containerPort: 80  selector:     matchLabels:       app: iis-2019 --- apiVersion: v1 kind: Service metadata:   name: iis spec:   type: LoadBalancer   ports:   - protocol: TCP     port: 80   selector:     app: iis-2019`

##  2.  Kubernetes 最佳实践

###  2.1.  Kubernetes 运行于多可用区环境
背景 
---

Kubernetes 从设计上允许同一个 Kubernetes 集群跨多个失效区来运行， 通常这些区位于某个称作 区域（region） 逻辑分组中。 主要的云提供商都将区域定义为一组失效区的集合（也称作 可用区（Availability Zones））， 能够提供一组一致的功能特性：每个区域内，各个可用区提供相同的 API 和服务。

典型的云体系结构都会尝试降低某个区中的失效影响到其他区中服务的概率。

控制面行为 
------

所有的控制面组件 都支持以一组可相互替换的资源池的形式来运行，每个组件都有多个副本。

当你部署集群控制面时，应将控制面组件的副本跨多个失效区来部署。 如果可用性是一个很重要的指标，应该选择至少三个失效区，并将每个 控制面组件（API 服务器、调度器、etcd、控制器管理器）复制多个副本， 跨至少三个失效区来部署。如果你在运行云控制器管理器，则也应该将 该组件跨所选的三个失效区来部署。

> Note:  
> Kubernetes 并不会为 API 服务器端点提供跨失效区的弹性。 你可以为集群 API 服务器使用多种技术来提升其可用性，包括使用 DNS 轮转、SRV 记录或者带健康检查的第三方负载均衡解决方案等等。

节点行为 
-----

Kubernetes 自动为负载资源（如Deployment 或 StatefulSet)） 跨集群中不同节点来部署其 Pods。 这种分布逻辑有助于降低失效带来的影响。

节点启动时，每个节点上的 kubelet 会向 Kubernetes API 中代表该 kubelet 的 Node 对象 添加 标签。 这些标签可能包含区信息。

如果你的集群跨了多个可用区或者地理区域，你可以使用节点标签，结合 Pod 拓扑分布约束 来控制如何在你的集群中多个失效域之间分布 Pods。这里的失效域可以是 地理区域、可用区甚至是特定节点。 这些提示信息使得调度器 能够更好地分布 Pods，以实现更好的可用性，降低因为某种失效给整个工作负载 带来的风险。

例如，你可以设置一种约束，确保某个 StatefulSet 中的三个副本都运行在 不同的可用区中，只要其他条件允许。你可以通过声明的方式来定义这种约束， 而不需要显式指定每个工作负载使用哪些可用区。

#### 跨多个区分布节点

Kubernetes 的核心逻辑并不会帮你创建节点，你需要自行完成此操作，或者使用 类似 [Cluster API](https://cluster-api.sigs.k8s.io/) 这类工具来替你管理节点。

使用类似 Cluster API 这类工具，你可以跨多个失效域来定义一组用做你的集群 工作节点的机器，以及当整个区的服务出现中断时如何自动治愈集群的策略。

为 Pods 手动指定区
------------

你可以应用节点选择算符约束 到你所创建的 Pods 上，或者为 Deployment、StatefulSet 或 Job 这类工作负载资源 中的 Pod 模板设置此类约束。

跨区的存储访问
-------

当创建持久卷时，​`PersistentVolumeLabel` ​准入控制器 会自动向那些链接到特定区的 PersistentVolume 添加区标签。 调度器通过其 ​`NoVolumeZoneConflict` ​断言确保申领给定 PersistentVolume 的 Pods 只会 被调度到该卷所在的可用区。

你可以为 PersistentVolumeClaim 指定StorageClass 以设置该类中的存储可以使用的失效域（区）。

网络 
---

Kubernetes 自身不提供与可用区相关的联网配置。 你可以使用网络插件 来配置集群的联网，该网络解决方案可能拥有一些与可用区相关的元素。 例如，如果你的云提供商支持 ​`type=LoadBalancer`​ 的 Service，则负载均衡器 可能仅会将请求流量发送到运行在负责处理给定连接的负载均衡器组件所在的区。 请查阅云提供商的文档了解详细信息。

对于自定义的或本地集群部署，也可以考虑这些因素 Service Ingress 的行为， 包括处理不同失效区的方法，在很大程度上取决于你的集群是如何搭建的。

失效恢复 
-----

在搭建集群时，你可能需要考虑当某区域中的所有失效区都同时掉线时，是否以及如何 恢复服务。例如，你是否要求在某个区中至少有一个节点能够运行 Pod？ 请确保任何对集群很关键的修复工作都不要指望集群中至少有一个健康节点。 例如：当所有节点都不健康时，你可能需要运行某个修复性的 Job， 该 Job 要设置特定的容忍度 以便修复操作能够至少将一个节点恢复为可用状态。

Kubernetes 对这类问题没有现成的解决方案；不过这也是要考虑的因素之一。

###  2.2.  Kubernetes 大规模集群的注意事项
大规模集群的注意事项
----------

集群是运行 Kubernetes 代理的、 由控制平面管理的一组 节点（物理机或虚拟机）。 Kubernetes v1.24 支持的最大节点数为 5000。 更具体地说，Kubernetes旨在适应满足以下所有标准的配置：

*   每个节点的 Pod 数量不超过 110
*   节点数不超过 5000
*   Pod 总数不超过 150000
*   容器总数不超过 300000

你可以通过添加或删除节点来扩展集群。集群扩缩的方式取决于集群的部署方式。

云供应商资源配额
--------

为避免遇到云供应商配额问题，在创建具有大规模节点的集群时，请考虑以下事项：

*   请求增加云资源的配额，例如：

*   计算实例
*   CPUs
*   存储卷
*   使用中的 IP 地址
*   数据包过滤规则集
*   负载均衡数量
*   网络子网
*   日志流

*   由于某些云供应商限制了创建新实例的速度，因此通过分批启动新节点来控制集群扩展操作，并在各批之间有一个暂停。

控制面组件
-----

对于大型集群，你需要一个具有足够计算能力和其他资源的控制平面。

通常，你将在每个故障区域运行一个或两个控制平面实例， 先垂直缩放这些实例，然后在到达下降点（垂直）后再水平缩放。

你应该在每个故障区域至少应运行一个实例，以提供容错能力。 Kubernetes 节点不会自动将流量引向相同故障区域中的控制平面端点。 但是，你的云供应商可能有自己的机制来执行此操作。

例如，使用托管的负载均衡器时，你可以配置负载均衡器发送源自故障区域 A 中的 kubelet 和 Pod 的流量， 并将该流量仅定向到也位于区域 A 中的控制平面主机。 如果单个控制平面主机或端点故障区域 A 脱机，则意味着区域 A 中的节点的所有控制平面流量现在都在区域之间发送。 在每个区域中运行多个控制平面主机能降低出现这种结果的可能性。

#### etcd 存储

为了提高大规模集群的性能，你可以将事件对象存储在单独的专用 etcd 实例中。

在创建集群时，你可以（使用自定义工具）：

*   启动并配置额外的 etcd 实例
*   配置 API 服务器，将它用于存储事件

#### 插件资源 

Kubernetes 资源限制 有助于最大程度地减少内存泄漏的影响以及 Pod 和容器可能对其他组件的其他方式的影响。 这些资源限制适用于插件资源， 就像它们适用于应用程序工作负载一样。

例如，你可以对日志组件设置 CPU 和内存限制

  `...   containers:   - name: fluentd-cloud-logging     image: fluent/fluentd-kubernetes-daemonset:v1     resources:       limits:         cpu: 100m         memory: 200Mi`

插件的默认限制通常基于从中小规模 Kubernetes 集群上运行每个插件的经验收集的数据。 插件在大规模集群上运行时，某些资源消耗常常比其默认限制更多。 如果在不调整这些值的情况下部署了大规模集群，则插件可能会不断被杀死，因为它们不断达到内存限制。 或者，插件可能会运行，但由于 CPU 时间片的限制而导致性能不佳。

为避免遇到集群插件资源问题，在创建大规模集群时，请考虑以下事项：

*   部分垂直扩展插件 —— 总有一个插件副本服务于整个集群或服务于整个故障区域。 对于这些附加组件，请在扩大集群时加大资源请求和资源限制。
*   许多水平扩展插件 —— 你可以通过运行更多的 Pod 来增加容量——但是在大规模集群下， 可能还需要稍微提高 CPU 或内存限制。 VerticalPodAutoscaler 可以在 recommender 模式下运行， 以提供有关请求和限制的建议数字。
*   一些插件在每个节点上运行一个副本，并由 DaemonSet 控制： 例如，节点级日志聚合器。与水平扩展插件的情况类似， 你可能还需要稍微提高 CPU 或内存限制。

###  2.3.  Kubernetes 校验节点设置
节点一致性测试 
--------

节点一致性测试 是一个容器化的测试框架，提供了针对节点的系统验证和功能测试。 测试验证节点是否满足 Kubernetes 的最低要求；通过测试的节点有资格加入 Kubernetes 集群。

该测试主要检测节点是否满足 Kubernetes 的最低要求，通过检测的节点有资格加入 Kubernetes 集群。

节点的前提条件 
--------

要运行节点一致性测试，节点必须满足与标准 Kubernetes 节点相同的前提条件。节点至少应安装以下守护程序：

*   容器运行时 (Docker)
*   Kubelet

运行节点一致性测试 
----------

要运行节点一致性测试，请执行以下步骤：

1.  得出 kubelet 的 ​`--kubeconfig`​ 的值；例如：​`--kubeconfig=/var/lib/kubelet/config.yaml`​。 由于测试框架启动了本地控制平面来测试 kubelet，因此使用 ​`http://localhost:8080`​ 作为API 服务器的 URL。 一些其他的 kubelet 命令行参数可能会被用到：

*   ​`--cloud-provider`​：如果使用 ​`--cloud-provider=gce`​，需要移除这个参数来运行测试。

3.  使用以下命令运行节点一致性测试：

`# $CONFIG_DIR 是你 Kubelet 的 pod manifest 路径。 # $LOG_DIR 是测试的输出路径。 sudo docker run -it --rm --privileged --net=host \   -v /:/rootfs -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \   k8s.gcr.io/node-test:0.2`

针对其他硬件体系结构运行节点一致性测试 
--------------------

Kubernetes 也为其他硬件体系结构的系统提供了节点一致性测试的 Docker 镜像：

架构

镜像

amd64

node-test-amd64

arm

node-test-arm

arm64

node-test-arm64

运行特定的测试 
--------

要运行特定测试，请使用你希望运行的测试的特定表达式覆盖环境变量 ​`FOCUS`​。

`sudo docker run -it --rm --privileged --net=host \   -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \   -e FOCUS=MirrorPod \ # Only run MirrorPod test   k8s.gcr.io/node-test:0.2`

要跳过特定的测试，请使用你希望跳过的测试的常规表达式覆盖环境变量 ​`SKIP`​。

`sudo docker run -it --rm --privileged --net=host \   -v /:/rootfs:ro -v $CONFIG_DIR:$CONFIG_DIR -v $LOG_DIR:/var/result \   -e SKIP=MirrorPod \ # 运行除 MirrorPod 测试外的所有一致性测试内容   k8s.gcr.io/node-test:0.2`

节点一致性测试是[节点端到端测试](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/e2e-node-tests.md)的容器化版本。

默认情况下，它会运行所有一致性测试。

理论上，只要合理地配置容器和挂载所需的卷，就可以运行任何的节点端到端测试用例。但是这里强烈建议只运行一致性测试，因为运行非一致性测试需要很多复杂的配置。

注意事项 
-----

*   测试会在节点上遗留一些 Docker 镜像，包括节点一致性测试本身的镜像和功能测试相关的镜像。
*   测试会在节点上遗留一些死的容器。这些容器是在功能测试的过程中创建的。

###  2.4.  Kubernetes PKI证书和要求
PKI 证书和要求
---------

Kubernetes 需要 PKI 证书才能进行基于 TLS 的身份验证。如果你是使用 kubeadm 安装的 Kubernetes， 则会自动生成集群所需的证书。你还可以生成自己的证书。 例如，不将私钥存储在 API 服务器上，可以让私钥更加安全。此页面说明了集群必需的证书。

集群是如何使用证书的 
-----------

Kubernetes 需要 PKI 才能执行以下操作：

*   Kubelet 的客户端证书，用于 API 服务器身份验证
*   Kubelet 服务端证书， 用于 API 服务器与 Kubelet 的会话
*   API 服务器端点的证书
*   集群管理员的客户端证书，用于 API 服务器身份认证
*   API 服务器的客户端证书，用于和 Kubelet 的会话
*   API 服务器的客户端证书，用于和 etcd 的会话
*   控制器管理器的客户端证书/kubeconfig，用于和 API 服务器的会话
*   调度器的客户端证书/kubeconfig，用于和 API 服务器的会话
*   前端代理 的客户端及服务端证书

> Note: 只有当你运行 kube-proxy 并要支持 扩展 API 服务器 时，才需要 ​`front-proxy`​ 证书  

etcd 还实现了双向 TLS 来对客户端和对其他对等节点进行身份验证。

证书存放的位置 
--------

假如通过 kubeadm 安装 Kubernetes，大多数证书都存储在 ​`/etc/kubernetes/pki`​。 本文档中的所有路径都是相对于该目录的，但用户账户证书除外，kubeadm 将其放在 ​`/etc/kubernetes`​ 中。

手动配置证书 
-------

如果你不想通过 kubeadm 生成这些必需的证书，你可以使用一个单一的根 CA 来创建这些证书或者直接提供所有证书。 

#### 单根 CA 

你可以创建一个单根 CA，由管理员控制器它。该根 CA 可以创建多个中间 CA，并将所有进一步的创建委托给 Kubernetes。

需要这些 CA：

路径

默认 CN

描述

ca.crt,key

kubernetes-ca

Kubernetes 通用 CA

etcd/ca.crt,key

etcd-ca

与 etcd 相关的所有功能

front-proxy-ca.crt,key

kubernetes-front-proxy-ca

用于 前端代理

上面的 CA 之外，还需要获取用于服务账户管理的密钥对，也就是 ​`sa.key`​ 和 ​`sa.pub`​。

下面的例子说明了上表中所示的 CA 密钥和证书文件。

`/etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/etcd/ca.key /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/front-proxy-ca.key`

#### 所有的证书 

如果你不想将 CA 的私钥拷贝至你的集群中，你也可以自己生成全部的证书。

需要这些证书：

默认 CN

父级 CA

O (位于 Subject 中)

类型

主机 (SAN)

kube-etcd

etcd-ca

server, client

`<hostname>`, `<Host_IP>`, `localhost`, `127.0.0.1`

kube-etcd-peer

etcd-ca

server, client

`<hostname>`, `<Host_IP>`, `localhost`, `127.0.0.1`

kube-etcd-healthcheck-client

etcd-ca

client

kube-apiserver-etcd-client

etcd-ca

system:masters

client

kube-apiserver

kubernetes-ca

server

`<hostname>`, `<Host_IP>`, `<advertise_IP>`, `[1]`

kube-apiserver-kubelet-client

kubernetes-ca

system:masters

client

front-proxy-client

kubernetes-front-proxy-ca

client

\[1\]: 用来连接到集群的不同 IP 或 DNS 名 （就像 kubeadm 为负载均衡所使用的固定 IP 或 DNS 名，​`kubernetes`​、​`kubernetes.default`​、​`kubernetes.default.svc`​、 ​`kubernetes.default.svc.cluster`​、​`kubernetes.default.svc.cluster.local`​）。

其中，​`kind` ​对应一种或多种类型的 [x509 密钥用途](https://pkg.go.dev/k8s.io/api/certificates/v1beta1 target=)：

kind

密钥用途

server

数字签名、密钥加密、服务端认证

client

数字签名、密钥加密、客户端认证

> Note:  
> 上面列出的 Hosts/SAN 是推荐的配置方式；如果需要特殊安装，则可以在所有服务器证书上添加其他 SAN。

> Note:  
> 对于 kubeadm 用户：  
> 
> *   不使用私钥，将证书复制到集群 CA 的方案，在 kubeadm 文档中将这种方案称为外部 CA。
> *   如果将以上列表与 kubeadm 生成的 PKI 进行比较，你会注意到，如果使用外部 etcd，则不会生成 ​`kube-etcd`​、​`kube-etcd-peer`​ 和 ​`kube-etcd-healthcheck-client`​ 证书。

#### 证书路径 

证书应放置在建议的路径中（以便 kubeadm 使用）。无论使用什么位置，都应使用给定的参数指定路径。

默认 CN

建议的密钥路径

建议的证书路径

命令

密钥参数

证书参数

etcd-ca

etcd/ca.key

etcd/ca.crt

kube-apiserver

\--etcd-cafile

kube-apiserver-etcd-client

apiserver-etcd-client.key

apiserver-etcd-client.crt

kube-apiserver

\--etcd-keyfile

\--etcd-certfile

kubernetes-ca

ca.key

ca.crt

kube-apiserver

\--client-ca-file

kubernetes-ca

ca.key

ca.crt

kube-controller-manager

\--cluster-signing-key-file

\--client-ca-file, --root-ca-file, --cluster-signing-cert-file

kube-apiserver

apiserver.key

apiserver.crt

kube-apiserver

\--tls-private-key-file

\--tls-cert-file

kube-apiserver-kubelet-client

apiserver-kubelet-client.key

apiserver-kubelet-client.crt

kube-apiserver

\--kubelet-client-key

\--kubelet-client-certificate

front-proxy-ca

front-proxy-ca.key

front-proxy-ca.crt

kube-apiserver

\--requestheader-client-ca-file

front-proxy-ca

front-proxy-ca.key

front-proxy-ca.crt

kube-controller-manager

\--requestheader-client-ca-file

front-proxy-client

front-proxy-client.key

front-proxy-client.crt

kube-apiserver

\--proxy-client-key-file

\--proxy-client-cert-file

etcd-ca

etcd/ca.key

etcd/ca.crt

etcd

\--trusted-ca-file, --peer-trusted-ca-file

kube-etcd

etcd/server.key

etcd/server.crt

etcd

\--key-file

\--cert-file

kube-etcd-peer

etcd/peer.key

etcd/peer.crt

etcd

\--peer-key-file

\--peer-cert-file

etcd-ca

etcd/ca.crt

etcdctl

\--cacert

kube-etcd-healthcheck-client

etcd/healthcheck-client.key

etcd/healthcheck-client.crt

etcdctl

\--key

\--cert

注意事项同样适用于服务帐户密钥对：

私钥路径

公钥路径

命令

参数

sa.key

kube-controller-manager

\--service-account-private-key-file

sa.pub

kube-apiserver

\--service-account-key-file

下面的例子展示了自行生成所有密钥和证书时所需要提供的文件路径。 这些路径基于前面的表格。

`/etc/kubernetes/pki/etcd/ca.key /etc/kubernetes/pki/etcd/ca.crt /etc/kubernetes/pki/apiserver-etcd-client.key /etc/kubernetes/pki/apiserver-etcd-client.crt /etc/kubernetes/pki/ca.key /etc/kubernetes/pki/ca.crt /etc/kubernetes/pki/apiserver.key /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver-kubelet-client.key /etc/kubernetes/pki/apiserver-kubelet-client.crt /etc/kubernetes/pki/front-proxy-ca.key /etc/kubernetes/pki/front-proxy-ca.crt /etc/kubernetes/pki/front-proxy-client.key /etc/kubernetes/pki/front-proxy-client.crt /etc/kubernetes/pki/etcd/server.key /etc/kubernetes/pki/etcd/server.crt /etc/kubernetes/pki/etcd/peer.key /etc/kubernetes/pki/etcd/peer.crt /etc/kubernetes/pki/etcd/healthcheck-client.key /etc/kubernetes/pki/etcd/healthcheck-client.crt /etc/kubernetes/pki/sa.key /etc/kubernetes/pki/sa.pub`

为用户帐户配置证书 
----------

你必须手动配置以下管理员帐户和服务帐户：

文件名

凭据名称

默认 CN

O (位于 Subject 中)

admin.conf

default-admin

kubernetes-admin

system:masters

kubelet.conf

default-auth

system:node:`<nodeName>` （参阅注释）

system:nodes

controller-manager.conf

default-controller-manager

system:kube-controller-manager

scheduler.conf

default-scheduler

system:kube-scheduler

> Note: ​`kubelet.conf`​ 中 ​`<nodeName>`​ 的值 必须 与 kubelet 向 apiserver 注册时提供的节点名称的值完全匹配。

1.  对于每个配置，请都使用给定的 CN 和 O 生成 x509 证书/密钥偶对。
2.  为每个配置运行下面的 ​`kubectl` ​命令：

`KUBECONFIG=<filename> kubectl config set-cluster default-cluster --server=https://<host ip>:6443 --certificate-authority <path-to-kubernetes-ca> --embed-certs KUBECONFIG=<filename> kubectl config set-credentials <credential-name> --client-key <path-to-key>.pem --client-certificate <path-to-cert>.pem --embed-certs KUBECONFIG=<filename> kubectl config set-context default-system --cluster default-cluster --user <credential-name> KUBECONFIG=<filename> kubectl config use-context default-system`

这些文件用途如下：

文件名

命令

说明

admin.conf

kubectl

配置集群的管理员

kubelet.conf

kubelet

集群中的每个节点都需要一份

controller-manager.conf

kube-controller-manager

必需添加到 `manifests/kube-controller-manager.yaml` 清单中

scheduler.conf

kube-scheduler

必需添加到 `manifests/kube-scheduler.yaml` 清单中

下面是前表中所列文件的完整路径。

`/etc/kubernetes/admin.conf /etc/kubernetes/kubelet.conf /etc/kubernetes/controller-manager.conf /etc/kubernetes/scheduler.conf`

###  2.5.  Kubernetes 强制实施Pod安全性标准
使用内置的 Pod 安全性准入控制器 
-------------------

FEATURE STATE: Kubernetes v1.23 \[beta\]

Pod 安全性准入控制器 尝试替换已被废弃的 PodSecurityPolicies。

#### 配置所有集群名字空间 

完全未经配置的名字空间应该被视为集群安全模型中的重大缺陷。 我们建议花一些时间来分析在每个名字空间中执行的负载的类型， 并通过引用 Pod 安全性标准来确定每个负载的合适级别。 未设置标签的名字空间应该视为尚未被评估。

针对所有名字空间中的所有负载都具有相同的安全性需求的场景， 我们提供了一个[示例](https://www.w3cschool.cn/kubernetes/kubernetes-pm913o6d.html) 用来展示如何批量应用 Pod 安全性标签。

#### 拥抱最小特权原则

在一个理想环境中，每个名字空间中的每个 Pod 都会满足 ​`restricted` ​策略的需求。 不过，这既不可能也不现实，某些负载会因为合理的原因而需要特权上的提升。

*   允许 ​`privileged` ​负载的名字空间需要建立并实施适当的访问控制机制。
*   对于运行在特权宽松的名字空间中的负载，需要维护其独特安全性需求的文档。 如果可能的话，要考虑如何进一步约束这些需求。

#### 采用多种模式的策略

Pod 安全性标准准入控制器的 ​`audit` ​和 ​`warn` ​模式（mode） 能够在不影响现有负载的前提下，让该控制器更方便地收集关于 Pod 的重要的安全信息。

针对所有名字空间启用这些模式是一种好的实践，将它们设置为你最终打算 ​`enforce` ​的 期望的 级别和版本。这一阶段中所生成的警告和审计注解信息可以帮助你到达这一状态。 如果你期望负载的作者能够作出变更以便适应期望的级别，可以启用 ​`warn` ​模式。 如果你希望使用审计日志了监控和驱动变更，以便负载能够适应期望的级别，可以启用 ​`audit` ​模式。

当你将 ​`enforce` ​模式设置为期望的取值时，这些模式在不同的场合下仍然是有用的：

*   通过将 ​`warn` ​设置为 ​`enforce` ​相同的级别，客户可以在尝试创建无法通过合法检查的 Pod （或者包含 Pod 模板的资源）时收到警告信息。这些信息会帮助于更新资源使其合规。
*   在将 ​`enforce` ​锁定到特定的非最新版本的名字空间中，将 ​`audit` ​和 ​`warn` ​模式设置为 ​`enforce` ​一样的级别而非 ​`latest` ​版本， 这样可以方便看到之前版本所允许但当前最佳实践中被禁止的设置。

第三方替代方案
-------

Kubernetes 生态系统中也有一些其他强制实施安全设置的替代方案处于开发状态中：

*   [Kubewarden](https://github.com/kubewarden)
*   [Kyverno](https://kyverno.io/policies/)
*   [OPA Gatekeeper](https://github.com/open-policy-agent/gatekeeper)

采用 内置的 方案（例如 PodSecurity 准入控制器）还是第三方工具， 这一决策完全取决于你自己的情况。在评估任何解决方案时，对供应链的信任都是至关重要的。 最终，使用前述方案中的 任何 一种都好过放任自流。

#  2.  Kubernetes 概述

##  1.  Kubernetes 简介
简介
--

Kubernetes 是一个可移植的、可扩展的开源平台，用于管理容器化的工作负载和服务，可促进声明式配置和自动化。 Kubernetes 拥有一个庞大且快速增长的生态系统。Kubernetes 的服务、支持和工具广泛可用。

Kubernetes 这个名字源于希腊语，意为“舵手”或“飞行员”。k8s 这个缩写是因为 k 和 s 之间有八个字符的关系。 Google 在 2014 年开源了 Kubernetes 项目。Kubernetes 建立在 [Google 在大规模运行生产工作负载方面拥有十几年的经验](https://research.google/pubs/pub43438/) 的基础上，结合了社区中最好的想法和实践。

时光回溯
----

让我们回顾一下为什么 Kubernetes 如此有用。

![](https://atts.w3cschool.cn/attachments/image/20220428/1651109065276234.jpg)  

### 传统部署时代：

早期，各个组织机构在物理服务器上运行应用程序。无法为物理服务器中的应用程序定义资源边界，这会导致资源分配问题。 例如，如果在物理服务器上运行多个应用程序，则可能会出现一个应用程序占用大部分资源的情况， 结果可能导致其他应用程序的性能下降。 一种解决方案是在不同的物理服务器上运行每个应用程序，但是由于资源利用不足而无法扩展， 并且维护许多物理服务器的成本很高。

### 虚拟化部署时代：

作为解决方案，引入了虚拟化。虚拟化技术允许你在单个物理服务器的 CPU 上运行多个虚拟机（VM）。 虚拟化允许应用程序在 VM 之间隔离，并提供一定程度的安全，因为一个应用程序的信息 不能被另一应用程序随意访问。

虚拟化技术能够更好地利用物理服务器上的资源，并且因为可轻松地添加或更新应用程序 而可以实现更好的可伸缩性，降低硬件成本等等。

每个 VM 是一台完整的计算机，在虚拟化硬件之上运行所有组件，包括其自己的操作系统。

### 容器部署时代：

容器类似于 VM，但是它们具有被放宽的隔离属性，可以在应用程序之间共享操作系统（OS）。 因此，容器被认为是轻量级的。容器与 VM 类似，具有自己的文件系统、CPU、内存、进程空间等。 由于它们与基础架构分离，因此可以跨云和 OS 发行版本进行移植。

容器因具有许多优势而变得流行起来。下面列出的是容器的一些好处：

*   敏捷应用程序的创建和部署：与使用 VM 镜像相比，提高了容器镜像创建的简便性和效率。
*   持续开发、集成和部署：通过快速简单的回滚（由于镜像不可变性），支持可靠且频繁的 容器镜像构建和部署。
*   关注开发与运维的分离：在构建/发布时而不是在部署时创建应用程序容器镜像， 从而将应用程序与基础架构分离。
*   可观察性：不仅可以显示操作系统级别的信息和指标，还可以显示应用程序的运行状况和其他指标信号。
*   跨开发、测试和生产的环境一致性：在便携式计算机上与在云中相同地运行。
*   跨云和操作系统发行版本的可移植性：可在 Ubuntu、RHEL、CoreOS、本地、 Google Kubernetes Engine 和其他任何地方运行。
*   以应用程序为中心的管理：提高抽象级别，从在虚拟硬件上运行 OS 到使用逻辑资源在 OS 上运行应用程序。
*   松散耦合、分布式、弹性、解放的微服务：应用程序被分解成较小的独立部分， 并且可以动态部署和管理 - 而不是在一台大型单机上整体运行。
*   资源隔离：可预测的应用程序性能。
*   资源利用：高效率和高密度。

为什么需要 Kubernetes，它能做什么?
-----------------------

容器是打包和运行应用程序的好方式。在生产环境中，你需要管理运行应用程序的容器，并确保不会停机。 例如，如果一个容器发生故障，则需要启动另一个容器。如果系统处理此行为，会不会更容易？

这就是 Kubernetes 来解决这些问题的方法！ Kubernetes 为你提供了一个可弹性运行分布式系统的框架。 Kubernetes 会满足你的扩展要求、故障转移、部署模式等。 例如，Kubernetes 可以轻松管理系统的 Canary 部署。

Kubernetes 为你提供：

*   服务发现和负载均衡
Kubernetes 可以使用 DNS 名称或自己的 IP 地址公开容器，如果进入容器的流量很大， Kubernetes 可以负载均衡并分配网络流量，从而使部署稳定。

*   存储编排

Kubernetes 允许你自动挂载你选择的存储系统，例如本地存储、公共云提供商等。

*   自动部署和回滚
    

你可以使用 Kubernetes 描述已部署容器的所需状态，它可以以受控的速率将实际状态 更改为期望状态。例如，你可以自动化 Kubernetes 来为你的部署创建新容器， 删除现有容器并将它们的所有资源用于新容器。

*   自动完成装箱计算
    

Kubernetes 允许你指定每个容器所需 CPU 和内存（RAM）。 当容器指定了资源请求时，Kubernetes 可以做出更好的决策来管理容器的资源。

*   自我修复
    

Kubernetes 重新启动失败的容器、替换容器、杀死不响应用户定义的 运行状况检查的容器，并且在准备好服务之前不将其通告给客户端。

*   密钥与配置管理
    

Kubernetes 允许你存储和管理敏感信息，例如密码、OAuth 令牌和 ssh 密钥。 你可以在不重建容器镜像的情况下部署和更新密钥和应用程序配置，也无需在堆栈配置中暴露密钥。

Kubernetes 不是什么
---------------

Kubernetes 不是传统的、包罗万象的 PaaS（平台即服务）系统。 由于 Kubernetes 在容器级别而不是在硬件级别运行，它提供了 PaaS 产品共有的一些普遍适用的功能， 例如部署、扩展、负载均衡、日志记录和监视。 但是，Kubernetes 不是单体系统，默认解决方案都是可选和可插拔的。 Kubernetes 提供了构建开发人员平台的基础，但是在重要的地方保留了用户的选择和灵活性。

Kubernetes：

*   不限制支持的应用程序类型。 Kubernetes 旨在支持极其多种多样的工作负载，包括无状态、有状态和数据处理工作负载。 如果应用程序可以在容器中运行，那么它应该可以在 Kubernetes 上很好地运行。
*   不部署源代码，也不构建你的应用程序。 持续集成(CI)、交付和部署（CI/CD）工作流取决于组织的文化和偏好以及技术要求。
*   不提供应用程序级别的服务作为内置服务，例如中间件（例如，消息中间件）、 数据处理框架（例如，Spark）、数据库（例如，mysql）、缓存、集群存储系统 （例如，Ceph）。这样的组件可以在 Kubernetes 上运行，并且/或者可以由运行在 Kubernetes 上的应用程序通过可移植机制（例如， [开放服务代理](https://www.openservicebrokerapi.org/)）来访问。
*   不要求日志记录、监视或警报解决方案。 它提供了一些集成作为概念证明，并提供了收集和导出指标的机制。
*   不提供或不要求配置语言/系统（例如 jsonnet），它提供了声明性 API， 该声明性 API 可以由任意形式的声明性规范所构成。
*   不提供也不采用任何全面的机器配置、维护、管理或自我修复系统。
*   此外，Kubernetes 不仅仅是一个编排系统，实际上它消除了编排的需要。 编排的技术定义是执行已定义的工作流程：首先执行 A，然后执行 B，再执行 C。 相比之下，Kubernetes 包含一组独立的、可组合的控制过程， 这些过程连续地将当前状态驱动到所提供的所需状态。 如何从 A 到 C 的方式无关紧要，也不需要集中控制，这使得系统更易于使用 且功能更强大、系统更健壮、更为弹性和可扩展。

##  2.  Kubernetes 组件
组件
--

当你部署完 Kubernetes, 即拥有了一个完整的集群。

一个 Kubernetes 集群由一组被称作节点的机器组成。这些节点上运行 Kubernetes 所管理的容器化应用。集群具有至少一个工作节点。

工作节点托管作为应用负载的组件的 Pod 。控制平面管理集群中的工作节点和 Pod 。 为集群提供故障转移和高可用性，这些控制平面一般跨多主机运行，集群跨多个节点运行。

本文档概述了交付正常运行的 Kubernetes 集群所需的各种组件。

![](https://atts.w3cschool.cn/attachments/image/20220428/1651110837317224.svg)  

控制平面组件（Control Plane Components）
--------------------------------

控制平面的组件对集群做出全局决策(比如调度)，以及检测和响应集群事件（例如，当不满足部署的 ​`replicas` ​字段时，启动新的 pod）。

控制平面组件可以在集群中的任何节点上运行。 然而，为了简单起见，设置脚本通常会在同一个计算机上启动所有控制平面组件， 并且不会在此计算机上运行用户容器。 

### kube-apiserver

API 服务器是 Kubernetes 控制面的组件， 该组件公开了 Kubernetes API。 API 服务器是 Kubernetes 控制面的前端。

Kubernetes API 服务器的主要实现是 kube-apiserver。 kube-apiserver 设计上考虑了水平伸缩，也就是说，它可通过部署多个实例进行伸缩。 你可以运行 kube-apiserver 的多个实例，并在这些实例之间平衡流量。

### etcd

etcd 是兼具一致性和高可用性的键值数据库，可以作为保存 Kubernetes 所有集群数据的后台数据库。

您的 Kubernetes 集群的 etcd 数据库通常需要有个备份计划。

### kube-scheduler

控制平面组件，负责监视新创建的、未指定运行节点（node）的 Pods，选择节点让 Pod 在上面运行。

调度决策考虑的因素包括单个 Pod 和 Pod 集合的资源需求、硬件/软件/策略约束、亲和性和反亲和性规范、数据位置、工作负载间的干扰和最后时限。

### kube-controller-manager

运行控制器进程的控制平面组件。

从逻辑上讲，每个控制器都是一个单独的进程， 但是为了降低复杂性，它们都被编译到同一个可执行文件，并在一个进程中运行。

这些控制器包括:

*   节点控制器（Node Controller）: 负责在节点出现故障时进行通知和响应
*   任务控制器（Job controller）: 监测代表一次性任务的 Job 对象，然后创建 Pods 来运行这些任务直至完成
*   端点控制器（Endpoints Controller）: 填充端点(Endpoints)对象(即加入 Service 与 Pod)
*   服务帐户和令牌控制器（Service Account & Token Controllers）: 为新的命名空间创建默认帐户和 API 访问令牌

### cloud-controller-manager

云控制器管理器是指嵌入特定云的控制逻辑的 控制平面组件。 云控制器管理器使得你可以将你的集群连接到云提供商的 API 之上， 并将与该云平台交互的组件同与你的集群交互的组件分离开来。

​`cloud-controller-manager`​ 仅运行特定于云平台的控制回路。 如果你在自己的环境中运行 Kubernetes，或者在本地计算机中运行学习环境， 所部署的环境中不需要云控制器管理器。

与 ​`kube-controller-manager`​ 类似，​`cloud-controller-manager`​ 将若干逻辑上独立的 控制回路组合到同一个可执行文件中，供你以同一进程的方式运行。 你可以对其执行水平扩容（运行不止一个副本）以提升性能或者增强容错能力。

下面的控制器都包含对云平台驱动的依赖：

*   节点控制器（Node Controller）: 用于在节点终止响应后检查云提供商以确定节点是否已被删除
*   路由控制器（Route Controller）: 用于在底层云基础架构中设置路由
*   服务控制器（Service Controller）: 用于创建、更新和删除云提供商负载均衡器

Node 组件 
--------

节点组件在每个节点上运行，维护运行的 Pod 并提供 Kubernetes 运行环境。

### kubelet

一个在集群中每个节点（node）上运行的代理。 它保证容器（containers）都 运行在 Pod 中。

kubelet 接收一组通过各类机制提供给它的 PodSpecs，确保这些 PodSpecs 中描述的容器处于运行状态且健康。 kubelet 不会管理不是由 Kubernetes 创建的容器。

### kube-proxy

kube-proxy 是集群中每个节点上运行的网络代理， 实现 Kubernetes 服务（Service） 概念的一部分。

kube-proxy 维护节点上的网络规则。这些网络规则允许从集群内部或外部的网络会话与 Pod 进行网络通信。

如果操作系统提供了数据包过滤层并可用的话，kube-proxy 会通过它来实现网络规则。否则， kube-proxy 仅转发流量本身。

### 容器运行时（Container Runtime） 

容器运行环境是负责运行容器的软件。

Kubernetes 支持容器运行时，例如 Docker、 containerd、CRI-O 以及 [Kubernetes CRI (容器运行环境接口)](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-node/container-runtime-interface.md) 的其他任何实现。

插件（Addons） 
-----------

插件使用 Kubernetes 资源（DaemonSet、 Deployment等）实现集群功能。 因为这些插件提供集群级别的功能，插件中命名空间域的资源属于 ​`kube-system`​ 命名空间。

下面描述众多插件中的几种。

### DNS 

尽管其他插件都并非严格意义上的必需组件，但几乎所有 Kubernetes 集群都应该 有集群 DNS， 因为很多示例都需要 DNS 服务。

集群 DNS 是一个 DNS 服务器，和环境中的其他 DNS 服务器一起工作，它为 Kubernetes 服务提供 DNS 记录。

Kubernetes 启动的容器自动将此 DNS 服务器包含在其 DNS 搜索列表中。

### Web 界面（仪表盘）

Dashboard 是 Kubernetes 集群的通用的、基于 Web 的用户界面。 它使用户可以管理集群中运行的应用程序以及集群本身并进行故障排除。

### 容器资源监控

容器资源监控 将关于容器的一些常见的时间序列度量值保存到一个集中的数据库中，并提供用于浏览这些数据的界面。

### 集群层面日志

集群层面日志 机制负责将容器的日志数据 保存到一个集中的日志存储中，该存储能够提供搜索和浏览接口。

##  3.  Kubernetes API
API
---

Kubernetes 控制面 的核心是 API 服务器。 API 服务器负责提供 HTTP API，以供用户、集群中的不同部分和集群外部组件相互通信。

Kubernetes API 使你可以查询和操纵 Kubernetes API 中对象（例如：Pod、Namespace、ConfigMap 和 Event）的状态。

大部分操作都可以通过 kubectl 命令行接口或 类似 kubeadm 这类命令行工具来执行， 这些工具在背后也是调用 API。不过，你也可以使用 REST 调用来访问这些 API。

如果你正在编写程序来访问 Kubernetes API，可以考虑使用 客户端库之一。

OpenAPI 规范
----------

完整的 API 细节是用 [OpenAPI](https://www.openapis.org/) 来表述的。

### OpenAPI V2

Kubernetes API 服务器通过 ​`/openapi/v2`​ 端点提供聚合的 OpenAPI v2 规范。 你可以按照下表所给的请求头部，指定响应的格式：

头部

可选值

说明

`Accept-Encoding`

`gzip`

_不指定此头部也是可以的_

`Accept`

`application/com.github.proto-openapi.spec.v2@v1.0+protobuf`

_主要用于集群内部_

`application/json`

_默认值_

`*`

_提供_`application/json`

OpenAPI v2 查询请求的合法头部值

Kubernetes 为 API 实现了一种基于 Protobuf 的序列化格式，主要用于集群内部通信。 关于此格式的详细信息，可参考 [Kubernetes Protobuf 序列化](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/api-machinery/protobuf.md) 设计提案。每种模式对应的接口描述语言（IDL）位于定义 API 对象的 Go 包中。

### OpenAPI V3

FEATURE STATE: Kubernetes v1.23 \[alpha\]

Kubernetes v1.23 提供将其 API 以 OpenAPI v3 形式发布的初始支持；这一功能特性处于 Alpha 状态，默认被禁用。 你可以通过为 kube-apiserver 组件启用 ​`OpenAPIV3` ​特性门控来启用此 Alpha 特性。

特性被启用时，Kubernetes API 服务器会在端点 ​`/openapi/v3/apis/<group>/<version>`​ 提供按 Kubernetes 组版本聚合的 OpenAPI v3 规范。 请参阅下表了解可接受的请求头部。

头部

可选值

说明

`Accept-Encoding`

`gzip`

_不提供此头部也是可接受的_

`Accept`

`application/com.github.proto-openapi.spec.v3@v1.0+protobuf`

_主要用于集群内部使用_

`application/json`

_默认_

`*`

_以_ `application/json` 形式返回

发现端点 ​`/openapi/v3`​ 被提供用来查看可用的所有组、版本列表。 此列表仅返回 JSON。

API 变更 
-------

任何成功的系统都要随着新的使用案例的出现和现有案例的变化来成长和变化。 为此，Kubernetes 的功能特性设计考虑了让 Kubernetes API 能够持续变更和成长的因素。 Kubernetes 项目的目标是 不要 引发现有客户端的兼容性问题，并在一定的时期内 维持这种兼容性，以便其他项目有机会作出适应性变更。

一般而言，新的 API 资源和新的资源字段可以被频繁地添加进来。 删除资源或者字段则要遵从 API 废弃策略。

Kubernetes 对维护达到正式发布（GA）阶段的官方 API 的兼容性有着很强的承诺， 通常这一 API 版本为 v1。此外，Kubernetes 在可能的时候还会保持 Beta API 版本的兼容性：如果你采用了 Beta API，你可以继续在集群上使用该 API， 即使该功能特性已进入稳定期也是如此。

> Note:  
> 尽管 Kubernetes 也努力为 Alpha API 版本维护兼容性，在有些场合兼容性是无法做到的。 如果你使用了任何 Alpha API 版本，需要在升级集群时查看 Kubernetes 发布说明， 以防 API 的确发生变更。

API 扩展 
-------

有两种途径来扩展 Kubernetes API：

1.  你可以使用自定义资源 来以声明式方式定义 API 服务器如何提供你所选择的资源 API。
2.  你也可以选择实现自己的 聚合层 来扩展 Kubernetes API。

#  3.  Kubernetes 安装

##  1.  Kubernetes Linux安装
kubectl 版本和集群版本之间的差异必须在一个小版本号内。 例如：v1.23 版本的客户端能与 v1.22、 v1.23 和 v1.24 版本的控制面通信。 用最新兼容版的 kubectl 有助于避免不可预见的问题。

在 Linux 系统中安装 kubectl
---------------------

在 Linux 系统中安装 kubectl 有如下几种方法：

*   用 curl 在 Linux 系统中安装 kubectl
*   用原生包管理工具安装
*   用其他包管理工具安装

用 curl 在 Linux 系统中安装 kubectl 
-----------------------------

1、用以下命令下载最新发行版：  

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"`

> Note:  
> 如需下载某个指定的版本，请用指定版本号替换该命令的这一部分：​ `$(curl -L -s https://dl.k8s.io/release/stable.txt)。` ​  
> 例如，要在 Linux 中下载 v1.23.0 版本，请输入：  
> 
> `curl -LO https://dl.k8s.io/release/v1.23.0/bin/linux/amd64/kubectl`

2、验证该可执行文件（可选步骤）  

*   下载 kubectl 校验和文件：

`curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256"`

*   基于校验和文件，验证 kubectl 的可执行文件：

`echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check`

*   验证通过时，输出为：

`kubectl: OK`

*   验证失败时，sha256 将以非零值退出，并打印如下输出：

`kubectl: FAILED sha256sum: WARNING: 1 computed checksum did NOT match`

> 下载的 kubectl 与校验和文件版本必须相同。

3、安装 kubectl

`sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl`

> 即使你没有目标系统的 root 权限，仍然可以将 kubectl 安装到目录 ~/.local/bin 中：
> 
> `chmod +x kubectl mkdir -p ~/.local/bin mv ./kubectl ~/.local/bin/kubectl # 之后将 ~/.local/bin 附加（或前置）到 $PATH`

4、执行测试，以保障你安装的版本是最新的：

`kubectl version --client`

*   或者使用如下命令来查看版本的详细信息：

`kubectl version --client --output=yaml`

用原生包管理工具安装
----------

### Ubuntu、Debian 或 HypriotOS

1、更新 ​`apt` ​包索引，并安装使用 Kubernetes apt 仓库所需要的包：

`sudo apt-get update sudo apt-get install -y apt-transport-https ca-certificates curl`

2、下载 Google Cloud 公开签名秘钥：

`sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg`

3、添加 Kubernetes ​`apt` ​仓库：

`echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list`

4、更新 ​`apt` ​包索引，使之包含新的仓库并安装 kubectl：

`sudo apt-get update sudo apt-get install -y kubectl`

### 基于 Red Hat 的发行版

`cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo [kubernetes] name=Kubernetes baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64 enabled=1 gpgcheck=1 repo_gpgcheck=1 gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg EOF sudo yum install -y kubectl`

用其他包管理工具安装
----------

### Snap

如果你使用的 Ubuntu 或其他 Linux 发行版，内建支持 [snap](https://snapcraft.io/docs/installing-snapd) 包管理工具， 则可用 [snap](https://snapcraft.io/) 命令安装 kubectl。

`snap install kubectl --classic kubectl version --client`

### Homebrew

如果你使用 Linux 系统，并且装了 [Homebrew](https://docs.brew.sh/Homebrew-on-Linux) 包管理工具， 则可以使用这种方式安装 kubectl。

`brew install kubectl kubectl version --client`

验证 kubectl 配置 
--------------

为了让 kubectl 能发现并访问 Kubernetes 集群，你需要一个 kubeconfig 文件， 该文件在 [kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh) 创建集群时，或成功部署一个 Miniube 集群时，均会自动生成。 通常，kubectl 的配置信息存放于文件 ​`~/.kube/config`​ 中。

通过获取集群状态的方法，检查是否已恰当的配置了 kubectl：

`kubectl cluster-info`

如果返回一个 URL，则意味着 kubectl 成功的访问到了你的集群。

如果你看到如下所示的消息，则代表 kubectl 配置出了问题，或无法连接到 Kubernetes 集群。

`The connection to the server <server-name:port> was refused - did you specify the right host or port? （访问 <server-name:port> 被拒绝 - 你指定的主机和端口是否有误？）`

例如，如果你想在自己的笔记本上（本地）运行 Kubernetes 集群，你需要先安装一个 Minikube 这样的工具，然后再重新运行上面的命令。

如果命令 ​`kubectl cluster-info`​ 返回了 url，但你还不能访问集群，那可以用以下命令来检查配置是否妥当：

`kubectl cluster-info dump`

kubectl 的可选配置和插件
----------------

### 启用 shell 自动补全功能

kubectl 为 Bash、Zsh、Fish 和 PowerShell 提供自动补全功能，可以为你节省大量的输入。

下面是为 Bash、Fish 和 Zsh 设置自动补全功能的操作步骤。

### Bash

kubectl 的 Bash 补全脚本可以用命令 ​`kubectl completion bash`​ 生成。 在 shell 中导入（Sourcing）补全脚本，将启用 kubectl 自动补全功能。

然而，补全脚本依赖于工具 [bash-completion](https://github.com/scop/bash-completion)， 所以要先安装它（可以用命令 ​`type _init_completion`​ 检查 bash-completion 是否已安装）。

#### 安装 bash-completion

很多包管理工具均支持 bash-completion（参见[这里](https://github.com/scop/bash-completion target=)）。 可以通过 ​`apt-get install bash-completion`​ 或 ​`yum install bash-completion`​ 等命令来安装它。

上述命令将创建文件 ​`/usr/share/bash-completion/bash_completion`​，它是 bash-completion 的主脚本。 依据包管理工具的实际情况，你需要在 ​`~/.bashrc`​ 文件中手工导入此文件。

要查看结果，请重新加载你的 shell，并运行命令 ​`type _init_completion`​。 如果命令执行成功，则设置完成，否则将下面内容添加到文件 ​`~/.bashrc`​ 中：

`source /usr/share/bash-completion/bash_completion`

重新加载 shell，再输入命令 ​`type _init_completion`​ 来验证 bash-completion 的安装状态。

#### 启动 kubectl 自动补全功能 

你现在需要确保一点：kubectl 补全脚本已经导入（sourced）到 shell 会话中。 可以通过以下两种方法进行设置：

*   当前用户

`echo 'source <(kubectl completion bash)' >>~/.bashrc`

*   系统全局

`kubectl completion bash | sudo tee /etc/bash_completion.d/kubectl > /dev/null`

如果 kubectl 有关联的别名，你可以扩展 shell 补全来适配此别名：

`echo 'alias k=kubectl' >>~/.bashrc echo 'complete -F __start_kubectl k' >>~/.bashrc`

> bash-completion 负责导入 ​`/etc/bash_completion.d`​ 目录中的所有补全脚本。

两种方式的效果相同。重新加载 shell 后，kubectl 自动补全功能即可生效。

### Fish

kubectl 通过命令 ​`kubectl completion fish`​ 生成 Fish 自动补全脚本。 在 shell 中导入（Sourcing）该自动补全脚本，将启动 kubectl 自动补全功能。

为了在所有的 shell 会话中实现此功能，请将下面内容加入到文件 ​`~/.config/fish/config.fish`​ 中。

`kubectl completion fish | source`

重新加载 shell 后，kubectl 自动补全功能将立即生效。

### Zsh

kubectl 通过命令 ​`kubectl completion zsh`​ 生成 Zsh 自动补全脚本。 在 shell 中导入（Sourcing）该自动补全脚本，将启动 kubectl 自动补全功能。

为了在所有的 shell 会话中实现此功能，请将下面内容加入到文件 ​`~/.zshrc`​ 中。

`source <(kubectl completion zsh)`

如果你为 kubectl 定义了别名，kubectl 自动补全将自动使用它。

重新加载 shell 后，kubectl 自动补全功能将立即生效。

如果你收到 ​`2: command not found: compdef`​ 这样的错误提示，那请将下面内容添加到 ​`~/.zshrc`​ 文件的开头：

`autoload -Uz compinit compinit`

### 安装 kubectl convert 插件

一个 Kubernetes 命令行工具 ​`kubectl` ​的插件，允许你将清单在不同 API 版本间转换。 这对于将清单迁移到新的 Kubernetes 发行版上未被废弃的 API 版本时尤其有帮助。

1、用以下命令下载最新发行版：

`curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert"`

2、验证该可执行文件（可选步骤）

*   下载 kubectl-convert 校验和文件：

`curl -LO "https://dl.k8s.io/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl-convert.sha256"`

*   基于校验和，验证 kubectl-convert 的可执行文件：

`echo "$(cat kubectl-convert.sha256) kubectl-convert" | sha256sum --check`

*   验证通过时，输出为：

`kubectl-convert: OK`

验证失败时，​`sha256` ​将以非零值退出，并打印输出类似于：

`kubectl-convert: FAILED sha256sum: WARNING: 1 computed checksum did NOT match`

> 下载相同版本的可执行文件和校验和。

3、安装 kubectl-convert

`sudo install -o root -g root -m 0755 kubectl-convert /usr/local/bin/kubectl-convert`

4、验证插件是否安装成功

`kubectl convert --help`

如果你没有看到任何错误就代表插件安装成功了。

##  2.  Kubernetes macOS安装
kubectl 版本和集群之间的差异必须在一个小版本号之内。 例如：v1.23 版本的客户端能与 v1.22、 v1.23 和 v1.24 版本的控制面通信。 用最新兼容版本的 kubectl 有助于避免不可预见的问题。

在 macOS 系统上安装 kubectl
---------------------

在 macOS 系统上安装 kubectl 有如下方法：

*   用 curl 在 macOS 系统上安装 kubectl
*   用 Homebrew 在 macOS 系统上安装
*   用 Macports 在 macOS 上安装
*   作为谷歌云 SDK 的一部分，在 macOS 上安装

用 curl 在 macOS 系统上安装 kubectl 
-----------------------------

1、下载最新的发行版：

*   Intel

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"`

*   Apple Silicon

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"`

> 如果需要下载某个指定的版本，用该指定版本号替换掉命令的这个部分：​`$(curl -L -s https://dl.k8s.io/release/stable.txt)`​。 例如：要为 Intel macOS 系统下载 v1.23.0 版本，则输入：
> 
> `curl -LO "https://dl.k8s.io/release/v1.23.0/bin/darwin/amd64/kubectl"`
> 
> 对于 Apple Silicon 版本的 macOS，输入：
> 
> `curl -LO "https://dl.k8s.io/release/v1.23.0/bin/darwin/arm64/kubectl"`

  

2、验证可执行文件（可选操作）

下载 kubectl 的校验和文件：

*   Intel

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl.sha256"`

*   Apple Silicon

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl.sha256"`

*   根据校验和文件，验证 kubectl：

`echo "$(cat kubectl.sha256)  kubectl" | shasum -a 256 --check`

*   验证通过时，输出如下：

`kubectl: OK`

*   验证失败时，​`shasum` ​将以非零值退出，并打印如下输出：

`kubectl: FAILED shasum: WARNING: 1 computed checksum did NOT match`

> 下载的 kubectl 与校验和文件版本要相同。

3、将 kubectl 置为可执行文件：

`chmod +x ./kubectl`

4、将可执行文件 kubectl 移动到系统可寻址路径 ​`PATH` ​内的一个位置：

`sudo mv ./kubectl /usr/local/bin/kubectl sudo chown root: /usr/local/bin/kubectl`

> 确保 ​`/usr/local/bin`​ 在你的 PATH 环境变量中。

5、测试一下，确保你安装的是最新的版本：

`kubectl version --client`

*   或者使用下面命令来查看版本的详细信息：

`kubectl version --client --output=yaml`

用 Homebrew 在 macOS 系统上安装
------------------------

如果你是 macOS 系统，且用的是 [Homebrew](https://brew.sh/) 包管理工具， 则可以用 Homebrew 安装 kubectl。

1、运行安装命令：

`brew install kubectl` 

*   或

`brew install kubernetes-cli`

2、测试一下，确保你安装的是最新的版本：

`kubectl version --client`

用 Macports 在 macOS 上安装
----------------------

如果你用的是 macOS，且用 [Macports](https://macports.org/) 包管理工具，则你可以用 Macports 安装kubectl。

1、运行安装命令：

`sudo port selfupdate sudo port install kubectl`

2、测试一下，确保你安装的是最新的版本：

`kubectl version --client`

验证 kubectl 配置
-------------

为了让 kubectl 能发现并访问 Kubernetes 集群，你需要一个 kubeconfig 文件， 该文件在 [kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh) 创建集群时，或成功部署一个 Miniube 集群时，均会自动生成。 通常，kubectl 的配置信息存放于文件 ​`~/.kube/config`​ 中。

通过获取集群状态的方法，检查是否已恰当的配置了 kubectl：

`kubectl cluster-info`

如果返回一个 URL，则意味着 kubectl 成功的访问到了你的集群。

如果你看到如下所示的消息，则代表 kubectl 配置出了问题，或无法连接到 Kubernetes 集群。

`The connection to the server <server-name:port> was refused - did you specify the right host or port? （访问 <server-name:port> 被拒绝 - 你指定的主机和端口是否有误？）`

例如，如果你想在自己的笔记本上（本地）运行 Kubernetes 集群，你需要先安装一个 Minikube 这样的工具，然后再重新运行上面的命令。

如果命令 ​`kubectl cluster-info`​ 返回了 url，但你还不能访问集群，那可以用以下命令来检查配置是否妥当：

`kubectl cluster-info dump`

可选的 kubectl 配置和插件
-----------------

### 启用 shell 自动补全功能

kubectl 为 Bash、Zsh、Fish 和 PowerShell 提供自动补全功能，可以为你节省大量的输入。

下面是为 Bash、Fish 和 Zsh 设置自动补全功能的操作步骤。

### Bash

kubectl 的 Bash 补全脚本可以通过 ​`kubectl completion bash`​ 命令生成。 在你的 shell 中导入（Sourcing）这个脚本即可启用补全功能。

此外，kubectl 补全脚本依赖于工具 [bash-completion](https://github.com/scop/bash-completion)， 所以你必须先安装它。

> **Warning**:  
> bash-completion 有两个版本：v1 和 v2。v1 对应 Bash3.2（也是 macOS 的默认安装版本），v2 对应 Bash 4.1+。 kubectl 的补全脚本**无法适配** bash-completion v1 和 Bash 3.2。 必须为它配备 **bash-completion v2** 和 **Bash 4.1+**。 有鉴于此，为了在 macOS 上使用 kubectl 补全功能，你必须要安装和使用 Bash 4.1+ ([说明](https://itnext.io/upgrading-bash-on-macos-7138bd1066ba?gi=9cba5d635a48))。 后续说明假定你用的是 Bash 4.1+（也就是 Bash 4.1 或更新的版本）

#### 升级 Bash

后续说明假定你已使用 Bash 4.1+。你可以运行以下命令检查 Bash 版本：

`echo $BASH_VERSION`

如果版本太旧，可以用 Homebrew 安装/升级：

`brew install bash`

重新加载 shell，并验证所需的版本已经生效：

`echo $BASH_VERSION $SHELL`

Homebrew 通常把它安装为 ​`/usr/local/bin/bash`​。

#### 安装 bash-completion 

> 如前所述，本说明假定你使用的 Bash 版本为 4.1+，这意味着你要安装 bash-completion v2 （不同于 Bash 3.2 和 bash-completion v1，kubectl 的补全功能在该场景下无法工作）。

你可以用命令 ​`type _init_completion`​ 测试 bash-completion v2 是否已经安装。 如未安装，用 Homebrew 来安装它：

`brew install bash-completion@2`

如命令的输出信息所显示的，将如下内容添加到文件 ​`~/.bash_profile`​ 中：

`export BASH_COMPLETION_COMPAT_DIR="/usr/local/etc/bash_completion.d" [[ -r "/usr/local/etc/profile.d/bash_completion.sh" ]] && . "/usr/local/etc/profile.d/bash_completion.sh"`

重新加载 shell，并用命令 ​`type _init_completion`​ 验证 bash-completion v2 已经恰当的安装。

#### 启用 kubectl 自动补全功能

你现在需要确保在所有的 shell 环境中均已导入（sourced） kubectl 的补全脚本， 有若干种方法可以实现这一点：

*   在文件 ​`~/.bash_profile`​ 中导入（Source）补全脚本：

`echo 'source <(kubectl completion bash)' >>~/.bash_profile`

*   将补全脚本添加到目录 ​`/usr/local/etc/bash_completion.d`​ 中：

`kubectl completion bash >/usr/local/etc/bash_completion.d/kubectl`

*   如果你为 kubectl 定义了别名，则可以扩展 shell 补全来兼容该别名：

`echo 'alias k=kubectl' >>~/.bash_profile echo 'complete -F __start_kubectl k' >>~/.bash_profile`

如果你是用 Homebrew 安装的 kubectl，则kubectl 补全脚本应该已经安装到目录 ​`/usr/local/etc/bash_completion.d/kubectl`​ 中了。这种情况下，你什么都不需要做。

> 用 Hommbrew 安装的 bash-completion v2 会初始化 目录 ​`BASH_COMPLETION_COMPAT_DIR`​ 中的所有文件，这就是后两种方法能正常工作的原因。

总之，重新加载 shell 之后，kubectl 补全功能将立即生效。

### Fish

kubectl 通过命令 ​`kubectl completion fish`​ 生成 Fish 自动补全脚本。 在 shell 中导入（Sourcing）该自动补全脚本，将启动 kubectl 自动补全功能。

为了在所有的 shell 会话中实现此功能，请将下面内容加入到文件 ​`~/.config/fish/config.fish`​ 中。

`kubectl completion fish | source`

重新加载 shell 后，kubectl 自动补全功能将立即生效。

### Zsh

kubectl 通过命令 ​`kubectl completion zsh`​ 生成 Zsh 自动补全脚本。 在 shell 中导入（Sourcing）该自动补全脚本，将启动 kubectl 自动补全功能。

为了在所有的 shell 会话中实现此功能，请将下面内容加入到文件 ​`~/.zshrc`​ 中。

`source <(kubectl completion zsh)`

如果你为 kubectl 定义了别名，kubectl 自动补全将自动使用它。

重新加载 shell 后，kubectl 自动补全功能将立即生效。

如果你收到 ​`2: command not found: compdef`​ 这样的错误提示，那请将下面内容添加到 ​`~/.zshrc`​ 文件的开头：

`autoload -Uz compinit compinit`

### 安装 kubectl convert 插件 

一个 Kubernetes 命令行工具 kubectl 的插件，允许你将清单在不同 API 版本间转换。 这对于将清单迁移到新的 Kubernetes 发行版上未被废弃的 API 版本时尤其有帮助。 更多信息请访问 迁移到非弃用 API

1、用以下命令下载最新发行版：

*   Intel

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl-convert"`

*   Apple Silicon

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl-convert"`

2、验证该可执行文件（可选步骤）

下载 kubectl-convert 校验和文件：

*   Intel

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl-convert.sha256"`

*   Apple Silicon

   `curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl-convert.sha256"`

基于校验和，验证 kubectl-convert 的可执行文件：

`echo "$(cat kubectl-convert.sha256)  kubectl-convert" | shasum -a 256 --check`

验证通过时，输出为：

`kubectl-convert: OK`

验证失败时，​`sha256` ​将以非零值退出，并打印输出类似于：

`kubectl-convert: FAILED shasum: WARNING: 1 computed checksum did NOT match`

> 下载相同版本的可执行文件和校验和。

3、使 kubectl-convert 二进制文件可执行

`chmod +x ./kubectl-convert`

4、将 kubectl-convert 可执行文件移动到系统 PATH 环境变量中的一个位置。

`sudo mv ./kubectl-convert /usr/local/bin/kubectl-convert sudo chown root: /usr/local/bin/kubectl-convert`

> 确保你的 PATH 环境变量中存在 ​`/usr/local/bin`​

5、验证插件是否安装成功

`kubectl convert --help`

如果你没有看到任何错误就代表插件安装成功了。

##  3.  Kubernetes Windows安装
kubectl 版本和集群版本之间的差异必须在一个小版本号内。 例如：v1.23 版本的客户端能与 v1.22、 v1.23 和 v1.24 版本的控制面通信。 用最新兼容版的 kubectl 有助于避免不可预见的问题。

在 Windows 上安装 kubectl 
----------------------

在 Windows 系统中安装 kubectl 有如下几种方法：

*   用 curl 在 Windows 上安装 kubectl
*   在 Windows 上用 Chocolatey 或 Scoop 安装

用 curl 在 Windows 上安装 kubectl
----------------------------

1、下载 [最新发行版 v1.23.0](https://storage.googleapis.com/kubernetes-release/release/v1.23.0/bin/windows/amd64/kubectl.exe)。

如果你已安装了 ​`curl`​,也可以使用此命令：

`curl -LO "https://dl.k8s.io/release/v1.23.0/bin/windows/amd64/kubectl.exe"`

2、验证该可执行文件（可选步骤）

下载 ​`kubectl` ​校验和文件：

`curl -LO "https://dl.k8s.io/v1.23.0/bin/windows/amd64/kubectl.exe.sha256"`

基于校验和文件，验证 kubectl 的可执行文件：

*   在命令行环境中，手工对比 CertUtil 命令的输出与校验和文件：

`CertUtil -hashfile kubectl.exe SHA256 type kubectl.exe.sha256`

用 PowerShell 自动验证，用运算符 ​`-eq`​ 来直接取得 ​`True` ​或 ​`False` ​的结果：

`$($(CertUtil -hashfile .\kubectl.exe SHA256)[1] -replace " ", "") -eq $(type .\kubectl.exe.sha256)`

3、将 ​`kubectl` ​二进制文件夹追加或插入到你的 ​`PATH` ​环境变量中。

4、测试一下，确保此 ​`kubectl` ​的版本和期望版本一致：

`kubectl version --client`

或者使用下面命令来查看版本的详细信息：

`kubectl version --client --output=yaml`

> [Windows 版的 Docker Desktop](https://docs.docker.com/desktop/windows/) 将其自带版本的 ​`kubectl` ​添加到 ​`PATH`​。 如果你之前安装过 Docker Desktop，可能需要把此 ​`PATH` ​条目置于 Docker Desktop 安装的条目之前， 或者直接删掉 Docker Desktop 的 ​`kubectl`​。

在 Windows 上用 Chocolatey 或 Scoop 安装
----------------------------------

1、要在 Windows 上安装 kubectl，你可以使用包管理器 [Chocolatey](https://chocolatey.org/) 或是命令行安装器 [Scoop](https://scoop.sh/)。

*   choco

`choco install kubernetes-cli`

*   scoop

`scoop install kubectl`

2、测试一下，确保安装的是最新版本：

`kubectl version --client`

3、导航到你的 home 目录：

`# 当你用 cmd.exe 时，则运行： cd %USERPROFILE% cd ~`

4、创建目录 ​`.kube`​：

`mkdir .kube`

5、切换到新创建的目录 ​`.kube` ​

`cd .kube`

6、配置 kubectl，以接入远程的 Kubernetes 集群：

`New-Item config -type file`

> 编辑配置文件，你需要先选择一个文本编辑器，比如 Notepad。

验证 kubectl 配置
-------------

为了让 kubectl 能发现并访问 Kubernetes 集群，你需要一个 kubeconfig 文件， 该文件在 [kube-up.sh](https://github.com/kubernetes/kubernetes/blob/master/cluster/kube-up.sh) 创建集群时，或成功部署一个 Miniube 集群时，均会自动生成。 通常，kubectl 的配置信息存放于文件 ​`~/.kube/config`​ 中。

通过获取集群状态的方法，检查是否已恰当的配置了 kubectl：

`kubectl cluster-info`

如果返回一个 URL，则意味着 kubectl 成功的访问到了你的集群。

如果你看到如下所示的消息，则代表 kubectl 配置出了问题，或无法连接到 Kubernetes 集群。

`The connection to the server <server-name:port> was refused - did you specify the right host or port? （访问 <server-name:port> 被拒绝 - 你指定的主机和端口是否有误？）`

例如，如果你想在自己的笔记本上（本地）运行 Kubernetes 集群，你需要先安装一个 Minikube 这样的工具，然后再重新运行上面的命令。

如果命令 ​`kubectl cluster-info`​ 返回了 url，但你还不能访问集群，那可以用以下命令来检查配置是否妥当：

`kubectl cluster-info dump`

kubectl 可选配置和插件
---------------

### 启用 shell 自动补全功能

kubectl 为 Bash、Zsh、Fish 和 PowerShell 提供自动补全功能，可以为你节省大量的输入。

下面是设置 PowerShell 自动补全功能的操作步骤。

使用命令 ​`kubectl completion powershell`​ 生成 PowerShell 的 kubectl 自动补全脚本。

如果需要自动补全在所有 shell 会话中生效，请将以下命令添加到 ​`$PROFILE`​ 文件中：

`kubectl completion powershell | Out-String | Invoke-Expression`

此命令将在每次 PowerShell 启动时重新生成自动补全脚本。你还可以将生成的自动补全脚本添加到 ​`$PROFILE`​ 文件中。

如果需要将自动补全脚本直接添加到 ​`$PROFILE`​ 文件中，请在 PowerShell 终端运行以下命令：

`kubectl completion powershell >> $PROFILE`

完成上述操作后重启 shell，kubectl的自动补全就可以工作了。

### 安装 kubectl convert 插件

一个 Kubernetes 命令行工具 ​`kubectl` ​的插件，允许你将清单在不同 API 版本间转换。 这对于将清单迁移到新的 Kubernetes 发行版上未被废弃的 API 版本时尤其有帮助。

1、用以下命令下载最新发行版：

`curl -LO "https://dl.k8s.io/release/v1.23.0/bin/windows/amd64/kubectl-convert.exe"`

2、验证该可执行文件（可选步骤）

下载 ​`kubectl-convert`​ 校验和文件：

`curl -LO "https://dl.k8s.io/v1.23.0/bin/windows/amd64/kubectl-convert.exe.sha256"`

基于校验和，验证 ​`kubectl-convert`​ 的可执行文件：

*   用提示的命令对 ​`CertUtil` ​的输出和下载的校验和文件进行手动比较。

`CertUtil -hashfile kubectl-convert.exe SHA256 type kubectl-convert.exe.sha256`

*   使用 PowerShell ​`-eq`​ 操作使验证自动化，获得 ​`True` ​或者 ​`False` ​的结果：

`$($(CertUtil -hashfile .\kubectl-convert.exe SHA256)[1] -replace " ", "") -eq $(type .\kubectl-convert.exe.sha256)`

3、将 kubectl-convert 二进制文件夹附加或添加到你的 PATH 环境变量中。

4、验证插件是否安装成功

`kubectl convert --help`

如果你没有看到任何错误就代表插件安装成功了。

#  4.  Kubernetes 对象

##  1.  Kubernetes 对象简介
本页说明了 Kubernetes 对象在 Kubernetes API 中是如何表示的，以及如何在 ​`.yaml`​ 格式的文件中表示。

理解 Kubernetes 对象
----------------

在 Kubernetes 系统中，Kubernetes 对象 是持久化的实体。 Kubernetes 使用这些实体去表示整个集群的状态。特别地，它们描述了如下信息：

*   哪些容器化应用在运行（以及在哪些节点上）
*   可以被应用使用的资源
*   关于应用运行时表现的策略，比如重启策略、升级策略，以及容错策略

Kubernetes 对象是 “目标性记录” —— 一旦创建对象，Kubernetes 系统将持续工作以确保对象存在。 通过创建对象，本质上是在告知 Kubernetes 系统，所需要的集群工作负载看起来是什么样子的， 这就是 Kubernetes 集群的 期望状态（Desired State）。

操作 Kubernetes 对象 —— 无论是创建、修改，或者删除 —— 需要使用 Kubernetes API。 比如，当使用 kubectl 命令行接口时，CLI 会执行必要的 Kubernetes API 调用， 也可以在程序中使用 客户端库直接调用 Kubernetes API。

对象规约（Spec）与状态（Status） 
----------------------

几乎每个 Kubernetes 对象包含两个嵌套的对象字段，它们负责管理对象的配置： 对象 ​`spec`​（规约） 和 对象 ​`status`​（状态） 。 对于具有 ​`spec` ​的对象，你必须在创建对象时设置其内容，描述你希望对象所具有的特征： 期望状态（Desired State） 。

​`status` ​描述了对象的 当前状态（Current State），它是由 Kubernetes 系统和组件 设置并更新的。在任何时刻，Kubernetes 控制平面 都一直积极地管理着对象的实际状态，以使之与期望状态相匹配。

例如，Kubernetes 中的 Deployment 对象能够表示运行在集群中的应用。 当创建 Deployment 时，可能需要设置 Deployment 的 ​`spec`​，以指定该应用需要有 3 个副本运行。 Kubernetes 系统读取 Deployment 规约，并启动我们所期望的应用的 3 个实例 —— 更新状态以与规约相匹配。 如果这些实例中有的失败了（一种状态变更），Kubernetes 系统通过执行修正操作 来响应规约和状态间的不一致 —— 在这里意味着它会启动一个新的实例来替换。

关于对象 spec、status 和 metadata 的更多信息，可参阅 [Kubernetes API 约定](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)。

描述 Kubernetes 对象
----------------

创建 Kubernetes 对象时，必须提供对象的规约，用来描述该对象的期望状态， 以及关于对象的一些基本信息（例如名称）。 当使用 Kubernetes API 创建对象时（或者直接创建，或者基于​`kubectl`​）， API 请求必须在请求体中包含 JSON 格式的信息。 大多数情况下，需要在 ​`.yaml`​ 文件中为 ​`kubectl` ​提供这些信息。 ​`kubectl` ​在发起 API 请求时，将这些信息转换成 JSON 格式。

这里有一个 ​`.yaml`​ 示例文件，展示了 Kubernetes Deployment 的必需字段和对象规约：

`apiVersion: apps/v1 kind: Deployment metadata:   name: nginx-deployment spec:   selector:     matchLabels:       app: nginx   replicas: 2 # tells deployment to run 2 pods matching the template   template:     metadata:       labels:         app: nginx     spec:       containers:       - name: nginx         image: nginx:1.14.2         ports:         - containerPort: 80`

使用类似于上面的 ​`.yaml`​ 文件来创建 Deployment的一种方式是使用 ​`kubectl` ​命令行接口（CLI）中的 [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands target=) 命令， 将 ​`.yaml`​ 文件作为参数。下面是一个示例：

`kubectl apply -f https://k8s.io/examples/application/deployment.yaml`

输出类似如下这样：

`deployment.apps/nginx-deployment created`

必需字段 
-----

在想要创建的 Kubernetes 对象对应的 ​`.yaml`​ 文件中，需要配置如下的字段：

*   ​`apiVersion` ​- 创建该对象所使用的 Kubernetes API 的版本
*   ​`kind` ​- 想要创建的对象的类别
*   ​`metadata` ​- 帮助唯一性标识对象的一些数据，包括一个 name 字符串、UID 和可选的 namespace
*   ​`spec` ​- 你所期望的该对象的状态

对象 ​`spec` ​的精确格式对每个 Kubernetes 对象来说是不同的，包含了特定于该对象的嵌套字段。

例如，参阅 Pod API 参考文档中 ​`spec`​ 字段。 对于每个 Pod，其 ​`.spec`​ 字段设置了 Pod 及其期望状态（例如 Pod 中每个容器的容器镜像名称）。 另一个对象规约的例子是 StatefulSet API 中的 ​`spec` ​字段。 对于 StatefulSet 而言，其 ​`.spec`​ 字段设置了 StatefulSet 及其期望状态。 在 StatefulSet 的 ​`.spec`​ 内，有一个为 Pod 对象提供的模板。该模板描述了 StatefulSet 控制器为了满足 StatefulSet 规约而要创建的 Pod。 不同类型的对象可以由不同的 ​`.status`​ 信息。API 参考页面给出了 ​`.status`​ 字段的详细结构， 以及针对不同类型 API 对象的具体内容。

##  2.  Kubernetes 对象管理
对象管理
----

​`kubectl` ​命令行工具支持多种不同的方式来创建和管理 Kubernetes 对象。 本文档概述了不同的方法。 阅读 [Kubectl book](https://kubectl.docs.kubernetes.io/) 来了解 kubectl 管理对象的详细信息。

管理技巧
----

> 应该只使用一种技术来管理 Kubernetes 对象。混合和匹配技术作用在同一对象上将导致未定义行为。

管理技术

作用于

建议的环境

支持的写者

学习难度

指令式命令

活跃对象

开发项目

1+

最低

指令式对象配置

单个文件

生产项目

1

中等

声明式对象配置

文件目录

生产项目

1+

最高

指令式命令
-----

使用指令式命令时，用户可以在集群中的活动对象上进行操作。用户将操作传给 ​`kubectl` ​命令作为参数或标志。

这是开始或者在集群中运行一次性任务的推荐方法。因为这个技术直接在活跃对象 上操作，所以它不提供以前配置的历史记录。

### 例子

通过创建 Deployment 对象来运行 nginx 容器的实例：

`kubectl create deployment nginx --image nginx`

### 权衡 

与对象配置相比的优点：

*   命令简单，易学且易于记忆。
*   命令仅需一步即可对集群进行更改。

与对象配置相比的缺点：

*   命令不与变更审查流程集成。
*   命令不提供与更改关联的审核跟踪。
*   除了实时内容外，命令不提供记录源。
*   命令不提供用于创建新对象的模板。

指令式对象配置
-------

在指令式对象配置中，kubectl 命令指定操作（创建，替换等），可选标志和 至少一个文件名。指定的文件必须包含 YAML 或 JSON 格式的对象的完整定义。

有关对象定义的详细信息，请查看 [API 参考](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.23/)。

> **Warning**:
> 
> ​`replace` ​指令式命令将现有规范替换为新提供的规范，并放弃对配置文件中 缺少的对象的所有更改。此方法不应与对象规约被独立于配置文件进行更新的 资源类型一起使用。比如类型为 ​`LoadBalancer` ​的服务，它的 ​`externalIPs` ​字段就是独立于集群配置进行更新。

### 例子

创建配置文件中定义的对象：

`kubectl create -f nginx.yaml`

删除两个配置文件中定义的对象：

`kubectl delete -f nginx.yaml -f redis.yaml`

通过覆盖活动配置来更新配置文件中定义的对象：

`kubectl replace -f nginx.yaml`

### 权衡

与指令式命令相比的优点：

*   对象配置可以存储在源控制系统中，比如 Git。
*   对象配置可以与流程集成，例如在推送和审计之前检查更新。
*   对象配置提供了用于创建新对象的模板。

与指令式命令相比的缺点：

*   对象配置需要对对象架构有基本的了解。
*   对象配置需要额外的步骤来编写 YAML 文件。

与声明式对象配置相比的优点：

*   指令式对象配置行为更加简单易懂。
*   从 Kubernetes 1.5 版本开始，指令对象配置更加成熟。

与声明式对象配置相比的缺点：

*   指令式对象配置更适合文件，而非目录。
*   对活动对象的更新必须反映在配置文件中，否则会在下一次替换时丢失。

声明式对象配置 
--------

使用声明式对象配置时，用户对本地存储的对象配置文件进行操作，但是用户 未定义要对该文件执行的操作。 ​`kubectl` ​会自动检测每个文件的创建、更新和删除操作。 这使得配置可以在目录上工作，根据目录中配置文件对不同的对象执行不同的操作。

> 声明式对象配置保留其他编写者所做的修改，即使这些更改并未合并到对象配置文件中。 可以通过使用 ​`patch` ​API 操作仅写入观察到的差异，而不是使用 ​`replace` ​API 操作来替换整个对象配置来实现。

### 例子

处理 ​`configs` ​目录中的所有对象配置文件，创建并更新活跃对象。 可以首先使用 ​`diff` ​子命令查看将要进行的更改，然后在进行应用：

`kubectl diff -f configs/ kubectl apply -f configs/`

递归处理目录：

`kubectl diff -R -f configs/ kubectl apply -R -f configs/`

### 权衡

与指令式对象配置相比的优点：

*   对活动对象所做的更改即使未合并到配置文件中，也会被保留下来。
*   声明性对象配置更好地支持对目录进行操作并自动检测每个文件的操作类型（创建，修补，删除）。

与指令式对象配置相比的缺点：

*   声明式对象配置难于调试并且出现异常时结果难以理解。
*   使用 diff 产生的部分更新会创建复杂的合并和补丁操作。

##  3.  Kubernetes 对象名称和IDs
对象名称和 IDs
---------

集群中的每一个对象都有一个名称 来标识在同类资源中的唯一性。

每个 Kubernetes 对象也有一个UID 来标识在整个集群中的唯一性。

比如，在同一个名字空间 中有一个名为 ​`myapp-1234`​ 的 Pod, 但是可以命名一个 Pod 和一个 Deployment 同为 ​`myapp-1234`​.

对于用户提供的非唯一性的属性，Kubernetes 提供了 标签（Labels）和 注解（Annotation）机制。

名称 
---

客户端提供的字符串，引用资源 url 中的对象，如​`/api/v1/pods/some name`​。

某一时刻，只能有一个给定类型的对象具有给定的名称。但是，如果删除该对象，则可以创建同名的新对象。

> 当对象所代表的是一个物理实体（例如代表一台物理主机的 Node）时， 如果在 Node 对象未被删除并重建的条件下，重新创建了同名的物理主机， 则 Kubernetes 会将新的主机看作是老的主机，这可能会带来某种不一致性。

以下是比较常见的四种资源命名约束。

### DNS 子域名 

很多资源类型需要可以用作 DNS 子域名的名称。 DNS 子域名的定义可参见 [RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123)。 这一要求意味着名称必须满足如下规则：

*   不能超过253个字符
*   只能包含小写字母、数字，以及'-' 和 '.'
*   须以字母数字开头
*   须以字母数字结尾

### RFC 1123 标签名 

某些资源类型需要其名称遵循 [RFC 1123](https://datatracker.ietf.org/doc/html/rfc1123) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

*   最多 63 个字符
*   只能包含小写字母、数字，以及 '-'
*   须以字母数字开头
*   须以字母数字结尾

### RFC 1035 标签名 

某些资源类型需要其名称遵循 [RFC 1035](https://tools.ietf.org/html/rfc1035) 所定义的 DNS 标签标准。也就是命名必须满足如下规则：

*   最多 63 个字符
*   只能包含小写字母、数字，以及 '-'
*   须以字母开头
*   须以字母数字结尾

### 路径分段名称 

某些资源类型要求名称能被安全地用作路径中的片段。 换句话说，其名称不能是 ​`.`​、​`..`​，也不可以包含 ​`/`​ 或 ​`%`​ 这些字符。

下面是一个名为​`nginx-demo`​的 Pod 的配置清单：

`apiVersion: v1 kind: Pod metadata:   name: nginx-demo spec:   containers:   - name: nginx     image: nginx:1.14.2     ports:     - containerPort: 80`

> 某些资源类型可能具有额外的命名约束。

UIDs
----

Kubernetes 系统生成的字符串，唯一标识对象。

在 Kubernetes 集群的整个生命周期中创建的每个对象都有一个不同的 uid，它旨在区分类似实体的历史事件。

Kubernetes UIDs 是全局唯一标识符（也叫 UUIDs）。 UUIDs 是标准化的，见 ISO/IEC 9834-8 和 ITU-T X.667.

##  4.  Kubernetes 名字空间
名字空间
----

在 Kubernetes 中，“名字空间（Namespace）”提供一种机制，将同一集群中的资源划分为相互隔离的组。 同一名字空间内的资源名称要唯一，但跨名字空间时没有这个要求。 名字空间作用域仅针对带有名字空间的对象，例如 Deployment、Service 等， 这种作用域对集群访问的对象不适用，例如 StorageClass、Node、PersistentVolume 等。

何时使用多个名字空间
----------

名字空间适用于存在很多跨多个团队或项目的用户的场景。对于只有几到几十个用户的集群，根本不需要创建或考虑名字空间。当需要名称空间提供的功能时，请开始使用它们。

名字空间为名称提供了一个范围。资源的名称需要在名字空间内是唯一的，但不能跨名字空间。 名字空间不能相互嵌套，每个 Kubernetes 资源只能在一个名字空间中。

名字空间是在多个用户之间划分集群资源的一种方法（通过资源配额）。

不必使用多个名字空间来分隔仅仅轻微不同的资源，例如同一软件的不同版本： 应该使用标签 来区分同一名字空间中的不同资源。

使用名字空间
------

### 创建名字空间

> 避免使用前缀 ​`kube-`​ 创建名字空间，因为它是为 Kubernetes 系统名字空间保留的。

新建一个名为 my-namespace.yaml 的 YAML 文件，并写入下列内容：

`apiVersion: v1 kind: Namespace metadata:   name: <insert-namespace-name-here>`

然后运行：

`kubectl create -f ./my-namespace.yaml`

2、或者，你可以使用下面的命令创建名字空间：

`kubectl create namespace <insert-namespace-name-here>`

请注意，名字空间的名称必须是一个合法的 DNS 标签。

可选字段 finalizers 允许观察者们在名字空间被删除时清除资源。记住如果指定了一个不存在的终结器，名字空间仍会被创建，但如果用户试图删除它，它将陷入 Terminating 状态。

更多有关 finalizers 的信息请查阅 [设计文档](https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/namespaces.md target=) 中名字空间部分。

### 删除名字空间 

删除名字空间使用命令：

`kubectl delete namespaces <insert-some-namespace-name>`

> **Warning**: 这会删除名字空间下的 所有内容 ！

删除是异步的，所以有一段时间你会看到名字空间处于 ​`Terminating` ​状态。

查看名字空间  

你可以使用以下命令列出集群中现存的名字空间：

`kubectl get namespace`

Kubernetes 会创建四个初始名字空间：

*   ​`default` ​没有指明使用其它名字空间的对象所使用的默认名字空间
*   ​`kube-system`​ Kubernetes 系统创建对象所使用的名字空间
*   ​`kube-public`​ 这个名字空间是自动创建的，所有用户（包括未经过身份验证的用户）都可以读取它。 这个名字空间主要用于集群使用，以防某些资源在整个集群中应该是可见和可读的。 这个名字空间的公共方面只是一种约定，而不是要求。
*   ​`kube-node-lease`​ 此名字空间用于与各个节点相关的 租约（Lease）对象。 节点租期允许 kubelet 发送心跳，由此控制面能够检测到节点故障。

### 为请求设置名字空间 

要为当前请求设置名字空间，请使用 ​`--namespace`​ 参数。

例如：

`kubectl run nginx --image=nginx --namespace=<名字空间名称> kubectl get pods --namespace=<名字空间名称>`

### 设置名字空间偏好 

你可以永久保存名字空间，以用于对应上下文中所有后续 kubectl 命令。

`kubectl config set-context --current --namespace=<名字空间名称> # 验证之 kubectl config view | grep namespace:`

名字空间和 DNS 
----------

当你创建一个服务 时， Kubernetes 会创建一个相应的 DNS 条目。

该条目的形式是 ​`<服务名称>.<名字空间名称>.svc.cluster.local`​，这意味着如果容器只使用 ​`<服务名称>`​，它将被解析到本地名字空间的服务。这对于跨多个名字空间（如开发、分级和生产） 使用相同的配置非常有用。如果你希望跨名字空间访问，则需要使用完全限定域名（FQDN）。

因此，所有的名字空间名称都必须是合法的 RFC 1123 DNS 标签。

> **Warning**:  
> 通过创建与[公共顶级域名](https://data.iana.org/TLD/tlds-alpha-by-domain.txt) 同名的名字空间，这些名字空间中的服务可以拥有与公共 DNS 记录重叠的、较短的 DNS 名称。 所有名字空间中的负载在执行 DNS 查找时，如果查找的名称没有 [尾部句点](https://datatracker.ietf.org/doc/html/rfc1034 target=)， 就会被重定向到这些服务上，因此呈现出比公共 DNS 更高的优先序。  
> 为了缓解这类问题，需要将创建名字空间的权限授予可信的用户。 如果需要，你可以额外部署第三方的安全控制机制，例如以 准入 Webhook 的形式，阻止用户创建与公共 [TLD](https://data.iana.org/TLD/tlds-alpha-by-domain.txt) 同名的名字空间。

并非所有对象都在名字空间中
-------------

大多数 kubernetes 资源（例如 Pod、Service、副本控制器等）都位于某些名字空间中。 但是名字空间资源本身并不在名字空间中。而且底层资源，例如 节点 和持久化卷不属于任何名字空间。

查看哪些 Kubernetes 资源在名字空间中，哪些不在名字空间中：

`# 位于名字空间中的资源 kubectl api-resources --namespaced=true  # 不在名字空间中的资源 kubectl api-resources --namespaced=false`

自动打标签 
------

FEATURE STATE: Kubernetes 1.21 \[beta\]

Kubernetes 控制面会为所有名字空间设置一个不可变更的 标签 ​`kubernetes.io/metadata.name`​，只要 ​`NamespaceDefaultLabelName`​ 这一 特性门控 被启用。标签的值是名字空间的名称。

##  5.  Kubernetes 标签和选择算符
标签和选择算符
-------

标签（Labels） 是附加到 Kubernetes 对象（比如 Pods）上的键值对。 标签旨在用于指定对用户有意义且相关的对象的标识属性，但不直接对核心系统有语义含义。 标签可以用于组织和选择对象的子集。标签可以在创建时附加到对象，随后可以随时添加和修改。 每个对象都可以定义一组键/值标签。每个键对于给定对象必须是唯一的。

`"metadata": {   "labels": {     "key1" : "value1",     "key2" : "value2"   } }`

标签能够支持高效的查询和监听操作，对于用户界面和命令行是很理想的。 应使用注解 记录非识别信息。

动机
--

标签使用户能够以松散耦合的方式将他们自己的组织结构映射到系统对象，而无需客户端存储这些映射。

服务部署和批处理流水线通常是多维实体（例如，多个分区或部署、多个发行序列、多个层，每层多个微服务）。 管理通常需要交叉操作，这打破了严格的层次表示的封装，特别是由基础设施而不是用户确定的严格的层次结构。

示例标签：

*   ​`"release" : "stable"`​, ​`"release" : "canary"` ​
*   ​`"environment" : "dev"`​, ​`"environment" : "qa"`​, ​`"environment" : "production"` ​
*   ​`"tier" : "frontend"`​, ​`"tier" : "backend"`​, ​`"tier" : "cache"` ​
*   ​`"partition" : "customerA"`​, ​`"partition" : "customerB"` ​
*   ​`"track" : "daily"`​, ​`"track" : "weekly"`​

有一些常用标签的例子; 你可以任意制定自己的约定。 请记住，标签的 Key 对于给定对象必须是唯一的。

语法和字符集 
-------

标签 是键值对。有效的标签键有两个段：可选的前缀和名称，用斜杠（​`/`​）分隔。 名称段是必需的，必须小于等于 63 个字符，以字母数字字符（​`[a-z0-9A-Z]`​）开头和结尾， 带有破折号（​`-`​），下划线（​`_`​），点（ ​`.`​）和之间的字母数字。 前缀是可选的。如果指定，前缀必须是 DNS 子域：由点（​`.`​）分隔的一系列 DNS 标签，总共不超过 253 个字符， 后跟斜杠（​`/`​）。

如果省略前缀，则假定标签键对用户是私有的。 向最终用户对象添加标签的自动系统组件（例如 ​`kube-scheduler`​、​`kube-controller-manager`​、 ​`kube-apiserver`​、​`kubectl`​ 或其他第三方自动化工具）必须指定前缀。

​`kubernetes.io/`​ 和 ​`k8s.io/`​ 前缀是为 Kubernetes 核心组件保留的。

有效标签值：

*   必须为 63 个字符或更少（可以为空）
*   除非标签值为空，必须以字母数字字符（​`[a-z0-9A-Z]`​）开头和结尾
*   包含破折号（​`-`​）、下划线（​`_`​）、点（​`.`​）和字母或数字。

标签选择算符 
-------

与名称和 UID 不同， 标签不支持唯一性。通常，我们希望许多对象携带相同的标签。

通过 标签选择算符，客户端/用户可以识别一组对象。标签选择算符是 Kubernetes 中的核心分组原语。

API 目前支持两种类型的选择算符：基于等值的 和 基于集合的。 标签选择算符可以由逗号分隔的多个 需求 组成。 在多个需求的情况下，必须满足所有要求，因此逗号分隔符充当逻辑 与（​`&&`​）运算符。

空标签选择算符或者未指定的选择算符的语义取决于上下文， 支持使用选择算符的 API 类别应该将算符的合法性和含义用文档记录下来。

> 对于某些 API 类别（例如 ReplicaSet）而言，两个实例的标签选择算符不得在命名空间内重叠， 否则它们的控制器将互相冲突，无法确定应该存在的副本个数。

> 对于基于等值的和基于集合的条件而言，不存在逻辑或（||）操作符。 你要确保你的过滤语句按合适的方式组织。

### 基于等值的需求

基于等值 或 基于不等值 的需求允许按标签键和值进行过滤。 匹配对象必须满足所有指定的标签约束，尽管它们也可能具有其他标签。 可接受的运算符有​`=`​、​`==`​ 和 ​`!=`​ 三种。 前两个表示 相等（并且只是同义词），而后者表示 不相等。例如：

`environment = production tier != frontend`

前者选择所有资源，其键名等于 ​`environment`​，值等于 ​`production`​。 后者选择所有资源，其键名等于 ​`tier`​，值不同于 ​`frontend`​，所有资源都没有带有 ​`tier` ​键的标签。 可以使用逗号运算符来过滤 ​`production` ​环境中的非 ​`frontend` ​层资源：​`environment=production,tier!=frontend`​。

基于等值的标签要求的一种使用场景是 Pod 要指定节点选择标准。 例如，下面的示例 Pod 选择带有标签 "​`accelerator=nvidia-tesla-p100`​"。

`apiVersion: v1 kind: Pod metadata:   name: cuda-test spec:   containers:     - name: cuda-test       image: "k8s.gcr.io/cuda-vector-add:v0.1"       resources:         limits:           nvidia.com/gpu: 1   nodeSelector:     accelerator: nvidia-tesla-p100`

### 基于集合的需求

基于集合 的标签需求允许你通过一组值来过滤键。 支持三种操作符：​`in`​、​`notin` ​和 ​`exists` ​(只可以用在键标识符上)。例如：

`environment in (production, qa) tier notin (frontend, backend) partition !partition`

*   第一个示例选择了所有键等于 ​`environment` ​并且值等于 ​`production` ​或者 ​`qa` ​的资源。
*   第二个示例选择了所有键等于 ​`tier` ​并且值不等于 ​`frontend` ​或者 ​`backend` ​的资源，以及所有没有 ​`tier` ​键标签的资源。
*   第三个示例选择了所有包含了有 ​`partition` ​标签的资源；没有校验它的值。
*   第四个示例选择了所有没有 ​`partition` ​标签的资源；没有校验它的值。

类似地，逗号分隔符充当 与 运算符。因此，使用 ​`partition` ​键（无论为何值）和 ​`environment` ​不同于 ​`qa`​ 来过滤资源可以使用 ​`partition, environment notin（qa)`​ 来实现。

基于集合 的标签选择算符是相等标签选择算符的一般形式，因为 ​`environment=production`​ 等同于 ​`environment in（production）`​；​`!=`​ 和 ​`notin` ​也是类似的。

基于集合 的要求可以与基于 相等 的要求混合使用。例如：​`partition in (customerA, customerB),environment!=qa`​。

API
---

### LIST 和 WATCH 过滤

LIST 和 WATCH 操作可以使用查询参数指定标签选择算符过滤一组对象。 两种需求都是允许的。（这里显示的是它们出现在 URL 查询字符串中）

*   基于等值 的需求: ​`?labelSelector=environment%3Dproduction,tier%3Dfrontend` ​
*   基于集合 的需求: ​`?labelSelector=environment+in+%28production%2Cqa%29%2Ctier+in+%28frontend%29`​

两种标签选择算符都可以通过 REST 客户端用于 list 或者 watch 资源。 例如，使用 ​`kubectl` ​定位 ​`apiserver`​，可以使用 基于等值 的标签选择算符可以这么写：

`kubectl get pods -l environment=production,tier=frontend`

或者使用 基于集合的 需求：

`kubectl get pods -l 'environment in (production),tier in (frontend)'`

正如刚才提到的，基于集合 的需求更具有表达力。例如，它们可以实现值的 或 操作：

`kubectl get pods -l 'environment in (production, qa)'`

或者通过 exists 运算符限制不匹配：

`kubectl get pods -l 'environment,environment notin (frontend)'`

### 在 API 对象中设置引用

一些 Kubernetes 对象，例如 ​`services` ​和 ​`replicationcontrollers` ​， 也使用了标签选择算符去指定了其他资源的集合，例如 pods。

#### Service 和 ReplicationController

一个 ​`Service` ​指向的一组 Pods 是由标签选择算符定义的。同样，一个 ​`ReplicationController` ​应该管理的 pods 的数量也是由标签选择算符定义的。

两个对象的标签选择算符都是在 ​`json` ​或者 ​`yaml` ​文件中使用映射定义的，并且只支持 基于等值 需求的选择算符：

`"selector": {     "component" : "redis", }`

或者

`selector:     component: redis`

这个选择算符(分别在 ​`json` ​或者 ​`yaml` ​格式中) 等价于 ​`component=redis`​ 或 ​`component in (redis)`​ 。

#### 支持基于集合需求的资源

比较新的资源，例如 ​`Job`​、 ​`Deployment`​、 ​`Replica Set`​ 和 ​`DaemonSet` ​， 也支持 基于集合的 需求。

`selector:   matchLabels:     component: redis   matchExpressions:     - {key: tier, operator: In, values: [cache]}     - {key: environment, operator: NotIn, values: [dev]}`

​`matchLabels`​ 是由 ​`{key,value}`​ 对组成的映射。 ​`matchLabels`​ 映射中的单个 ​`{key,value }`​ 等同于 ​`matchExpressions` ​的元素， 其 ​`key` ​字段为 "key"，​`operator` ​为 "In"，而 ​`values` ​数组仅包含 "value"。 ​`matchExpressions` ​是 Pod 选择算符需求的列表。 有效的运算符包括 ​`In`​、​`NotIn`​、​`Exists` ​和 ​`DoesNotExist`​。 在 ​`In` ​和 ​`NotIn` ​的情况下，设置的值必须是非空的。 来自 ​`matchLabels` ​和 ​`matchExpressions` ​的所有要求都按逻辑与的关系组合到一起 -- 它们必须都满足才能匹配。

#### 选择节点集 

通过标签进行选择的一个用例是确定节点集，方便 Pod 调度。

##  6.  Kubernetes 注解
注解
--

你可以使用 Kubernetes 注解为对象附加任意的非标识的元数据。客户端程序（例如工具和库）能够获取这些元数据信息。

为对象附加元数据
--------

你可以使用标签或注解将元数据附加到 Kubernetes 对象。 标签可以用来选择对象和查找满足某些条件的对象集合。 相反，注解不用于标识和选择对象。 注解中的元数据，可以很小，也可以很大，可以是结构化的，也可以是非结构化的，能够包含标签不允许的字符。

注解和标签一样，是键/值对:

`"metadata": {   "annotations": {     "key1" : "value1",     "key2" : "value2"   } }`

> Map 中的键和值必须是字符串。 换句话说，你不能使用数字、布尔值、列表或其他类型的键或值。

以下是一些例子，用来说明哪些信息可以使用注解来记录:

*   由声明性配置所管理的字段。 将这些字段附加为注解，能够将它们与客户端或服务端设置的默认值、 自动生成的字段以及通过自动调整大小或自动伸缩系统设置的字段区分开来。
*   构建、发布或镜像信息（如时间戳、发布 ID、Git 分支、PR 数量、镜像哈希、仓库地址）。
*   指向日志记录、监控、分析或审计仓库的指针。
*   可用于调试目的的客户端库或工具信息：例如，名称、版本和构建信息。
*   用户或者工具/系统的来源信息，例如来自其他生态系统组件的相关对象的 URL。
*   轻量级上线工具的元数据信息：例如，配置或检查点。
*   负责人员的电话或呼机号码，或指定在何处可以找到该信息的目录条目，如团队网站。
*   从用户到最终运行的指令，以修改行为或使用非标准功能。

你可以将这类信息存储在外部数据库或目录中而不使用注解， 但这样做就使得开发人员很难生成用于部署、管理、自检的客户端共享库和工具。

语法和字符集
------

注解（Annotations） 存储的形式是键/值对。有效的注解键分为两部分： 可选的前缀和名称，以斜杠（​`/`​）分隔。 名称段是必需项，并且必须在63个字符以内，以字母数字字符（​`[a-z0-9A-Z]`​）开头和结尾， 并允许使用破折号（​`-`​），下划线（​`_`​），点（​`.`​）和字母数字。 前缀是可选的。如果指定，则前缀必须是DNS子域：一系列由点（​`.`​）分隔的DNS标签， 总计不超过253个字符，后跟斜杠（​`/`​）。 如果省略前缀，则假定注解键对用户是私有的。 由系统组件添加的注解 （例如，​`kube-scheduler`​，​`kube-controller-manager`​，​`kube-apiserver`​，​`kubectl`​ 或其他第三方组件），必须为终端用户添加注解前缀。

​`kubernetes.io/`​ 和 ​`k8s.io/`​ 前缀是为Kubernetes核心组件保留的。

例如，下面是一个 Pod 的配置文件，其注解中包含 ​`imageregistry: https://hub.docker.com/`​：

`apiVersion: v1 kind: Pod metadata:   name: annotations-demo   annotations:     imageregistry: "https://hub.docker.com/" spec:   containers:   - name: nginx     image: nginx:1.7.9     ports:     - containerPort: 80`

##  7.  Kubernetes Finalizers
Finalizers
----------

Finalizer 是带有命名空间的键，告诉 Kubernetes 等到特定的条件被满足后， 再完全删除被标记为删除的资源。 Finalizer 提醒控制器清理被删除的对象拥有的资源。

当你告诉 Kubernetes 删除一个指定了 Finalizer 的对象时， Kubernetes API 通过填充 ​`.metadata.deletionTimestamp`​ 来标记要删除的对象， 并返回​`202`​状态码 (HTTP "已接受") 使其进入只读状态。 此时控制平面或其他组件会采取 Finalizer 所定义的行动， 而目标对象仍然处于终止中（Terminating）的状态。 这些行动完成后，控制器会删除目标对象相关的 Finalizer。 当 ​`metadata.finalizers`​ 字段为空时，Kubernetes 认为删除已完成。

你可以使用 Finalizer 控制资源的垃圾收集。 例如，你可以定义一个 Finalizer，在删除目标资源前清理相关资源或基础设施。

你可以通过使用 Finalizers 提醒控制器 在删除目标资源前执行特定的清理任务， 来控制资源的垃圾收集。

Finalizers 通常不指定要执行的代码。 相反，它们通常是特定资源上的键的列表，类似于注解。 Kubernetes 自动指定了一些 Finalizers，但你也可以指定你自己的。

Finalizers 如何工作 
----------------

当你使用清单文件创建资源时，你可以在 ​`metadata.finalizers`​ 字段指定 Finalizers。 当你试图删除该资源时，处理删除请求的 API 服务器会注意到 ​`finalizers`​ 字段中的值， 并进行以下操作：

*   修改对象，将你开始执行删除的时间添加到 ​`metadata.deletionTimestamp`​ 字段。
*   禁止对象被删除，直到其 ​`metadata.finalizers`​ 字段为空。
*   返回 ​`202`​ 状态码（HTTP "Accepted"）。

管理 finalizer 的控制器注意到对象上发生的更新操作，对象的 ​`metadata.deletionTimestamp`​ 被设置，意味着已经请求删除该对象。然后，控制器会试图满足资源的 Finalizers 的条件。 每当一个 Finalizer 的条件被满足时，控制器就会从资源的 ​`finalizers` ​字段中删除该键。 当 ​`finalizers` ​字段为空时，​`deletionTimestamp` ​字段被设置的对象会被自动删除。 你也可以使用 Finalizers 来阻止删除未被管理的资源。

一个常见的 Finalizer 的例子是 ​`kubernetes.io/pv-protection`​， 它用来防止意外删除 ​`PersistentVolume` ​对象。 当一个 ​`PersistentVolume` ​对象被 Pod 使用时， Kubernetes 会添加 ​`pv-protection`​ Finalizer。 如果你试图删除 ​`PersistentVolume`​，它将进入 ​`Terminating` ​状态， 但是控制器因为该 Finalizer 存在而无法删除该资源。 当 Pod 停止使用 ​`PersistentVolume` ​时， Kubernetes 清除 ​`pv-protection`​ Finalizer，控制器就会删除该卷。

属主引用、标签和 Finalizers
-------------------

与标签类似， 属主引用 描述了 Kubernetes 中对象之间的关系，但它们作用不同。 当一个控制器 管理类似于 Pod 的对象时，它使用标签来跟踪相关对象组的变化。 例如，当 Job 创建一个或多个 Pod 时， Job 控制器会给这些 Pod 应用上标签，并跟踪集群中的具有相同标签的 Pod 的变化。

Job 控制器还为这些 Pod 添加了属主引用，指向创建 Pod 的 Job。 如果你在这些 Pod 运行的时候删除了 Job， Kubernetes 会使用属主引用（而不是标签）来确定集群中哪些 Pod 需要清理。

当 Kubernetes 识别到要删除的资源上的属主引用时，它也会处理 Finalizers。

在某些情况下，Finalizers 会阻止依赖对象的删除， 这可能导致目标属主对象被保留的时间比预期的长，而没有被完全删除。 在这些情况下，你应该检查目标属主和附属对象上的 Finalizers 和属主引用，来排查原因。

> 在对象卡在删除状态的情况下，要避免手动移除 Finalizers，以允许继续删除操作。 Finalizers 通常因为特殊原因被添加到资源上，所以强行删除它们会导致集群出现问题。 只有了解 finalizer 的用途时才能这样做，并且应该通过一些其他方式来完成 （例如，手动清除其余的依赖对象）。

##  8.  Kubernetes 字段选择器

##  9.  Kubernetes 属主与附属

##  10.  Kubernetes 推荐使用的标签

#  5.  Kubernetes 架构

##  1.  Kubernetes 节点

##  2.  Kubernetes 控制面到节点通信

##  3.  Kubernetes 控制器

##  4.  Kubernetes 云控制器管理器

##  5.  Kubernetes 垃圾收集

##  6.  Kubernetes 容器运行时接口（CRI）

#  6.  Kubernetes 容器

##  1.  Kubernetes 镜像

##  2.  Kubernetes 容器环境

##  3.  Kubernetes 容器运行时类（Runtime Class）

##  4.  Kubernetes 容器生命周期回调

#  7.  Kubernetes Pods

##  1.  Kubernetes Pod的生命周期

##  2.  Kubernetes Init容器

##  3.  Kubernetes Pod拓扑分布约束

##  4.  Kubernetes 干扰（Disruptions）

##  5.  Kubernetes 临时容器

#  8.  Kubernetes 工作负载资源

##  1.  Kubernetes Deployments

##  2.  Kubernetes ReplicaSet

##  3.  Kubernetes StatefulSets

##  4.  Kubernetes DaemonSet

##  5.  Kubernetes Jobs

##  6.  Kubernetes 已完成 Job 的自动清理

##  7.  Kubernetes CronJob

##  8.  Kubernetes ReplicationController

#  9.  Kubernetes 服务、负载均衡和联网

##  1.  Kubernetes 使用拓扑键实现拓扑感知的流量路由

##  2.  Kubernetes 服务

##  3.  Kubernetes Pod 与 Service 的 DNS

##  4.  Kubernetes 使用 Service 连接到应用

##  5.  Kubernetes Ingress

##  6.  Kubernetes Ingress 控制器

##  7.  Kubernetes 拓扑感知提示

##  8.  Kubernetes 服务内部流量策略

##  9.  Kubernetes 端点切片（Endpoint Slices）

##  10.  Kubernetes 网络策略

##  11.  Kubernetes IPv4/IPv6 双协议栈

#  10.  Kubernetes 存储

##  1.  Kubernetes 卷

##  2.  Kubernetes 持久卷

##  3.  Kubernetes 投射卷

##  4.  Kubernetes 临时卷

##  5.  Kubernetes 存储类

#  11.  Kubernetes 配置

##  1.  Kubernetes 配置最佳实践

##  2.  Kubernetes ConfigMap

##  3.  Kubernetes Secret

##  4.  Kubernetes 为 Pod 和容器管理资源

##  5.  Kubernetes 使用 kubeconfig 文件组织集群访问

##  6.  Kubernetes Windows 节点的资源管理

#  12.  Kubernetes 安全

##  1.  Kubernetes 云原生安全概述

##  2.  Kubernetes Pod安全性标准

##  3.  Kubernetes Pod安全性准入

##  4.  Kubernetes Pod安全策略

##  5.  Kubernetes Windows节点的安全性

##  6.  Kubernetes API访问控制

##  7.  Kubernetes 基于角色的访问控制良好实践

#  13.  Kubernetes 策略

##  1.  Kubernetes 限制范围

##  2.  Kubernetes 资源配额

##  3.  Kubernetes 进程ID约束与预留

##  4.  Kubernetes 节点资源管理器

#  14.  Kubernetes 调度，抢占和驱逐

##  1.  Kubernetes 调度器

##  2.  Kubernetes 将Pod指派给节点

##  3.  Kubernetes Pod开销

##  4.  Kubernetes 污点和容忍度

##  5.  Kubernetes Pod优先级和抢占

##  6.  Kubernetes 节点压力驱逐

##  7.  Kubernetes API发起的驱逐

##  8.  Kubernetes 扩展资源的资源装箱

##  9.  Kubernetes 调度框架

##  10.  Kubernetes 调度器性能调优

#  15.  Kubernetes 集群管理

##  1.  Kubernetes 管理资源

##  2.  Kubernetes 集群网络系统

##  3.  Kubernetes 系统组件指标

##  4.  Kubernetes 日志架构

##  5.  Kubernetes 系统日志

##  6.  Kubernetes 追踪系统组件

##  7.  Kubernetes 代理

##  8.  Kubernetes API优先级和公平性

##  9.  Kubernetes 安装扩展（Addons）

#  16.  Kubernetes 扩展

##  1.  Kubernetes 扩展API

###  1.  Kubernetes 定制资源

###  2.  Kubernetes 通过聚合层扩展API

##  2.  Kubernetes Operator模式

##  3.  Kubernetes 计算、存储和网络扩展

###  1.  Kubernetes 网络插件

###  2.  Kubernetes 设备插件

##  4.  Kubernetes 服务目录

#  17.  Kubernetes 应用故障排除

##  1.  Kubernetes 调试Pod

##  2.  Kubernetes 调试Service

##  3.  Kubernetes 调试StatefulSet

##  4.  Kubernetes 调试Init容器

##  5.  Kubernetes 确定Pod失败的原因

##  6.  Kubernetes 获取正在运行容器的Shell

##  7.  Kubernetes 调试运行中的Pod

#  18.  Kubernetes 集群故障排查

##  1.  Kubernetes 资源指标管道

##  2.  Kubernetes 节点健康监测

##  3.  Kubernetes 使用crictl对Kubernetes节点进行调试

##  4.  Kubernetes Windows调试提示

##  5.  Kubernetes 使用telepresence在本地开发和调试服务

##  6.  Kubernetes 审计

##  7.  Kubernetes 资源监控工具

#  19.  Kubernetes 管理集群

##  1.  Kubernetes 从dockershim迁移

###  1.  Kubernetes 将节点上的容器运行时从Docker Engine改为containerd

###  2.  Kubernetes 将Docker Engine节点从dockershim迁移到cri-dockerd

###  3.  Kubernetes CNI插件相关错误故障排除

###  4.  Kubernetes 查明节点上所使用的容器运行时

###  5.  Kubernetes 检查弃用Dockershim是否对你有影响

###  6.  Kubernetes 从dockershim迁移遥测和安全代理

##  2.  Kubernetes 用kubeadm进行管理

###  1.  Kubernetes 使用kubeadm进行证书管理

###  2.  Kubernetes 配置cgroup驱动

###  3.  Kubernetes 重新配置kubeadm集群

###  4.  Kubernetes 升级kubeadm集群

###  5.  Kubernetes 添加Windows节点

###  6.  Kubernetes 升级Windows节点

##  3.  Kubernetes 手动生成证书

##  4.  Kubernetes 管理内存，CPU和API资源

###  1.  Kubernetes 为命名空间配置默认的内存请求和限制

###  2.  Kubernetes 为命名空间配置默认的CPU请求和限制

###  3.  Kubernetes 配置命名空间的最小和最大内存约束

###  4.  Kubernetes 为命名空间配置CPU最小和最大约束

###  5.  Kubernetes 为命名空间配置内存和CPU配额

###  6.  Kubernetes 配置命名空间下Pod配额

##  5.  Kubernetes 安装网络策略驱动

###  1.  Kubernetes 使用Antrea提供NetworkPolicy

###  2.  Kubernetes 使用Calico提供NetworkPolicy

###  3.  Kubernetes 使用Cilium提供NetworkPolicy

###  4.  Kubernetes 使用kube-router提供NetworkPolicy

###  5.  Kubernetes 使用Romana提供NetworkPolicy

###  6.  Kubernetes 使用Weave Net提供NetworkPolicy

##  6.  Kubernetes IP Masquerade Agent用户指南

##  7.  Kubernetes 云管理控制器

##  8.  Kubernetes 验证签名的容器镜像

##  9.  Kubernetes 运行 etcd 集群

##  10.  Kubernetes 为系统守护进程预留计算资源

##  11.  Kubernetes 为节点发布扩展资源

##  12.  Kubernetes 以非root用户身份运行Kubernetes节点组件

##  13.  Kubernetes 使用CoreDNS进行服务发现

##  14.  Kubernetes 使用KMS驱动进行数据加密

##  15.  Kubernetes 使用Kubernetes API访问集群

##  16.  Kubernetes 使用NUMA感知的内存管理器

##  17.  Kubernetes 保护集群

##  18.  Kubernetes 关键插件Pod的调度保证

##  19.  Kubernetes 升级集群

##  20.  Kubernetes 名字空间演练

##  21.  Kubernetes 启用/禁用Kubernetes API

##  22.  Kubernetes 在Kubernetes集群中使用NodeLocal DNSCache

##  23.  Kubernetes 在Kubernetes集群中使用sysctl

##  24.  Kubernetes 在运行中的集群上重新配置节点的kubelet

##  25.  Kubernetes 在集群中使用级联删除
