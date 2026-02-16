###############################################################################
# 0. ZIEL
#
# - Host darf NICHT öffentlich administrierbar sein
# - SSH ausschließlich über Custos (WireGuard)
# - Provider-Firewall minimal (nur UDP 51820)
# - Host-Firewall mit nftables
# - Reboot-sicherer Zustand
###############################################################################



###############################################################################
# 1. CUSTOS (WireGuard) INSTALLATION
###############################################################################

# WireGuard installieren
sudo apt update
sudo apt install -y wireguard

# Prüfen ob installiert
wg --version



###############################################################################
# 2. SCHLÜSSEL ERZEUGEN
###############################################################################

# Private Key erzeugen (wird im Terminal angezeigt)
sudo wg genkey

# Private Key speichern
sudo nano /etc/wireguard/custos_private.key

# Rechte korrekt setzen (nur root darf lesen)
sudo chmod 600 /etc/wireguard/custos_private.key


# Public Key erzeugen
sudo wg pubkey

# Public Key speichern
sudo nano /etc/wireguard/custos_public.key

# Rechte setzen (Public darf lesbar sein)
sudo chmod 644 /etc/wireguard/custos_public.key



###############################################################################
# 3. CUSTOS KONFIGURATION
###############################################################################

sudo nano /etc/wireguard/custos.conf

# Inhalt:

[Interface]
Address = 10.100.0.1/24
ListenPort = 51820
PrivateKey = <Inhalt aus custos_private.key>
SaveConfig = false


# Windows-Peer hinzufügen:

[Peer]
PublicKey = <Windows Public Key>
AllowedIPs = 10.100.0.2/32



###############################################################################
# 4. CUSTOS STARTEN UND AKTIVIEREN
###############################################################################

# Interface starten
sudo wg-quick up custos

# Autostart aktivieren
sudo systemctl enable wg-quick@custos

# Status prüfen
sudo wg
ip addr show custos



###############################################################################
# 5. PROVIDER-FIREWALL
###############################################################################

# Nur folgende Regel aktiv:
# UDP 51820 inbound erlauben (0.0.0.0/0)
#
# Alle anderen TCP Ports schließen:
# - 22
# - 80
# - 443
# - 8443
# - 8447
#
# Ziel: Nur Custos öffentlich erreichbar



###############################################################################
# 6. HOST-FIREWALL (nftables)
###############################################################################

# Installation
sudo apt install -y nftables

# Konfigurationsdatei erstellen
sudo nano /etc/nftables.conf


# Inhalt:

flush ruleset

table inet filter {

  chain input {
    type filter hook input priority 0;
    policy drop;

    # Loopback erlauben
    iif "lo" accept

    # Bereits etablierte Verbindungen erlauben
    ct state established,related accept

    # Custos (WireGuard) öffentlich erlauben
    udp dport 51820 accept

    # SSH nur über Custos-Interface erlauben
    iifname "custos" tcp dport 22 accept

    # Ping nur über Custos-Interface erlauben
    iifname "custos" ip protocol icmp accept
    iifname "custos" ip6 nexthdr icmpv6 accept
  }

  chain forward {
    type filter hook forward priority 0;
    policy drop;
  }

  chain output {
    type filter hook output priority 0;
    policy accept;
  }
}


###############################################################################
# 7. REGELN TESTEN UND AKTIVIEREN
###############################################################################

# Syntax prüfen
sudo nft -c -f /etc/nftables.conf

# Regeln laden
sudo nft -f /etc/nftables.conf

# Autostart aktivieren
sudo systemctl enable --now nftables



###############################################################################
# 8. FUNKTIONSTEST
###############################################################################

# Windows:
# Tunnel aktivieren
# ping 10.100.0.1
# ssh custos@10.100.0.1

# Public SSH testen (muss blockiert sein)



###############################################################################
# 9. REBOOT-TEST
###############################################################################

sudo reboot

# Nach Reboot:
# Tunnel aktivieren
# SSH über 10.100.0.1 testen
# Public SSH weiterhin blockiert



###############################################################################
# DEBUGGING
###############################################################################

# Custos Status
sudo wg

# Custos Interface
ip addr show custos

# nftables Status
sudo systemctl status nftables

# Aktive Firewall-Regeln
sudo nft list ruleset

# Lauscht UDP 51820?
sudo ss -ulnp | grep 51820

# SSH Logs
sudo tail -n 50 /var/log/auth.log
