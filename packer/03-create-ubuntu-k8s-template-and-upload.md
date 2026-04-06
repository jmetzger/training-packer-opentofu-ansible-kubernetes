# Übung: Packer – Ubuntu 24.04 Kubernetes-Template für Proxmox

## Ziel

Wir erstellen mit Packer ein Proxmox-VM-Template, das Ubuntu 24.04 mit allen Kubernetes-Voraussetzungen enthält:

- containerd (SystemdCgroup)
- kubeadm, kubelet, kubectl (v1.35)
- Kernel-Module: `overlay`, `br_netfilter`
- Sysctl: IP-Forwarding, Bridge-Netfilter
- Swap deaktiviert
- kubeadm-Images vorgezogen
- cloud-init bereit für Klonen

Template-ID: **900\<tln-nr\>** (z.B. TLN 1 → `9001`, TLN 5 → `9005`)

---

## Voraussetzungen

- Zugang zum Proxmox-Server
- Ubuntu 24.04 Server ISO bereits auf Proxmox hochgeladen (`local:iso/ubuntu-24.04.4-live-server-amd64.iso`)
- Packer installiert auf eurem Arbeitsrechner

### Packer installieren (falls noch nicht vorhanden)

```bash
# Variante 1: Hashicorp Repo (Debian/Ubuntu)
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" \
  | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt-get update && sudo apt-get install -y packer

# Variante 2: Binary Download
# https://developer.hashicorp.com/packer/downloads
```

---

## Schritt 1: Teilnehmernummer setzen und Verzeichnisstruktur anlegen

```bash
export TLN_NR=1   # <-- EURE Nummer hier!
```

```bash
cd
mkdir -p packer/template-ubuntu/{http,scripts}
cd packer/template-ubuntu
```

Die Struktur sieht so aus:

```
packer/template-ubuntu/
├── ubuntu-k8s.pkr.hcl          # Packer-Template (HCL)
├── variables.pkrvars.hcl        # Eure Variablen (nicht committen!)
├── http/
│   ├── user-data                # Autoinstall Cloud-Config
│   └── meta-data                # (leer, aber nötig)
└── scripts/
    ├── 01-system-base.sh        # Updates, Swap off, Sysctl
    ├── 02-kernel-modules.sh     # overlay, br_netfilter
    ├── 03-containerd.sh         # containerd aus Docker-Repo
    ├── 04-kubernetes.sh         # kubeadm, kubelet, kubectl
    └── 05-cleanup.sh            # Aufräumen für Template
```

---

## Schritt 2: Dateien anlegen

> **Wichtig:** Alle folgenden Befehle im Verzeichnis `packer/template-ubuntu/` ausführen!

### 2.1 – Packer-Template (ubuntu-k8s.pkr.hcl)

> `$TLN_NR` wird durch die Shell automatisch mit eurer Nummer ersetzt.

```bash
cat > ubuntu-k8s.pkr.hcl << ENDOFFILE
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.2.2"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

variable "proxmox_url" {
  type    = string
  default = "https://176.9.38.183:8006/api2/json"
}

variable "proxmox_token_id" {
  type        = string
  description = "API Token ID, z.B. root@pam!packer"
}

variable "proxmox_token_secret" {
  type      = string
  sensitive = true
}

variable "proxmox_node" {
  type    = string
  default = "pve"
}

variable "iso_file" {
  type    = string
  default = "local:iso/ubuntu-24.04.4-live-server-amd64.iso"
}

variable "ssh_username" {
  type    = string
  default = "trainee"
}

variable "ssh_password" {
  type      = string
  default   = "training"
  sensitive = true
}

source "proxmox-iso" "ubuntu-k8s" {
  proxmox_url              = var.proxmox_url
  username                 = var.proxmox_token_id
  token                    = var.proxmox_token_secret
  insecure_skip_tls_verify = true
  node                     = var.proxmox_node

  vm_id                = 900$TLN_NR
  vm_name              = "ubuntu-k8s-tln$TLN_NR"
  template_description = "Ubuntu 24.04 mit kubeadm - TLN $TLN_NR"

  boot_iso {
    iso_file = var.iso_file
  }

  os           = "l26"
  cpu_type     = "host"
  cores        = 2
  memory       = 4096
  machine      = "q35"
  bios         = "ovmf"
  scsi_controller = "virtio-scsi-single"

  efi_config {
    efi_storage_pool  = "local-lvm"
    efi_type          = "4m"
    pre_enrolled_keys = false
  }

  disks {
    type         = "scsi"
    disk_size    = "30G"
    # normalerweise schon local-lvm
    # storage_pool = "local-lvm"
    # in unserer Installation gibt es aber kein lvm
    storage_pool = "local"
    format       = "raw"
  }

  network_adapters {
    model    = "virtio"
    bridge   = "vmbr0"
    firewall = false
  }

  cloud_init              = true
  cloud_init_storage_pool = "local-lvm"

  boot_command = [
    "<wait5><wait5>",
    "c<wait3>",
    "linux /casper/vmlinuz autoinstall ds='nocloud;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/' ---<enter>",
    "initrd /casper/initrd<enter>",
    "boot<enter>"
  ]

  boot_wait = "5s"

  http_directory = "http"

  ssh_username = var.ssh_username
  ssh_password = var.ssh_password
  ssh_timeout  = "30m"
}

build {
  sources = ["source.proxmox-iso.ubuntu-k8s"]

  provisioner "shell" {
    inline = ["while [ ! -f /var/lib/cloud/instance/boot-finished ]; do sleep 2; done"]
  }

  provisioner "shell" {
    execute_command = "echo '\${var.ssh_password}' | sudo -S bash -c '{{ .Path }}'"
    scripts = [
      "scripts/01-system-base.sh",
      "scripts/02-kernel-modules.sh",
      "scripts/03-containerd.sh",
      "scripts/04-kubernetes.sh",
      "scripts/05-cleanup.sh"
    ]
  }
}
ENDOFFILE
```

### 2.2 – Autoinstall: user-data und meta-data

```bash
cat > http/user-data << 'ENDOFFILE'
#cloud-config
autoinstall:
  version: 1
  locale: en_US.UTF-8
  keyboard:
    layout: de
  identity:
    hostname: ubuntu-k8s
    username: trainee
    password: "$6$rounds=4096$randomsalt$VZzF0L7Kj5GJxKxKqVxDmC.MjRqQ5bN5YZKf0MhJxWlKjnDBa8dqm1CB/LJhD6sR9.d5XlE2qWhYv6Fx9aXH0"
    # Passwort: training
  ssh:
    install-server: true
    allow-pw: true
  storage:
    layout:
      name: lvm
  packages:
    - qemu-guest-agent
    - cloud-init
    - curl
    - apt-transport-https
    - ca-certificates
    - gnupg
    - software-properties-common
  late-commands:
    - "systemctl enable qemu-guest-agent --root=/target"
ENDOFFILE
```

```bash
# meta-data muss existieren, bleibt aber leer
touch http/meta-data
```

### 2.3 – Script: 01-system-base.sh

```bash
cat > scripts/01-system-base.sh << 'ENDOFFILE'
#!/bin/bash
set -euo pipefail

echo ">>> System-Updates"
apt-get update
apt-get upgrade -y

echo ">>> Swap deaktivieren"
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab

echo ">>> IPv4 Forwarding & Bridge-Netfilter"
cat > /etc/sysctl.d/99-kubernetes.conf <<EOF
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system
ENDOFFILE
```

### 2.4 – Script: 02-kernel-modules.sh

```bash
cat > scripts/02-kernel-modules.sh << 'ENDOFFILE'
#!/bin/bash
set -euo pipefail

echo ">>> Kernel-Module für Kubernetes"
cat > /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter
ENDOFFILE
```

### 2.5 – Script: 03-containerd.sh

```bash
cat > scripts/03-containerd.sh << 'ENDOFFILE'
#!/bin/bash
set -euo pipefail

echo ">>> containerd installieren (Docker-Repo)"

install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${VERSION_CODENAME}") stable" > /etc/apt/sources.list.d/docker.list

apt-get update
apt-get install -y containerd.io

echo ">>> containerd Default-Config mit SystemdCgroup"
mkdir -p /etc/containerd
containerd config default > /etc/containerd/config.toml
sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml

systemctl restart containerd
systemctl enable containerd
ENDOFFILE
```

### 2.6 – Script: 04-kubernetes.sh

```bash
cat > scripts/04-kubernetes.sh << 'ENDOFFILE'
#!/bin/bash
set -euo pipefail

KUBE_VERSION="1.35"

echo ">>> Kubernetes ${KUBE_VERSION} Repo hinzufügen"
mkdir -p /etc/apt/keyrings
curl -fsSL "https://pkgs.k8s.io/core:/stable:/v${KUBE_VERSION}/deb/Release.key" \
  | gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${KUBE_VERSION}/deb/ /" \
  > /etc/apt/sources.list.d/kubernetes.list

apt-get update
apt-get install -y kubelet kubeadm kubectl
apt-mark hold kubelet kubeadm kubectl

echo ">>> crictl konfigurieren"
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
EOF

echo ">>> kubeadm Images vorziehen"
kubeadm config images pull

systemctl enable kubelet
ENDOFFILE
```

### 2.7 – Script: 05-cleanup.sh

```bash
cat > scripts/05-cleanup.sh << 'ENDOFFILE'
#!/bin/bash
set -euo pipefail

echo ">>> Cleanup für Template"
apt-get autoremove -y
apt-get clean
rm -rf /var/lib/apt/lists/*

# Cloud-init zurücksetzen für Template
cloud-init clean --logs

# Machine-ID leeren (wird beim Klonen neu generiert)
truncate -s 0 /etc/machine-id
rm -f /var/lib/dbus/machine-id
ln -s /etc/machine-id /var/lib/dbus/machine-id

# SSH Host-Keys entfernen (werden beim Boot neu erzeugt)
rm -f /etc/ssh/ssh_host_*

echo ">>> Template bereit"
ENDOFFILE
```

### 2.8 – Scripts ausführbar machen

```bash
chmod +x scripts/*.sh
```

---

## Schritt 3: Variablen anpassen

```bash
# Dateien, die auf ".auto.pkrvars.hcl" enden werden automatisch geladen 
cat > variables.auto.pkrvars.hcl << 'ENDOFFILE'
proxmox_url          = "https://176.9.38.183:8006/api2/json"
proxmox_token_id     = "root@pam!automation"
proxmox_token_secret = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
proxmox_node         = "pve"
iso_file             = "local:iso/ubuntu-24.04.4-live-server-amd64.iso"
ENDOFFILE
```

**Jetzt die Datei editieren** – Proxmox-Zugangsdaten eintragen:

```bash
nano variables.auto.pkrvars.hcl
```

> **Hinweis Token-ID:** Das Format ist `user@realm!tokenname`, z.B. `root@pam!packer`.
> Der Token-Secret ist die UUID, die beim Erstellen des Tokens angezeigt wurde.

---

## ISO-Datei

Das Ubuntu 24.04 Server ISO muss **vorab auf dem Proxmox-Server** im Storage liegen.
Der Trainer lädt es einmal hoch – ihr referenziert es dann über `iso_file`.

Falls ihr es selbst hochladen müsst:

```bash
# Variante 1: Über die Proxmox-Weboberfläche
# Node (z.B. pve) → local → ISO Images → Upload

# Variante 2: Direkt auf dem Proxmox-Host
cd /var/lib/vz/template/iso/
wget https://releases.ubuntu.com/24.04.4/ubuntu-24.04.4-live-server-amd64.iso
```

Danach ist das ISO unter `local:iso/ubuntu-24.04.4-live-server-amd64.iso` verfügbar.

---

## Schritt 4: Packer initialisieren

```bash
packer init ubuntu-k8s.pkr.hcl
```

Das lädt das Proxmox-Plugin herunter.

---

## Schritt 5: Validieren

```bash
# validiert alle pkr.hcl Dateien 
packer validate .
```

---

## Schritt 6: Template bauen

```bash
packer build .
```

> **Dauer:** ca. 10–15 Minuten. Packer startet die VM, führt autoinstall durch, provisioniert per SSH und konvertiert am Ende zu einem Template.

---

## Schritt 7: Überprüfen

In der Proxmox-Oberfläche solltet ihr jetzt ein Template sehen:

- **ID:** 900\<eure-nr\> (z.B. 9001)
- **Name:** ubuntu-k8s-tln\<eure-nr\>

### Schnelltest: VM aus Template klonen

```bash
# Auf dem Proxmox-Host:
qm clone 9001 201 --name k8s-test --full
qm start 201

# Per SSH verbinden und prüfen:
kubeadm version
kubelet --version
kubectl version --client
containerd --version
cat /etc/modules-load.d/k8s.conf
sysctl net.ipv4.ip_forward
```

---

## Was steckt im Template?

| Komponente | Details |
|---|---|
| OS | Ubuntu 24.04 LTS (Noble) |
| Container Runtime | containerd (Docker-Repo), SystemdCgroup=true |
| Kubernetes | kubeadm, kubelet, kubectl v1.35.x (apt-mark hold) |
| Kernel-Module | `overlay`, `br_netfilter` (persistent in `/etc/modules-load.d/`) |
| Sysctl | `ip_forward=1`, `bridge-nf-call-iptables=1` |
| Swap | Deaktiviert (fstab + swapoff) |
| Images | kubeadm-Images vorgezogen (`kubeadm config images pull`) |
| Cloud-Init | Bereit – Machine-ID + SSH-Keys geleert |
| QEMU Guest Agent | Installiert und aktiviert |
