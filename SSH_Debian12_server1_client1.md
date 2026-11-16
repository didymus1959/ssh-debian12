# SSH unter Debian 12 – Installation, Konfiguration und Betrieb

> **Betriebshandbuch für zukünftige Administratoren**  
> Systeme: `server1` (SSH-Server) und `client1` (Administrationsclient)  
> Betriebssystem: Debian GNU/Linux 12 (Bookworm)  
> Stand: 2026-11-15 · Status: Vorlage – standortspezifische Angaben prüfen

## 1. Ziel und Geltungsbereich

Dieses Dokument beschreibt Installation, Erstkonfiguration, Härtung, Prüfung, Betrieb und Fehlerbehebung von OpenSSH auf zwei Debian-12-Systemen.

**Annahmen (vor Umsetzung anpassen):**

| Eigenschaft | `server1` | `client1` |
|---|---|---|
| Rolle | SSH-Zielsystem | SSH-Ausgangssystem |
| Beispiel-IP | `192.168.1.191` | `192.168.1.223` |
| Beispiel-DNS | `server1.srv.lcd` | `client1.srv.lcd` |
| Administrationskonto | `adminuser` | `adminuser` |
| SSH-Port | `22/tcp` | ausgehend zu `22/tcp` |

Die Adressen `192.168.1.0/24` und die Domain `srev.lcd` sind **Dokumentationsbeispiele**, keine produktiven Netzwerte. Der Benutzer `adminuser` ist ein Platzhalter. Befehle mit `sudo` erfordern entsprechende Berechtigungen.

### 1.1 Sicherheitsgrundsätze

- Administrationszugang bevorzugt mit individuellen Ed25519-Schlüsseln.
- Root-Login über SSH deaktivieren; kein gemeinsames Administratorkonto, wenn individuelle Konten möglich sind.
- Passwort- und Keyboard-Interactive-Login erst abschalten, **nachdem** der Schlüsselzugang nachweislich funktioniert.
- Firewall und SSH-Konfiguration nur bei vorhandener Konsolen-/Out-of-Band-Rettungsmöglichkeit ändern.
- Bestehende SSH-Sitzung bis zur erfolgreichen Prüfung einer **zweiten** Verbindung offen lassen.
- Schlüssel und Berechtigungen regelmäßig überprüfen; ausgeschiedene Administratoren unverzüglich entziehen.

## 2. Voraussetzungen und Bestandsaufnahme

Auf **beiden** Systemen:

```bash
cat /etc/os-release
hostnamectl
ip -br address
ip route
getent hosts server1.example.internal
```

Auf `server1` zusätzlich:

```bash
sudo ss -ltnp | grep ':22 ' || true
sudo systemctl status ssh --no-pager
```

DNS muss den richtigen Zielhost auflösen. Fehlt internes DNS, kann vorübergehend die Datei `/etc/hosts` auf `client1` ergänzt werden:

```text
192.168.1.191 server1.example.internal server1
```

`/etc/hosts` nur entsprechend der realen Netzwerkplanung pflegen.

## 3. Installation auf server1

### 3.1 Pakete installieren

```bash
sudo apt update
sudo apt install openssh-server
```

Das Paket installiert den OpenSSH-Server und die zugehörigen systemd-Komponenten. Status und Aktivierung prüfen:

```bash
sudo systemctl enable --now ssh
sudo systemctl status ssh --no-pager
sudo sshd -t
sudo ss -ltnp | grep ':22 '
```

**Hinweis:** Unter Debian heißt der systemd-Dienst üblicherweise `ssh.service` (nicht `sshd.service`). `sshd -t` prüft die Syntax und grundlegende Gültigkeit der Serverkonfiguration; es prüft nicht die Erreichbarkeit aus dem Netzwerk.

### 3.2 Administrationskonto vorbereiten

Wenn noch kein individuelles Administrationskonto existiert:

```bash
sudo adduser adminuser
sudo usermod -aG sudo adminuser
id adminuser
```

Die Mitgliedschaft in `sudo` nur erteilen, wenn sie benötigt und freigegeben ist. Die `sudo`-Berechtigung kann nach einer neuen Anmeldung mit `sudo -v` geprüft werden.

## 4. Installation auf client1

```bash
sudo apt update
sudo apt install openssh-client
ssh -V
```

Erreichbarkeit des Servers prüfen:

```bash
getent hosts server1.srv.lcd
nc -vz server1.srv.lcd 22
```

`nc` ist optional und wird gegebenenfalls mit `sudo apt install netcat-openbsd` installiert. Alternativ direkt mit `ssh -v adminuser@server1.srv.lcd` testen. Ein erfolgreicher TCP-Verbindungsaufbau bedeutet noch keine erfolgreiche SSH-Authentifizierung.

## 5. Schlüsselpaar auf client1 erstellen

**Als lokaler Benutzer `adminuser` auf `client1` ausführen, nicht mit `sudo`:**

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519_server1 -C 'adminuser@client1 -> server1'
chmod 600 ~/.ssh/id_ed25519_server1
chmod 644 ~/.ssh/id_ed25519_server1.pub
```

Eine starke Passphrase für den privaten Schlüssel vergeben. Die Option `-a 100` erhöht den Aufwand beim Ableiten des Schlüssels aus der Passphrase. Der öffentliche Schlüssel (`.pub`) darf verteilt werden; der private Schlüssel **niemals**.

Öffentlichen Schlüssel anzeigen:

```bash
cat ~/.ssh/id_ed25519_server1.pub
ssh-keygen -lf ~/.ssh/id_ed25519_server1.pub
```

### 5.1 Öffentlichen Schlüssel auf server1 installieren

Solange auf `server1` noch ein erlaubter initialer Anmeldeweg besteht:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_server1.pub adminuser@server1.srv.lcd
```

Wenn `ssh-copy-id` nicht verfügbar ist: öffentlichen Schlüssel über einen **vertrauenswürdigen** Konsolenzugang als einzelne Zeile in `/home/adminuser/.ssh/authorized_keys` eintragen. Auf `server1` müssen Eigentümer und Rechte stimmen:

```bash
sudo install -d -m 700 -o adminuser -g adminuser /home/adminuser/.ssh
sudo chown adminuser:adminuser /home/adminuser/.ssh/authorized_keys
sudo chmod 600 /home/adminuser/.ssh/authorized_keys
```

Die letzten beiden Befehle erst ausführen, wenn die Datei existiert. Bei zentral verwalteten Home-Verzeichnissen die dortigen Berechtigungsvorgaben beachten.

### 5.2 Ersten Schlüsseltest durchführen

```bash
ssh -i ~/.ssh/id_ed25519_server1 -o IdentitiesOnly=yes adminuser@server1.srv.lcd
```

Auf dem Zielsystem prüfen:

```bash
whoami
hostname
sudo -v
exit
```

**Host-Key-Vertrauen:** Beim ersten Verbindungsaufbau den angezeigten Host-Key-Fingerprint **nicht blind bestätigen**. Fingerprint auf `server1` über die Konsole ermitteln und über einen unabhängigen, vertrauenswürdigen Kanal vergleichen:

```bash
sudo ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Der Host-Key identifiziert den **Server**, der Benutzer-Schlüssel authentifiziert den **Administrator**. Eine unerwartete Host-Key-Änderung muss untersucht werden; `StrictHostKeyChecking=no` ist keine sichere Abkürzung.

## 6. SSH-Serverkonfiguration auf server1

### 6.1 Vorhandene Einstellungen sichern

```bash
sudo cp -a /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%Y%m%d-%H%M%S)
sudo ls -la /etc/ssh/sshd_config.d/
sudo sshd -T | less
```

Debian nutzt `/etc/ssh/sshd_config` und kann zusätzliche Dateien aus `/etc/ssh/sshd_config.d/*.conf` einlesen. **Wichtig:** OpenSSH verwendet bei vielen Optionen den **ersten** gefundenen Wert. Daher sind Reihenfolge, bestehende Include-Dateien und etwaige `Match`-Blöcke zu berücksichtigen. Nicht davon ausgehen, dass eine alphabetisch spätere Datei frühere Werte überschreibt.

### 6.2 Härtung als Drop-in

Vorher vorhandene Definitionen der folgenden Optionen prüfen:

```bash
sudo grep -RniE '^[[:space:]]*(Include|Port|PermitRootLogin|PubkeyAuthentication|PasswordAuthentication|KbdInteractiveAuthentication|AuthenticationMethods|AllowUsers|X11Forwarding|AllowTcpForwarding|MaxAuthTries|LoginGraceTime)' /etc/ssh/sshd_config /etc/ssh/sshd_config.d/ 2>/dev/null
```

**Beispiel für `/etc/ssh/sshd_config.d/00-local-hardening.conf`:**

```sshconfig
# Lokale SSH-Sicherheitsvorgaben für server1 (OpenSSH-Server)
# Root darf sich nicht direkt per SSH anmelden; Administration über sudo.
PermitRootLogin no

# Anmeldung mit SSH-Schlüsseln zulassen.
PubkeyAuthentication yes

# Passwortanmeldung deaktivieren: vorher Schlüsselanmeldung erfolgreich testen!
PasswordAuthentication no

# Interaktive Authentifizierung (z. B. PAM-Passwortabfragen) deaktivieren.
KbdInteractiveAuthentication no

# Konten ohne Passwort dürfen sich nicht per Passwort anmelden.
PermitEmptyPasswords no

# X11-Grafikweiterleitung über SSH deaktivieren.
X11Forwarding no

# TCP-Portweiterleitungen (lokal, remote und dynamisch) verbieten.
# Nicht verwenden, wenn SSH-Tunnel für die Administration benötigt werden.
AllowTcpForwarding no

# Höchstens drei fehlgeschlagene Authentifizierungsversuche pro Verbindung.
MaxAuthTries 3

# Nicht authentifizierte Verbindungen nach 30 Sekunden beenden.
LoginGraceTime 30

# Nur diesem Benutzer SSH-Anmeldungen erlauben.
# WICHTIG: 'adminuser' durch den tatsächlichen Admin-Account ersetzen!
# Vor Aktivierung prüfen, dass der Account existiert und sich per Schlüssel anmelden kann.
AllowUsers adminuser
```

Datei mit Root-Rechten anlegen, z. B. `sudoedit /etc/ssh/sshd_config.d/00-local-hardening.conf`, und Berechtigungen setzen:

```bash
sudo chown root:root /etc/ssh/sshd_config.d/00-local-hardening.conf
sudo chmod 644 /etc/ssh/sshd_config.d/00-local-hardening.conf
```

**Bedeutung und Anpassung:** `AllowUsers` lässt ausschließlich genannte Konten zu und kann andere Administratoren oder Automationskonten aussperren. `AllowTcpForwarding no` deaktiviert SSH-Tunnel, `X11Forwarding no` die X11-Weiterleitung. Diese Optionen nur setzen, wenn die Funktionen nicht benötigt werden. Für SFTP sind zusätzliche Anforderungen und gegebenenfalls `Match`-Regeln zu berücksichtigen. `PasswordAuthentication no` und `KbdInteractiveAuthentication no` verhindern die üblichen passwortbasierten SSH-Anmeldewege; die Wirksamkeit ist mit der effektiven Konfiguration und realen Tests zu verifizieren.

### 6.3 Effektive Konfiguration prüfen und aktivieren

```bash
sudo sshd -t
sudo sshd -T -C user=adminuser,host=server1,addr=192.168.1.223 | grep -E '^(port|permitrootlogin|pubkeyauthentication|passwordauthentication|kbdinteractiveauthentication|allowusers|allowtcpforwarding|x11forwarding|maxauthtries|logingracetime) '
```

Der Parameter `addr=` ist die **Quell-IP von client1**, nicht die IP des SSH-Servers. `-C` berücksichtigt bedingte `Match`-Regeln für den Testkontext. Weichen Werte ab, Include-Reihenfolge und andere Konfigurationsdateien korrigieren.

**Erst wenn Schlüsselanmeldung getestet ist**, Dienst neu laden:

```bash
sudo systemctl reload ssh
sudo systemctl status ssh --no-pager
```

Eine bestehende Sitzung offen halten und von einem **zweiten Terminal** auf `client1` testen:

```bash
ssh -i ~/.ssh/id_ed25519_server1 -o IdentitiesOnly=yes adminuser@server1.srv.lcd
```

Wenn der neue Login scheitert, die ursprüngliche Sitzung für Diagnose und Rollback nutzen. Erst nach erfolgreichem Zweittest die erste Sitzung beenden.

## 7. SSH-Clientkonfiguration auf client1

Als `adminuser` die Datei `~/.ssh/config` anlegen:

```sshconfig
Host server1
    HostName server1.srv.lcd
    User adminuser
    Port 22
    IdentityFile ~/.ssh/id_ed25519_server1
    IdentitiesOnly yes
    ForwardAgent no
    StrictHostKeyChecking ask
```

Dateirechte:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
ssh -G server1 | grep -E '^(hostname|user|port|identityfile|identitiesonly|forwardagent|stricthostkeychecking) '
ssh server1
```

`StrictHostKeyChecking ask` fordert bei bislang unbekannten Host-Keys eine Bestätigung an; der Fingerprint muss vor der Annahme unabhängig geprüft werden. Bei einem geänderten bekannten Host-Key darf die Warnung nicht ohne Ursachenklärung umgangen werden.

Optional für passphrasegeschützte Schlüssel:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_server1
ssh-add -l
```

Ein Agent speichert entschlüsselte Schlüssel temporär im Benutzerkontext; Agent-Forwarding bleibt standardmäßig deaktiviert.

## 8. Netzwerk und Firewall

Auf `server1` muss eingehendes TCP/22 von den **autorisierten Quellnetzen** erreichbar sein. Eine Firewall-Regel sollte den Zugriff möglichst auf Administrations-IP-Adressen bzw. ein Managementnetz begrenzen.

### 8.1 Beispiel mit UFW (nur wenn UFW als Firewall vorgesehen ist)

```bash
sudo apt install ufw
sudo ufw status verbose
sudo ufw allow from 192.168.1.223 to any port 22 proto tcp comment 'SSH client1'
```

**Vor `ufw enable`:** vorhandene Firewall-Architektur, weitere benötigte Dienste, IPv6, Konsolenzugriff und die Erreichbarkeit des Administrationsclients prüfen. UFW nicht unkoordiniert parallel zu bestehenden nftables-/iptables-Regeln betreiben.

```bash
sudo ufw enable
sudo ufw status numbered
```

Auf `client1` ist normalerweise keine eingehende SSH-Freigabe nötig; ausgehende Verbindungen zum Zielport müssen jedoch erlaubt sein. Ein anderer SSH-Port ist allein keine belastbare Sicherheitsmaßnahme und muss konsistent in Server, Firewall und Client eingetragen werden.

## 9. Funktions- und Sicherheitsprüfungen

| Test | Befehl / Vorgehen | Erwartetes Ergebnis |
|---|---|---|
| Dienst läuft | `sudo systemctl is-active ssh` auf `server1` | `active` |
| Konfiguration gültig | `sudo sshd -t` | keine Fehlermeldung |
| Port erreichbar | `ssh -v server1` auf `client1` | Verbindung möglich |
| Schlüssel-Login | `ssh -o PreferredAuthentications=publickey server1` | Anmeldung erfolgreich |
| Passwortlogin deaktiviert | `ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password,keyboard-interactive server1` | Anmeldung abgelehnt (kein Passwort-Login) |
| Root-Login deaktiviert | Test mit separatem, bewusst begrenztem Prüfversuch | Root-Anmeldung nicht möglich |
| Effektive Richtlinie | `sudo sshd -T -C user=adminuser,host=server1,addr=192.168.1.223` | erwartete Werte |
| Rechte | `stat -c '%a %U:%G %n' ~/.ssh ~/.ssh/authorized_keys` auf `server1` | i. d. R. `700`, `600` |

**Wichtig:** Passworttest nicht in einer Schleife durchführen; Sperrmechanismen können greifen. Ein fehlgeschlagener Login allein beweist nicht, dass der gewünschte Authentifizierungsweg deaktiviert ist: `sshd -T` und Logs zusätzlich prüfen.

## 10. Protokolle und Diagnose

Auf `server1`:

```bash
sudo journalctl -u ssh -n 100 --no-pager
sudo journalctl -u ssh --since '1 hour ago' --no-pager
sudo systemctl status ssh --no-pager
sudo ss -ltnp | grep ssh
```

Je nach Logging-Konfiguration stehen SSH-Ereignisse auch in `/var/log/auth.log`. Die Datei muss nicht in jeder Installation vorhanden sein.

Auf `client1`:

```bash
ssh -v server1
ssh -vvv server1
ssh -G server1
```

`-vvv` erzeugt umfangreiche Diagnoseinformationen. Vor dem Teilen von Logs personenbezogene Angaben, interne Adressen und sensible Daten prüfen.

### 10.1 Typische Fehler

| Meldung / Symptom | Mögliche Ursache | Prüfung / Lösung |
|---|---|---|
| `Connection refused` | Dienst nicht aktiv oder falscher Port | `systemctl status ssh`, `ss -ltnp`, Port prüfen |
| `Connection timed out` | Routing, ACL, Firewall, Ziel nicht erreichbar | DNS, `ip route`, Firewall und Netzpfad prüfen |
| `Permission denied (publickey)` | falscher Schlüssel, falscher Benutzer, Rechteproblem | `ssh -vvv`, `authorized_keys`, Eigentümer/Rechte, Logs |
| `REMOTE HOST IDENTIFICATION HAS CHANGED` | Host-Key geändert oder Angriff möglich | Fingerprint über unabhängigen Kanal verifizieren; **nicht blind löschen** |
| `Bad configuration option` | Tippfehler / inkompatible Direktive | `sudo sshd -t`, betroffene Datei korrigieren |
| `Too many authentication failures` | Agent bietet zu viele Schlüssel an | `IdentitiesOnly yes`, `IdentityFile` prüfen |
| `sudo` nicht erlaubt | fehlende Berechtigung | `id`, Gruppen und sudoers über Konsole prüfen |

## 11. Betrieb und Wartung

### 11.1 Sicherheitsupdates

Auf beiden Systemen regelmäßig:

```bash
sudo apt update
apt list --upgradable
sudo apt upgrade
apt-cache policy openssh-server openssh-client
```

Updates nach Wartungs- und Freigabeprozess durchführen. Bei SSH-/Netzwerkänderungen eine Rettungssitzung offen halten und nach dem Update die Verbindung erneut testen. Debian-12-Sicherheits- und Supportstatus regelmäßig anhand offizieller Debian-Hinweise prüfen.

### 11.2 Neue Administratoren aufnehmen

Für jede Person ein eigenes Konto und einen eigenen öffentlichen Schlüssel vorsehen. Berechtigungen dokumentieren und freigeben; Schlüssel **nicht** zwischen Personen teilen. Auf `server1` den geprüften öffentlichen Schlüssel in der passenden `authorized_keys`-Datei ergänzen, Eigentümer und Dateirechte kontrollieren und einen Testlogin durchführen. Bei Verwendung von `AllowUsers` das neue Konto dort berücksichtigen und nach `sshd -t` neu laden.

### 11.3 Zugänge entziehen

Bei Austritt oder Schlüsselkompromittierung den betreffenden öffentlichen Schlüssel aus `authorized_keys` entfernen, ggf. das Konto sperren, Gruppen-/sudo-Rechte entziehen und aktive Sitzungen nach dem festgelegten Incident-/Offboarding-Prozess beenden. Auf gemeinsam genutzten Systemen weitere Berechtigungen (Cron, Automationsschlüssel, SSH-Zertifikate, zentrale Identitätsverwaltung) prüfen.

### 11.4 Backups und Wiederherstellung

Sichern: `/etc/ssh/sshd_config`, `/etc/ssh/sshd_config.d/`, freigegebene Benutzer-/Berechtigungsinformationen und – entsprechend der Sicherheitsrichtlinie – Host-Keys unter `/etc/ssh/ssh_host_*`. **Private Host-Keys** nur verschlüsselt und zugriffsbeschränkt sichern; niemals in Git committen. Benutzer-private Schlüssel bleiben auf den jeweiligen Clients bzw. in einer genehmigten Schlüsselverwaltung.

Bei einer Neuinstallation ist zu entscheiden, ob die alte Host-Identität kontrolliert wiederhergestellt oder ein neuer Host-Key mit dokumentierter Fingerprint-Verteilung verwendet wird. Unkontrollierte Host-Key-Wechsel erzeugen Sicherheitswarnungen bei Clients.

## 12. Rollback bei fehlerhafter SSH-Konfiguration

Nur über eine **noch funktionierende SSH-Sitzung oder die lokale/Out-of-Band-Konsole**:

1. Fehler mit `sudo sshd -t` und `sudo journalctl -u ssh -n 100` prüfen.
2. Zuletzt hinzugefügtes Drop-in deaktivieren, beispielsweise:

   ```bash
   sudo mv /etc/ssh/sshd_config.d/00-local-hardening.conf /etc/ssh/sshd_config.d/00-local-hardening.conf.disabled
   sudo sshd -t
   sudo systemctl reload ssh
   ```

3. Von `client1` in einem zweiten Terminal erneut verbinden.
4. Ursache, Änderung, Zeit und Ergebnis im Betriebs-/Change-Protokoll dokumentieren.

Bei Änderungen an der Hauptdatei die zuvor erstellte Sicherung gezielt zurückspielen; nicht blind eine unbekannte ältere Konfiguration übernehmen.

## 13. Git-Repository und Umgang mit Geheimnissen

Empfohlene Struktur:

```text
infrastructure-docs/
├── README.md
├── docs/
│   └── ssh-debian12-server1-client1.md
└── .gitignore
```

Diese Datei kann als `docs/ssh-debian12-server1-client1.md` abgelegt werden. In Git gehören **keine** privaten SSH-Schlüssel, Passphrasen, Zugangsdaten, vollständigen personenbezogenen Logdateien oder ungeschützt sicherheitskritische Betriebsgeheimnisse. Öffentliche Schlüssel und interne IP-Adressen nur nach Organisationsrichtlinie veröffentlichen.

Beispiel für `.gitignore` (zusätzlicher Schutz, **kein** Ersatz für Prüfung vor dem Commit):

```gitignore
# SSH private keys / sensitive exports
id_rsa
id_rsa_*
id_ed25519
id_ed25519_*
*.pem
*.key
*.p12
*.pfx
.env
.env.*
```

Die Muster können auch harmlose Dateien treffen und schützen nicht vor bereits getrackten Dateien. Vor jedem Commit `git diff --cached` kontrollieren und gegebenenfalls einen Secret-Scanner im CI einsetzen.

```bash
git add docs/ssh-debian12-server1-client1.md
git diff --cached
git commit -m "docs: SSH-Betriebshandbuch für Debian 12 ergänzen"
```

## 14. Dokumentationspflicht und Übergabe

Folgende Werte müssen vor der Produktivfreigabe **intern und zugriffsgeschützt** vervollständigt werden:

| Feld | Einzutragender Wert |
|---|---|
| Verantwortliches Team / Kontakt | TODO |
| Reale IP und DNS von `server1` | TODO |
| Reale IP und DNS von `client1` | TODO |
| Administrationskonten / Rollen | TODO |
| Erlaubte Quellnetze / Firewall-Regeln | TODO |
| Host-Key-Fingerprint (`server1`, Ed25519) | TODO |
| Backup- und Wiederherstellungsort | TODO |
| Konsolen-/Out-of-Band-Rettungsweg | TODO |
| Change-/Ticket-Referenz | TODO |
| Datum des letzten erfolgreichen SSH-Tests | TODO |
| Nächster Review-Termin | TODO |

### 14.1 Abnahmecheckliste

- [ ] Debian-Version und Hostnamen bestätigt
- [ ] Reale Adressen, DNS und Routing geprüft
- [ ] OpenSSH auf beiden Systemen installiert
- [ ] `ssh.service` auf `server1` aktiv
- [ ] Individuelles Administrationskonto eingerichtet
- [ ] Ed25519-Schlüssel auf `client1` erstellt und geschützt
- [ ] Öffentlicher Schlüssel auf `server1` hinterlegt
- [ ] Host-Key-Fingerprint unabhängig verifiziert
- [ ] Schlüsselanmeldung in zweiter Sitzung erfolgreich
- [ ] Effektive `sshd`-Konfiguration kontrolliert
- [ ] Root- und Passwort-Login gemäß Richtlinie deaktiviert
- [ ] Firewall-Zugriff auf autorisierte Quellen begrenzt
- [ ] `~/.ssh/config` auf `client1` getestet
- [ ] Wiederherstellungsweg und Backup dokumentiert
- [ ] Verantwortliche, Freigabe und Review-Termin erfasst
- [ ] Git-Diff auf Geheimnisse geprüft

## 15. Weiterführende Quellen

- Debian OpenSSH-Paketinformationen: <https://packages.debian.org/bookworm/openssh-server>
- Debian Administrator's Handbook: <https://debian-handbook.info/>
- OpenSSH-Handbuchseiten auf Debian: `man ssh`, `man sshd`, `man ssh_config`, `man sshd_config`, `man ssh-keygen`, `man ssh-copy-id`
- Debian Security: <https://www.debian.org/security/>

---

**Änderungshistorie**

| Datum | Version | Änderung | Verantwortlich |
|---|---|---|---|
| 2026-11-15 | 1.0 | Erstdokumentation als anpassbare Vorlage | TODO |
