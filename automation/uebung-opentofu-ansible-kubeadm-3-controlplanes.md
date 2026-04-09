# Übung: HA Kubernetes-Cluster mit OpenTofu und Ansible auf Proxmox ( 3 Controlplanes )

## Ziel

Du erstellst mit OpenTofu einen hochverfügbaren Kubernetes-Cluster auf Proxmox:
- 2 HAProxy/Keepalived Nodes (Load Balancer mit VIP)
- 3 Control Plane Nodes
- N Worker Nodes

OpenTofu generiert automatisch das Ansible-Inventory. Ansible übernimmt das komplette Bootstrapping.

## Voraussetzungen

- Proxmox-Server mit fertigem VM-Template (VM-ID `900<TLN_NR>`, erstellt mit Packer)
  - Im Template bereits vorbereitet: containerd, kubelet, kubeadm, kubectl, Kernel-Module, Sysctl, Swap deaktiviert
- OpenTofu installiert (`tofu version`)
- Ansible installiert (`ansible --version`)
- SSH-Keypair vorhanden (`ssh-keygen -t ed25519` falls nicht)

## Architektur

```
                    ┌─────────────┐
                    │     VIP     │
                    │ 10.10.10.1X0│
                    └──────┬──────┘
                           │
              ┌────────────┴────────────┐
              │                         │
      ┌───────┴───────┐       ┌────────┴───────┐
      │   HAProxy 1   │       │   HAProxy 2    │
      │  + Keepalived  │       │  + Keepalived  │
      │ 10.10.10.1X1  │       │ 10.10.10.1X2   │
      └───────┬───────┘       └────────┬───────┘
              │    Round-Robin :6443    │
       ┌──────┼──────────┬─────────────┘
       │      │          │
  ┌────┴──┐ ┌─┴───┐ ┌───┴──┐
  │ CP 1  │ │ CP 2│ │ CP 3 │
  │  .1X3 │ │ .1X4│ │ .1X5 │
  └───────┘ └─────┘ └──────┘
       │
  ┌────┴────┐
  │Worker(s)│
  │  .1X6+  │
  └─────────┘
```

## Variablen für die Übung

| Variable | Beispielwert | Beschreibung |
|----------|-------------|--------------|
| `TLN_NR` | `1` | Teilnehmernummer (1, 2, 3, 4, 5, ...) |
| `PROXMOX_TOKEN_ID` | `root@pam!automation` | Proxmox API Token ID |
| `PROXMOX_TOKEN_SECRET` | `xxxxxxxx-xxxx-...` | Proxmox API Token Secret |

**VM-ID-Schema (Beispiel TLN_NR=1):**

| Rolle | Formel | VM-ID |
|-------|--------|-------|
| HAProxy 1 | `3${TLN_NR}1` | **311** |
| HAProxy 2 | `3${TLN_NR}2` | **312** |
| CP 1 | `3${TLN_NR}3` | **313** |
| CP 2 | `3${TLN_NR}4` | **314** |
| CP 3 | `3${TLN_NR}5` | **315** |
| Worker 1 | `3${TLN_NR}6` | **316** |

**IP-Schema (Beispiele):**

| TLN_NR | VIP | HAProxy 1 | HAProxy 2 | CP 1 | CP 2 | CP 3 | Worker 1 |
|--------|-----|-----------|-----------|------|------|------|----------|
| 1 | .110 | .111 | .112 | .113 | .114 | .115 | .116 |
| 2 | .120 | .121 | .122 | .123 | .124 | .125 | .126 |

---

## Schritt 1: SSH-Keypair erstellen

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -N ""
```

---

## Schritt 2: Variablen setzen

```bash
export TLN_NR="1"
export PROXMOX_TOKEN_ID='root@pam!automation'
export PROXMOX_TOKEN_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

---

## Schritt 3: Projektstruktur anlegen

```bash
mkdir -p ~/opentofu-k8s/templates
mkdir -p ~/opentofu-k8s/inventory
mkdir -p ~/opentofu-k8s/ansible
cd ~/opentofu-k8s
```

---

## Schritt 4: Provider und VMs (main.tf)

```bash
cat > main.tf <<'EOF'
terraform {
  required_providers {
    proxmox = {
      source  = "bpg/proxmox"
      version = "~> 0.78"
    }
  }
}

provider "proxmox" {
  endpoint  = var.proxmox_url
  api_token = "${var.proxmox_token_id}=${var.proxmox_token_secret}"
  insecure  = true
}

locals {
  ip_prefix = join(".", slice(split(".", var.vip), 0, 3))
  vip_last  = tonumber(split(".", var.vip)[3])
}

# --- HAProxy/Keepalived Nodes: VM-ID = 3<TLN_NR>1, 3<TLN_NR>2 ---
resource "proxmox_virtual_environment_vm" "haproxy" {
  count     = 2
  name      = "k8s-tln${var.tln_nr}-lb${count.index + 1}"
  node_name = var.proxmox_node
  vm_id     = tonumber("3${var.tln_nr}${count.index + 1}")

  clone {
    vm_id = var.template_vm_id
    full  = true
  }

  cpu {
    cores = 1
    type  = "host"
  }

  memory {
    dedicated = 2048
  }

  disk {
    datastore_id = var.datastore
    interface    = "scsi0"
    size         = 20
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    datastore_id = var.datastore
    ip_config {
      ipv4 {
        address = "${local.ip_prefix}.${local.vip_last + count.index + 1}/24"
        gateway = var.gateway
      }
    }
    dns {
      servers = [var.dns_server]
    }
    user_account {
      keys     = [file(var.ssh_public_key_path)]
      username = var.vm_user
    }
  }
}

# --- Control Plane Nodes: VM-ID = 3<TLN_NR>3, 3<TLN_NR>4, 3<TLN_NR>5 ---
resource "proxmox_virtual_environment_vm" "controlplane" {
  count     = var.cp_count
  name      = "k8s-tln${var.tln_nr}-cp${count.index + 1}"
  node_name = var.proxmox_node
  vm_id     = tonumber("3${var.tln_nr}${count.index + 3}")

  clone {
    vm_id = var.template_vm_id
    full  = true
  }

  cpu {
    cores = 2
    type  = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = var.datastore
    interface    = "scsi0"
    size         = 40
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    datastore_id = var.datastore
    ip_config {
      ipv4 {
        address = "${local.ip_prefix}.${local.vip_last + count.index + 3}/24"
        gateway = var.gateway
      }
    }
    dns {
      servers = [var.dns_server]
    }
    user_account {
      keys     = [file(var.ssh_public_key_path)]
      username = var.vm_user
    }
  }
}

# --- Worker Nodes: VM-ID = 3<TLN_NR>6, 3<TLN_NR>7, ... ---
resource "proxmox_virtual_environment_vm" "worker" {
  count     = var.worker_count
  name      = "k8s-tln${var.tln_nr}-worker${count.index + 1}"
  node_name = var.proxmox_node
  vm_id     = tonumber("3${var.tln_nr}${count.index + 3 + var.cp_count}")

  clone {
    vm_id = var.template_vm_id
    full  = true
  }

  cpu {
    cores = 2
    type  = "host"
  }

  memory {
    dedicated = 4096
  }

  disk {
    datastore_id = var.datastore
    interface    = "scsi0"
    size         = 40
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    datastore_id = var.datastore
    ip_config {
      ipv4 {
        address = "${local.ip_prefix}.${local.vip_last + count.index + 3 + var.cp_count}/24"
        gateway = var.gateway
      }
    }
    dns {
      servers = [var.dns_server]
    }
    user_account {
      keys     = [file(var.ssh_public_key_path)]
      username = var.vm_user
    }
  }
}
EOF
```

---

## Schritt 5: Variablen (variables.tf)

```bash
cat > variables.tf <<'EOF'
variable "tln_nr" {
  description = "Teilnehmernummer (1, 2, 3, ...)"
  type        = string
}

variable "proxmox_url" {
  type = string
}

variable "proxmox_token_id" {
  description = "Proxmox API Token ID (z.B. root@pam!automation)"
  type        = string
}

variable "proxmox_token_secret" {
  description = "Proxmox API Token Secret"
  type        = string
  sensitive   = true
}

variable "proxmox_node" {
  type    = string
  default = "pve"
}

variable "template_vm_id" {
  description = "VM-ID des Packer-Templates (900<TLN_NR>)"
  type        = number
}

variable "datastore" {
  type    = string
  default = "local-lvm"
}

variable "cp_count" {
  description = "Anzahl Control Plane Nodes"
  type        = number
  default     = 3
}

variable "worker_count" {
  description = "Anzahl Worker Nodes"
  type        = number
  default     = 1
}

variable "vip" {
  description = "Virtual IP für den API-Server (HAProxy/Keepalived)"
  type        = string
}

variable "gateway" {
  type = string
}

variable "dns_server" {
  type = string
}

variable "vm_user" {
  type    = string
  default = "trainee"
}

variable "bridge" {
  description = "Proxmox Network Bridge"
  type        = string
  default     = "vmbr0"
}

variable "ssh_public_key_path" {
  type    = string
  default = "~/.ssh/id_ed25519.pub"
}
EOF
```

---

## Schritt 6: Inventory-Template (templates/inventory.tpl)

```bash
cat > templates/inventory.tpl <<'EOF'
[loadbalancer]
%{ for lb in lbs ~}
${lb.name} ansible_host=${lb.ip}
%{ endfor ~}

[controlplane]
%{ for cp in cps ~}
${cp.name} ansible_host=${cp.ip}
%{ endfor ~}

[workers]
%{ for w in workers ~}
${w.name} ansible_host=${w.ip}
%{ endfor ~}

[k8s_cluster:children]
controlplane
workers

[all:vars]
ansible_user=${vm_user}
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
vip=${vip}
EOF
```

---

## Schritt 7: Outputs und Inventory-Generierung (outputs.tf)

```bash
cat > outputs.tf <<'EOF'
resource "local_file" "ansible_inventory" {
  content = templatefile("${path.module}/templates/inventory.tpl", {
    lbs = [
      for i, vm in proxmox_virtual_environment_vm.haproxy : {
        name = vm.name
        ip   = "${local.ip_prefix}.${local.vip_last + i + 1}"
      }
    ]
    cps = [
      for i, vm in proxmox_virtual_environment_vm.controlplane : {
        name = vm.name
        ip   = "${local.ip_prefix}.${local.vip_last + i + 3}"
      }
    ]
    workers = [
      for i, vm in proxmox_virtual_environment_vm.worker : {
        name = vm.name
        ip   = "${local.ip_prefix}.${local.vip_last + i + 3 + var.cp_count}"
      }
    ]
    vm_user = var.vm_user
    vip     = var.vip
  })
  filename        = "${path.module}/inventory/hosts.ini"
  file_permission = "0644"

  depends_on = [
    proxmox_virtual_environment_vm.haproxy,
    proxmox_virtual_environment_vm.controlplane,
    proxmox_virtual_environment_vm.worker
  ]
}

output "vip" {
  value = var.vip
}

output "haproxy_ips" {
  value = [
    for i in range(2) :
    "${local.ip_prefix}.${local.vip_last + i + 1}"
  ]
}

output "controlplane_ips" {
  value = [
    for i in range(var.cp_count) :
    "${local.ip_prefix}.${local.vip_last + i + 3}"
  ]
}

output "worker_ips" {
  value = [
    for i in range(var.worker_count) :
    "${local.ip_prefix}.${local.vip_last + i + 3 + var.cp_count}"
  ]
}

output "inventory_path" {
  value = local_file.ansible_inventory.filename
}
EOF
```

---

## Schritt 8: tfvars generieren

```bash
cat > terraform.tfvars <<EOF
tln_nr           = "${TLN_NR}"
proxmox_url      = "https://176.9.38.183:8006"
proxmox_token_id     = "${PROXMOX_TOKEN_ID}"
proxmox_token_secret = "${PROXMOX_TOKEN_SECRET}"
proxmox_node     = "pve"
datastore        = "local"
template_vm_id   = 900${TLN_NR}

cp_count        = 3
worker_count    = 1
vip             = "10.10.10.1${TLN_NR}0"
gateway         = "10.10.10.1"
dns_server      = "8.8.8.8"

vm_user             = "trainee"
bridge              = "vmbr1"
ssh_public_key_path = "~/.ssh/id_ed25519.pub"
EOF
```

**Hinweis:** `terraform.tfvars` enthält das API-Token — in `.gitignore` aufnehmen!

---

## Schritt 9: Ansible-Playbook (ansible/site.yml)

```bash
cat > ansible/site.yml <<'EOF'
---
# =============================================================
# 1) HAProxy + Keepalived auf den Load Balancer Nodes
# =============================================================
- name: HAProxy und Keepalived einrichten
  hosts: loadbalancer
  become: true
  vars:
    cp_hosts: "{{ groups['controlplane'] | map('extract', hostvars, 'ansible_host') | list }}"
  tasks:

    - name: HAProxy und Keepalived installieren
      apt:
        name:
          - haproxy
          - keepalived
        state: present
        update_cache: true

    - name: HAProxy konfigurieren
      copy:
        dest: /etc/haproxy/haproxy.cfg
        content: |
          global
              log /dev/log local0 warning
              chroot /var/lib/haproxy
              pidfile /var/run/haproxy.pid
              maxconn 4000
              user haproxy
              group haproxy
              daemon
              stats socket /var/lib/haproxy/stats

          defaults
              log     global
              option  tcplog
              option  dontlognull
              timeout connect 5000
              timeout client  50000
              timeout server  50000

          frontend k8s-api
              bind *:6443
              mode tcp
              default_backend k8s-api

          backend k8s-api
              mode tcp
              option tcp-check
              balance roundrobin
              default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
          {% for host in cp_hosts %}
              server cp{{ loop.index }} {{ host }}:6443 check
          {% endfor %}
      notify: haproxy restart

    - name: Keepalived Health-Check Script erstellen
      copy:
        dest: /etc/keepalived/check_apiserver.sh
        mode: '0755'
        content: |
          #!/bin/sh
          /usr/bin/killall -0 haproxy

    - name: Keepalived konfigurieren
      copy:
        dest: /etc/keepalived/keepalived.conf
        content: |
          vrrp_script chk_haproxy {
              script "/etc/keepalived/check_apiserver.sh"
              interval 3
              weight -2
              fall 10
              rise 2
          }

          vrrp_instance haproxy-vip {
              state {{ 'MASTER' if inventory_hostname == groups['loadbalancer'][0] else 'BACKUP' }}
              priority {{ '200' if inventory_hostname == groups['loadbalancer'][0] else '100' }}
              interface eth0
              virtual_router_id 51
              advert_int 1
              authentication {
                  auth_type PASS
                  auth_pass kubernetes
              }
              virtual_ipaddress {
                  {{ vip }}
              }
              track_script {
                  chk_haproxy
              }
          }
      notify: keepalived restart

    - name: Services aktivieren
      systemd:
        name: "{{ item }}"
        enabled: true
        state: started
      loop:
        - haproxy
        - keepalived

  handlers:
    - name: haproxy restart
      systemd:
        name: haproxy
        state: restarted

    - name: keepalived restart
      systemd:
        name: keepalived
        state: restarted

# =============================================================
# 2) Ersten Control Plane Node initialisieren
# =============================================================
- name: Ersten Control Plane initialisieren
  hosts: controlplane[0]
  become: true
  tasks:

    - name: kubeadm init mit HA-Endpoint
      command: >
        kubeadm init
        --control-plane-endpoint "{{ vip }}:6443"
        --upload-certs
        --pod-network-cidr=192.168.0.0/16
        --apiserver-advertise-address={{ ansible_host }}
      args:
        creates: /etc/kubernetes/admin.conf

    - name: kubeconfig für User einrichten
      shell: |
        mkdir -p /home/{{ ansible_user }}/.kube
        cp /etc/kubernetes/admin.conf /home/{{ ansible_user }}/.kube/config
        chown -R {{ ansible_user }}:{{ ansible_user }} /home/{{ ansible_user }}/.kube

    - name: Calico Operator installieren
      become: false
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/tigera-operator.yaml
      register: calico_operator
      changed_when: calico_operator.rc == 0
      failed_when: calico_operator.rc != 0 and 'AlreadyExists' not in calico_operator.stderr

    - name: Warten bis Calico CRDs registriert sind
      become: false
      command: kubectl wait --for=condition=Established crd/installations.operator.tigera.io --timeout=30s
      retries: 10
      delay: 10
      register: crd_wait
      until: crd_wait.rc == 0

    - name: Calico Custom Resources installieren
      become: false
      command: kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.31.4/manifests/custom-resources.yaml
      register: calico_cr
      changed_when: calico_cr.rc == 0
      failed_when: calico_cr.rc != 0 and 'AlreadyExists' not in calico_cr.stderr

    - name: Warten bis Calico-Node Pods ready sind
      become: false
      shell: kubectl wait --for=condition=Ready pods -l k8s-app=calico-node -n calico-system --timeout=180s
      retries: 5
      delay: 15
      register: calico_wait
      until: calico_wait.rc == 0

    - name: Certificate-Key erzeugen
      command: kubeadm init phase upload-certs --upload-certs
      register: cert_key_output

    - name: Certificate-Key extrahieren
      set_fact:
        cert_key: "{{ cert_key_output.stdout_lines | last }}"

    - name: Worker-Join-Command erzeugen
      command: kubeadm token create --print-join-command
      register: join_command

    - name: Join-Commands als Facts setzen
      set_fact:
        k8s_join_command: "{{ join_command.stdout }}"
        k8s_cp_join_command: "{{ join_command.stdout }} --control-plane --certificate-key {{ cert_key }}"

# =============================================================
# 3) Weitere Control Plane Nodes joinen
# =============================================================
- name: Weitere Control Planes joinen
  hosts: controlplane[1:]
  become: true
  serial: 1
  tasks:

    - name: Als Control Plane joinen
      command: >
        {{ hostvars[groups['controlplane'][0]].k8s_cp_join_command }}
        --apiserver-advertise-address={{ ansible_host }}
      args:
        creates: /etc/kubernetes/admin.conf

    - name: kubeconfig für User einrichten
      shell: |
        mkdir -p /home/{{ ansible_user }}/.kube
        cp /etc/kubernetes/admin.conf /home/{{ ansible_user }}/.kube/config
        chown -R {{ ansible_user }}:{{ ansible_user }} /home/{{ ansible_user }}/.kube

# =============================================================
# 4) kubeconfig lokal kopieren (zeigt auf VIP)
# =============================================================
- name: kubeconfig lokal bereitstellen
  hosts: controlplane[0]
  become: true
  tasks:

    - name: kubeconfig vom Control Plane holen
      fetch:
        src: /etc/kubernetes/admin.conf
        dest: /tmp/kubeconfig
        flat: true

    - name: Lokales .kube Verzeichnis erstellen
      delegate_to: localhost
      become: false
      file:
        path: "{{ lookup('env', 'HOME') }}/.kube"
        state: directory
        mode: '0700'

    - name: kubeconfig nach ~/.kube/config kopieren
      delegate_to: localhost
      become: false
      copy:
        src: /tmp/kubeconfig
        dest: "{{ lookup('env', 'HOME') }}/.kube/config"
        mode: '0600'

    - name: Temp-Datei aufräumen
      delegate_to: localhost
      become: false
      file:
        path: /tmp/kubeconfig
        state: absent

# =============================================================
# 5) Worker Nodes joinen
# =============================================================
- name: Worker dem Cluster hinzufügen
  hosts: workers
  become: true
  tasks:

    - name: Worker joinen
      command: "{{ hostvars[groups['controlplane'][0]].k8s_join_command }}"
      args:
        creates: /etc/kubernetes/kubelet.conf
EOF
```

**Hinweise:**
- `--control-plane-endpoint "{{ vip }}:6443"` sorgt dafür, dass alle Nodes die VIP als API-Endpunkt verwenden
- `--upload-certs` verschlüsselt die Zertifikate im Cluster, damit weitere CPs sie automatisch abrufen können (gültig 2h)
- `serial: 1` bei den weiteren CPs stellt sicher, dass sie nacheinander joinen
- Die kubeconfig zeigt bereits auf die VIP — kein URL-Rewrite nötig

---

## Schritt 10: OpenTofu ausführen

```bash
tofu init
tofu plan
tofu apply -auto-approve
```

### Ergebnis prüfen

```bash
cat inventory/hosts.ini
```

Erwartete Ausgabe (bei TLN_NR=1):

```ini
[loadbalancer]
k8s-tln1-lb1 ansible_host=10.10.10.111
k8s-tln1-lb2 ansible_host=10.10.10.112

[controlplane]
k8s-tln1-cp1 ansible_host=10.10.10.113
k8s-tln1-cp2 ansible_host=10.10.10.114
k8s-tln1-cp3 ansible_host=10.10.10.115

[workers]
k8s-tln1-worker1 ansible_host=10.10.10.116

[k8s_cluster:children]
controlplane
workers

[all:vars]
ansible_user=trainee
ansible_ssh_private_key_file=~/.ssh/id_ed25519
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
vip=10.10.10.110
```

```bash
# SSH-Erreichbarkeit testen
ansible -i inventory/hosts.ini all -m ping
```

---

## Schritt 11: Ansible-Playbook ausführen

```bash
ansible-playbook -i inventory/hosts.ini ansible/site.yml
```

---

## Schritt 12: Cluster prüfen

Die kubeconfig zeigt auf die VIP — du kannst direkt lokal arbeiten:

```bash
kubectl get nodes
```

Erwartete Ausgabe:

```
NAME              STATUS   ROLES           AGE   VERSION
k8s-tln1-cp1      Ready    control-plane   5m    v1.31.x
k8s-tln1-cp2      Ready    control-plane   3m    v1.31.x
k8s-tln1-cp3      Ready    control-plane   2m    v1.31.x
k8s-tln1-worker1  Ready    <none>          1m    v1.31.x
```

HA prüfen:

```bash
# VIP erreichbar?
curl -k https://10.10.10.110:6443/healthz

# HAProxy-Status auf LB-Nodes
ssh trainee@10.10.10.111 systemctl status haproxy
ssh trainee@10.10.10.112 systemctl status keepalived

# Welcher LB-Node hält die VIP?
ssh trainee@10.10.10.111 ip addr show | grep 10.10.10.110

# Calico-Status
kubectl get pods -n calico-system
```

---

## Schritt 13: Aufräumen

```bash
tofu destroy
```

---

## Optional: Alles in einem Script (deploy.sh)

```bash
cat > deploy.sh <<'SCRIPT'
#!/bin/bash
set -e

echo "=== Step 1: OpenTofu - VMs erstellen ==="
tofu init
tofu apply -auto-approve

echo ""
echo "=== Warte 30s auf VM-Boot ==="
sleep 30

INVENTORY="$(tofu output -raw inventory_path)"
echo "Inventory: $INVENTORY"
cat "$INVENTORY"

echo ""
echo "=== Step 2: Ansible - HA-Cluster mit Calico bootstrappen ==="
ansible-playbook -i "$INVENTORY" ansible/site.yml

echo ""
echo "=== Fertig! Cluster prüfen: ==="
echo "kubectl get nodes"
echo "curl -k https://$(tofu output -raw vip):6443/healthz"
SCRIPT

chmod +x deploy.sh
```
