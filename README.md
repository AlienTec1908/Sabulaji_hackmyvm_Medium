# Sabulaji - HackMyVM Writeup

![Sabulaji Icon](Sabulaji.png)

## Übersicht

*   **VM:** Sabulaji
*   **Plattform:** [HackMyVM](https://hackmyvm.eu/machines/machine.php?vm=Sabulaji)
*   **Schwierigkeit:** Medium
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 25. Juni 2025
*   **Original-Writeup:** https://alientec1908.github.io/Sabulaji_hackmyvm_Medium/
*   **Autor:** Ben C.

---

**Disclaimer:**

Dieser Writeup dient ausschließlich zu Bildungszwecken und dokumentiert Techniken, die in einer kontrollierten Testumgebung (HackTheBox/HackMyVM) angewendet wurden. Die Anwendung dieser Techniken auf Systeme, für die keine ausdrückliche Genehmigung vorliegt, ist illegal und ethisch nicht vertretbar. Der Autor und der Ersteller dieses README übernehmen keine Verantwortung für jeglichen Missbrauch der hier beschriebenen Informationen.

---

## Zusammenfassung

Die Box "Sabulaji" begann mit einer Standard-Enumeration offener Dienste. Dabei fielen der offene FTP-Port (21), ein Apache Webserver (80) und besonders der rsync-Dienst (873) auf. Der rsync-Dienst enthüllte zwei Module: "public" und das passwortgeschützte "epages". Das Herunterladen der öffentlichen Dateien lieferte eine To-Do-Liste, die den Benutzernamen "sabulaji" und Passwort-Hinweise enthielt. Durch einen Brute-Force-Angriff auf das rsync-Modul "epages" konnte das Passwort für "sabulaji" gefunden werden.

Der Download der "epages"-Dateien enthüllte ein Dokument mit einem Hinweis auf einen Benutzer "welcome" und ein Passwort "P@ssw0rd123!", das für SSH verwendet werden konnte. Der SSH-Login als "welcome" war erfolgreich.

Als Benutzer "welcome" wurde eine `sudo`-Regel gefunden, die das Ausführen des Skripts `/opt/sync.sh` als Benutzer "sabulaji" ohne Passwort erlaubte. Die Analyse dieses Skripts und die Untersuchung des Dateisystems (insbesondere der `mlocate.db`) führten zur Entdeckung einer Datei namens "creds.txt" im persönlichen Verzeichnis von "sabulaji". Das Ausführen des `/opt/sync.sh`-Skripts mit "creds.txt" als Parameter enthüllte mittels eines `diff`-Outputs Base64-kodierte Anmeldedaten.

Die dekodierten Anmeldedaten (gasparin) oder die Base64-kodierte Form (Z2FzcGFyaW4=) ermöglichten den Login als Benutzer "sabulaji". Als "sabulaji" wurde eine weitere `sudo`-Regel gefunden, die die Ausführung von `/usr/bin/rsync` als Root ohne Passwort erlaubte. Die Ausnutzung einer bekannten Schwachstelle in `rsync` mit Sudo-Berechtigungen ermöglichte schließlich die Erlangung einer Root-Shell.

## Technische Details

*   **Betriebssystem:** Debian (basierend auf Nmap-Erkennung)
*   **Offene Ports:**
    *   `22/tcp`: SSH (OpenSSH 8.4p1)
    *   `80/tcp`: HTTP (Apache httpd 2.4.62)
    *   `873/tcp`: rsync (Protokoll Version 31)

## Enumeration

1.  **ARP-Scan:** Identifizierung der Ziel-IP (192.168.2.36).
2.  **`/etc/hosts` Eintrag:** Hinzufügen von `sabu.hmv` zur lokalen hosts-Datei.
3.  **Nmap Scan:** Identifizierung offener Ports 22 (SSH), 80 (HTTP) und 873 (rsync).
4.  **Web Enumeration (Port 80):** Zeigte eine "epages"-Seite und einen Hinweis auf "Sabulaji" in der chinesischen Internetkultur, was auf den Namen "sabulaji" als potenziellen Benutzer oder Thema hindeutete.
5.  **Rsync Enumeration (Port 873):** Der `rsync-list-modules` Nmap-Script oder `nc` zeigte, dass der rsync-Dienst verfügbar war und die Module `public` und `epages` anbot. Tests zeigten, dass `epages` passwortgeschützt war.

## Initialer Zugriff (rsync & SSH als welcome)

1.  **Rsync Pull (public):** Das Modul `public` war ohne Authentifizierung zugänglich. `rsync -av rsync://192.168.2.36/public/ ./public_files/` lud die Datei `todo.list` herunter.
2.  **`todo.list` Analyse:** Die To-Do-Liste enthielt Einträge, die auf den Benutzernamen "sabulaji" abzielten ("Remove private sharing settings", "Change to a strong password").
3.  **Rsync Passwort Brute-Force:** Basierend auf dem Benutzernamen "sabulaji" wurde ein Brute-Force-Angriff auf das rsync-Modul `epages` durchgeführt. Das Skript `while read -r p; do export RSYNC_PASSWORD="$p"; ... rsync -q --no-motd rsync://sabulaji@192.168.2.36/epages/ ...` fand das Passwort `admin123`.
4.  **Rsync Pull (epages):** Mit den Anmeldedaten `sabulaji:admin123` konnte das rsync-Modul `epages` heruntergeladen werden (`rsync -av rsync://sabulaji@192.168.2.36/epages/ ./epages_files/`).
5.  **`secrets.doc` Fund:** In den heruntergeladenen "epages"-Dateien wurde ein Dokument gefunden (`secrets.doc` oder ähnlich), das einen Hinweis auf den Benutzer "welcome" und das Passwort "P@ssw0rd123!" für einen FTP-Dienst enthielt (trotz der Nmap-Ergebnisse, die auch SSH offen zeigten). Der Hinweis schlug vor, es für den SSH-Login zu verwenden.
6.  **SSH Login:** Mit den Anmeldedaten `welcome:P@ssw0rd123!` war der SSH-Login erfolgreich.

## Lateral Movement (welcome -> sabulaji)

1.  **Systemerkundung als `welcome`:** Im Home-Verzeichnis von `welcome` wurde die `user.txt` gefunden.
2.  **User Flag:** Der Inhalt von `/home/welcome/user.txt` war `flag{user-cf7883184194add6adfa5f20b5061ac7}`.
3.  **Sudo-Regel für `welcome`:** `sudo -l` als `welcome` zeigte: `(sabulaji) NOPASSWD: /opt/sync.sh`. `welcome` konnte `/opt/sync.sh` als `sabulaji` ohne Passwort ausführen.
4.  **`/opt/sync.sh` Analyse:** Das Skript nimmt einen Dateipfad als Argument, vergleicht diese Datei mit `/home/sabulaji/personal/notes.txt` und gibt die Unterschiede aus, bevor die angegebene Datei nach `/home/sabulaji/personal/notes.txt` kopiert wird. Es gab eine Filterung, die das Wort "sabulaji" im Input verbot.
5.  **`mlocate.db` Analyse:** Die Datei `/var/lib/mlocate/mlocate.db` konnte von `welcome` gelesen werden. Durch das Auslesen der Strings (`strings /var/lib/mlocate/mlocate.db`) und Filtern nach Pfaden zu sabulajis Home-Verzeichnis und potentiell interessanten Dateinamen wurde die Datei `/home/sabulaji/personal/creds.txt` identifiziert.
6.  **Information Leak via `sync.sh`:** Das Skript `/opt/sync.sh` wurde ausgenutzt, um den Inhalt von `/home/sabulaji/personal/creds.txt` zu lesen, indem es mit diesem Pfad als Argument ausgeführt wurde (`sudo -u sabulaji /opt/sync.sh /home/sabulaj*/personal/creds.txt`). Der `diff`-Output enthüllte den Inhalt der Datei: `Sensitive Credentials:Z2FzcGFyaW4=`.
7.  **Base64 Decode:** Die Base64-kodierte Zeichenkette `Z2FzcGFyaW4=` wurde dekodiert und ergab `gasparin`. Dies war das Passwort für den Benutzer `sabulaji`.
8.  **Wechsel zu sabulaji:** Mit den Anmeldedaten `sabulaji:Z2FzcGFyaW4=` (die Base64-kodierte Form, `gasparin` funktionierte nicht direkt mit `su`) war der Wechsel zum Benutzer `sabulaji` mittels `su sabulaji` erfolgreich.

## Privilegieneskalation (sabulaji -> root)

1.  **Sudo-Regel für `sabulaji`:** Als Benutzer `sabulaji` wurde `sudo -l` ausgeführt. Die entscheidende `sudo`-Regel erlaubte die Ausführung von `/usr/bin/rsync` als Root ohne Passwort: `(ALL) NOPASSWD: /usr/bin/rsync`.
2.  **rsync Sudo Exploit:** Das Binary `/usr/bin/rsync` kann mit Sudo-Berechtigungen ausgenutzt werden, um Befehle als Root auszuführen (bekannter Exploit auf GTFOBins).
3.  **Root Shell:** Der Befehl `sudo rsync -e 'sh -c "sh 0<&2 1>&2"' 127.0.0.1:/dev/null` wurde ausgeführt, was zur Etablierung einer interaktiven Root-Shell führte.

## Finalisierung (Root Shell)

1.  **Root-Zugriff:** In der Root-Shell konnte auf das Root-Verzeichnis zugegriffen werden.
2.  **Root Flag:** Die Datei `root.txt` im `/root`-Verzeichnis wurde gefunden und ihr Inhalt ausgelesen.

## Flags

*   **user.txt:** `flag{user-cf7883184194add6adfa5f20b5061ac7}` (Gefunden unter `/home/welcome/user.txt`)
*   **root.txt:** `flag{root-89e62d8807f7986edb259eb2237d011c}` (Gefunden unter `/root/root.txt`)

---
