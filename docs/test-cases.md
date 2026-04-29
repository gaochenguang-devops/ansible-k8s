# 测试用例

本文用于完整验证 `ansible-k8s` 当前功能。测试覆盖部署、清理、断点续跑、幂等、HA 控制面、CNI、扩容、缩容、版本适配、多系统兼容和常见失败场景。

这些用例会真实修改目标节点系统配置，部分用例会清理 Kubernetes、iptables、CNI 网卡、containerd 数据和软件包。请只在测试环境执行，不要直接在生产集群跑全量测试。

## 测试约定

### 用例级别

| 级别 | 说明 |
| --- | --- |
| P0 | 核心功能，必须通过。 |
| P1 | 常用能力，建议每次发版或大改后执行。 |
| P2 | 兼容、异常和边界场景，可按环境选择执行。 |

### 测试环境建议

至少准备两套环境：

| 环境 | 节点 | 用途 |
| --- | --- | --- |
| 单 master 环境 | 1 master + 2 worker | 基础部署、CNI、断点、幂等、清理。 |
| HA 环境 | 3 master + 2 worker | VIP、Keepalived、HAProxy、master 扩缩容。 |

多系统兼容建议准备：

| 系统族 | 建议系统 | 覆盖点 |
| --- | --- | --- |
| Debian family | Ubuntu 22.04/24.04 或 Debian 12 | apt repo、DEB 包版本解析、hold/unhold。 |
| RedHat family | Rocky 8/9、CentOS 7、openEuler 22.03+ | yum/dnf repo、RPM 包版本解析、cgroup v1/v2。 |

### 通用前置条件

1. 控制端可执行 `ansible` 和 `ansible-playbook`。
2. 控制端能 SSH 到所有目标节点。
3. 目标节点时间同步正常。
4. 目标节点之间网络互通。
5. `inventory/sample/hosts.yml` 已按测试环境填写。
6. `playbooks/group_vars/all.yml` 已按测试场景设置。
7. 测试前确认目标节点可以被清理或重装。

### 通用检查命令

```bash
ansible k8s_cluster -m ping
ansible-playbook --syntax-check playbooks/cluster.yml
ansible-playbook --syntax-check playbooks/reset.yml
ansible-playbook playbooks/preflight.yml
```

部署后常用检查：

```bash
ansible-playbook playbooks/verify.yml
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes -o wide"
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A -o wide"
```

清理后常用检查：

```bash
ansible k8s_cluster -m shell -a "test ! -e /etc/kubernetes/kubelet.conf"
ansible k8s_cluster -m shell -a "test ! -d /var/lib/etcd"
ansible k8s_cluster -m shell -a "ip link show | egrep 'cni0|flannel.1|cilium|vxlan.calico|cali' || true"
ansible k8s_cluster -m shell -a "iptables-save | egrep 'KUBE-|CALI-|CILIUM|FLANNEL' || true"
```

## 测试矩阵

| 编号 | 场景 | 系统 | 拓扑 | CNI | 优先级 |
| --- | --- | --- | --- | --- | --- |
| M1 | 基础单 master 部署 | Rocky/Ubuntu 任一 | 1 master + 2 worker | Calico | P0 |
| M2 | HA 控制面部署 | Rocky/Ubuntu 任一 | 3 master + 2 worker | Calico | P0 |
| M3 | Flannel 部署 | 任一 | 1 master + 1 worker | Flannel | P1 |
| M4 | Cilium 部署 | kernel >= 5.10 | 1 master + 1 worker | Cilium | P1 |
| M5 | RedHat 系包版本解析 | RedHat family | 任一 | 任一 | P1 |
| M6 | Debian 系包版本解析 | Debian family | 任一 | 任一 | P1 |
| M7 | 老系统 cgroup v1 兼容 | CentOS 7 等 | 任一 | Calico/Flannel | P2 |
| M8 | 清理和重复清理 | 任一 | 已部署集群 | 任一 | P0 |
| M9 | 扩容和缩容 | HA 环境 | 3 master + worker | Calico | P0 |
| M10 | kubeadm 证书续期 | 任一 | 单 master/HA | 任一 | P0 |

## 静态和预检

### TC-001 语法检查

级别：P0

目的：确认所有主要 playbook 能被 Ansible 正常解析。

步骤：

```bash
ansible-playbook --syntax-check playbooks/preflight.yml
ansible-playbook --syntax-check playbooks/cluster.yml
ansible-playbook --syntax-check playbooks/cluster-resume.yml
ansible-playbook --syntax-check playbooks/reset.yml
ansible-playbook --syntax-check playbooks/verify.yml
ansible-playbook --syntax-check playbooks/scale-up.yml
ansible-playbook --syntax-check playbooks/remove-node.yml
ansible-playbook --syntax-check playbooks/update-certs.yml
ansible-playbook --syntax-check playbooks/resume-status.yml
```

预期结果：

- 所有命令返回成功。
- 无 YAML、变量解析、role 找不到等错误。

### TC-002 inventory 连通性

级别：P0

目的：确认控制端能连接所有目标节点。

步骤：

```bash
ansible k8s_cluster -m ping
```

预期结果：

- 所有节点返回 `pong`。
- 没有 unreachable、permission denied、sudo 失败。

### TC-003 preflight 成功

级别：P0

目的：确认当前 inventory 和变量满足部署条件。

步骤：

```bash
ansible-playbook playbooks/preflight.yml
```

预期结果：

- 所有节点通过系统、版本、CNI、runtime、node_ip、cgroup 校验。
- 多 master 场景下必须已经设置 `kube_control_plane_vip`。

### TC-004 preflight 缺少 VIP 失败

级别：P1

目的：验证多 master 必须配置稳定 API Server endpoint。

前置条件：

- inventory 中 `kube_control_plane` 至少 2 台。
- 临时设置 `kube_control_plane_vip: ""`。

步骤：

```bash
ansible-playbook playbooks/preflight.yml
```

预期结果：

- playbook 失败。
- 报错提示需要设置 `kube_control_plane_vip` 或稳定负载均衡地址。

### TC-004A 单 master 误设置 VIP 失败

级别：P1

目的：验证单 master 不会静默使用一个未部署的 VIP 作为 kubeadm endpoint。

前置条件：

- inventory 中 `kube_control_plane` 只有 1 台。
- `apiserver_lb_enabled: false` 或保持默认。
- 临时设置 `kube_control_plane_vip: "192.168.200.181"`。

步骤：

```bash
ansible-playbook playbooks/preflight.yml
```

预期结果：

- playbook 失败。
- 报错提示普通单 master 应设置 `kube_control_plane_vip: ""`。
- 如果确实要单 master 也使用 VIP，必须同时设置 `apiserver_lb_enabled: true`。

### TC-005 preflight 非法 CNI 失败

级别：P1

目的：验证 CNI 枚举校验。

步骤：

```bash
ansible-playbook playbooks/preflight.yml -e cni_plugin=invalid
```

预期结果：

- playbook 失败。
- 报错提示 `cni_plugin` 必须是 `calico`、`flannel` 或 `cilium`。

### TC-006 Kubernetes 1.35+ cgroup v1 拦截

级别：P2

目的：验证老系统 cgroup v1 保护逻辑。

前置条件：

- 目标节点使用 cgroup v1。
- `kube_version` 为 `1.35.0` 或更高。
- `allow_legacy_cgroup_v1: false`。

步骤：

```bash
ansible-playbook playbooks/preflight.yml
```

预期结果：

- playbook 失败。
- 报错提示升级到 cgroup v2、降级 Kubernetes，或设置 `allow_legacy_cgroup_v1: true`。

## 基础部署

### TC-101 单 master + worker 部署

级别：P0

目的：验证最小可用集群部署。

变量建议：

```yaml
kube_control_plane_vip: ""
cni_plugin: "calico"
pod_network_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- 第一个 master 生成 `/etc/kubernetes/admin.conf`。
- worker 生成 `/etc/kubernetes/kubelet.conf`。
- 所有节点 Ready。
- CoreDNS Ready。
- Calico DaemonSet rollout 成功。

### TC-102 部署幂等性

级别：P0

目的：验证重复执行不会重复 init/join，也不会破坏已有集群。

前置条件：

- TC-101 已成功。

步骤：

```bash
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- `kubeadm init` 被 `creates: /etc/kubernetes/admin.conf` 跳过。
- 已加入节点跳过 `kubeadm join`。
- 集群仍然 Ready。

### TC-103 分阶段执行

级别：P1

目的：验证常用 tags 可单独运行。

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags preflight
ansible-playbook playbooks/cluster.yml --tags os
ansible-playbook playbooks/cluster.yml --tags runtime
ansible-playbook playbooks/cluster.yml --tags repo
ansible-playbook playbooks/cluster.yml --tags packages
ansible-playbook playbooks/cluster.yml --tags cni
ansible-playbook playbooks/cluster.yml --tags workers
ansible-playbook playbooks/cluster.yml --tags certs
```

预期结果：

- 每个 tag 都能执行到对应阶段。
- 重复执行不破坏已有集群。

## 证书续期

### TC-151 部署后自动设置 100 年证书

级别：P0

目的：验证完整部署后 kubeadm 主要证书有效期被设置为约 100 年。

前置条件：

- TC-101 或 TC-201 已成功。
- `kubeadm_cert_renewal_enabled: true`。

步骤：

```bash
ansible kube_control_plane -m shell -a "kubeadm certs check-expiration"
ansible kube_control_plane -m shell -a "openssl x509 -checkend $((36400*86400)) -noout -in /etc/kubernetes/pki/apiserver.crt"
ansible kube_control_plane -m shell -a "openssl x509 -checkend $((36400*86400)) -noout -in /etc/kubernetes/pki/apiserver-kubelet-client.crt"
ansible kube_control_plane -m shell -a "openssl x509 -checkend $((36400*86400)) -noout -in /etc/kubernetes/pki/front-proxy-client.crt"
```

预期结果：

- `kubeadm certs check-expiration` 显示主要证书接近 100 年。
- `openssl -checkend` 返回成功。
- Kubernetes 1.31+ 使用 kubeadm v1beta4 时，CA 证书也应接近 100 年。

### TC-152 手动续期剧本幂等

级别：P0

目的：验证 `update-certs.yml` 可以单独执行，且证书剩余有效期足够时不会反复重签。

前置条件：

- TC-151 已成功。

步骤：

```bash
ansible-playbook playbooks/update-certs.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- 剩余有效期高于 `kubeadm_cert_renewal_min_remaining_days` 时，`Renew kubeadm certificates` 任务被跳过。
- 集群仍然 Ready。

### TC-153 强制重签证书

级别：P1

目的：验证需要立即刷新证书时可以强制重签。

前置条件：

- 已部署可用集群。

步骤：

```bash
ansible-playbook playbooks/update-certs.yml -e kubeadm_cert_renewal_force=true
ansible-playbook playbooks/verify.yml
ansible kube_control_plane -m shell -a "kubeadm certs check-expiration"
ansible kube_control_plane -m shell -a "ls -d /etc/kubernetes.old-$(date +%Y%m%d)"
```

预期结果：

- control-plane 节点上生成 `/usr/local/sbin/update-kubeadm-cert.sh`。
- `/etc/kubernetes.old-YYYYMMDD` 备份目录存在。
- apiserver、controller-manager、scheduler、kubelet 重启后集群恢复 Ready。
- 证书有效期符合 `kubeadm_cert_renewal_days`。

## HA 控制面

### TC-201 三 master HA 部署

级别：P0

目的：验证 Keepalived + HAProxy 内置 apiserver 入口。

前置条件：

- inventory 中有 3 台 `kube_control_plane`。
- VIP 是未占用 IP，且和 master 在同一二层网络。

变量建议：

```yaml
kube_control_plane_vip: "192.168.200.181"
apiserver_lb_enabled: true
apiserver_lb_port: 16443
kube_apiserver_port: 6443
cni_plugin: "calico"
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- 每台 master 上 HAProxy、Keepalived 运行。
- VIP 出现在其中一台 master。
- HAProxy 监听 `16443`。
- kubeadm endpoint 为 `VIP:16443`。
- 3 台 master 都 Ready。

检查命令：

```bash
ansible kube_control_plane -m shell -a "systemctl is-active haproxy keepalived"
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.181 || true"
ansible kube_control_plane -m shell -a "ss -lntp | grep 16443"
```

### TC-202 Keepalived unicast 模式

级别：P1

目的：验证 VRRP 组播不可用网络中的 unicast 配置。

变量：

```yaml
apiserver_lb_keepalived_unicast: true
```

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags lb
ansible kube_control_plane -m shell -a "grep -n 'unicast' /etc/keepalived/keepalived.conf"
```

预期结果：

- Keepalived 配置包含 `unicast_src_ip` 和 `unicast_peer`。
- VIP 正常漂移。

### TC-203 VIP 故障漂移

级别：P1

目的：验证当前 VIP owner 故障后 VIP 可迁移。

步骤：

1. 找到 VIP 所在 master：

```bash
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.181 || true"
```

2. 在 VIP owner 上停止 Keepalived：

```bash
systemctl stop keepalived
```

3. 在其他 master 检查 VIP：

```bash
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.181 || true"
```

4. 恢复服务：

```bash
ansible kube_control_plane -m shell -a "systemctl start keepalived"
```

预期结果：

- VIP 从原 master 漂移到其他 master。
- `kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes` 仍可访问 API。

## CNI 插件

### TC-301 Calico 部署

级别：P0

变量：

```yaml
cni_plugin: "calico"
calico_version: "3.31.5"
pod_network_cidr: "10.244.0.0/16"
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- `calico-node` DaemonSet rollout 成功。
- 节点 Ready。
- Calico manifest 中 Pod CIDR 已被渲染为 `pod_network_cidr`。

检查命令：

```bash
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get ds calico-node"
ansible "kube_control_plane[0]" -m shell -a "grep CALICO_IPV4POOL_CIDR -A1 /opt/k8s/manifests/calico.yaml"
```

### TC-302 Flannel 部署

级别：P1

变量：

```yaml
cni_plugin: "flannel"
flannel_version: "0.28.2"
pod_network_cidr: "10.244.0.0/16"
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- `kube-flannel-ds` DaemonSet rollout 成功。
- 节点 Ready。
- Flannel manifest 中 Network 等于 `pod_network_cidr`。

### TC-303 Cilium 部署

级别：P1

前置条件：

- 所有节点内核版本不低于 `cilium_min_kernel`，默认 `5.10.0`。
- 目标节点可以访问 Helm 下载地址和 Cilium chart 仓库，或已配置镜像。

变量：

```yaml
cni_plugin: "cilium"
cilium_version: "1.19.3"
cilium_kube_proxy_replacement: false
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- Helm 已安装到 `/usr/local/bin/helm`。
- `cilium` DaemonSet rollout 成功。
- 节点 Ready。

检查命令：

```bash
ansible "kube_control_plane[0]" -m shell -a "/usr/local/bin/helm -n kube-system list | grep cilium"
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get ds cilium"
```

## 断点续跑

### TC-401 断点状态生成

级别：P0

目的：验证 `cluster-resume.yml` 会生成阶段 checkpoint。

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster-resume.yml
ansible-playbook playbooks/resume-status.yml
```

预期结果：

- `.ansible/resume/<cluster_name>/` 下生成阶段 `.done` 文件。
- `resume-status.yml` 能显示已完成阶段。
- `verify.yml` 成功。

### TC-402 断点跳过已完成阶段

级别：P1

前置条件：

- TC-401 已成功。

步骤：

```bash
ansible-playbook playbooks/cluster-resume.yml
```

预期结果：

- 已完成阶段被跳过。
- 集群状态保持 Ready。

### TC-403 强制重跑指定阶段

级别：P1

步骤：

```bash
ansible-playbook playbooks/cluster-resume.yml -e '{"resume_force_stages":["cni"]}'
ansible-playbook playbooks/verify.yml
```

预期结果：

- `cni` 阶段重新执行。
- 其他已完成阶段按 checkpoint 跳过。
- 集群仍 Ready。

### TC-404 reset 后清理断点

级别：P0

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/resume-status.yml
```

预期结果：

- reset 成功。
- 断点状态被清理，避免后续部署误跳过阶段。

## 清理集群

### TC-501 全量清理

级别：P0

目的：验证 reset 不漏清理 Kubernetes、CNI、HA、repo、日志和网络状态。

步骤：

```bash
ansible-playbook playbooks/reset.yml
```

预期结果：

- kubelet 停止并禁用。
- HAProxy、Keepalived 停止并禁用。
- `/etc/kubernetes`、`/var/lib/etcd`、`/var/lib/kubelet` 不存在。
- `/etc/cni`、`/var/lib/cni` 不存在。
- `/var/log/pods`、`/var/log/containers` 不存在。
- 常见 CNI 虚拟网卡不存在。
- Kubernetes repo 文件不存在。
- `/etc/hosts` 中没有 `ANSIBLE MANAGED K8S CLUSTER HOSTS` 块。

检查命令：

```bash
ansible k8s_cluster -m shell -a "systemctl is-active kubelet || true"
ansible k8s_cluster -m shell -a "systemctl is-active haproxy keepalived || true"
ansible k8s_cluster -m shell -a "test ! -d /etc/kubernetes && test ! -d /var/lib/etcd && test ! -d /var/lib/kubelet"
ansible k8s_cluster -m shell -a "test ! -d /etc/cni && test ! -d /var/lib/cni"
ansible k8s_cluster -m shell -a "ip link show | egrep 'cni0|flannel.1|cilium|vxlan.calico|cali' || true"
ansible k8s_cluster -m shell -a "grep 'ANSIBLE MANAGED K8S CLUSTER HOSTS' /etc/hosts || true"
```

### TC-502 重复清理幂等性

级别：P0

目的：验证 reset 可重复执行，不因文件不存在或服务不存在失败。

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/reset.yml
```

预期结果：

- 两次执行均成功。
- 不因 kubeadm 不存在、目录不存在、服务不存在而失败。

### TC-503 快速重装清理

级别：P1

目的：验证保留 containerd 数据和配置时仍能重新部署。

步骤：

```bash
ansible-playbook playbooks/reset.yml \
  -e reset_remove_containerd_data=false \
  -e reset_remove_containerd_config=false \
  -e reset_remove_kubernetes_repo=false
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- reset 成功。
- containerd 可保留或恢复运行。
- 后续部署成功。

### TC-504 清理后重新部署

级别：P0

目的：验证 reset 后可以再次从零部署。

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

预期结果：

- 新部署成功。
- 所有节点 Ready。

## 扩容

### TC-601 新增 worker

级别：P0

前置条件：

- 已有集群健康。
- 新 worker 已加入 inventory 的 `kube_node`。
- 新 worker SSH 可达。

步骤：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,runtime,packages,workers
ansible-playbook playbooks/verify.yml
```

预期结果：

- 新 worker 加入集群。
- 新 worker 状态 Ready。
- 已存在节点未被重复 join。

检查命令：

```bash
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes -o wide"
```

### TC-602 新增 master

级别：P0

前置条件：

- 已有 HA 集群健康。
- 新 master 已加入 inventory 的 `kube_control_plane`。
- `kube_control_plane_vip` 已配置。

步骤：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
ansible-playbook playbooks/verify.yml
```

预期结果：

- 新 master 加入控制面。
- HAProxy backend 包含新 master。
- 所有 master Ready。
- VIP 仍可访问。

检查命令：

```bash
ansible kube_control_plane -m shell -a "grep '^    server' /etc/haproxy/haproxy.cfg"
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes"
```

### TC-602A 单 master 扩成 HA 时迁移 endpoint

级别：P0

目的：验证单 master 初始部署后新增 master，会把 `masterIP:6443` 迁移为 `VIP:16443`。

前置条件：

- 先按单 master 部署成功，初始 `kube_control_plane_vip: ""`。
- 后续 inventory 新增至少 1 台 master。
- 设置 `kube_control_plane_vip`，且 `apiserver_lb_enabled` 为 `true`。

步骤：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
ansible-playbook playbooks/verify.yml
```

预期结果：

- `kubeadm-config` 中 `controlPlaneEndpoint` 为 `VIP:16443`。
- `kubeadm token create --print-join-command` 输出的 join endpoint 为 `VIP:16443`。
- 已有节点 `/etc/kubernetes/kubelet.conf` 指向 `VIP:16443`。
- 各 master 的 apiserver 证书 SAN 包含 VIP。

检查命令：

```bash
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf -n kube-system get cm kubeadm-config -o jsonpath='{.data.ClusterConfiguration}' | grep 'controlPlaneEndpoint:'"
ansible "kube_control_plane[0]" -m shell -a "kubeadm token create --print-join-command"
ansible k8s_cluster -m shell -a "grep 'server: https://' /etc/kubernetes/kubelet.conf || true"
ansible kube_control_plane -m shell -a "openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -text | grep 192.168.200.181"
```

## 缩容

### TC-701 删除 worker

级别：P0

前置条件：

- 要删除的 worker 仍在 inventory 中。

步骤：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-2"]}'
ansible-playbook playbooks/verify.yml
```

预期结果：

- 目标 worker 被 drain。
- 目标 worker 执行 reset。
- Kubernetes Node 对象被删除。
- 剩余节点 Ready。

检查命令：

```bash
ansible "kube_control_plane[0]" -m shell -a "kubectl --kubeconfig /etc/kubernetes/admin.conf get node k8s-worker-2 --ignore-not-found"
ansible k8s-worker-2 -m shell -a "test ! -e /etc/kubernetes/kubelet.conf"
```

### TC-702 删除 master

级别：P0

前置条件：

- 至少有 2 台 master。
- 要删除的 master 仍在 inventory 中。
- 删除后至少保留 1 台 master。

步骤：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-master-3"]}'
ansible-playbook playbooks/verify.yml
```

预期结果：

- 被删除 master 上 HAProxy、Keepalived 停止。
- 剩余 master 的 HAProxy backend 不再包含被删除 master。
- 被删除 master 执行 reset。
- Kubernetes Node 对象被删除。
- 剩余 master 和 worker Ready。

### TC-703 删除多个节点

级别：P1

步骤：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-1","k8s-worker-2"]}'
ansible-playbook playbooks/verify.yml
```

预期结果：

- 多个目标节点均被 drain、reset、delete node。
- 剩余集群健康。

### TC-704 删除最后一个 master 被拦截

级别：P1

目的：验证保护逻辑，避免把控制面删空。

步骤：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-master-1","k8s-master-2","k8s-master-3"]}'
```

预期结果：

- playbook 在输入校验阶段失败。
- 报错提示至少需要保留一个 control-plane 节点。

## 主机名和 hosts 管理

### TC-801 设置 hostname

级别：P1

变量：

```yaml
manage_hostname: true
```

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags os
ansible k8s_cluster -m shell -a "hostname"
```

预期结果：

- 每台机器 hostname 等于 inventory hostname。

### TC-802 维护 `/etc/hosts`

级别：P1

变量：

```yaml
manage_etc_hosts: true
```

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags os
ansible k8s_cluster -m shell -a "grep -A20 'ANSIBLE MANAGED K8S CLUSTER HOSTS' /etc/hosts"
```

预期结果：

- `/etc/hosts` 包含所有 `k8s_cluster` 节点解析。
- 配置 VIP 时包含 `<cluster_name>-api` 解析。

### TC-803 reset 清理 `/etc/hosts`

级别：P1

变量：

```yaml
reset_remove_etc_hosts: true
```

步骤：

```bash
ansible-playbook playbooks/reset.yml
ansible k8s_cluster -m shell -a "grep 'ANSIBLE MANAGED K8S CLUSTER HOSTS' /etc/hosts || true"
```

预期结果：

- `/etc/hosts` 中 Ansible 管理块被删除。

## 版本、软件源和系统兼容

### TC-901 RedHat 系 Kubernetes 包版本解析

级别：P1

前置条件：

- 目标节点为 RedHat family。

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags repo,packages
```

预期结果：

- 能解析到单个 RPM 包版本，例如 `1.36.0-150500.1.1`。
- kubelet、kubeadm、kubectl 逐个安装成功。
- 不出现“space separated string of packages”错误。

检查命令：

```bash
ansible k8s_cluster -m shell -a "rpm -q kubelet kubeadm kubectl"
```

### TC-902 Debian 系 Kubernetes 包版本解析

级别：P1

前置条件：

- 目标节点为 Debian family。

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags repo,packages
```

预期结果：

- 能解析到单个 DEB 包版本。
- kubelet、kubeadm、kubectl 安装成功。
- `kube_hold_packages: true` 时包被 hold。

检查命令：

```bash
ansible k8s_cluster -m shell -a "dpkg -l kubelet kubeadm kubectl | cat"
ansible k8s_cluster -m shell -a "apt-mark showhold | egrep 'kubelet|kubeadm|kubectl' || true"
```

### TC-903 openEuler containerd 安装

级别：P2

目的：验证 openEuler 使用系统仓库中的 `containerd` 包。

前置条件：

- 目标节点为 openEuler。

步骤：

```bash
ansible-playbook playbooks/cluster.yml --tags runtime
ansible k8s_cluster -m shell -a "systemctl is-active containerd"
```

预期结果：

- containerd 安装成功并运行。
- 不强制使用 Docker upstream `containerd.io`。

### TC-904 代理变量

级别：P2

目的：验证外网访问需要代理时 playbook 使用 `proxy_env`。

变量示例：

```yaml
proxy_env:
  http_proxy: "http://proxy.example.com:8080"
  https_proxy: "http://proxy.example.com:8080"
  no_proxy: "127.0.0.1,localhost,10.96.0.0/12,10.244.0.0/16,192.168.200.0/24"
```

步骤：

```bash
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml --tags repo,runtime,cni
```

预期结果：

- repo、containerd、CNI 下载阶段可以通过代理完成。

## 验收清单

全功能验收建议至少满足：

- TC-001、TC-002、TC-003 通过。
- TC-101、TC-102、TC-501、TC-502、TC-504 通过。
- TC-151、TC-152 通过；需要验证强制重签时 TC-153 通过。
- 如果启用多 master，TC-201 通过。
- 如果使用 Calico，TC-301 通过。
- 如果使用 Flannel，TC-302 通过。
- 如果使用 Cilium，TC-303 通过。
- 如果需要断点续跑，TC-401 到 TC-404 通过。
- 如果需要扩缩容，TC-601、TC-602、TC-701、TC-702 通过。
- 至少在一个 RedHat family 和一个 Debian family 系统上完成 `repo,packages` 阶段测试。

## 建议测试顺序

完整回归可以按下面顺序执行：

1. `TC-001` 到 `TC-003`
2. `TC-101` 到 `TC-103`
3. `TC-151` 到 `TC-153`
4. `TC-301`、`TC-302`、`TC-303`
5. `TC-401` 到 `TC-404`
6. `TC-201` 到 `TC-203`
7. `TC-601`、`TC-602`
8. `TC-701` 到 `TC-704`
9. `TC-801` 到 `TC-803`
10. `TC-901` 到 `TC-904`
11. `TC-501` 到 `TC-504`

每完成一个破坏性用例后，建议执行：

```bash
ansible-playbook playbooks/verify.yml
```

如果 verify 不适用于已经 reset 的集群，则使用清理后常用检查命令确认节点状态。
