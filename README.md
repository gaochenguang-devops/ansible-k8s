# ansible-k8s

`ansible-k8s` 是一套基于 Ansible + kubeadm 的 Kubernetes 集群部署剧本。它把系统初始化、containerd、Kubernetes 软件源、kubeadm 初始化、CNI 安装、HA API Server 入口、扩容、缩容、清理和验证拆成独立角色，目标是让部署过程标准化、可重复、可排错。

这份 README 按新手第一次使用的顺序编写。你可以先照着“快速开始”跑通一套集群，再根据后面的章节调整版本、网络插件、高可用、扩容和清理策略。

## 功能概览

- 支持系统：Ubuntu、Debian、CentOS、Rocky、AlmaLinux、RedHat、OracleLinux、openEuler，以及常见 Debian/RedHat 系发行版。
- 容器运行时：containerd。
- Kubernetes 安装：使用官方 `pkgs.k8s.io` 新版软件源，按 minor 仓库和 patch 版本解析安装包。
- 网络插件：Calico、Flannel、Cilium。
- 高可用控制面：多 master 时自动部署 Keepalived + HAProxy，提供 `VIP:16443` 作为 kube-apiserver 入口。
- 幂等部署：重复执行会跳过已初始化 master、已加入节点，并用 `kubectl apply` 或 `helm upgrade --install` 收敛 CNI。
- 断点续跑：支持按阶段记录 checkpoint，中断后继续跑。
- 证书有效期：部署后默认把 kubeadm 主要证书设置为 100 年，并支持单独执行续期剧本。
- 运维操作：支持验证、清理、扩容 master/worker、删除 master/worker。
- 主机名和 hosts：可按 inventory 自动设置主机名，并维护、清理目标机 `/etc/hosts` 中的集群解析块。

## 快速开始

如果你只是想先跑通一套集群，按这个顺序走：

1. 修改 `inventory/sample/hosts.yml`，把示例 IP 换成你的服务器 IP。
2. 修改 `playbooks/group_vars/all.yml`，至少确认 `kube_version`、`cni_plugin`、`pod_network_cidr`、`service_cidr`、`kube_control_plane_vip`。
3. 测试 SSH：

```bash
ansible k8s_cluster -m ping
```

4. 预检查：

```bash
ansible-playbook playbooks/preflight.yml
```

5. 部署：

```bash
ansible-playbook playbooks/cluster.yml
```

6. 验证：

```bash
ansible-playbook playbooks/verify.yml
```

如果中途失败，修复问题后可以直接重复执行 `cluster.yml`。如果希望跳过已经成功的阶段，用：

```bash
ansible-playbook playbooks/cluster-resume.yml
```

## 目录结构

```text
ansible-k8s/
├── ansible.cfg                         # Ansible 默认配置，默认使用 inventory/sample/hosts.yml
├── inventory/
│   └── sample/hosts.yml                # 示例 inventory，填写节点 IP、SSH 用户等
├── playbooks/
│   ├── group_vars/all.yml              # 核心变量：版本、网络、VIP、CNI、清理策略等
│   ├── preflight.yml                   # 只做部署前检查
│   ├── cluster.yml                     # 完整部署入口
│   ├── cluster-resume.yml              # 开启断点续跑的部署入口
│   ├── verify.yml                      # 部署后健康检查
│   ├── reset.yml                       # 清理集群
│   ├── scale-up.yml                    # 新增 master 或 worker
│   ├── remove-node.yml                 # 删除 master 或 worker
│   ├── update-certs.yml                # 手动更新 kubeadm 证书有效期
│   └── resume-status.yml               # 查看断点状态
├── roles/
│   ├── preflight                       # 参数、系统、CNI、cgroup 等前置检查
│   ├── os_prepare                      # swap、内核模块、sysctl、SELinux、firewalld、hostname、hosts
│   ├── apiserver_lb                    # Keepalived + HAProxy
│   ├── containerd                      # containerd 安装和配置
│   ├── kubernetes_repo                 # Kubernetes DEB/RPM 软件源
│   ├── kubernetes_packages             # kubelet/kubeadm/kubectl 安装和 hold
│   ├── kubeadm_control_plane           # 初始化或加入控制面
│   ├── kubeadm_workers                 # worker 加入集群
│   ├── kubeadm_cert_renewal            # 更新 kubeadm 证书有效期
│   ├── cni                             # Calico/Flannel/Cilium
│   ├── verify_cluster                  # 健康检查
│   ├── reset_cluster                   # kubeadm reset 和状态清理
│   └── resume                          # 断点续跑状态管理
└── docs/
    ├── install-flow.md                 # 安装流程和阶段说明
    ├── test-cases.md                   # 全功能测试用例
    ├── ha-control-plane.md             # 高可用控制面说明
    ├── scale.md                        # 扩容和缩容说明
    ├── operations.md                   # 常用操作
    └── variables.md                    # 变量说明
```

重点记住两个文件：

- `inventory/sample/hosts.yml`：写有哪些机器、每台机器的 SSH 地址和节点地址。
- `playbooks/group_vars/all.yml`：写集群版本、CNI、VIP、网段、镜像源、清理策略等变量。

本项目当前把全局变量放在 `playbooks/group_vars/all.yml`。执行 `ansible-playbook playbooks/xxx.yml` 时会自动加载它。不要只把变量放到项目根目录的 `group_vars/all.yml` 后又删除 `playbooks/group_vars/all.yml`，否则可能出现 `resume_enabled is undefined` 这类变量未定义报错。

## 部署前准备

### 1. 准备控制端

控制端是执行 Ansible 的机器，建议使用 Linux、macOS 或 Windows WSL。目标节点是要部署 Kubernetes 的 Linux 服务器。

控制端需要：

- Ansible，建议 2.14 或更新版本。
- 能通过 SSH 访问所有目标节点。
- SSH 用户具备 root 权限，或可以 sudo 到 root。
- 能访问 Kubernetes、Docker、CNI、Helm 等下载地址，或者你已经配置好内网镜像源。

常见安装方式：

```bash
# Ubuntu/Debian 控制端
sudo apt update
sudo apt install -y ansible openssh-client

# RHEL/Rocky/CentOS/openEuler 控制端
sudo yum install -y ansible openssh-clients
```

检查 Ansible：

```bash
ansible --version
```

### 2. 准备目标节点

目标节点建议：

- CPU：至少 2 核。
- 内存：至少 2 GB，生产环境按业务规模增加。
- 磁盘：至少 30 GB 可用空间。
- 时间同步正常。
- 节点之间网络互通。
- 每台机器的 `node_ip` 固定，不要使用会变化的 DHCP 地址。
- Pod 网段和 Service 网段不要和现有机房、办公网、VPN 网段冲突。

如果要部署 Cilium，默认要求内核不低于 `5.10.0`。如果使用 Kubernetes 1.35 及以上，建议使用 cgroup v2 系统，例如 Rocky 9、Ubuntu 22.04+、openEuler 22.03+ 等。

### 3. 配置 SSH 免密

以 root 用户为例：

```bash
ssh-keygen -t ed25519
ssh-copy-id root@192.168.200.10
ssh-copy-id root@192.168.200.11
ssh-copy-id root@192.168.200.12
```

测试连通性：

```bash
ansible k8s_cluster -m ping
```

如果你没有修改 `ansible.cfg`，默认 inventory 是 `inventory/sample/hosts.yml`。

## 第一步：配置 inventory

打开 `inventory/sample/hosts.yml`，按你的服务器修改。

### 单 master 示例

适合测试环境或小规模环境：

```yaml
---
all:
  children:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    kube_control_plane:
      hosts:
        k8s-master-1:
          ansible_host: 192.168.200.10
          node_ip: 192.168.200.10
    kube_node:
      hosts:
        k8s-worker-1:
          ansible_host: 192.168.200.11
          node_ip: 192.168.200.11
        k8s-worker-2:
          ansible_host: 192.168.200.12
          node_ip: 192.168.200.12

  vars:
    ansible_user: root
    ansible_port: 22
```

单 master 时，建议在 `playbooks/group_vars/all.yml` 中设置：

```yaml
kube_control_plane_vip: ""
```

单 master 默认不会部署 Keepalived/HAProxy，也不会使用 VIP 作为 kubeadm endpoint。不要只设置 `kube_control_plane_vip`；否则 preflight 会拦截。确实想让单 master 也通过 VIP 访问时，需要同时设置：

```yaml
kube_control_plane_vip: "192.168.200.181"
apiserver_lb_enabled: true
apiserver_lb_port: 16443
```

### 三 master 高可用示例

适合生产或准生产环境：

```yaml
---
all:
  children:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    kube_control_plane:
      hosts:
        k8s-master-1:
          ansible_host: 192.168.200.10
          node_ip: 192.168.200.10
        k8s-master-2:
          ansible_host: 192.168.200.11
          node_ip: 192.168.200.11
        k8s-master-3:
          ansible_host: 192.168.200.12
          node_ip: 192.168.200.12
    kube_node:
      hosts:
        k8s-worker-1:
          ansible_host: 192.168.200.21
          node_ip: 192.168.200.21
        k8s-worker-2:
          ansible_host: 192.168.200.22
          node_ip: 192.168.200.22

  vars:
    ansible_user: root
    ansible_port: 22
```

字段解释：

- `k8s-master-1`、`k8s-worker-1`：inventory 主机名。默认会被设置成目标机 hostname。
- `ansible_host`：Ansible SSH 连接地址。
- `node_ip`：Kubernetes 节点对集群通告的 IP，通常和 `ansible_host` 一样。
- `kube_control_plane`：控制面节点，也就是 master。
- `kube_node`：工作节点，也就是 worker。
- `k8s_cluster`：所有 Kubernetes 节点的集合。

## 第二步：修改核心变量

打开 `playbooks/group_vars/all.yml`。第一次部署先重点检查下面这些变量。

### Kubernetes 版本

```yaml
kube_version: "1.36.0"
kube_repo_minor_version: "1.36"
kube_package_version: ""
kube_image_repository: "registry.k8s.io"
```

说明：

- `kube_version` 建议写精确 patch 版本，例如 `1.36.0`，便于重复部署得到同一版本。
- `kube_repo_minor_version` 是 Kubernetes 官方软件源的 minor 仓库，例如 `1.36`。
- `kube_version: ""` 表示安装该 minor 仓库里的最新 patch，不如固定 patch 可控。
- `kube_package_version` 一般留空，剧本会在目标节点解析 DEB/RPM 真实包版本。只有特殊发行版解析失败时才手动写。

RPM 系系统可能解析到类似 `1.36.0-150500.1.1` 的包版本，Debian/Ubuntu 可能是类似 `1.36.0-1.1`。这个差异由剧本自动处理，不建议手写混用。

### 集群网络

```yaml
pod_network_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"
kube_dns_domain: "cluster.local"
```

注意：

- Pod 网段必须和 CNI 配置一致。
- Pod 网段、Service 网段不要和物理网络、VPN、办公网冲突。
- Flannel 默认常用 `10.244.0.0/16`。
- Calico 可以使用 `10.244.0.0/16`，也可以按你的规划调整。

### CNI 网络插件

三选一：

```yaml
cni_plugin: "calico"
# cni_plugin: "flannel"
# cni_plugin: "cilium"
```

对应版本：

```yaml
calico_version: "3.31.5"
flannel_version: "0.28.2"
cilium_version: "1.19.3"
```

如果选择 Cilium，还要注意：

```yaml
cilium_min_kernel: "5.10.0"
cilium_kube_proxy_replacement: false
cilium_extra_helm_args: []
```

### 高可用控制面 VIP

单 master：

```yaml
kube_control_plane_vip: ""
```

普通单 master 必须留空。多 master 或显式启用内置 LB 时才使用 VIP。

多 master：

```yaml
kube_control_plane_vip: "192.168.200.181"
apiserver_lb_enabled: "{{ (groups['kube_control_plane'] | default([]) | length) > 1 }}"
apiserver_lb_port: 16443
kube_apiserver_port: 6443
```

多 master 时，剧本会在每个 master 上部署：

- Keepalived：负责漂移 VIP。
- HAProxy：监听 `16443`，转发到各 master 的 `6443`。
- kubeadm endpoint：使用 `VIP:16443`。

VIP 不需要提前存在，但必须满足：

- 这个 IP 没有被其他机器占用。
- 和 master 节点在同一个二层网络，Keepalived 能漂移过去。
- 不在 DHCP 自动分配范围内，避免冲突。

`apiserver_lb_port` 默认用 `16443`，是为了避免 HAProxy 在 master 本机和 kube-apiserver 的 `6443` 端口冲突。

### Keepalived 常用变量

```yaml
apiserver_lb_interface: ""
apiserver_lb_vip_prefix: 32
apiserver_lb_keepalived_vrid: 51
apiserver_lb_keepalived_auth_pass: "k8sha"
apiserver_lb_keepalived_unicast: false
```

说明：

- `apiserver_lb_interface` 为空时自动使用默认网卡。如果 VIP 没漂上来，建议手动写网卡名，例如 `eth0`、`ens33`。
- `apiserver_lb_keepalived_auth_pass` 最长 8 位。
- 如果网络禁用了 VRRP 组播，设置 `apiserver_lb_keepalived_unicast: true`。
- Keepalived 模板里所有节点都是 `state BACKUP` 是正常设计，真正谁持有 VIP 由 priority 决定。这样可以避免固定 MASTER 恢复后强行抢占造成抖动。

### cgroup v1 兼容

Kubernetes 1.35 及以上默认对 cgroup v1 更严格。老系统，例如 CentOS 7，可能出现 kubeadm preflight 失败。

推荐优先选择：

- 升级到支持 cgroup v2 的系统或内核。
- 或把 Kubernetes 降到 1.35 以下。

如果你明确接受风险，可以启用：

```yaml
allow_legacy_cgroup_v1: true
```

启用后，剧本会：

- 在 kubelet 配置中设置 `failCgroupV1: false`。
- 给 kubeadm 加上 `SystemVerification` 的 ignore preflight。

### 主机名和 hosts 管理

```yaml
manage_hostname: true
manage_etc_hosts: true
reset_remove_etc_hosts: true
```

开启后：

- 目标节点 hostname 会设置为 inventory 主机名，例如 `k8s-master-1`。
- 目标节点 `/etc/hosts` 会写入一个 Ansible 管理块，包含所有集群节点和 VIP 解析。
- 执行 `reset.yml` 时，如果 `reset_remove_etc_hosts: true`，会清理这个管理块。

### 代理和镜像源

需要通过代理访问外网时：

```yaml
proxy_env:
  http_proxy: "http://proxy.example.com:8080"
  https_proxy: "http://proxy.example.com:8080"
  no_proxy: "127.0.0.1,localhost,10.96.0.0/12,10.244.0.0/16,192.168.200.0/24"
```

需要替换软件源或镜像源时，优先改这些变量：

```yaml
kube_image_repository: "registry.k8s.io"
kubernetes_repo_base_url: "https://pkgs.k8s.io/core:/stable:/v{{ effective_kube_repo_minor_version }}"
docker_apt_repo_url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}"
docker_rpm_repo_url: "https://download.docker.com/linux/centos/{{ ansible_distribution_major_version }}/$basearch/stable"
helm_download_base_url: "https://get.helm.sh"
cilium_helm_repo_url: "https://helm.cilium.io"
```

containerd 镜像加速示例：

```yaml
containerd_registry_mirrors:
  docker.io:
    - "https://registry-1.docker.io"
  registry.k8s.io:
    - "https://registry.k8s.io"
```

## 第三步：部署集群

### 1. 检查 inventory 连通性

```bash
ansible k8s_cluster -m ping
```

如果这里失败，先解决 SSH、用户、端口、防火墙、sudo 权限问题。

### 2. 执行部署前检查

```bash
ansible-playbook playbooks/preflight.yml
```

preflight 会检查：

- inventory 分组是否正确。
- Kubernetes 版本格式是否正确。
- 证书续期变量是否合理。
- 多 master 是否配置了 VIP。
- CNI 是否支持。
- containerd 是否是当前支持的运行时。
- 系统发行版是否在支持列表内。
- `node_ip` 是否可用。
- Kubernetes 1.35+ 和 cgroup v1 是否兼容。
- Cilium 内核版本是否满足要求。

### 3. 部署集群

```bash
ansible-playbook playbooks/cluster.yml
```

部署阶段顺序：

1. `preflight`：部署前检查。
2. `os_prepare`：关闭 swap，配置内核模块、sysctl、SELinux、firewalld、hostname、hosts。
3. `apiserver_lb`：多 master 时部署 Keepalived + HAProxy。
4. `containerd`：安装和配置 containerd。
5. `kubernetes_repo`：配置 Kubernetes 软件源。
6. `kubernetes_packages`：安装 kubelet、kubeadm、kubectl。
7. `control_plane`：初始化第一个 master，加入其他 master。
8. `cni`：安装 Calico、Flannel 或 Cilium。
9. `workers`：加入 worker 节点。
10. `certs`：更新 kubeadm 证书有效期，默认更新为 100 年。

### 4. 验证集群

```bash
ansible-playbook playbooks/verify.yml
```

验证会检查：

- Kubernetes API `/readyz`。
- 所有节点是否 Ready。
- CoreDNS 是否正常。
- 当前 CNI 的 DaemonSet 是否正常。
- 输出 `kubectl get nodes -o wide` 和 `kubectl get pods -A -o wide`。

你也可以在第一个 master 上手动查看：

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes -o wide
kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A -o wide
```

## 断点续跑

普通 `cluster.yml` 已经具备幂等性，失败后通常可以直接重复执行：

```bash
ansible-playbook playbooks/cluster.yml
```

如果你希望明确跳过已经成功完成的阶段，使用断点续跑入口：

```bash
ansible-playbook playbooks/cluster-resume.yml
```

查看断点状态：

```bash
ansible-playbook playbooks/resume-status.yml
```

清空断点后重新生成：

```bash
ansible-playbook playbooks/cluster-resume.yml -e resume_reset_state=true
```

强制重跑某个阶段，其他阶段仍按断点跳过：

```bash
ansible-playbook playbooks/cluster-resume.yml -e '{"resume_force_stages":["cni"]}'
```

可用阶段名：

```text
preflight
os_prepare
apiserver_lb
containerd
kubernetes_repo
kubernetes_packages
control_plane
cni
workers
certs
```

断点文件保存在控制端：

```text
.ansible/resume/<cluster_name>/
```

执行 `reset.yml` 后默认会清理断点，避免 reset 后再次部署时误跳过阶段。

## 证书有效期

部署完成后，剧本默认会在每个 control-plane 节点运行 `kubeadm_cert_renewal`，把 kubeadm 生成的主要证书更新为 100 年：

```yaml
kubeadm_certificate_validity_period: "876000h"
kubeadm_ca_certificate_validity_period: "876000h"
kubeadm_cert_renewal_enabled: true
kubeadm_cert_renewal_days: 36500
kubeadm_cert_renewal_min_remaining_days: 36400
kubeadm_cert_renewal_scope: "all"
kubeadm_cert_renewal_force: false
```

Kubernetes 1.31+ 会通过 kubeadm v1beta4 配置直接生成 100 年证书和 CA 证书；旧版本不支持这些 kubeadm 字段时，会在部署后通过续期脚本重签主要证书。旧版本的 CA 证书不由脚本静默替换，如需 CA 也变为 100 年，建议使用 Kubernetes 1.31+ 重新部署或单独规划 CA 轮换。

这个功能基于项目内的 `update-kubeadm-cert.sh` 思路实现，实际下发的是 `roles/kubeadm_cert_renewal/templates/update-kubeadm-cert.sh.j2`，已改成适配 containerd/crictl。它会更新 apiserver、apiserver-kubelet-client、front-proxy-client、etcd server/peer/healthcheck-client、apiserver-etcd-client，以及 admin/super-admin/controller-manager/scheduler/kubelet kubeconfig 中的客户端证书。

手动更新：

```bash
ansible-playbook playbooks/update-certs.yml
```

强制重签：

```bash
ansible-playbook playbooks/update-certs.yml -e kubeadm_cert_renewal_force=true
```

查看有效期：

```bash
ansible kube_control_plane -m shell -a "kubeadm certs check-expiration"
```

## 分阶段执行和排错

只跑 preflight：

```bash
ansible-playbook playbooks/cluster.yml --tags preflight
```

只做系统准备：

```bash
ansible-playbook playbooks/cluster.yml --tags os
```

只安装 containerd：

```bash
ansible-playbook playbooks/cluster.yml --tags runtime
```

只配置 Kubernetes 软件源：

```bash
ansible-playbook playbooks/cluster.yml --tags repo
```

只安装 kubelet/kubeadm/kubectl：

```bash
ansible-playbook playbooks/cluster.yml --tags packages
```

只部署 HAProxy + Keepalived：

```bash
ansible-playbook playbooks/cluster.yml --tags lb
```

只处理控制面：

```bash
ansible-playbook playbooks/cluster.yml --tags control-plane
```

只安装 CNI：

```bash
ansible-playbook playbooks/cluster.yml --tags cni
```

只加入 worker：

```bash
ansible-playbook playbooks/cluster.yml --tags workers
```

常用 tag 汇总：

```text
preflight
os
prepare
lb
apiserver-lb
haproxy
keepalived
runtime
containerd
repo
packages
kubeadm
control-plane
workers
certs
cni
verify
reset
cleanup
remove
drain
delete
```

## 高可用控制面说明

当 `kube_control_plane` 里有多个节点时，默认：

```yaml
apiserver_lb_enabled: "{{ (groups['kube_control_plane'] | default([]) | length) > 1 }}"
```

也就是多 master 自动启用内置 HA 入口。

工作方式：

```text
kubectl/kubelet/kubeadm
        |
        v
VIP:16443
        |
        v
HAProxy on current VIP owner
        |
        v
master-1:6443 / master-2:6443 / master-3:6443
```

检查 HAProxy 和 Keepalived：

```bash
ansible kube_control_plane -m shell -a "systemctl status haproxy keepalived --no-pager"
```

检查 VIP 漂在哪台 master：

```bash
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.181"
```

检查 HAProxy 监听端口：

```bash
ansible kube_control_plane -m shell -a "ss -lntp | grep 16443"
```

如果 VIP 没出现，优先检查：

- `kube_control_plane_vip` 是否是空闲 IP。
- `apiserver_lb_interface` 是否选对网卡。
- 云平台或交换网络是否允许 VRRP。
- 如果组播不通，设置 `apiserver_lb_keepalived_unicast: true`。
- `apiserver_lb_keepalived_auth_pass` 是否超过 8 位。

## 清理集群

清理所有 inventory 中的 Kubernetes 节点：

```bash
ansible-playbook playbooks/reset.yml
```

默认清理内容由 `playbooks/group_vars/all.yml` 控制：

```yaml
reset_remove_packages: true
reset_remove_state: true
reset_remove_containerd_data: true
reset_remove_downloads: true
reset_remove_etc_hosts: true
reset_flush_iptables: true
reset_flush_ip6tables: true
reset_remove_network_links: true
reset_remove_kubernetes_repo: true
reset_remove_containerd_config: true
reset_remove_pod_logs: true
reset_remove_os_tuning: true
resume_clear_on_reset: true
```

默认会做这些事：

- 执行 `kubeadm reset -f`。
- 停止 kubelet。
- 停止并禁用 HAProxy、Keepalived，释放 VIP。
- 停止 containerd。
- 卸载 Kubernetes、CNI、Calico、BPF 相关挂载点。
- 删除 `/etc/kubernetes`、`/var/lib/etcd`、`/var/lib/kubelet`、`/var/lib/cni`、`/etc/cni` 等状态目录。
- 清理 Calico runtime 状态，避免 `/var/run/calico/cgroup/io.pressure` 这类 cgroup 挂载残留导致删除失败。
- 删除常见 CNI 虚拟网卡，例如 `cni0`、`flannel.1`、`cilium_host`、`vxlan.calico`、`cali*`。
- 删除 Pod 和容器日志目录。
- 删除 containerd 数据。
- 删除 containerd 配置和 crictl 配置。
- 删除下载的 CNI manifest 和 Helm 临时包。
- 删除证书续期脚本 `/usr/local/sbin/update-kubeadm-cert.sh`。
- 清理目标机 `/etc/hosts` 中 Ansible 管理的集群解析块。
- 清理 iptables、ip6tables 和 IPVS。
- 删除 Kubernetes 软件源文件。
- 删除本项目创建的 sysctl 和内核模块配置。
- 卸载 kubelet、kubeadm、kubectl、kubernetes-cni、cri-tools。
- 清理断点续跑状态。

如果你想保留软件包和 containerd 镜像，方便快速重装：

```yaml
reset_remove_packages: false
reset_remove_containerd_data: false
reset_remove_downloads: false
reset_remove_kubernetes_repo: false
reset_remove_containerd_config: false
```

然后执行：

```bash
ansible-playbook playbooks/reset.yml
```

注意：`reset.yml` 会清理目标节点状态，不会自动修改控制端的 `inventory/sample/hosts.yml`。如果是永久删除某台机器，清理完成后再手动从 inventory 删除。

## 扩容节点

扩容前先修改 `inventory/sample/hosts.yml`，把新节点加入对应分组。

### 新增 worker

1. 把新节点加入 `kube_node`：

```yaml
kube_node:
  hosts:
    k8s-worker-1:
      ansible_host: 192.168.200.21
      node_ip: 192.168.200.21
    k8s-worker-2:
      ansible_host: 192.168.200.22
      node_ip: 192.168.200.22
    k8s-worker-3:
      ansible_host: 192.168.200.23
      node_ip: 192.168.200.23
```

2. 执行 worker 扩容：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,runtime,packages,workers
ansible-playbook playbooks/verify.yml
```

已加入过的 worker 会因为 `/etc/kubernetes/kubelet.conf` 存在而跳过 join，重复执行是安全的。

### 新增 master

建议一开始就按 HA 规划部署多 master。如果你已经是多 master 集群，新增 master 的步骤如下。

1. 把新 master 加入 `kube_control_plane`。

2. 确认已经配置 VIP：

```yaml
kube_control_plane_vip: "192.168.200.181"
apiserver_lb_enabled: true
apiserver_lb_port: 16443
```

3. 执行 master 扩容：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
ansible-playbook playbooks/verify.yml
```

剧本会：

- 在所有 master 上重新收敛 HAProxy/Keepalived 配置。
- 如果原来是单 master endpoint，自动把 kubeadm 的 `controlPlaneEndpoint` 迁移到 `VIP:16443`。
- 必要时重新生成控制面 apiserver 证书，让证书 SAN 包含 VIP。
- 把已有节点 kubeconfig 中的 API Server 地址收敛到新的控制面 endpoint。
- 给新 master 安装 containerd 和 Kubernetes 软件包。
- 生成 join 命令和 certificate key。
- 把未加入的 master 作为 control-plane 加入集群。

如果你是从单 master 改造成多 master，扩容前必须设置 `kube_control_plane_vip`，并确认 `apiserver_lb_enabled` 为 `true`。扩容时会短暂重启 kube-apiserver 静态 Pod 来刷新证书，建议在维护窗口执行。

## 删除节点

删除节点时，不要先从 inventory 删除。正确顺序是：

1. 节点仍保留在 `inventory/sample/hosts.yml`。
2. 执行 `remove-node.yml`。
3. 成功后再从 inventory 删除该节点。

### 删除 worker

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-2"]}'
```

### 删除 master

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-master-3"]}'
```

要求至少保留一个 master。对于内置 HA 模式，剧本会：

- 停止被删除节点上的 HAProxy 和 Keepalived。
- 从剩余 master 的 HAProxy backend 中移除该节点。
- drain 节点。
- 在目标节点执行 reset。
- 删除 Kubernetes Node 对象。
- 再次收敛剩余 master 的 HAProxy/Keepalived。

### 一次删除多个节点

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-1","k8s-worker-2"]}'
```

也可以传逗号分隔字符串：

```bash
ansible-playbook playbooks/remove-node.yml -e remove_nodes=k8s-worker-1,k8s-worker-2
```

常用变量：

```yaml
remove_node_drain: true
remove_node_drain_timeout: "10m"
remove_node_force: true
remove_node_ignore_daemonsets: true
remove_node_delete_emptydir_data: true
remove_node_reset: true
remove_node_delete_from_cluster: true
remove_node_ignore_missing: true
```

如果节点已经完全宕机且无法 SSH，不能执行目标节点 reset。可以按实际情况关闭部分步骤或只执行可达节点上的 LB 更新和 Kubernetes Node 删除。

## 切换 CNI

第一次部署前直接改：

```yaml
cni_plugin: "calico"
```

可选值：

```yaml
cni_plugin: "calico"
cni_plugin: "flannel"
cni_plugin: "cilium"
```

已经运行的集群不建议直接无脑切换 CNI。不同 CNI 会写入不同 CRD、DaemonSet、CNI 配置和节点网络状态。更稳妥的方式是：

1. 备份业务和集群配置。
2. 执行 `reset.yml` 清理所有节点。
3. 修改 `cni_plugin` 和相关网段。
4. 重新执行 `cluster.yml`。

## 版本和系统兼容建议

### Kubernetes 软件源

Kubernetes 官方新仓库按 minor 版本区分：

```text
https://pkgs.k8s.io/core:/stable:/v1.36/deb/
https://pkgs.k8s.io/core:/stable:/v1.36/rpm/
```

本项目用：

```yaml
kubernetes_repo_base_url: "https://pkgs.k8s.io/core:/stable:/v{{ effective_kube_repo_minor_version }}"
```

然后自动生成 DEB/RPM 源地址。

### RedHat 系包版本

RedHat 系安装时会使用：

```bash
yum --disableexcludes=kubernetes --showduplicates list kubelet
```

剧本会选择和 `kube_version` 匹配的最高 release，例如：

```text
1.36.0-150500.1.1
```

然后逐个安装：

```text
kubelet-1.36.0-150500.1.1
kubeadm-1.36.0-150500.1.1
kubectl-1.36.0-150500.1.1
```

这可以避免把多个包拼成一个带空格字符串导致 yum 模块报错。

### openEuler

openEuler 走 RedHat family 逻辑。containerd 默认使用系统仓库里的 `containerd` 包，而不是 Docker upstream 的 `containerd.io` 包。

如果你的 openEuler 环境软件源较旧，建议先确认系统仓库里有可用 containerd，或者准备内部源。

### Kubernetes 1.35+ 和老系统

如果看到类似错误：

```text
cgroups v1 support is deprecated and will be removed in a future release
```

说明当前系统使用 cgroup v1，而 Kubernetes 1.35+ 默认更严格。

处理方式：

1. 推荐：升级系统或内核，使用 cgroup v2。
2. 推荐：使用 Kubernetes 1.34 或更低版本。
3. 临时兼容：设置 `allow_legacy_cgroup_v1: true`。

如果还看到 containerd `RuntimeConfig` warning，说明 containerd 版本偏旧。Kubernetes 1.36 里通常还是 warning，但面向 1.37+ 应尽快升级运行时。

## 常见问题

### 1. `resume_enabled is undefined`

通常是变量文件没有被加载。

检查：

```bash
ls playbooks/group_vars/all.yml
ansible-playbook playbooks/preflight.yml
```

本项目默认变量文件位置是：

```text
playbooks/group_vars/all.yml
```

如果你移动了变量文件，确认 Ansible 能加载它。最简单的做法是保持当前目录结构不变。

### 2. 多 master 报 `kube_control_plane_vip` 没设置

多 master 必须有稳定的 API Server 入口。设置：

```yaml
kube_control_plane_vip: "192.168.200.181"
```

VIP 不需要提前存在，Keepalived 会创建和漂移它。但它必须是空闲 IP，并且网络允许 VRRP 或配置了 unicast。

### 3. 为什么 Keepalived 三个节点都是 BACKUP

这是正常设计。所有节点都配置成 BACKUP，通过 priority 选出实际持有 VIP 的节点。这样 failover 更平滑，也能避免固定 MASTER 恢复时立刻抢占导致连接抖动。

查看谁持有 VIP：

```bash
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.181"
```

### 4. VIP 不漂移或 HAProxy 端口不通

检查：

```bash
ansible kube_control_plane -m shell -a "systemctl status keepalived haproxy --no-pager"
ansible kube_control_plane -m shell -a "ip addr"
ansible kube_control_plane -m shell -a "ss -lntp | grep 16443"
```

常见原因：

- VIP 被占用。
- 网卡自动识别错误，手动设置 `apiserver_lb_interface`。
- 网络不允许 VRRP 组播，设置 `apiserver_lb_keepalived_unicast: true`。
- 防火墙或安全组阻止 16443/6443。
- HAProxy backend 中 master 的 `node_ip` 写错。

### 5. yum 安装 Kubernetes 包时报空格字符串错误

当前剧本已经把 RedHat 系 Kubernetes 包拆成列表逐个安装。如果你仍遇到类似错误，先确认本地代码是最新版本，并检查 `roles/kubernetes_packages/tasks/main.yml` 中 RedHat 安装任务是否使用 loop 安装 `kube_rpm_packages`。

### 6. kubeadm init 报 cgroup v1 错误

优先升级系统或使用 Kubernetes 1.34 及以下。确实要在老系统上跑 Kubernetes 1.35+ 时：

```yaml
allow_legacy_cgroup_v1: true
```

然后重新执行：

```bash
ansible-playbook playbooks/cluster.yml
```

### 7. reset 清理 Calico 目录时报 `io.pressure Operation not permitted`

这是 Calico runtime 目录里存在 cgroup 挂载残留。当前 `reset_cluster` 角色会先查找并 lazy unmount 相关挂载点，再删除 `/var/run/calico` 和 `/run/calico`。如果仍失败，可以手动查看：

```bash
mount | grep calico
```

然后重新执行：

```bash
ansible-playbook playbooks/reset.yml
```

### 8. 节点一直 NotReady

常见原因：

- CNI 没安装成功。
- Pod CIDR 和 CNI 配置不一致。
- 节点之间网络不通。
- kubelet 使用的 `node_ip` 不对。
- containerd 未正常启动。

排查：

```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf get nodes -o wide
kubectl --kubeconfig /etc/kubernetes/admin.conf get pods -A -o wide
systemctl status kubelet containerd --no-pager
journalctl -u kubelet -n 200 --no-pager
```

### 9. Cilium 安装失败

先确认内核版本：

```bash
uname -r
```

默认要求：

```yaml
cilium_min_kernel: "5.10.0"
```

然后检查 Helm 仓库和网络是否能访问：

```bash
curl -I https://helm.cilium.io
```

### 10. 执行 playbook 时下载失败

如果目标节点不能访问外网，需要配置代理或内网源。

代理：

```yaml
proxy_env:
  http_proxy: "http://proxy.example.com:8080"
  https_proxy: "http://proxy.example.com:8080"
  no_proxy: "127.0.0.1,localhost,10.96.0.0/12,10.244.0.0/16,192.168.200.0/24"
```

内网源：

```yaml
kubernetes_repo_base_url: "https://your-mirror.example.com/core:/stable:/v{{ effective_kube_repo_minor_version }}"
kube_image_repository: "your-registry.example.com/registry.k8s.io"
```

## 推荐执行流程

新集群：

```bash
ansible k8s_cluster -m ping
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

部署中断后继续：

```bash
ansible-playbook playbooks/cluster-resume.yml
ansible-playbook playbooks/verify.yml
```

彻底重装：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

新增 worker：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,runtime,packages,workers
ansible-playbook playbooks/verify.yml
```

新增 master：

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
ansible-playbook playbooks/verify.yml
```

删除节点：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-2"]}'
ansible-playbook playbooks/verify.yml
```

## 生产环境建议

- Kubernetes、containerd、CNI 都固定精确版本，不要生产环境长期使用“最新 patch”自动漂移。
- 多 master 建议从一开始就规划 VIP 和 HA endpoint。
- VIP 使用专门保留地址，不要和 DHCP、云平台浮动 IP、其他业务 VIP 冲突。
- 准备内网 yum/apt 源、镜像仓库、CNI manifest 或 Helm chart 镜像。
- 定期备份 etcd。
- 变更 CNI、Pod CIDR、Service CIDR 前先做完整备份，生产环境不要直接在线切换。
- 老系统和 cgroup v1 只作为兼容方案，长期建议升级系统。
- 每次扩容、缩容、升级后执行 `playbooks/verify.yml`。

## 继续阅读

- [docs/install-flow.md](docs/install-flow.md)：完整安装流程和阶段说明。
- [docs/test-cases.md](docs/test-cases.md)：全功能测试用例和验收清单。
- [docs/ha-control-plane.md](docs/ha-control-plane.md)：高可用控制面说明。
- [docs/scale.md](docs/scale.md)：扩容和缩容说明。
- [docs/operations.md](docs/operations.md)：常用运维操作。
- [docs/variables.md](docs/variables.md)：变量说明。
