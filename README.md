# Libki dokumentation

## Server

Sunne bibliotek använder Libki som hanteringssystem för gästdatorer. Libki finns på https://github.com/libki/libki-server. Libki körs på en Debian Jessieserver (installationshänvisningar till senare serverversioner håller på att skrivas) och är installerat utifrån anvisningarna i Libkis README 2018-12-11. Ett hum om linux och hur du arbetar i terminal är bra om något ska ändras. Du behöver kunna hantera mappstruktur och en texteditor, förslagsvis nano. Har du aldrig arbetat i terminal tidigare är det nog bra att kika på en av de tusentals snabbkurser i terminal som finns på nätet. Sök på "linux terminal 101" och allihop kommer fungera för det vi behöver använda den till här.

### NTP automatisk klocka

Eftersom Debians klocka inte automatiskt synkas mot en tidsserver är ett paket installerat för att ta hand om automatisk uppdatering av klockan, för att försäkra att den alltid går rätt.

```
sudo apt-get install ntp
```

NTP går att konfigurera så den kopplas mot en lokal svensk tidsserver, men vår är helt enkelt inställd på debians standardserver.

### Automatiska uppdateringar

Servern uppdateras automatiskt via paketet Unattended-upgrades (`apt-get install unattended-upgrades`) och är därmed i princip underhållsfri.

Följande konfigureringar har gjorts för Unattended-upgrades:

`/etc/apt/apt.conf.d/20auto-upgrades` innehåller följande text:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
APT::Periodic::Download-Upgradeable-Packages "1";
APT::Periodic::AutoCleanInterval "7";
```

I `/etc/apt/apt.conf.d/50unattended-upgrades` ersätts allt inom Unattended-Upgrade::Origions-Pattern {}; med:

```
Unattended-Upgrade::Origions-Pattern {
    "o=*";
};
```

### Backup

Serverns databas backas upp varje söndag via skriptet /root/backup_db.sh och läggs på den anslutna externa hårddisken som är monterad på /mnt/backup/. Backupen kopieras sen till /root/backup/ så den är sparad på två ställen, skulle något hända.


```
#!/bin/bash

DATE=`date +%y-%m-%d`;
HOST='[FTP-SERVER]';
USER='[FTP ANVÄNDARNAMN]';
PASSWD='[FTP LÖSENORD]';

mysqldump -u[MYSQL ANVÄNDARNAMN] -p[MYSQL LÖSENORD] libki | gzip > /mnt/backup/libki-$DATE.sql.gz

cp /mnt/backup/libki-$DATE.sql.gz /root/backup

ftp -n -v $HOST << EOT
ascii
user $USER $PASSWD
prompt
cd db_backup
put /root/backup/libki-$DATE.sql.gz
bye
EOT
```

Backupen körs via crontab (schemaläggare). Du når crontab för att göra förändringar med crontab -e. Syntax för crontab är fem stjärnor för att visa tid.

```
* * * * * command to be executed
- - - - -
| | | | |
| | | | +----- day of week (0 - 6) (Sunday=0)
| | | +------- month (1 - 12)
| | +--------- day of        month (1 - 31)
| +----------- hour (0 - 23)
+------------- min (0 - 59)
```

I vårt fall är därmed kommandot `* * * * 7 /root/backup_db.sh`

För att återställa backupen behöver libki-databasen tömmas, backupen zippas upp och importeras i databasen. 

Tömma databasen:
```
mysql -u[MYSQL ANVÄNDARNAMN] -p[MYSQL LÖSENORD]
DROP DATABASE libki;
exit
```

Zippa upp backupen:
```
gzip -d [backupens filnamn].sql.gz
```

Importera backupen:
```
mysql -u[MYSQL ANVÄNDARNAMN] -p[MYSQL LÖSENORD] libki < [backupens filnamn].sql
```

## Klienterna

Klienterna kör Linux Mint Cinnamon. Just nu är versionen 18.1 och har support till april 2021.

Den här versionen (och senare) finns på https://linuxmint.com/download_all.php. Installationsguide finns på https://linuxmint-installation-guide.readthedocs.io/en/latest/.

Klienterna är precis som servern konfigurerad till att ha automatiska uppdateringar med samma inställningar som för servern.

Följande modifieringar är gjorda och körs:

Tre program är skrivna och installerade. Här följer de i tur och ordning:

backup

```
#!/bin/bash

rm -rf /opt/public
cp -a /home/public /opt/public
echo "Backup klar."
```

Det finns alltså en mapp som användaren public (den som normalt är inloggad) inte har skrivrättigheter till. Den heter `/opt`, och där sparas backupen. Hemmappen för användaren public heter `/home/public`. Programmet tar alltså först bort `/opt/public` (den uppbackade mappen) om den finns, kopierar sedan hela `/home/public` och lägger den i `/opt/public` för att avslutningsvis meddela att backupen är klar. 

Eftersom det behöver köras med rootprivilegier är kommandot i terminal: `sudo backup`

restore

```
#!/bin/bash

rsync -qrpog --delete --exclude '.X*- /opt/public /home
echo "" > /home/public/.local/share/recently-used.xbel
rm /var/spool/cups/*
```

Det här jämför `/home/public` med den uppbackade `/opt/public` och återställer alla ändringar som gjorts/uppstått sedan programmet kördes senast. Sedan tömmer den filen "senaste använda dokument". Eventuellt är den raden redundant, men jag har inte undersökt det närmare. Avslutningsvis tömmer den skrivarens loggfil över färdigutskrivna dokument.

demapper

```
#!/bin/bash

xmodmap -e 'keycode 133='
xmodmap -e 'keycode 64='
```

Det här kopplar bort tangenterna 133 (Windowstangenten) och 64 (vänster alt-tangent) från användaren och gör dem obrukbara. Detta eftersom det via Windowstangenten och alt-tangenten går att komma förbi Libkis inloggningsruta. 

När de olika programmen skrivits behöver de göras exekverbara, och dess ägare behöver ändras till root så att de inte kan skrivas om.

```
sudo chmod +x [programmets namn]

sudo chown root:root [programmets namn]
```

När det väl är gjort behöver de placeras i en mapp som ingår i systemets `$PATH`, vilket innebär att de kan anropas utan att man behöver ange dess fulla sökväg.

```
sudo mv [programmets namn] /usr/local/bin
```

Backup körs när vi vill att det ska köras, exempelvis efter en uppdatering av något slag, nyinstallerad programvara eller dylikt.

Demapper körs när användaren loggas in, och är tillagt via det grafiska gränssnittet och "Uppstartsprogram", med en kort fördröjning på 5 sekunder.

Restore körs också när användaren loggas in, men eftersom det behöver rootprivilegier och ska göras innan användaren loggats in är det lagt inuti displayhanterarens uppstartsrutin.

öppna `/etc/mdm/PreSession/Default` med nano.

```
sudo nano /etc/mnm/PreSession/Default
```

Gå längst ner, och lägg till en rad innan `exit 0` som bara innehåller ordet `restore`.

Sätt upp autologin om det inte redan gjordes när datorn installerades.

Öppna `/etc/mdm/mdm.conf` och hitta `[daemon]`. Informationen däri ska vara som följer:

```
AutomaticLoginEnable=true
AutomaticLogin=public
TimedLoginEnable=true
TimedLogin=public
TimedLoginDelay=1
```

Libkiklienten ska installeras. Den går att kompilera genom att följa anvisningarna på https://github.com/Libki/libki-client/wiki/libki-client-installation eller använda det bifogade färdigkompilerade programmet, byggd för just Linux Mint 18.2.

Programmet ska göras exekverbart (precis som tidigare, `sudo chmod +x libkiclient`) och placeras i `/usr/local/bin`. Använder du en usb-sticka finns den under `/media/public/[namnet på usb-stickan]`.

Avslutningsvis ska libki få ett par .ini-filer som styr hur programmet fungerar, adress till server och lite sådana saker.

`Libki.ini` ska läggas i `/home/public/.config` och innehålla följande saker:

```
[server]
host=192.168.199.223
port=80
scheme=http

[node]

name=[Det namn du vill att datorn ska ha, exempelvis Dator 7]

location=[Den plats datorn finns på. Har biblioteket bara en placering av datorer kan denna rad vara utkommenterad med ett semikolon.]

; Kommentera ut de två som inte ska användas. 
; Eftersom vi använder utloggning för att återställa allt är den förinställd på det

logoutAction=logout           ;Log out of OS
; logoutAction=reboot           ;Reboot computer
; logoutAction=no_action        ;Just redisplay the Libki login screen

; Tar bort lösenord för inloggning om satt till 1
no_passwords=0

; Bakdörr, om man behöver komma in på datorn utan att gå via libki. Lösenordet sparat som md5. 
; Använd exempelvis https://www.md5hashgenerator.com/.
password=[LIBKI-LÖSENORD]

[labels]
; Vad som står på login-skärmen
username="Personnummer:"
password="PIN-kod:"
```

Mappen `Libki` innehållandes `Libki Kiosk Management System.ini` behöver inte förändras, utan ska enbart kopieras till `/home/public/.config`.

Släktforskningsprogram som ska installeras på släktforskningsdatorn och Dator 7 respektive 8 kommer dokumenteras vid ett senare tillfälle.
