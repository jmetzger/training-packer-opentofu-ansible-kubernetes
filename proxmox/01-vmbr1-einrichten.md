# vmbr1 einrichten (Setup hetzner)

## Konstellation:

  * Wir werden ohne externe IP's arbeiten, weil bei hetzner mit Einrichtungsgebühren pro ip von Euro 20,- verbunden sind.
  * Diese Einrichtung brauchen wir so nur für proxmox auf hetzner (und nur wenn wir ausschliesslich mit internen ip's arbeiten)

## Einrichtung (Teil 1: ip selbst)

```
cat >> /etc/network/interfaces << 'EOF'

auto vmbr1
iface vmbr1 inet static
        address 10.10.10.1/24
        bridge-ports none
        bridge-stp off
        bridge-fd 0
        post-up   echo 1 > /proc/sys/net/ipv4/ip_forward
        post-up   iptables -t nat -A POSTROUTING -s '10.10.10.0/24' -o enp7s0 -j MASQUERADE
        post-down iptables -t nat -D POSTROUTING -s '10.10.10.0/24' -o enp7s0 -j MASQUERADE
EOF
```

```
ifreload -a
```
