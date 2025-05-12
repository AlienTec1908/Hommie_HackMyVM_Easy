# Hommie (HackMyVM) - Penetration Test Bericht

![Hommie.png](Hommie.png)

**Datum des Berichts:** 9. Oktober 2022  
**VM:** Hommie  
**Plattform:** HackMyVM ([Link zur VM](https://hackmyvm.eu/machines/machine.php?vm=Hommie))  
**Autor der VM:** DarkSpirit  
**Original Writeup:** [https://alientec1908.github.io/Hommie_HackMyVM_Easy/](https://alientec1908.github.io/Hommie_HackMyVM_Easy/)

---

## Disclaimer

**Wichtiger Hinweis:** Dieser Bericht und die darin enthaltenen Informationen dienen ausschließlich zu Bildungs- und Forschungszwecken im Bereich der Cybersicherheit. Die hier beschriebenen Techniken und Werkzeuge dürfen nur in legalen und autorisierten Umgebungen (z.B. auf eigenen Systemen oder mit ausdrücklicher Genehmigung des Eigentümers) angewendet werden. Jegliche illegale Nutzung der hier bereitgestellten Informationen ist strengstens untersagt. Der Autor übernimmt keine Haftung für Schäden, die durch Missbrauch dieser Informationen entstehen. Handeln Sie stets verantwortungsbewusst und ethisch.

---

## Inhaltsverzeichnis

1.  [Zusammenfassung](#zusammenfassung)
2.  [Verwendete Tools](#verwendete-tools)
3.  [Phase 1: Reconnaissance](#phase-1-reconnaissance)
4.  [Phase 2: Information Disclosure & Initial Access](#phase-2-information-disclosure--initial-access)
5.  [Phase 3: Privilege Escalation (SUID Binary)](#phase-3-privilege-escalation-suid-binary)
6.  [Proof of Concept (Root)](#proof-of-concept-root)
7.  [Flags](#flags)
8.  [Empfohlene Maßnahmen (Mitigation)](#empfohlene-maßnahmen-mitigation)

---

## Zusammenfassung

Dieser Bericht dokumentiert die Kompromittierung der virtuellen Maschine "Hommie" von HackMyVM (Schwierigkeitsgrad: Easy). Die initiale Erkundung offenbarte offene FTP-, SSH- und HTTP-Dienste. Obwohl Nmap einen anonymen FTP-Zugriff meldete, war der entscheidende Hinweis eine Nachricht in der `index.html` des Webservers. Diese Nachricht wies auf einen exponierten SSH-Schlüssel (`id_rsa`) für den Benutzer `alexia` hin.

Dieser private SSH-Schlüssel wurde über einen ungesicherten **TFTP-Dienst** (UDP Port 69, der von Nmap initial nicht gescannt wurde) heruntergeladen. Der Login als `alexia` per SSH war damit erfolgreich. Die Privilegieneskalation zu Root-Rechten erfolgte durch Ausnutzung eines benutzerdefinierten SUID-Root-Binaries `/opt/showMetheKey`. Die Ausführung dieses Programms gab entweder direkt eine Root-Shell oder, wahrscheinlicher, den privaten SSH-Schlüssel des Root-Benutzers aus, was den finalen Zugriff ermöglichte.

---

## Verwendete Tools

*   `arp-scan`
*   `nmap`
*   `gobuster`
*   `tftp`
*   `chmod`
*   `ssh`
*   `cat`
*   `find`
*   `/opt/showMetheKey` (benutzerdefiniertes SUID-Binary)
*   `export` (für Shell-Variablen, falls verwendet)
*   `id`, `cd`, `grep`, `tail`

---

## Phase 1: Reconnaissance

1.  **Netzwerk-Scan:**
    *   `arp-scan -l` identifizierte das Ziel `192.168.2.100` (VirtualBox VM).

2.  **Port-Scan (Nmap):**
    *   Ein umfassender `nmap`-Scan (`nmap -sS -sC -T5 -sV -A 192.168.2.100 -p-`) offenbarte:
        *   **Port 21 (FTP):** vsftpd 3.0.3. **Kritisch:** Anonymer Login erlaubt (`ftp-anon: Anonymous FTP login allowed`).
        *   **Port 22 (SSH):** OpenSSH 7.9p1 Debian
        *   **Port 80 (HTTP):** nginx 1.14.2

---

## Phase 2: Information Disclosure & Initial Access

1.  **Web Enumeration:**
    *   `gobuster dir` auf Port 80 fand nur `/index.html`.
    *   Der Inhalt der `/index.html` (oder ein Kommentar darin) enthielt eine entscheidende Nachricht von "nobody" an "alexia":
        ```
        alexia, Your id_rsa is exposed, please move it!!!!!
        Im fighting regarding reverse shells! -nobody
        ```
        Dies bestätigte den Benutzernamen `alexia` und das Vorhandensein eines exponierten privaten SSH-Schlüssels.

2.  **Auffinden des SSH-Schlüssels via TFTP:**
    *   Obwohl Nmap keinen TFTP-Dienst (UDP Port 69) anzeigte (da kein expliziter UDP-Scan durchgeführt wurde), wurde basierend auf dem Hinweis versucht, die Datei `id_rsa` über TFTP herunterzuladen:
        ```bash
        tftp 192.168.2.100
        tftp> get id_rsa
        # Received 1850 bytes in 0.0 seconds
        tftp> quit
        ```
    *   Der private SSH-Schlüssel für `alexia` wurde erfolgreich über TFTP erlangt.

3.  **SSH-Login als `alexia`:**
    *   Die Berechtigungen des heruntergeladenen Schlüssels wurden angepasst: `chmod 600 id_rsa`.
    *   Der SSH-Login als `alexia` mit dem Schlüssel war erfolgreich:
        ```bash
        ssh -i id_rsa alexia@192.168.2.100
        ```
    *   Initialer Zugriff als `alexia` auf dem System `hommie` wurde erlangt.

---

## Phase 3: Privilege Escalation (SUID Binary)

1.  **Enumeration als `alexia`:**
    *   Die User-Flag wurde in `/home/alexia/user.txt` gefunden: `Imnotroot`.
    *   Die Suche nach SUID-Binaries (`find / -type f -perm -4000 -ls 2>/dev/null`) offenbarte ein benutzerdefiniertes SUID-Root-Binary: `/opt/showMetheKey`.

2.  **Ausnutzung des SUID-Binaries:**
    *   Die Ausführung von `/opt/showMetheKey` als `alexia`:
        ```bash
        alexia@hommie:~$ /opt/showMetheKey
        ```
    *   Das Programm gab entweder direkt eine Root-Shell oder, was im Log wahrscheinlicher dargestellt wird, den privaten SSH-Schlüssel des **Root-Benutzers** aus. Das Log zeigt direkt einen Wechsel zu einem Root-Prompt (`root@hommie:/opt#`) nach der Ausführung, was auf eine direkte Shell hindeutet, oder dass der ausgegebene Schlüssel umgehend für einen SSH-Login als Root verwendet wurde (dieser Schritt wäre dann impliziert).

---

## Proof of Concept (Root)

**Kurzbeschreibung:** Die Privilegieneskalation erfolgte durch Ausführung des SUID-Root-Binaries `/opt/showMetheKey`. Dieses Programm gab entweder direkt eine Root-Shell oder den privaten SSH-Schlüssel des Root-Benutzers aus, was den Zugriff als Root ermöglichte.

**Schritte (als `alexia`):**
1.  Führe das SUID-Binary aus:
    ```bash
    /opt/showMetheKey
    ```
**Ergebnis (basierend auf Log-Fortschritt):**
*   Eine Shell mit `uid=0(root)` wird gestartet, ODER
*   Der private SSH-Schlüssel des Root-Benutzers wird ausgegeben, der dann für einen SSH-Login als `root` verwendet wird.

(Der Befehl `id` nach Erhalt der Root-Shell bestätigte: `uid=0(root) gid=0(root) groups=0(root),alexia`)

---

## Flags

*   **User Flag (`/home/alexia/user.txt`):**
    ```
    Imnotroot
    ```
*   **Root Flag (`/usr/include/root.txt`):**
    ```
    Imnotbatman
    ```

---

## Empfohlene Maßnahmen (Mitigation)

*   **TFTP-Sicherheit:**
    *   **DRINGEND:** Deaktivieren Sie den TFTP-Dienst, wenn er nicht betriebsnotwendig ist. TFTP ist von Natur aus unsicher und sollte nicht in unsicheren Umgebungen oder für den Transfer sensibler Daten verwendet werden.
    *   Wenn TFTP benötigt wird, konfigurieren Sie ihn sicher: Beschränken Sie den Zugriff auf spezifische, nicht-sensible Verzeichnisse und auf autorisierte IP-Adressen.
*   **SSH-Schlüssel-Management:**
    *   **Speichern Sie private SSH-Schlüssel niemals an öffentlich zugänglichen Orten oder über unsichere Dienste wie TFTP.**
    *   Schützen Sie private SSH-Schlüssel immer mit starken Passphrasen.
    *   Im Falle einer Kompromittierung müssen SSH-Schlüssel umgehend widerrufen und neue, sichere Schlüssel generiert werden.
*   **SUID/GUID-Binaries:**
    *   **DRINGEND:** Überprüfen Sie alle benutzerdefinierten SUID/GUID-Binaries (wie `/opt/showMetheKey`) sorgfältig auf Sicherheitslücken.
    *   Entfernen Sie das SUID/GUID-Bit von Programmen, bei denen es nicht absolut notwendig ist.
    *   Stellen Sie sicher, dass SUID-Programme keine sensiblen Informationen (wie private Schlüssel) preisgeben oder unkontrollierte Shell-Zugriffe ermöglichen.
*   **FTP-Sicherheit:**
    *   Deaktivieren Sie den anonymen FTP-Zugriff, wenn er nicht explizit und sicher benötigt wird.
*   **Informationspreisgabe:**
    *   Vermeiden Sie es, Hinweise auf Benutzernamen oder interne Sicherheitslücken in öffentlichen Dateien (wie `index.html`) zu hinterlassen.
*   **Allgemeine Systemhärtung:**
    *   Halten Sie alle Dienste (SSH, Webserver, FTP) auf dem neuesten Stand.
    *   Überwachen Sie Systemlogs auf verdächtige Aktivitäten und Zugriffsversuche.
    *   Führen Sie regelmäßige Sicherheitsaudits durch.

---

**Ben C. - Cyber Security Reports**
