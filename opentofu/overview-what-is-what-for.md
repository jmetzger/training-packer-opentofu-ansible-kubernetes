# OpenTofu – Was ist das ? Was damit machen, was lieber nicht.

## Was ist OpenTofu ?

- **Open-Source-Fork** von Terraform (nach HashiCorps Lizenzwechsel zu BSL)
- **Infrastructure-as-Code-Tool** – Infrastruktur deklarativ in HCL beschreiben
- Provisionierung per `tofu plan` / `tofu apply`
- **Terraform-Provider kompatibel** (AWS, Azure, GCP, DigitalOcean, Proxmox etc.)
- Lizenz: **MPL-2.0**
- Verwaltet von der **Linux Foundation**

## Dafür ist OpenTofu gemacht

| Bereich | Beispiele |
|---|---|
| **Cloud-Infrastruktur provisionieren** | VMs, Kubernetes-Cluster (DOKS, EKS, GKE), Netzwerke, Firewalls, Load Balancer |
| **Multi-Cloud & Hybrid** | Gleicher Workflow für DigitalOcean, AWS, Azure, vSphere, Proxmox etc. |
| **DNS & Domains** | Records anlegen/verwalten (z.B. DigitalOcean DNS, Cloudflare) |
| **State Management** | Zentraler State (S3, Consul, Postgres) als Single Source of Truth |
| **Modulare Infrastruktur** | Wiederverwendbare Module für wiederkehrende Patterns (z.B. Training-Cluster) |
| **Drift Detection** | `tofu plan` zeigt Abweichungen zwischen State und Realität |
| **Secrets-Infrastruktur** | OpenBao/Vault-Cluster aufsetzen, Backends konfigurieren |
| **Netzwerk & Security Groups** | VPCs, Subnetze, Firewall-Regeln deklarativ verwalten |
| **CI/CD-Integration** | `tofu plan` in MR/PR, `tofu apply` nach Approval (GitLab CI, GitHub Actions) |
| **Kosten-Transparenz** | Infra als Code → reviewbar, nachvollziehbar, auditierbar |

## Das kann OpenTofu auch (wird oft übersehen)

- **`for_each` / `count`** – Dynamisch N Teilnehmer-VMs oder Cluster hochfahren
- **`moved`-Blöcke** – Ressourcen refactoren ohne Destroy/Recreate
- **State Encryption** – Neu in OpenTofu, bei Terraform nicht verfügbar
- **`import`-Blöcke** – Bestehende Infra deklarativ in den State übernehmen
- **Provider-Funktionen** – Custom Functions von Providern nutzen (OpenTofu-exklusiv)
- **`-generate-config-out`** – Nach Import automatisch HCL generieren lassen

## Finger weg – dafür ist OpenTofu NICHT gedacht

 - **OpenTofu kann schlecht mit Dateien umgehen. Zustände von Dateien (das kann ansible besser !)**

| Anti-Pattern | Warum nicht | Besser |
|---|---|---|
| **Software auf VMs installieren/konfigurieren** | OpenTofu provisioniert, konfiguriert nicht | **Ansible**, cloud-init |
| **Application Deployment** | Kein Rolling Update, kein Health Check | **Helm, Flux CD, ArgoCD** |
| **Secrets direkt im State speichern** | State enthält Plaintext (auch mit State Encryption: defense in depth) | **OpenBao/Vault + ESO** |
| **Komplexe Logik / Scripting** | HCL ist deklarativ, keine Programmiersprache | **Python, Go, Bash** |
| **Kubernetes-Manifeste verwalten** | Geht technisch (K8s-Provider), funktioniert in der Praxis schlecht | **Helm, Kustomize, GitOps** |
| **Manuelle Änderungen neben OpenTofu** | State Drift → nächster Apply zerstört manuelle Änderungen | Alles durch Code |
| **Riesige Monolith-States** | Langsam, riskant, Blast Radius zu groß | **Workspaces / State-Splitting** |
| **Datenbank-Inhalte verwalten** | Keine Migration, kein Schema-Management | **Flyway, Liquibase, Django Migrations** |
| **Monitoring/Alerting konfigurieren** | Geht (Grafana-Provider etc.), wird aber schnell unhandlich | **Jsonnet, Grafana-as-Code, Ansible** |

## Faustregel

```
OpenTofu = Infrastruktur-Lifecycle (Create → Read → Update → Delete)
         ≠ Configuration Management
         ≠ Application Deployment
         ≠ Orchestrierung
```

> **Provisioniere mit OpenTofu, konfiguriere mit Ansible, deploye mit GitOps.**
