# Packer – Machine Images automatisiert bauen

## Was ist Packer?

Packer ist ein **Open-Source-Tool von HashiCorp**, mit dem sich Machine Images automatisiert und reproduzierbar erstellen lassen. Statt VM-Templates manuell zusammenzuklicken, definierst du den gesamten Build-Prozess als Code.

## Unterstützte Plattformen

| Plattform       | Image-Typ          |
|-----------------|---------------------|
| Proxmox         | VM-Template         |
| VMware / vSphere| OVF / VM-Template   |
| Hyper-V         | VHDX                |
| AWS             | AMI                 |
| DigitalOcean    | Snapshot            |
| Azure           | Managed Image       |
| GCP             | Machine Image       |

## Wie funktioniert es?

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│  Basis-ISO / │     │  Provisioner │     │  Fertiges    │
│  Cloud-Image │ ──▶ │  (Shell,     │ ──▶ │  Template /  │
│              │     │   Ansible…)  │     │  Image       │
└─────────────┘     └──────────────┘     └──────────────┘
```

1. **Source** – Packer startet eine temporäre VM aus einem Basis-Image oder ISO.
2. **Provisioner** – Shell-Skripte, Ansible-Playbooks o.ä. installieren und konfigurieren Software.
3. **Post-Processor** – Das Ergebnis wird als Template/Image gespeichert, die temp. VM wird gelöscht.

## Minimalbeispiel (HCL)

```hcl
packer {
  required_plugins {
    proxmox = {
      version = ">= 1.1.0"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

source "proxmox-iso" "ubuntu" {
  proxmox_url = "https://pve.example.com:8006/api2/json"
  node        = "pve1"
  iso_file    = "local:iso/ubuntu-24.04-server.iso"
  # ...
}

build {
  sources = ["source.proxmox-iso.ubuntu"]

  provisioner "shell" {
    inline = ["apt-get update && apt-get upgrade -y"]
  }
}
```

## Typischer Workflow

```
Packer          →  OpenTofu / Terraform  →  Ansible
(Image bauen)      (VMs provisionieren)     (Konfiguration)
```

Packer kümmert sich um das **Golden Image**, OpenTofu/Terraform um die Infrastruktur, Ansible um die laufende Konfiguration.

## Was ist ein Golden Image?

Ein Golden Image ist ein **vorkonfiguriertes, gehärtetes VM-/Container-Image**, das als einheitliche Basis für alle neuen Instanzen dient – die "Master-Vorlage".

**Typischer Inhalt:**
- OS-Updates, Standardpakete
- Security-Hardening
- Monitoring-Agent
- SSH-Keys / Basiskonfiguration

**Nicht enthalten:** anwendungsspezifische Konfiguration – das übernimmt Ansible o.ä. zur Laufzeit.

**Vorteil:** Jede VM startet von einem identischen, getesteten Ausgangszustand – kein Konfigurationsdrift, schnelleres Provisioning, reproduzierbare Umgebungen.

## Kernbefehle

| Befehl              | Beschreibung                          |
|----------------------|---------------------------------------|
| `packer init .`      | Plugins herunterladen                 |
| `packer validate .`  | Template auf Syntaxfehler prüfen      |
| `packer build .`     | Image bauen                           |
| `packer fmt .`       | HCL-Dateien formatieren               |

## Weiterführende Links

- [Packer Dokumentation](https://developer.hashicorp.com/packer/docs)
- [Plugin-Registry](https://developer.hashicorp.com/packer/integrations)
