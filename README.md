# Multi Node Kubernetes Cluster with Ansible

Build a highly available Kubernetes cluster from 6 VMs, with kubeadm and HAProxy, using simple Ansible roles. No prechecks, no extra tooling. Each role follows the PDF step by step, so you can run it live in a session and explain every task as it runs.

Source: `multi-node-cluster.pdf`, Kubernetes Cluster Setup Guide.

## What you build

```
                HAProxy (haproxy1)
                192.168.x.x:6443
                        |
        --------------------------------
        |              |               |
     master1        master2         master3
   control plane  control plane   control plane
        |              |               |
     worker1        worker2
```

| Item | Value from the PDF |
|---|---|
| Masters | 3 |
| Workers | 2 |
| Load balancer | 1 HAProxy node |
| Kubernetes | 1.35.0 (kubeadm, kubelet, kubectl, cri-tools pinned) |
| CNI | Calico v3.32.1 |
| Pod network CIDR | 192.168.0.0/16 |
| Service CIDR | 10.127.44.0/24 |
| API endpoint | `<HAProxy_IP>:6443` |
| HAProxy balancing | roundrobin, tcp-check, inter 5s, fall 3, rise 2 |

## Prerequisites

1. 6 VMs with Ubuntu or Debian, apt, systemd, and sudo. The PDF uses apt and hostnamectl throughout.
2. SSH access from your Ansible controller to all 6 VMs, with a user that can sudo without a password prompt, or with `--ask-become-pass` at run time.
3. Ansible installed on the controller machine only. The nodes need no Ansible.
4. The VMs can reach each other on the ports below, and can reach the internet to fetch apt packages and the Calico manifest.

| Node type | Ports | Purpose |
|---|---|---|
| Master | 6443 | Kubernetes API server |
| Master | 2379 to 2380 | etcd client API |
| Master and worker | 10250 | Kubelet API |
| Worker | 30000 to 32767 | NodePort service range |
| HAProxy | 6443 | API frontend |

## Quick start

### 1. Clone the repo

```
git clone https://github.com/vishuops0210/multi-node-cluster-role-session.git
cd multi-node-cluster-role-session
```

### 2. Fill in your inventory

Edit `inventory/hosts.yml`. Replace every `CHANGE_ME` with the real IP of that VM. Check `ansible_user` matches the SSH user on your VMs.

```
all:
  children:
    control_plane:
      hosts:
        master1:
          ansible_host: 192.168.1.11
        master2:
          ansible_host: 192.168.1.12
        master3:
          ansible_host: 192.168.1.13
    workers:
      hosts:
        worker1:
          ansible_host: 192.168.1.14
        worker2:
          ansible_host: 192.168.1.15
    haproxy:
      hosts:
        haproxy1:
          ansible_host: 192.168.1.10
```

### 3. Fill in the HAProxy IP

Edit `group_vars/all.yml` and set `haproxy_ip` to the HAProxy VM IP. This is the IP every node uses as the control plane endpoint. Leave `group_vars/join.yml` as `CHANGE_ME` for now.

### 4. Check you can reach all nodes

```
ansible -i inventory/hosts.yml all -m ping
```

Every node must answer `pong`. If one does not, fix SSH and the IP in the inventory before you run anything else.

### 5. Run step 1: HAProxy, prep, and init

```
ansible-playbook -i inventory/hosts.yml playbook-init.yml
```

This runs, in order:

1. `haproxy` role on haproxy1.
2. `common` role on all 5 Kubernetes nodes.
3. `control-plane` role on master1 only: `kubeadm init`, kubeconfig setup, Calico install.

Watch the output on master1. `kubeadm init` prints two join commands at the end. Copy these three values from them: the token, the discovery token ca cert hash, and the certificate key.

### 6. Fill in the join values

Edit `group_vars/join.yml` and paste the values from step 5:

```
kubeadm_token: abcdef.0123456789abcdef
kubeadm_ca_hash: "sha256:0123456789abcdef..."
kubeadm_cert_key: 0123456789abcdef...
```

### 7. Run step 2: join the rest

```
ansible-playbook -i inventory/hosts.yml playbook-join.yml
```

This joins master2 and master3 as control plane nodes, then joins worker1 and worker2 as workers.

### 8. Verify the cluster

On master1:

```
kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -A
curl -k https://<HAProxy_IP>:6443/version
```

You should see all 5 Kubernetes nodes in `Ready` state, Calico pods in `Running` state in `kube-system`, and the API answering on the HAProxy IP.

## Roles

| Role | Runs on | PDF phase | What it does |
|---|---|---|---|
| `common` | All masters and workers | 2 to 5 | apt update, hostname, /etc/hosts, kernel modules overlay and br_netfilter, sysctl, containerd with SystemdCgroup true, Kubernetes repo for v1.35, pinned kubeadm, kubelet, kubectl, cri-tools 1.35.0-1.1, package hold |
| `control-plane` | Masters | 6 | First master: kubeadm init with the HAProxy endpoint, upload-certs, pod and service CIDRs, kubeconfig, Calico v3.32.1. Other masters: kubeadm join with --control-plane and the certificate key |
| `worker` | Workers | 6 | kubeadm join only. No init, no Calico, no etcd |
| `haproxy` | HAProxy node | 7 | Install HAProxy, back up the stock config, render haproxy.cfg with one server line per master, restart HAProxy |

Full step notes per node type are in `SESSION_NOTES.md`.

## Why two playbooks

The join values only exist after init prints them. A single playbook would need to parse init output or guess the values. Two runs keep each role readable line by line for the session, and match the PDF flow: init prints the join commands, you run them on the other nodes.

## Why HAProxy runs first

The PDF lists HAProxy as Phase 7, last. But `kubeadm init` points at the HAProxy IP, and the other nodes join through it, so HAProxy must be up before init and join. `playbook-init.yml` runs the HAProxy role first for that reason. Call this out in the session, it is the one place the run order differs from the PDF order.

## Two small gaps in the PDF that the roles fill in

1. The PDF says enable SystemdCgroup, but its containerd commands never edit `config.toml`. The `common` role sets `SystemdCgroup = true`.
2. The PDF pipes the repo key into `/etc/apt/keyrings/` without creating that directory. The `common` role creates it first.

## Troubleshooting

| Symptom | Check first |
|---|---|
| Join fails with an invalid or expired token | Tokens expire after 24 hours. On master1 run `kubeadm token create --print-join-command`, update `group_vars/join.yml`, rerun `playbook-join.yml` |
| A node stays `NotReady` | On master1 run `kubectl describe node <name>` and `kubectl get pods -n kube-system`. A missing or failing Calico pod is the usual cause |
| API not reachable on the HAProxy IP | On haproxy1 run `systemctl status haproxy` and `cat /etc/haproxy/haproxy.cfg`. Check the backend server lines carry the real master IPs, and that port 6443 is open |
| `kubeadm init` fails | Rerun it only after a reset, see below. Check that containerd is running and SystemdCgroup is true in `/etc/containerd/config.toml` |
| Rerunning init or join does nothing | The roles use `creates:` guards on `/etc/kubernetes/admin.conf` and `/etc/kubernetes/kubelet.conf`, so a node that already joined is skipped. That is intentional |

## Reset a node and start again

Run on the node itself, then rerun the matching playbook:

```
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd
sudo systemctl restart containerd
```

On master1 only, also remove the old kubeconfig first: `rm -rf $HOME/.kube/config`.

## Layout

```
ansible.cfg
inventory/hosts.yml
group_vars/
  all.yml
  join.yml
playbook-init.yml
playbook-join.yml
SESSION_NOTES.md
roles/
  common/
    tasks/main.yml
    templates/hosts.j2
  control-plane/
    tasks/main.yml
  worker/
    tasks/main.yml
  haproxy/
    tasks/main.yml
    templates/haproxy.cfg.j2
```
