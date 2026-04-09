# IIS-Bereitstellung mit Packer, Ansible & PowerShell

> **Ziel:** Ein Windows Server 2022 Image mit vorinstalliertem IIS bauen und als Hyper-V VM deployen – vollständig automatisiert.

---

## Pipeline-Überblick

```
Linux Control Node
  │
  ├─ Packer (Image bauen)
  │    └─ Ansible via WinRM (Provisionierung)
  │         ├─ IIS installieren       (win_feature)
  │         └─ Konfiguration          (win_copy, win_template, ...)
  │
  └─ Ansible via WinRM (VM-Deployment)
       └─ PowerShell auf Hyper-V Host (win_powershell)
```

Packer erstellt das Basis-Image, Ansible übernimmt sowohl die Provisionierung innerhalb des Images als auch das spätere VM-Deployment. PowerShell wird dabei als Ausführungsmotor unter der Haube von Ansible genutzt.

> **Wichtig:** Die Ansible Control Node (dort wo `ansible-playbook` ausgeführt wird) **muss ein Linux-System** sein. Ansible selbst läuft nicht auf Windows. Die Verbindung zu den Windows-Zielsystemen (Managed Nodes) erfolgt über **WinRM**.

---

## 1. Packer – Windows-Image bauen

Packer erstellt ein VHDX-Image auf Basis einer Windows Server ISO und übergibt die Provisionierung an Ansible.

```hcl
# windows-iis.pkr.hcl

source "hyperv-iso" "win2022" {
  iso_url        = "path/to/windows_server_2022.iso"
  generation     = 2
  cpus           = 2
  memory         = 4096
  disk_size      = 40960
  communicator   = "winrm"
  winrm_username = "Administrator"
  winrm_password = "PackerPass123!"
  winrm_timeout  = "30m"
}

build {
  sources = ["source.hyperv-iso.win2022"]

  provisioner "ansible" {
    playbook_file   = "./ansible/iis.yml"
    use_proxy       = false
    extra_arguments = ["-e", "ansible_winrm_transport=ntlm"]
  }
}
```

### Ausführung

```bash
packer init .
packer build windows-iis.pkr.hcl
```

**Output:** `C:\packer-output\win2022-iis.vhdx`

---

## 2. Ansible – IIS installieren und konfigurieren

Dieses Playbook wird von Packer während des Image-Builds aufgerufen.

```yaml
# ansible/iis.yml

- hosts: all
  gather_facts: false
  tasks:

    - name: IIS mit Management-Tools installieren
      ansible.windows.win_feature:
        name:
          - Web-Server
          - Web-Asp-Net45
          - Web-Mgmt-Tools
        state: present
        include_management_tools: true

    - name: Default-Seite deployen
      ansible.windows.win_copy:
        src: files/index.html
        dest: C:\inetpub\wwwroot\index.html

    - name: IIS-Dienst sicherstellen
      ansible.windows.win_service:
        name: W3SVC
        state: started
        start_mode: auto
```

### Voraussetzung

Die Ansible Collection für Windows muss installiert sein:

```bash
ansible-galaxy collection install ansible.windows
```

---

## 3. Ansible – VM aus dem Image deployen

Nach dem Image-Build kann ein zweites Playbook die VM auf dem Hyper-V Host erstellen und starten – ebenfalls über Ansible, mit PowerShell als Ausführungsmotor.

```yaml
# ansible/deploy-vm.yml

- hosts: hyperv_hosts
  gather_facts: false
  vars:
    vm_name: "iis-prod-01"
    vhdx_path: "C:\\packer-output\\win2022-iis.vhdx"
    switch_name: "Default Switch"

  tasks:

    - name: VM erstellen
      ansible.windows.win_powershell:
        script: |
          if (-not (Get-VM -Name "{{ vm_name }}" -ErrorAction SilentlyContinue)) {
            New-VM -Name "{{ vm_name }}" `
              -MemoryStartupBytes 4GB `
              -VHDPath "{{ vhdx_path }}" `
              -SwitchName "{{ switch_name }}" `
              -Generation 2
            Set-VM -Name "{{ vm_name }}" -ProcessorCount 2
          }

    - name: VM starten
      ansible.windows.win_powershell:
        script: |
          Start-VM -Name "{{ vm_name }}"
```

### Ausführung

```bash
ansible-playbook -i inventory.ini ansible/deploy-vm.yml
```

---

## Projektstruktur

```
project/
├── windows-iis.pkr.hcl          # Packer Template
├── ansible/
│   ├── iis.yml                   # IIS-Provisionierung (im Image)
│   ├── deploy-vm.yml             # VM-Deployment (auf Hyper-V Host)
│   └── files/
│       └── index.html            # Default-Webseite
└── inventory.ini                 # Ansible Inventory (Hyper-V Hosts)
```

---

## Zusammenfassung

| Schritt | Tool | Läuft auf | Aufgabe |
|---------|------|-----------|---------|
| Image bauen | **Packer** | Linux Control Node | Windows Server ISO → VHDX |
| IIS installieren | **Ansible** (`win_feature`) | Linux → WinRM → Windows | Rolle + Features ins Image |
| Konfiguration | **Ansible** (`win_copy`, `win_template`) | Linux → WinRM → Windows | Webseite, Einstellungen |
| VM erstellen | **Ansible** (`win_powershell`) | Linux → WinRM → Hyper-V Host | Hyper-V VM aus VHDX |

Der Vorteil dieses Ansatzes: Alles ist in Playbooks beschrieben, versionierbar in Git, und es werden keine separaten PowerShell-Skripte gepflegt. PowerShell bleibt das Werkzeug – Ansible die Steuerungsebene.
