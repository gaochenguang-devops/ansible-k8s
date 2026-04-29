# Operations

## 部署

```bash
ansible-playbook playbooks/preflight.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

## 重复部署

同一批节点重复执行 `playbooks/cluster.yml` 是允许的。已初始化的控制面会跳过 `kubeadm init`，已加入的节点会跳过 `kubeadm join`，CNI 会通过 apply 或 Helm 的版本判断保持收敛。

## 重新部署

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

当前默认 reset 偏向彻底清理，适合把节点恢复到可重新部署的状态。默认会清理 kubeadm 状态、CNI 状态、Pod 日志、证书续期脚本、HAProxy/Keepalived 配置、Kubernetes 软件源、iptables/ip6tables/IPVS、`/etc/hosts` 管理块和断点状态。

核心变量：

```yaml
reset_remove_packages: true
reset_remove_state: true
reset_remove_containerd_data: true
reset_remove_downloads: true
reset_remove_etc_hosts: true
reset_remove_kubernetes_repo: true
reset_remove_containerd_config: true
reset_remove_network_links: true
```

如果只是快速重建集群，并希望保留 containerd 镜像缓存和已安装软件包：

```bash
ansible-playbook playbooks/reset.yml \
  -e reset_remove_packages=false \
  -e reset_remove_containerd_data=false \
  -e reset_remove_containerd_config=false \
  -e reset_remove_kubernetes_repo=false
```

需要连 containerd、HAProxy、Keepalived 包也卸载时再显式打开：

```yaml
reset_remove_containerd_package: true
reset_remove_apiserver_lb_packages: true
```

## 分段排错

```bash
ansible-playbook playbooks/cluster.yml --tags preflight
ansible-playbook playbooks/cluster.yml --tags os
ansible-playbook playbooks/cluster.yml --tags runtime
ansible-playbook playbooks/cluster.yml --tags repo
ansible-playbook playbooks/cluster.yml --tags packages
ansible-playbook playbooks/cluster.yml --tags lb
ansible-playbook playbooks/cluster.yml --tags control-plane
ansible-playbook playbooks/cluster.yml --tags cni
ansible-playbook playbooks/cluster.yml --tags workers
ansible-playbook playbooks/cluster.yml --tags certs
```

## 证书续期

完整部署完成后，`cluster.yml` 默认会在 control-plane 节点执行证书续期阶段。需要单独续期时运行：

```bash
ansible-playbook playbooks/update-certs.yml
```

默认只有证书剩余有效期低于 `kubeadm_cert_renewal_min_remaining_days` 才会重签。需要立即强制重签时运行：

```bash
ansible-playbook playbooks/update-certs.yml -e kubeadm_cert_renewal_force=true
```

查看当前证书有效期：

```bash
ansible kube_control_plane -m shell -a "kubeadm certs check-expiration"
```

## 断点续跑

部署中断后继续：

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

强制重跑指定阶段：

```bash
ansible-playbook playbooks/cluster-resume.yml -e '{"resume_force_stages":["kubernetes_packages","cni"]}'
```

执行 `playbooks/reset.yml` 后默认会清理断点状态，避免 reset 后仍跳过部署阶段。
