# DC04 - HackMyVM (Hard)
 
![DC04.png](DC04.png)

## Übersicht

*   **VM:** DC04
*   **Plattform:** HackMyVM (https://hackmyvm.eu/machines/machine.php?vm=DC04)
*   **Schwierigkeit:** Hard
*   **Autor der VM:** DarkSpirit
*   **Datum des Writeups:** 24. Juli 2025
*   **Original-Writeup:** https://alientec1908.github.io/DC04_HackMyVM_Hard/
*   **Autor:** Ben C.

## Kurzbeschreibung

Dieses Writeup beschreibt die Kompromittierung eines Windows Active Directory Domain Controllers. Der Angriffspfad war vielschichtig und umfasste die Enumeration von Web- und AD-Diensten, die Entdeckung von Informationslecks über eine Apache `server-status`-Seite und das Erzwingen einer SMB-Authentifizierung. Ein abgefangener NTLMv2-Hash wurde offline geknackt, um erste Anmeldeinformationen zu erhalten. Die weitere Eskalation erfolgte durch das Auffinden zusätzlicher Klartext-Passwörter in AD-Benutzerbeschreibungen und das Knacken eines passwortgeschützten RAR-Archivs, das einen alten Pentest-Bericht enthielt. Dieser Bericht enthielt den NTHash des `krbtgt`-Kontos, was schließlich einen "Golden Ticket"-Angriff ermöglichte, um volle Domain-Admin-Rechte zu erlangen.

## Disclaimer / Wichtiger Hinweis

Die in diesem Writeup beschriebenen Techniken und Werkzeuge dienen ausschließlich zu Bildungszwecken im Rahmen von legalen Capture-The-Flag (CTF)-Wettbewerben und Penetrationstests auf Systemen, für die eine ausdrückliche Genehmigung vorliegt. Die Anwendung dieser Methoden auf Systeme ohne Erlaubnis ist illegal. Der Autor übernimmt keine Verantwortung für missbräuchliche Verwendung der hier geteilten Informationen. Handeln Sie stets ethisch und verantwortungsbewusst.

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `enum4linux`
*   `curl`
*   `nikto`
*   `gobuster`
*   `responder`
*   `hashcat`
*   `impacket` (smbexec, psexec, GetUserSPNs, lookupsid, ticketer, wmiexec)
*   `smbclient`
*   `nxc` (netexec)
*   `john` (rar2john)
*   Standard Linux-Befehle (`vi`, `cat`, `grep`, `awk`, `date`, etc.)

## Lösungsweg (Zusammenfassung)

Der Angriff auf die Maschine "DC04" gliederte sich in folgende Phasen:

1.  **Reconnaissance & Enumeration:**
    *   Identifizierung der Ziel-IP (`192.168.2.200`) als Windows Domain Controller der Domain `SOUPEDECODE.LOCAL`.
    *   Ein `nmap`-Scan deckte zahlreiche AD-Dienste (Kerberos, LDAP, SMB) und einen Apache-Webserver auf Port 80 auf. Ein signifikanter `clock-skew` (Zeitunterschied) wurde festgestellt.

2.  **Web & AD Enumeration:**
    *   Der Webserver leitete auf einen virtuellen Host `soupedecode.local` um, der eine Endlos-Umleitungsschleife aufwies.
    *   Eine öffentlich zugängliche `/server-status`-Seite des Apache-Servers offenbarte die Existenz eines internen Webdienstes auf `localhost:8080`.
    *   Da die Webanwendung fehlerhaft war, wurde die Taktik geändert, um eine SMB-Authentifizierung vom Server zu provozieren.

3.  **Initial Access (NTLM Hash Cracking):**
    *   `Responder` wurde gestartet, um auf SMB-Authentifizierungsversuche zu lauschen.
    *   Durch eine Interaktion mit der Webanwendung wurde der Server dazu gebracht, eine Verbindung zu Responder aufzubauen.
    *   Der NTLMv2-Hash des Dienstkontos `soupedecode\websvc` wurde abgefangen.
    *   `Hashcat` wurde verwendet, um den Hash mit der `rockyou.txt`-Wortliste zu knacken. Das Passwort wurde als `jordan23` identifiziert.

4.  **Post-Exploitation / Privilege Escalation (von `websvc` zu Domain User):**
    *   Das Passwort für `websvc` war abgelaufen und musste zunächst geändert werden (hier: `password`).
    *   Mit `smbclient` und den neuen Anmeldedaten wurde auf das Dateisystem zugegriffen und die User-Flag gefunden.
    *   Durch die Abfrage von AD-Benutzerbeschreibungen wurde ein Klartextpasswort für den Benutzer `rtina979` gefunden.
    *   Im Home-Verzeichnis von `rtina979` wurde eine passwortgeschützte `Report.rar`-Datei entdeckt. Das Passwort (`PASSWORD123`) wurde mit `john` geknackt.

5.  **Privilege Escalation (Golden Ticket zu Domain Admin):**
    *   Die `Report.rar` enthielt einen alten Pentest-Bericht. In diesem Bericht wurde der NTHash des `krbtgt`-Kontos gefunden.
    *   Die Systemzeit des Angreifer-Systems wurde mit der des Domain Controllers synchronisiert, um Kerberos-Fehler zu vermeiden.
    *   Mit `impacket-ticketer` wurde unter Verwendung des `krbtgt`-Hashes ein "Golden Ticket" (ein gefälschtes Kerberos TGT) für den `Administrator`-Benutzer erstellt.
    *   Dieses Ticket wurde in die Umgebung geladen und `impacket-wmiexec` wurde verwendet, um sich mit Kerberos-Authentifizierung am Domain Controller anzumelden und eine administrative Shell zu erhalten.

## Wichtige Schwachstellen und Konzepte

*   **Informationslecks (Apache Server-Status):** Eine öffentlich zugängliche Statusseite deckte interne Dienste und Konfigurationen auf.
*   **LLMNR/NBT-NS Poisoning:** Das Ausnutzen veralteter Namensauflösungsprotokolle, um SMB-Authentifizierungs-Hashes abzufangen.
*   **Schwache Passwörter:** Das Knacken des `websvc`-Passworts war aufgrund seiner Einfachheit schnell möglich. Ein weiteres Passwort wurde im Klartext in einem AD-Benutzerattribut gefunden.
*   **Golden Ticket Attack:** Eine fortgeschrittene Active Directory-Angriffstechnik. Durch den Besitz des `krbtgt`-Hashes können willkürliche Kerberos-Tickets erstellt werden, was einem persistenten Domain-Admin-Zugriff gleichkommt.
*   **Unsichere Speicherung sensibler Daten:** Ein alter Pentest-Bericht mit kritischen Informationen (Credentials, Hashes) wurde unsicher im Netzwerk belassen.

## Flags

*   **User Flag (`C:\Users\websvc\Desktop\user.txt`):** `709e449a996a85aa7deaf18c79515d6a`
*   **Root Flag (`C:\users\administrator\desktop\root.txt`):** `1c66eabe105636d7e0b82ec1fa87cb7a`

## Tags

`HackMyVM`, `DC04`, `Hard`, `Active Directory`, `Windows`, `Golden Ticket`, `Kerberos`, `Responder`, `Hashcat`, `Impacket`
