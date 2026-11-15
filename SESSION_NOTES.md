# Multi node cluster: steps noted from the PDF

Source: multi-node-cluster.pdf, Kubernetes Cluster Setup Guide.
Topology from the PDF: 3 master nodes, 2 worker nodes, 1 HAProxy node.
Kubernetes 1.35.0, Calico v3.32.1, pod CIDR 192.168.0.0/16, service CIDR 10.127.44.0/24.

## What runs where

| PDF phase | Control plane nodes | Worker nodes | HAProxy node |
|---|---|---|---|
| 1. Ports | 6443 API, 2379 to 2380 etcd, 10250 kubelet | 10250 kubelet, 30000 to 32767 NodePort | 6443 frontend |
| 2. Hostname, /etc/hosts, apt update | Yes | Yes | No, only HAProxy install later |
| 3. Kernel modules, sysctl | Yes | Yes | No |
| 4. containerd, SystemdCgroup | Yes | Yes | No |
| 5. kubeadm, kubelet, kubectl pinned 1.35.0 | Yes | Yes | No |
| 6. kubeadm init, Calico, join | First master inits, other two join as control plane | Join as workers | No |
| 7. HAProxy | No | No | Yes, backend is the 3 masters |

## Control plane nodes, step by step

Common prep, same as workers, Phases 2 to 5:
1. apt update, set hostname to master1, master2, master3.
2. Add all 6 nodes to /etc/hosts.
3. modprobe overlay and br_netfilter, persist in /etc/modules-load.d/containerd.conf.
4. Write /etc/sysctl.d/kubernetes.conf (bridge nf call iptables, bridge nf call ip6tables, ip_forward), run sysctl --system.
5. Install containerd, generate /etc/containerd/config.toml, set SystemdCgroup to true, start and enable containerd.
6. Add Kubernetes repo for v1.35, install kubeadm, kubelet, kubectl, cri-tools pinned to 1.35.0-1.1, hold the packages.

First master only, Phase 6:
7. kubeadm init --control-plane-endpoint="<HAProxy_IP>:6443" --upload-certs --pod-network-cidr=192.168.0.0/16 --service-cidr=10.127.44.0/24
8. Copy /etc/kubernetes/admin.conf to $HOME/.kube/config.
9. kubectl apply -f calico.yaml (Calico v3.32.1).
10. Save the two join commands printed by init. They carry the token, the ca cert hash, and the certificate key.

Other two masters, Phase 6:
11. kubeadm join <HAProxy_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <cert-key>

## Worker nodes, step by step

1. Same common prep as control plane, Phases 2 to 5: hostname, /etc/hosts, kernel modules, sysctl, containerd with SystemdCgroup true, pinned kubeadm, kubelet, kubectl.
2. No init, no Calico, no etcd. Workers only join.
3. kubeadm join <HAProxy_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

## HAProxy node, step by step

1. apt install haproxy.
2. Back up /etc/haproxy/haproxy.cfg to haproxy.cfg.backup.
3. Frontend kubernetes, bind *:6443, mode tcp, default backend kubernetes-api.
4. Backend kubernetes-api, mode tcp, balance roundrobin, option tcp-check, one server line per master: server master1 <IP1>:6443 check inter 5s fall 3 rise 2, same for master2 and master3.
5. systemctl restart haproxy.

## Verification, from the PDF

1. kubectl get nodes: every master and worker in Ready state.
2. kubectl get pods -n kube-system: Calico pods Running.
3. API reachable on the HAProxy IP, port 6443, from all nodes.

## Two things to call out in the session

1. Run order is not PDF order. The PDF lists HAProxy as Phase 7, last. But kubeadm init points at the HAProxy IP, and the other nodes join through it, so HAProxy must be up before init and join. The playbooks run HAProxy first.
2. Join tokens expire after 24 hours. If a token lapses, generate a new one on master1 with kubeadm token create --print-join-command.

## Small gaps in the PDF that the roles fill in

1. The PDF says enable SystemdCgroup, but its containerd commands never edit config.toml. The common role sets SystemdCgroup to true.
2. The PDF pipes the repo key into /etc/apt/keyrings/ without creating that directory. The common role creates it first.
