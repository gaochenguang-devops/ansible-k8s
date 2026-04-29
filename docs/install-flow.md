# 安装流程

本文按 playbook 实际执行顺序梳理 Kubernetes 集群安装流程，用来做上线前检查、分阶段排错和断点续跑。

## 总体入口

完整部署：

```bash
ansible-playbook playbooks/cluster.yml
```

开启断点续跑：

```bash
ansible-playbook playbooks/cluster-resume.yml
```

只做预检查：

```bash
ansible-playbook playbooks/preflight.yml
```

部署后验证：

```bash
ansible-playbook playbooks/verify.yml
```

## 执行阶段

| 顺序 | 阶段 | 作用 | 常用 tag |
| --- | --- | --- | --- |
| 1 | `preflight` | 校验 inventory、系统、版本、CNI、VIP、cgroup 兼容性 | `preflight` |
| 2 | `os_prepare` | 安装基础包，关闭 swap，配置内核模块、sysctl、SELinux、firewalld、hostname、`/etc/hosts` | `os`, `prepare` |
| 3 | `apiserver_lb` | 多 master 时部署 HAProxy + Keepalived，提供 `VIP:16443` | `lb`, `apiserver-lb`, `haproxy`, `keepalived` |
| 4 | `containerd` | 安装 containerd，写入 `/etc/containerd/config.toml` 和 `/etc/crictl.yaml` | `runtime`, `containerd` |
| 5 | `kubernetes_repo` | 配置 Kubernetes DEB/RPM 软件源 | `repo` |
| 6 | `kubernetes_packages` | 安装 kubelet、kubeadm、kubectl，并配置 kubelet node IP | `packages` |
| 7 | `control_plane` | 初始化第一个 master，并把其他 master 加入控制面 | `kubeadm`, `control-plane` |
| 8 | `cni` | 安装 Calico、Flannel 或 Cilium | `cni` |
| 9 | `workers` | 把 worker 节点加入集群 | `kubeadm`, `workers` |
| 10 | `certs` | 更新 kubeadm 主要证书有效期，默认 100 年 | `certs`, `certificates` |

## 推荐新装流程

1. 修改 `inventory/sample/hosts.yml`。
2. 修改 `playbooks/group_vars/all.yml`。
3. 检查 SSH：

```bash
ansible k8s_cluster -m ping
```

4. 做预检查：

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

## 分阶段排错

如果失败点很明确，可以只跑某个阶段：

```bash
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

完整部署会自动执行 `certs` 阶段。后续需要单独检查并续期时，可以直接运行：

```bash
ansible-playbook playbooks/update-certs.yml
```

如果已经是 100 年证书，默认会跳过重签。需要强制重新生成时：

```bash
ansible-playbook playbooks/update-certs.yml -e kubeadm_cert_renewal_force=true
```

## 断点续跑

断点续跑会在控制端记录每个阶段的成功状态：

```text
.ansible/resume/<cluster_name>/
```

中断后继续：

```bash
ansible-playbook playbooks/cluster-resume.yml
```

查看断点：

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

## 重装流程

彻底重装建议先 reset，再重新部署：

```bash
ansible-playbook playbooks/reset.yml
ansible-playbook playbooks/cluster.yml
ansible-playbook playbooks/verify.yml
```

`reset.yml` 会清理目标节点上的 Kubernetes、CNI、HAProxy/Keepalived、iptables/IPVS、软件源、日志和下载物等状态。执行 reset 后默认也会清理断点，避免下一次部署误跳过阶段。

如果只是想快速重装并保留 containerd 镜像缓存和配置，可以临时覆盖：

```bash
ansible-playbook playbooks/reset.yml \
  -e reset_remove_containerd_data=false \
  -e reset_remove_containerd_config=false
```

## 扩容和缩容

新增节点时，先把节点加入 inventory，再运行扩容：

```bash
# 新增 worker
ansible-playbook playbooks/scale-up.yml --tags preflight,os,runtime,packages,workers

# 新增 master
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
```

删除节点时，先执行删除 playbook，成功后再从 inventory 移除：

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-2"]}'
```

更多细节见 [scale.md](scale.md)。
