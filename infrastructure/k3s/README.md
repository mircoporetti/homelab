# K3s with Ansible

This project contains an Ansible playbook to install and configure a k3s bare metal cluster with a control plane and multiple worker nodes.

---

## Prerequisites

- Ansible installed on your local machine.
- The nodes run with SSH access enabled.
- You have SSH key-based login set up from your control machine to the worker node.
- You know the k3s node token and nodes IP addresses.

---

## Files

- `control-plane-playbook.yml` — The Ansible playbook that installs the k3s server on the control plane node.
- `worker-playbook.yml` — The Ansible playbook that installs the k3s agent on the worker nodes.
- `inventory.ini` — The Ansible inventory file listing the target nodes.

---

## Setup

1. **Configure your inventory**

Edit `inventory.ini` and add your control plane and worker node IPs and SSH users, for example:

```ini
[control_plane_nodes]
mycontrolplane ansible_host=192.168.1.100 ansible_user=user ansible_playbook=control-plane-playbook.yml

[worker_nodes]
worker1 ansible_host=192.168.1.101 ansible_user=user node_label=worker-1 ansible_playbook=worker-playbook.yml
worker2 ansible_host=192.168.1.102 ansible_user=user node_label=worker-2 ansible_playbook=worker-playbook.yml
```

## Run

Run the playbook:

```ini
ansible-playbook -i inventory.ini --ask-become-pass
```