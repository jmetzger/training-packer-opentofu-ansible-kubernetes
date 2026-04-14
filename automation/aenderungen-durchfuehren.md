# Änderungen durchführen

```
# Änderung, dann 
tofu apply -auto-approve 
# maschinen nacheinander neu starten 
 ansible loadbalancer -m reboot -b -f 1 -i inventory/hosts.ini
```
