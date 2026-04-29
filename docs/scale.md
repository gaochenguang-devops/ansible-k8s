# Scale Operations

## Add Control-Plane Nodes

1. Add the new host to `inventory/.../hosts.yml` under `kube_control_plane`.
2. Keep at least one existing healthy control-plane node in the same inventory.
3. If this is a multi-control-plane cluster, set a VIP:

```yaml
kube_control_plane_vip: "192.168.200.100"
apiserver_lb_enabled: true
apiserver_lb_port: 16443
```

4. Run:

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,lb,runtime,packages,control-plane,certs
ansible-playbook playbooks/verify.yml
```

The playbook updates HAProxy/Keepalived, installs runtime and Kubernetes packages, then joins unjoined control-plane nodes.

When a cluster was originally created as a single-control-plane cluster, kubeadm stores the first master address as the cluster endpoint. During control-plane scale-up, this project migrates that stored endpoint to `kube_control_plane_vip:apiserver_lb_port`, refreshes kubeconfigs, and regenerates apiserver certificates when the VIP is missing from the certificate SANs. This can briefly restart kube-apiserver static pods, so run single-master to HA migration in a maintenance window.

## Add Worker Nodes

1. Add the new host to `inventory/.../hosts.yml` under `kube_node`.
2. Run:

```bash
ansible-playbook playbooks/scale-up.yml --tags preflight,os,runtime,packages,workers
ansible-playbook playbooks/verify.yml
```

Existing joined nodes are skipped by kubeadm checks, so the playbook is safe to repeat.

## Remove Worker Nodes

Run before deleting the host from inventory:

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-2"]}'
```

The playbook drains the node, resets Kubernetes state on the target host, and deletes the Node object.

## Remove Control-Plane Nodes

Run before deleting the host from inventory:

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-master-3"]}'
```

At least one control-plane node must remain. For embedded HA, the playbook stops HAProxy/Keepalived on the removed node, removes it from remaining HAProxy backends, drains and resets it, then deletes the Node object.

## Remove Multiple Nodes

```bash
ansible-playbook playbooks/remove-node.yml -e '{"remove_nodes":["k8s-worker-1","k8s-worker-2"]}'
```

You can also pass a comma-separated string:

```bash
ansible-playbook playbooks/remove-node.yml -e remove_nodes=k8s-worker-1,k8s-worker-2
```

Useful variables:

```yaml
remove_node_drain: true
remove_node_drain_timeout: "10m"
remove_node_reset: true
remove_node_delete_from_cluster: true
remove_node_ignore_missing: true
```
