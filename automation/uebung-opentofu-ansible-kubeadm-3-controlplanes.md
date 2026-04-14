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

## Hintergrund: Keepalived

Keepalived verwaltet eine **Virtual IP (VIP)** — eine "schwebende" IP-Adresse, die immer auf einem gesunden Node liegt. Es läuft auf beiden HAProxy-Nodes und handelt per **VRRP-Protokoll** (Virtual Router Redundancy Protocol) aus, wer die VIP bekommt:

- Ein Node ist **MASTER** (hält die VIP), der andere ist **BACKUP**
- Wenn der Master ausfällt, übernimmt der Backup die VIP (Failover in 2-3 Sekunden)
- Für alle Clients (kubectl, kubelet) ist immer nur die VIP sichtbar — welcher LB-Node dahinter steht, ist transparent

**Keepalived-Konfiguration erklärt:**

```
vrrp_script chk_haproxy {
    script "/etc/keepalived/check_apiserver.sh"   # Prüft ob HAProxy läuft
    interval 3                                     # Alle 3 Sekunden prüfen
    weight -2                                      # Bei Fehler: Priorität -2
    fall 10                                        # Nach 10 Fehlern → Script "failed"
    rise 2                                         # Nach 2 Erfolgen → Script "ok"
}

vrrp_instance haproxy-vip {
    state MASTER|BACKUP          # Startrolle (MASTER auf LB1, BACKUP auf LB2)
    priority 200|100             # Höhere Priorität = bevorzugter Master
    interface eth0               # Netzwerk-Interface für die VIP
    virtual_router_id 51         # Muss auf beiden Nodes gleich sein
    advert_int 1                 # VRRP-Heartbeat alle 1 Sekunde
    authentication {
        auth_type PASS
        auth_pass kubernetes     # Shared Secret zwischen den Nodes
    }
    virtual_ipaddress {
        10.10.10.1X0             # Die VIP-Adresse
    }
    track_script {
        chk_haproxy              # Wenn HAProxy stirbt → Priorität sinkt → Failover
    }
}
```

**Failover-Ablauf:** MASTER sendet jede Sekunde einen VRRP-Heartbeat. Bleibt er aus (Node down) oder sinkt die Priorität (HAProxy down → `weight -2`), übernimmt der BACKUP die VIP.

---

## Hintergrund: HAProxy

HAProxy ist ein TCP/HTTP Load Balancer. In unserem Setup verteilt er API-Server-Traffic (TCP Port 6443) per Round-Robin auf die 3 Control Plane Nodes und prüft deren Gesundheit.

**HAProxy-Konfiguration erklärt:**

```
global
    log /dev/log local0 warning     # Logging: nur Warnungen und Fehler
    chroot /var/lib/haproxy          # Sicherheit: eingesperrtes Verzeichnis
    pidfile /var/run/haproxy.pid     # PID-Datei für Systemd
    maxconn 4000                     # Max gleichzeitige Verbindungen global
    user haproxy                     # Unprivilegierter User
    group haproxy
    daemon                           # Als Hintergrund-Dienst laufen
    stats socket /var/lib/haproxy/stats  # Unix-Socket für Statistiken
```

```
defaults
    log     global                   # Logging von global erben
    option  tcplog                   # TCP-Verbindungen loggen
    option  dontlognull              # Keine leeren Verbindungen loggen
    timeout connect 5000             # 5s Timeout für Verbindungsaufbau zum Backend
    timeout client  50000            # 50s Timeout für Client-Inaktivität
    timeout server  50000            # 50s Timeout für Server-Inaktivität
```

```
frontend k8s-api
    bind *:6443                      # Lauscht auf allen Interfaces, Port 6443
    mode tcp                         # Layer 4 (TCP), kein HTTP-Parsing
    default_backend k8s-api          # Leitet an Backend weiter
```

```
backend k8s-api
    mode tcp                         # TCP-Modus (TLS-Termination macht der API-Server)
    option httpchk GET /healthz      # Health Check: HTTPS-Request an /healthz
    http-check expect status 200     # Nur Status 200 = gesund
    balance roundrobin               # Neue Verbindungen reihum verteilen
    default-server                   # Standardwerte für alle server-Zeilen:
        inter 10s                    #   Health Check alle 10 Sekunden
        downinter 5s                 #   Wenn down: alle 5s prüfen (schnellere Recovery)
        rise 2                       #   Nach 2 Erfolgen → Server gilt als gesund
        fall 2                       #   Nach 2 Fehlern → Server gilt als down
        slowstart 60s                #   Nach Recovery: 60s langsam Traffic hochfahren
        maxconn 250                  #   Max 250 gleichzeitige Verbindungen pro Server
        maxqueue 256                 #   Max 256 wartende Verbindungen in der Queue
        weight 100                   #   Gewichtung (alle gleich = gleichmäßig)
    server cp1 10.10.10.113:6443 check check-ssl verify none  # check-ssl: Health Check über HTTPS
    server cp2 10.10.10.114:6443 check check-ssl verify none  # verify none: selbst-signiertes Zert. akzeptieren
    server cp3 10.10.10.115:6443 check check-ssl verify none
```

**Hinweis zu Round-Robin im TCP-Modus:** HAProxy verteilt pro neuer TCP-Verbindung. Da kubectl HTTP/2 mit persistenten Verbindungen nutzt, bleibt eine kubectl-Session auf demselben CP. Erst bei einer neuen Verbindung (neuer Client, Reconnect) wird der nächste CP gewählt. Der Hauptvorteil ist nicht die Lastverteilung, sondern die automatische Erkennung und Umgehung ausgefallener CPs.

---

## Hintergrund: HA Control Plane mit kubeadm

Ein HA Control Plane besteht aus 3 Nodes, die jeweils API-Server, Controller-Manager, Scheduler und etcd laufen haben. Das Bootstrapping erfolgt in zwei Phasen:

### Phase 1: Erster Control Plane (kubeadm init)

```bash
kubeadm init \
  --control-plane-endpoint "VIP:6443" \
  --upload-certs \
  --pod-network-cidr=192.168.0.0/16 \
  --apiserver-advertise-address=<CP1-IP>
```

**Was passiert:**

1. kubeadm erstellt alle **Zertifikate** (CA, API-Server, etcd, Front-Proxy, ServiceAccount Keys)
2. Statische Pod-Manifeste werden unter `/etc/kubernetes/manifests/` angelegt (API-Server, Controller-Manager, Scheduler, etcd)
3. kubelet startet diese als Static Pods
4. `--control-plane-endpoint "VIP:6443"` wird in die kubeconfig und ins Cluster geschrieben — alle Nodes verwenden die VIP als API-Endpunkt
5. `--upload-certs` verschlüsselt die CA-Zertifikate und speichert sie als Secret (`kubeadm-certs`) im Cluster

### Phase 2: Weitere Control Planes (kubeadm join)

```bash
kubeadm join VIP:6443 \
  --control-plane \
  --certificate-key <KEY> \
  --apiserver-advertise-address=<CPn-IP>
```

**Was passiert:**

1. Der neue Node kontaktiert die VIP (→ HAProxy → ein gesunder CP)
2. Mit dem **Certificate-Key** holt er das verschlüsselte `kubeadm-certs` Secret und entschlüsselt es
3. Daraus erhält er die CA-Zertifikate und generiert seine eigenen API-Server/etcd-Zertifikate
4. `--control-plane` sagt kubeadm: erstelle API-Server, Controller-Manager, Scheduler und etcd auf diesem Node
5. Der neue etcd-Node tritt dem bestehenden etcd-Cluster bei (Raft-Konsens)
6. `--apiserver-advertise-address` setzt die IP, auf der dieser API-Server erreichbar ist

### Certificate-Key Gültigkeit

Der Certificate-Key ist nur **2 Stunden gültig**. Nach Ablauf wird das Secret automatisch gelöscht. Falls man später einen weiteren CP hinzufügen will:

```bash
# Auf einem bestehenden CP — lädt Zertifikate erneut hoch
kubeadm init phase upload-certs --upload-certs
```

### Warum serial: 1 im Playbook?

Die weiteren CPs joinen im Ansible-Playbook mit `serial: 1` — also nacheinander. Grund: Jeder neue Node muss dem etcd-Cluster beitreten. etcd braucht dafür Quorum und eine stabile Cluster-Mitgliedschaft. Paralleles Joinen kann zu Race Conditions führen, bei denen etcd den Überblick über seine Mitglieder verliert.

### etcd Quorum

etcd verwendet den **Raft-Konsensalgorithmus**. Für Lese- und Schreiboperationen braucht es eine Mehrheit (Quorum):

| CPs gesamt | Quorum | Tolerierte Ausfälle |
|-----------|--------|---------------------|
| 1 | 1 | 0 |
| 3 | 2 | 1 |
| 5 | 3 | 2 |

Bei 3 CPs darf **maximal 1 Node** ausfallen. Fallen 2 aus, ist etcd (und damit der gesamte API-Server) nicht mehr verfügbar — auch nicht für Lesezugriffe, da etcd standardmäßig linearizable Reads verwendet.

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
mkdir -p ~/opentofu-k8s-ha/templates
mkdir -p ~/opentofu-k8s-ha/inventory
mkdir -p ~/opentofu-k8s-ha/ansible
cd ~/opentofu-k8s-ha
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
  ip_prefix = join(".", slice(split(".", var.node_base_ip), 0, 3))
  ip_start  = tonumber(split(".", var.node_base_ip)[3])
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
    size         = 30
  }

  network_device {
    bridge = var.bridge
  }

  initialization {
    datastore_id = var.datastore
    ip_config {
      ipv4 {
        address = "${local.ip_prefix}.${local.ip_start + count.index}/24"
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
        address = "${local.ip_prefix}.${local.ip_start + 2 + count.index}/24"
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
        address = "${local.ip_prefix}.${local.ip_start + 2 + var.cp_count + count.index}/24"
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

variable "node_base_ip" {
  description = "IP des ersten Nodes (LB1), weitere werden hochgezählt"
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
        ip   = "${local.ip_prefix}.${local.ip_start + i}"
      }
    ]
    cps = [
      for i, vm in proxmox_virtual_environment_vm.controlplane : {
        name = vm.name
        ip   = "${local.ip_prefix}.${local.ip_start + 2 + i}"
      }
    ]
    workers = [
      for i, vm in proxmox_virtual_environment_vm.worker : {
        name = vm.name
        ip   = "${local.ip_prefix}.${local.ip_start + 2 + var.cp_count + i}"
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
    "${local.ip_prefix}.${local.ip_start + i}"
  ]
}

output "controlplane_ips" {
  value = [
    for i in range(var.cp_count) :
    "${local.ip_prefix}.${local.ip_start + 2 + i}"
  ]
}

output "worker_ips" {
  value = [
    for i in range(var.worker_count) :
    "${local.ip_prefix}.${local.ip_start + 2 + var.cp_count + i}"
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
node_base_ip    = "10.10.10.1${TLN_NR}1"
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
              option httpchk GET /healthz
              http-check expect status 200
              balance roundrobin
              default-server inter 10s downinter 5s rise 2 fall 2 slowstart 60s maxconn 250 maxqueue 256 weight 100
          {% for host in cp_hosts %}
              server cp{{ loop.index }} {{ host }}:6443 check check-ssl verify none
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
tofu apply -parallelism=1 -auto-approve
```

**Hinweis:** `-parallelism=1` ist nötig, da Proxmox bei parallelen Clones auf denselben Storage ein Lock setzt und der Apply sonst fehlschlägt.

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
tofu destroy -parallelism=1
```

---

## Optional: Alles in einem Script (deploy.sh)

```bash
cat > deploy.sh <<'SCRIPT'
#!/bin/bash
set -e

echo "=== Step 1: OpenTofu - VMs erstellen ==="
tofu init
tofu apply -parallelism=1 -auto-approve

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
