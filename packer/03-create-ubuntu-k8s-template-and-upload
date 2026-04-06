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

## Schritt 1: Verzeichnisstruktur anlegen

```bash
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

Alle Dateien aus dem bereitgestellten `packer/template-ubuntu/`-Ordner in euer Verzeichnis kopieren.

---

## Schritt 3: Variablen anpassen

```bash
cp variables.pkrvars.hcl.example variables.pkrvars.hcl
```

Datei `variables.pkrvars.hcl` editieren – **eure Teilnehmernummer und Proxmox-Zugangsdaten eintragen**:

```hcl
proxmox_url          = "https://proxmox.training.local:8006/api2/json"
proxmox_token_id     = "root@pam!packer"
proxmox_token_secret = "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
proxmox_node         = "pve"
iso_file             = "local:iso/ubuntu-24.04.4-live-server-amd64.iso"
tln_nr               = "1"    # <-- EURE Nummer hier!
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
# Datacenter → Storage (local) → ISO Images → Upload

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
packer validate -var-file=variables.pkrvars.hcl ubuntu-k8s.pkr.hcl
```

---

## Schritt 6: Template bauen

```bash
packer build -var-file=variables.pkrvars.hcl ubuntu-k8s.pkr.hcl
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
