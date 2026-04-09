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
