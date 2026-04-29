# Variables

核心变量集中在 `playbooks/group_vars/all.yml`。推荐复制一份 inventory 目录并在自己的环境中维护变量覆盖，不要直接把生产地址写进示例 inventory。

## 版本

| 变量 | 说明 |
| --- | --- |
| `kube_version` | 精确 Kubernetes patch 版本。留空时安装 minor 仓库内最新 patch。 |
| `kube_repo_minor_version` | `kube_version` 留空时使用的 minor 仓库，如 `1.36`。 |
| `kube_package_version` | 极少数场景手动指定发行版包版本后缀。 |
| `kube_image_repository` | kubeadm 拉取控制面镜像的仓库，默认 `registry.k8s.io`。 |
| `calico_version` / `flannel_version` / `cilium_version` | CNI 版本。 |
| `helm_version` | 安装 Cilium 时使用的 Helm 版本。 |

## 网络

| 变量 | 说明 |
| --- | --- |
| `kube_control_plane_vip` | 多控制面必须配置为负载均衡地址或稳定 DNS。普通单 master 应留空；单 master 只有显式 `apiserver_lb_enabled: true` 时才应设置。 |
| `pod_network_cidr` | Pod 网段，需要和 CNI 配置一致。 |
| `service_cidr` | Service 网段。 |
| `cni_plugin` | `calico`、`flannel` 或 `cilium`。 |

## 软件源和镜像源

| 变量 | 说明 |
| --- | --- |
| `kubernetes_repo_base_url` | Kubernetes 官方包仓库根地址，可替换为镜像站。 |
| `docker_apt_repo_url` / `docker_rpm_repo_url` | containerd.io 的 Docker upstream 仓库地址。 |
| `helm_download_base_url` | Helm 二进制下载地址。 |
| `cilium_helm_repo_url` | Cilium chart 仓库。 |
| `containerd_registry_mirrors` | containerd 镜像仓库 mirror 配置。 |
| `proxy_env` | 需要代理访问外网时设置 HTTP/HTTPS/NO_PROXY 环境变量。 |

`containerd_registry_mirrors` 示例：

```yaml
containerd_registry_mirrors:
  docker.io:
    - "https://registry-1.docker.io"
  registry.k8s.io:
    - "https://registry.k8s.io"
```

代理示例：

```yaml
proxy_env:
  http_proxy: "http://proxy.example.com:8080"
  https_proxy: "http://proxy.example.com:8080"
  no_proxy: "127.0.0.1,localhost,10.96.0.0/12,10.244.0.0/16"
```

## 清理

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `reset_remove_packages` | `true` | 清理时是否卸载 kubelet/kubeadm/kubectl、kubernetes-cni、cri-tools。 |
| `reset_remove_containerd_data` | `true` | 清理时是否删除 containerd 镜像、容器和运行时数据。 |
| `reset_remove_downloads` | `true` | 清理时是否删除下载的 CNI manifest、Helm 临时包。 |
| `reset_remove_etc_hosts` | `true` | 是否删除 `/etc/hosts` 中 Ansible 管理的集群解析块。 |
| `reset_flush_iptables` | `true` | 清理时是否清空常见 iptables/IPVS 状态。 |
| `reset_flush_ip6tables` | `true` | 清理 iptables 时是否同时处理 ip6tables。 |
| `reset_flush_nftables` | `false` | 是否执行 `nft flush ruleset`。这个动作影响较大，默认关闭。 |
| `reset_stop_apiserver_lb` | `true` | 是否停止并禁用 HAProxy/Keepalived。 |
| `reset_remove_apiserver_lb_config` | `true` | 是否删除 HAProxy/Keepalived 配置和 HAProxy 运行目录。 |
| `reset_remove_apiserver_lb_packages` | `false` | 是否卸载 HAProxy/Keepalived 包。 |
| `reset_remove_kubernetes_repo` | `true` | 是否删除 Kubernetes apt/yum 软件源文件和 apt keyring。 |
| `reset_remove_containerd_repo` | `false` | 是否删除 Docker upstream containerd 软件源。 |
| `reset_remove_containerd_config` | `true` | 是否删除 `/etc/containerd/config.toml` 和 `/etc/crictl.yaml`。 |
| `reset_remove_containerd_package` | `false` | 是否卸载 containerd 包。 |
| `reset_remove_kubelet_config` | `true` | 是否删除 kubelet host 配置文件。 |
| `reset_remove_cni_binaries` | `false` | 是否删除 `/opt/cni/bin`。默认保留，避免误删系统包安装的 CNI 二进制。 |
| `reset_remove_pod_logs` | `true` | 是否删除 `/var/log/pods` 和 `/var/log/containers`。 |
| `reset_remove_network_links` | `true` | 是否删除常见 Kubernetes/CNI 虚拟网卡，如 `cni0`、`flannel.1`、`cilium_host`、`vxlan.calico`、`cali*`。 |
| `reset_network_link_name_regex` | 见 `all.yml` | 匹配要删除的虚拟网卡名，特殊环境可收窄范围。 |
| `reset_remove_os_tuning` | `true` | 是否删除本项目创建的内核模块和 sysctl 配置文件。 |
| `reset_start_containerd_after_reset` | `true` | 保留 containerd 数据、配置和包时，reset 后是否重新启动 containerd。 |

清理路径集中在这些列表中，特殊环境可以自行增删：

```yaml
reset_kubernetes_state_paths: []
reset_kubelet_config_paths: []
reset_containerd_config_paths: []
reset_pod_log_paths: []
reset_os_tuning_paths: []
reset_apiserver_lb_config_paths: []
```

## 断点续跑

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `resume_enabled` | `false` | 是否启用 checkpoint 跳过已完成阶段。 |
| `resume_state_dir` | `.ansible/resume` | 控制端保存断点状态的目录。 |
| `resume_cluster_key` | `cluster_name` 清洗后结果 | 断点目录名。 |
| `resume_force_stages` | `[]` | 忽略指定阶段的 checkpoint，强制重跑这些阶段。 |
| `resume_reset_state` | `false` | 本次执行前清空当前集群断点状态。 |
| `resume_clear_on_reset` | `true` | 执行 `reset.yml` 后自动清空断点状态。 |

阶段名：

```text
preflight
os_prepare
containerd
kubernetes_repo
kubernetes_packages
control_plane
cni
workers
```
# Compatibility Notes

## Kubernetes 1.35+ on cgroup v1

Kubernetes 1.35 and newer fail kubeadm preflight on cgroup v1 by default. This affects older systems such as CentOS 7 with a 3.10 kernel.

Recommended options:

1. Use a newer OS/kernel with cgroup v2, such as Rocky 9, openEuler 22.03+, Ubuntu 22.04+, or another kernel 5.x/6.x system.
2. Use Kubernetes below 1.35 on legacy cgroup v1 hosts.
3. If you explicitly accept the risk, enable legacy mode:

```yaml
allow_legacy_cgroup_v1: true
```

When enabled, the playbook:

- writes `failCgroupV1: false` into the kubeadm-managed `KubeletConfiguration`;
- adds `SystemVerification` to kubeadm `--ignore-preflight-errors`;
- keeps the behavior explicit so unsupported legacy hosts do not pass silently.

The container runtime warning about `RuntimeConfig` means the runtime should be upgraded before Kubernetes 1.37. It is a warning in Kubernetes 1.36, but should still be treated as a maintenance item.
