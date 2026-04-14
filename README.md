# Training Packer/OpenTofu/Ansible -> Kubernetes (auf Proxmox)

## Agenda

  1. Vorbereitung proxmox
     * [vmbr1 für interne IP's aufsetzen in proxmox (nur Hetzner !)](proxmox/01-vmbr1-einrichten.md)
     * [API - Token einrichten](proxmox/api-token-einrichten.md)
     * [Ubuntu 24.04 - image in proxmox hochladen](proxmox/ubuntu-24-04-iso-hochladen.md)
  
  1. Überblick Installation Kubernetes (Automation) 
     * [Packer/Opentofu/Ansible - Workflow Überblick](automation/packer-opentofu-ansible-kubeadm.md)
  
  1. Packer
     * [Was ist Packer ?](packer/00-overview.md)
     * [Packer installieren](packer/01-installation.md)
     * [Packer bash-completion aktivieren](packer/02-bash-completion.md)
     * [Ubuntu-k8s-template mit Packer erstellen und hochladen](packer/03-create-ubuntu-k8s-template-and-upload.md)
  
  1. OpenTofu
     * [Was ist/kann OpenTofu, Was nicht ?](opentofu/overview-what-is-what-for.md)
     * [OpenTofu installieren auf client](opentofu/installation.md)
     * [OpenTofu bash completion](opentofu/bash-completion.md)
  
  1. ansible
     * [Was ist ansible](ansible/00-overview.md)
     * [ansible installation](ansible/installation.md)    
  
  1. Kubernetes-Installation mit Opentofu und Ansible
     * [Kubernetes (kubeadm) - Installation mit opentofu und ansible](automation/uebung-opentofu-ansible-kubeadm.md)
     * [Kubernetes HA-Cluster (kubeadm) - Installation mit opentofu und ansible](automation/uebung-opentofu-ansible-kubeadm-3-controlplanes.md)

  1. Misc
     * [HyperV und opentofu ? Alternativen ?](misc/01-hyperv-opentofu.md)
     * [IIS deployen mit packer und ansible auf HyperV](misc/02-iis-auf-windows-deployen-in-hyperv.md)
     

    
