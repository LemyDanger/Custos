###############################################################################
# 0. AUSGANGSSITUATION
# - Frischer Debian 13 Server
# - Erstlogin als root per SSH
# - Ziel: Admin-User "custos" mit Key-Login
# - Root-SSH deaktivieren
# - Passwort-Login deaktivieren
###############################################################################


###############################################################################
# 1. ADMIN-USER ANLEGEN
###############################################################################

# Neuen Benutzer erstellen
adduser custos

# Benutzer in sudo-Gruppe aufnehmen
usermod -aG sudo custos

# Prüfen ob Gruppe korrekt gesetzt ist
groups custos



###############################################################################
# 2. SSH-KEY FÜR CUSTOS HINTERLEGEN
###############################################################################

# Zum Benutzer wechseln (sauberer als alles als root anzulegen)
su - custos

# .ssh Verzeichnis anlegen
mkdir -p ~/.ssh

# Public Key einfügen
nano ~/.ssh/authorized_keys
# → hier kompletten Inhalt von custos_worker1.pub einfügen

# Rechte korrekt setzen (wichtig für SSH!)
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Zurück zu root
exit



###############################################################################
# 3. SSH KONFIGURATION HÄRTEN
###############################################################################

# SSH-Konfig öffnen
nano /etc/ssh/sshd_config

# Folgende Einstellungen setzen oder prüfen:
#
# PermitRootLogin no
# PasswordAuthentication no
# KbdInteractiveAuthentication no
# PubkeyAuthentication yes
# UsePAM yes



###############################################################################
# 4. CLOUD-INIT KONFIGURATION ANPASSEN
# Debian/Cloud-Images überschreiben oft PasswordAuthentication
###############################################################################

nano /etc/ssh/sshd_config.d/50-cloud-init.conf

# Sicherstellen:
# PasswordAuthentication no



###############################################################################
# 5. KONFIGURATION VALIDIEREN
###############################################################################

# Syntaxprüfung der SSH-Konfiguration
sshd -t

# Effektive Werte anzeigen
sshd -T | egrep 'permitrootlogin|passwordauthentication|kbdinteractiveauthentication'



###############################################################################
# 6. SSH NEU LADEN (ohne bestehende Sessions zu beenden)
###############################################################################

systemctl reload ssh



###############################################################################
# 7. TESTS (in neuer Session)
###############################################################################

# Login als custos muss funktionieren
# ssh custos@<server-ip>

# root darf NICHT mehr funktionieren
# ssh root@<server-ip>



###############################################################################
# DEBUGGING & FEHLERSUCHE
###############################################################################

# Letzte SSH-Logeinträge prüfen
tail -n 50 /var/log/auth.log

# Prüfen welche SSH-Config greift
sshd -T | grep permitrootlogin

# Prüfen ob Include-Dateien etwas überschreiben
grep -R PermitRootLogin /etc/ssh/
grep -R PasswordAuthentication /etc/ssh/

# Rechte der SSH-Struktur prüfen
ls -ld /home/custos
ls -ld /home/custos/.ssh
ls -l /home/custos/.ssh/authorized_keys

# Owner & Rechte exakt anzeigen
stat -c '%U:%G %a %n' /home/custos \
/home/custos/.ssh \
/home/custos/.ssh/authorized_keys

# SSH-Dienststatus prüfen
systemctl status ssh --no-pager
