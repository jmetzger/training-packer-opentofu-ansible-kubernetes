# Überblick 


Packer (proxmox-iso) → VM-Template (ID 9000) mit containerd + kubeadm
    ↓
OpenTofu (bpg/proxmox) → klont 2 VMs (cp1 + w1) mit statischen IPs
                        → schreibt inventory.ini via local_file
    ↓
tofu apply fertig
    ↓
ansible-playbook -i inventory.ini cluster.yml
    ├── Role: kubeadm-init  → kubeadm init auf cp1, register join-command
    ├── Role: kubeadm-join  → join-command auf w1 via hostvars
    └── Role: cni           → Cilium/Flannel auf cp1 deployen
    ↓
Cluster ready
