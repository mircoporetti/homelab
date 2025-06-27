# Add New k3s Worker Node with Ansible

This project contains an Ansible playbook to install and configure a k3s worker node, joining it to an existing k3s control plane.

---

## Prerequisites

- You already have Ansible installed on your local machine.
- You already have a running k3s control plane.
- The new worker node runs with SSH access enabled.
- You have SSH key-based login set up from your control machine to the worker node.
- You know the k3s node token and control plane IP address.

---

## Files

- `worker-playbook.yml` — The Ansible playbook that installs the k3s agent on the worker node.
- `inventory.ini` — The Ansible inventory file listing the target worker node(s).

---

## Setup

1. **Configure your inventory**

Edit `inventory.ini` and add your worker node IP and SSH user, for example:

```ini
[worker_nodes]
minipc ansible_host=192.168.1.101 ansible_user=debian
```

## Run

Run the playbook:

```ini
ansible-playbook -i inventory.ini worker-playbook.yml --ask-become-pass