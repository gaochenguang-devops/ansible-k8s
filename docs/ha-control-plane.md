# HA Control Plane

For three control-plane nodes, enable the embedded API server load balancer by setting a VIP:

```yaml
kube_control_plane_vip: "192.168.200.100"
apiserver_lb_enabled: true
apiserver_lb_port: 16443
```

The playbook deploys HAProxy and Keepalived on every `kube_control_plane` node.

- Keepalived owns `kube_control_plane_vip`.
- HAProxy listens on `*:{{ apiserver_lb_port }}`.
- HAProxy forwards TCP traffic to every control-plane node on `kube_apiserver_port`, usually `6443`.
- `kube_control_plane_endpoint` becomes `VIP:16443`.

The LB port is intentionally different from `6443`. If HAProxy listens on `6443` on the control-plane nodes, it can conflict with the local kube-apiserver static pod.

Useful commands:

```bash
ansible-playbook playbooks/cluster.yml --tags lb
ansible kube_control_plane -m shell -a "systemctl status haproxy keepalived --no-pager"
ansible kube_control_plane -m shell -a "ip addr | grep 192.168.200.100"
ansible kube_control_plane -m shell -a "ss -lntp | grep 16443"
```

If VRRP multicast is blocked in your network, enable unicast mode:

```yaml
apiserver_lb_keepalived_unicast: true
```
