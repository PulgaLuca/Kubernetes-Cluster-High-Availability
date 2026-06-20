# k3s Kubernetes Cluster on Raspberry Pi

A high-availability Kubernetes cluster built on six Raspberry Pi 4 (4GB) boards, using k3s as the Kubernetes distribution. The cluster runs 3 control plane nodes and 3 worker nodes on ARM64 hardware.

## Hardware

6x Raspberry Pi 4 (4GB RAM, ARM64), connected via wired ethernet on a dedicated subnet (10.39.100.0/24). All nodes run Raspberry Pi OS Lite 64-bit (Bookworm) in headless mode.

## Stack

| Component | Role |
|---|---|
| k3s | Lightweight Kubernetes distribution |
| kube-vip | Virtual IP and control plane load balancing |
| ingress-nginx | HTTP/HTTPS ingress (DaemonSet mode) |
| Prometheus | Metrics collection |
| Grafana | Dashboards and visualization |
| k6 | Load testing |

## What was built

**High availability control plane.** Three master nodes each run the full k3s control plane stack including an embedded etcd instance. The three etcd nodes form a Raft consensus cluster, which tolerates the loss of one master without any interruption to the cluster. A second master failure causes etcd to become read-only, but running workloads on the worker nodes continue unaffected.

**Virtual IP with kube-vip.** A floating virtual IP (10.39.100.250) is managed by kube-vip, running as a static pod on each of the three master nodes. At any given time one master holds the VIP on its network interface. If that master goes down, the remaining kube-vip instances elect a new leader, which claims the VIP and announces it to the LAN via gratuitous ARP. Failover takes between 3 and 10 seconds.

**Ingress.** Nginx ingress controller is deployed as a DaemonSet with host networking, placing one Nginx instance on each worker node and binding directly to ports 80 and 443. This avoids the need for an external load balancer or a cloud provider integration.

**Monitoring.** The kube-prometheus-stack Helm chart deploys Prometheus, Grafana, Alertmanager, Node Exporter and kube-state-metrics. Grafana is accessible on port 30300. Custom PromQL queries expose per-node CPU, memory and Raspberry Pi CPU temperature.

**Autoscaling.** The Metrics Server feeds resource usage data to a Horizontal Pod Autoscaler configured on the demo API deployment, scaling between 3 and 15 replicas based on CPU utilization.

## Failure scenarios tested

**One master down.** k3s and kube-vip continue operating normally. The VIP migrates automatically and the cluster accepts reads and writes throughout.

**Two masters down.** etcd loses quorum and becomes read-only. Existing pods on worker nodes keep running. Restoring one master re-establishes quorum within 60 seconds.

**Network partition (split-brain).** Simulated by blocking traffic between one master and the other two using iptables. The isolated master loses quorum and its kubectl operations time out. The two-node partition maintains quorum, elects a new Raft leader, and claims the VIP. After the partition is removed, the previously isolated master rejoins as a follower without manual intervention.

**Worker drain.** Pods are evicted from a worker using kubectl drain and reschedule onto the remaining workers in roughly 30 seconds.
