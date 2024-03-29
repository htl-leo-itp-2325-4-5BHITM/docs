# Deployen zur Leocloud

## Generelles
Alles was `<zwischen spizklammern>` steht muss durch irgendwas ersetzt werden. Immer auch die spizklammern ersetzten und wenn vorhanden das trennzeichen "-" oder so beachten. `<dein-name>` wird zu `yanik-kendler`

Screen ist eine utility für den install von sachen, dass die nicht abgebrochen werden wenn der user von ssh disconnected. Alle apt installs sollten im `screen` ausgeführt werden. Mit `exit` kann man screen verlassen.
\
Falls ein Screen prozess im Hintergrund läuft und nicht von selbst stopt: `pkill screen` killt alle screens.

Mit ssh kann man sich auf einen Server verbinden. In unserem fall ist der Server so konfiguriert, dass er nur Verbindungen zulässt die den passenden private key zum angegebenen public key haben.
\
Der langform befehl wäre `ssh -i <private_key_file> user@server` also `ssh -i id_rsa ubuntu@<ip.adresse>` . Durch die verwendung von einem config file wird das verkürzt und es reicht den namen des Host Eintrag im config file anzugeben (geht nur wenn das key file id_rsa heißt) `ssh <host-name>`
\
 Mit `exit` kann man die ssh verbindung Verlassen.

**Info**: die meisten Befehle sollten auf allen Betriebsystemen identisch sein, alle auf der VM sind OS unabhängig. Manche der windows befehle sind vlt auf mac/linux anders.

## Ubuntu Instanz erstellen
1. auf `oracle.com/cloud` einloggen
2. unter compute > instances eine neue Instanz anlegen
3. instanz benennen
4. unter "Image and Shape"
   1. edit
   2. change Image
   3. Ubuntu
   4. 22.04 Minimal
5. "add ssh keys"
   1. "paste public keys" auswählen
   2. entweder einen vorhanden public key pasten oder einen erstellen (geht auch auf windows)
      1. terminal öffnen
      2. im user directory in .ssh ordner wechseln bzw erstellen
      3. `ssh-keygen` ausführen
         1. key name leer lassen
         2. passphrase leer lassen
      4. `code .` ausführen und inhalt von id_rsa.pub kopieren
      5. in inputfeld pasten
6. "create"

## "Instance details" bearbeiten
1. virtual cloud network: "vcn..." anclicken
2. "subnet.."
3. "Default Security List.."
4. add ingress Rules
   1. Source CIDR: 0.0.0.0/0
   2. Destination Port Range: 80
   3. Add

## SSH verbindung einrichten
1. In oracle - burger menu - pinned>instances - `<deine-instanz>` - unter "Instance access" ip kopieren
2. in vscode im .ssh ordner(c:/users/name/.ssh) ein file "config" erstellen und folgendes einfügen 
   ```
   Host <project-name>
      HostName <kopierte.ip>
      User ubuntu
   ```
1. im Terminal `ssh <project-name>`
2. `sudo apt update`
3. `sudo apt install screen` immer y bzw yes
4. `screen`
5. `sudo apt install nano` immer y bzw yes
6. `sudo apt install nginx` immer y bzw yes
7.  `exit` (screen)
8.  `cd /etc/iptables/`
9.  `sudo nano rules.v4`
10. alle lines zwischen "-A OUTPUT..." und "COMMIT" mit strg+k löschen
11. zu line "-A INPUT -p tcp -m state --state NEW -m tcp --dport 22 -j ACCEPT" mit pfeiltasten
12. strg+k
13. zwei mal strg+u
14. bei einer von beiden den --dport von 22 auf 80 ändern
15. str+x
16. y
17. enter

## Nginx configuration (temporäre Lösung)
1. `cd /etc/nginx/sites-available` 
2. `sudo nano default`
3. nach unten scrollen und unter "location / {... }" folgendes einfügen
    ```
   location /api/ {
      proxy_pass http://localhost:8080;
      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Host $host;
      proxy_set_header content-type $content_type;
      proxy_cache_bypass $http_upgrade;
      proxy_set_header Connection 'upgrade';
   }
   ```
4. strg+x, y, enter
5. `sudo systemctl start nginx` bzw  `sudo systemctl restart nginx`

## Postgres auf der VM
1. `screen`
2. `sudo apt install postgresql` y bzw yes
3. bei der restart abfrage `15` restarten
4. `exit` (screen)
5. `sudo su - postgres`
6. `nano setup.sql`
   ```
   create database <projectname>;

   create user <projectname> with password '<password>';

   alter user <projectname> with superuser;
   ```
7. strg+x
8. y
9. enter
10. `plsql -f setup.sql`

## Quarkus anpassen
1. in den application.propperties die line `quarkus.package.type=uber-jar` hinzufügen
2. falls noch nicht vorhanden `quarkus.hibernate-orm.sql-load-script=import.sql` ergänzen (nur relevant wenn ihr ein import.sql skript verwendet)
3. falls noch nicht in den einzelnen resources "api" vorm pfad steht `quarkus.http.root-path=/api` hinzufügen
   - aberger mag das wenn es so gelöst ist und nicht in den einzelnen resources
4. terminal im backend öffnen
5. `mvn clean package` oder `./mvnw clean package`
6. wenn ein test error `mvn -D skipTests=true clean package` oder `./mvnw -D skipTests=true clean package`
   \
   *(wenn anderer error gehts auch über IntelliJ)*
7. in den target folder cd'n
8. `scp <dein-jar-runner-file> <project-name>:` (gleicher name wie oben im config file)

## Java auf der VM
1. im terminal am server:
2. `screen`
3. `sudo apt list | grep jre`
4. openJDK version 21 headless bis zum leerzeichen kopieren (ohne datum und "amd..")
5. `sudo apt install <kopiertes-ding>`
6. y
7. bei der restart abfrage `15` restarten
8. `exit` (screen)
9. `java -jar <dein-jar-file>`

Euer Projekt sollte jetzt am Server laufen: Kann getestet werden indem ihr die `<ip.des.servers>/api/<euer/api/pfad>` im browser eingebt.. der Browser kann aber nur GET requests stellen. Für komplexere routen einfach ein .http file anlegen und dort den POST ausführen.
\
Ihr könnt theoretisch auch euer Frontend schon hoch pushen bringt aber gerade nichts.

## Änderungen am Projekt zum Server pushen
Temporäre Lösung die in Zukunft automatisiert wird
1. Nach [Quarkus anpassen 3.-7.](#quarkus-anpassen) ein jar file packagen und auf den server laden
2. Über ssh auf die Instanz verbinden und `java -jar <dein-jar-file>`

## Teammitglieder Serverzugriff gewähren
1. Auf dem Rechner des Teammitglieds einen vorhandenen public key kopieren oder nach [Ubuntu instanz erstellen 5.2](#ubuntu-instanz-erstellen) einen erstellen und an den Instanz Besitzter senden
2. auf dem Besitzter Rechner auf die Instanz ssh'n `ssh <project-name>`
3. `cd .ssh`
4. nano authorized_keys
5. ACHTUNG: wenn ihr die erste Zeile aus diesem File löscht oder verändert könnt ihr nicht mehr auf die Instanz zugreifen.
6. in einer neuen zeile den public key des teammitglieds einfügen (rechtsklick)
