# Multi Node Kubernetes Cluster with Ansible

Build a highly available Kubernetes cluster from 8 AWS VMs, with kubeadm and HAProxy, using simple Ansible roles. No prechecks, no extra tooling. Each role follows the PDF step by step, so you can run it live in a session and explain every task as it runs.

Source: `multi-node-cluster.pdf`, Kubernetes Cluster Setup Guide.

Status, 2026-11-15: the AWS infra below is built and verified Running. The Kubernetes cluster itself has not been built yet, because the Ansible run needs SSH from the controller to all 7 other VMs, and that key setup is still open. See "SSH setup on the controller" below.

## What you build

8 VMs, 2 vCPU and 4 GB RAM each:

| Role | Count | Region | Hostnames |
|---|---|---|---|
| Ansible controller | 1 | Mumbai ap-south-1 | controller |
| Control plane masters | 3 | Mumbai ap-south-1 | master1, master2, master3 |
| Workers | 3 | Virginia us-east-1 | worker1, worker2, worker3 |
| HAProxy | 1 | Virginia us-east-1 | haproxy1 |

```
Mumbai ap-south-1                  Virginia us-east-1
controller                         worker1, worker2, worker3
master1, master2, master3          haproxy1 44.193.12.95:6443
        |                                  |
        ------------------ public IPs ------
```

| Item | Value |
|---|---|
| Kubernetes | 1.35.0 (kubeadm, kubelet, kubectl, cri-tools pinned) |
| CNI | Calico v3.32.1 |
| Pod network CIDR | 192.168.0.0/16 |
| Service CIDR | 10.127.44.0/24 |
| API endpoint | `<HAProxy_IP>:6443`, here 44.193.12.95:6443 |
| HAProxy balancing | roundrobin, tcp-check, inter 5s, fall 3, rise 2 |
| OS | Ubuntu Server 24.04 LTS |
| Instance type | c7i-flex.large, 2 vCPU, 4 GiB RAM |

## Part A: Set up the VMs on AWS, as done for this test

### 1. Why two regions

Mumbai Service Quotas, EC2 quota L-1216C47A, Running On-Demand Standard instances, is 8 vCPUs on this account. The 4 Mumbai VMs use 4 times 2 vCPUs, so Mumbai is full. Eight VMs at 2 vCPUs each would need 16 vCPUs. A quota increase goes through AWS Support and takes time, so this test uses 4 VMs in Mumbai and 4 in Virginia us-east-1 instead. Hyderabad ap-south-2 was checked and returned Unauthorized, the region is not enabled for this account, so it was not used.

### 2. Mumbai, ap-south-1: 4 VMs

These 4 were already running when the test started, launched 15 Nov 2026 at 17:02 IST. To build the same 4 again:

1. EC2 Console, region ap-south-1, Launch instances.
2. AMI: Ubuntu Server 24.04 LTS, ami-09d8c1e69d114dbf7, ubuntu-noble-24.04-amd64-server-20261108.
3. Instance type: c7i-flex.large.
4. Key pair: restore.
5. Network: VPC vpc-0c319615cd4c43a82, subnet subnet-05835600bd1ae51bd, AZ ap-south-1a.
6. Security group: launch-wizard-10, sg-083a5d6c03d2c3c4c. See step 4 below for its rules.
7. Number of instances: 4. Tag them with Role tags controller, master1, master2, master3. Note: in this test run all 4 were named master, with no Role tag at launch, so the controller and master split is by launch order only. Tag them properly at launch if you rebuild.
8. Launch, wait for Running and 3/3 status checks.

Result in this test:

| Role | Instance ID | Public IP | Private IP |
|---|---|---|---|
| controller | i-0cbc0a85d9a11cc09 | 13.201.54.123 | 172.31.38.36 |
| master1 | i-0fb2a19ad71c44ac8 | 65.2.34.247 | 172.31.34.236 |
| master2 | i-03af2492f3429091e | 52.66.204.75 | 172.31.43.14 |
| master3 | i-0cbdf8b3b4410777c | 13.201.51.173 | 172.31.47.63 |

### 3. Virginia, us-east-1: 4 VMs

Launched 15 Nov 2026 at 17:35 IST, same spec as Mumbai:

1. EC2 Console, region us-east-1, Launch instances.
2. AMI: Ubuntu Server 24.04 LTS, ami-08b275ac6cc5dbefb, same 20261108 Noble build, the ID differs because it is region specific.
3. Instance type: c7i-flex.large.
4. Key pair: create a new key pair named k8s-multi-node-us-east-1, RSA, .pem, and download it once. The Mumbai key pair restore does not exist in us-east-1, so you cannot reuse it there.
5. Network: default VPC vpc-0585a2e05eb6bdc1e, subnet subnet-0f61bf1a80bef0215, AZ us-east-1a.
6. Security group: create k8s-multi-node, sg-0098fa1cd38251f7f. See step 4 below for its rules.
7. Storage: 30 GiB gp3 root volume.
8. Number of instances: 4. Tag each with its Name and Role: worker1, worker2, worker3, haproxy1.
9. Launch, wait for Running and 3/3 status checks.

Result in this test:

| Role | Instance ID | Public IP | Private IP |
|---|---|---|---|
| worker1 | i-0a671c332e6e59d91 | 3.91.209.251 | 172.31.13.66 |
| worker2 | i-092981b47965b5a2c | 32.196.116.31 | 172.31.0.136 |
| worker3 | i-07c4715015239e164 | 13.222.98.232 | 172.31.15.214 |
| haproxy1 | i-086ea07f0da132093 | 44.193.12.95 | 172.31.7.106 |

### 4. Security groups, both regions

The two halves are in different VPCs in different regions, so cross-region traffic runs over public IPs. Both security groups allow the same inbound rules, for this test only:

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 22 | TCP | 0.0.0.0/0 | SSH |
| 6443 | TCP | 0.0.0.0/0 | Kubernetes API server, HAProxy frontend |
| 2379 to 2380 | TCP | 0.0.0.0/0 | etcd client API |
| 10250 | TCP | 0.0.0.0/0 | Kubelet API |
| 30000 to 32767 | TCP | 0.0.0.0/0 | NodePort service range |

Mumbai group launch-wizard-10, sg-083a5d6c03d2c3c4c, previously allowed only SSH 22. The four Kubernetes rules above were added to it, nothing was removed. Virginia group k8s-multi-node, sg-0098fa1cd38251f7f, was created with all five rules.

Warning: 0.0.0.0/0 is wide open, and it is only for this test. For anything longer lived, restrict the Kubernetes ports to the 8 public IPs of the cluster instead.

### 5. Check the infra is ready

In each region, EC2 Instances should show 4 instances, all Running, all 3/3 status checks passed. Note the public IP of each VM, you need them for the inventory in Part B.

## Part B: Run Ansible from the controller

### 1. SSH setup on the controller

This is the open step for the test run. Ansible runs from the controller VM, 13.201.54.123 in this test, and it must be able to SSH as ubuntu to all 7 other VMs, with sudo.

The Mumbai VMs use key pair restore. The Virginia VMs use key pair k8s-multi-node-us-east-1. Those are two different private keys, so the controller needs both, or a single new key distributed to all nodes.

Simplest route, do this on the controller:

```
ssh-keygen -t ed25519 -f ~/.ssh/k8s-controller -C "k8s-controller"
cat ~/.ssh/k8s-controller.pub
```

Then, for each of the other 7 VMs, SSH in once with its region key pair and append that public key to `~/.ssh/authorized_keys` on the VM:

```
ssh -i <region-key>.pem ubuntu@<VM-public-IP> "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys" < ~/.ssh/k8s-controller.pub
```

Test it from the controller, with no key prompt and no password prompt:

```
ssh -i ~/.ssh/k8s-controller ubuntu@<VM-public-IP> "hostname"
```

Do that for all 7 VMs before you run any playbook.

### 2. Install Ansible and clone the repo, on the controller

```
sudo apt update
sudo apt install -y ansible git
git clone https://github.com/vishuops0210/multi-node-cluster-role-session.git
cd multi-node-cluster-role-session
```

### 3. Fill in your inventory

Edit `inventory/hosts.yml`. In this test run it is already filled with the public IPs from Part A, and `group_vars/all.yml` already has `haproxy_ip: 44.193.12.95`. If you rebuild the infra, replace the IPs with your new public IPs, and set `haproxy_ip` to your new HAProxy public IP. Check `ansible_user` is ubuntu.

### 4. Check you can reach all nodes

```
ansible -i inventory/hosts.yml control_plane:workers:haproxy -m ping
```

Every node must answer `pong`. If one does not, fix SSH and the IP in the inventory before you run anything else. The controller itself is not an Ansible target, so it is not in that ping.

### 5. Run step 1: HAProxy, prep, and init

```
ansible-playbook -i inventory/hosts.yml playbook-init.yml
```

This runs, in order:

1. `haproxy` role on haproxy1.
2. `common` role on all 6 Kubernetes nodes.
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

This joins master2 and master3 as control plane nodes, then joins worker1, worker2, and worker3 as workers.

### 8. Verify the cluster

On master1:

```
kubectl get nodes
kubectl get pods -n kube-system
kubectl get pods -A
curl -k https://<HAProxy_IP>:6443/version
```

You should see all 6 Kubernetes nodes in `Ready` state, Calico pods in `Running` state in `kube-system`, and the API answering on the HAProxy IP.

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
| Ansible cannot SSH to a node | Check the key setup in Part B step 1, and that port 22 is open in that region's security group |

## Reset a node and start again

Run on the node itself, then rerun the matching playbook:

```
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d /var/lib/etcd
sudo systemctl restart containerd
```

On master1 only, also remove the old kubeconfig first: `rm -rf $HOME/.kube/config`.

## Tear down

When the test is done, terminate all 8 instances in both regions from the EC2 Console, or the VMs keep billing. The Virginia key pair and security groups can be deleted after the instances are gone.

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
