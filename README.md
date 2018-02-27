# Sätta upp en server i kursen TDDD82 (grupprojektet)

Servrarna givna av IDA (2018) kör operativsystemet [Debian 8](https://en.wikipedia.org/wiki/Debian).

Kontakta handledare om `sudo` inte är installerat (har hänt).

# Login och användare
Logga in på servern med:
```bash
ssh liuid@itkand-X-X.tddd82-20XX.ida.liu.se
```

Vid först inlogg kan det verka som terminalen är trasig (e.g. piltangener fungerar inte), detta är för att IDA skapar sina använda på ett väldigt primitivt (läs retarded) sätt. "Det är en väldigt barskrapad config, men det gör också att ni kan anpassa per grupp". Fixa detta problem genom att ändra din login shell med
```bash
chsh -s /bin/bash
```
Du behöver sedan logga ut (`exit`) och logga in igen för att se någon skillnad.

Du kan göra detta för andra användare med (`ls /home` för att se alla användare)
```bash
sudo chsh -s /bin/bash liuid
```
# En webmasters snabbguide

## Webbserver - nginx
[nginx](https://en.wikipedia.org/wiki/Nginx) är själva webbservern och är det som hanterar inkommande requests till servern. I en config-fil för din sida säger man vad som ska hända när användaren når en viss url, till exempel 'skicka all requests till den här python-appen'. Detta kommer också hantera certifikat för HTTPS.

Config-filer som nginx automatiskt använder ligger i mappen:
```bash
/etc/nginx/sites-enabled
```
När ändringar görs av dessa filer måste man ladda om config-filerna manuellt med:
```bash
sudo service nginx reload
```
Debugging:
```bash
# Testa config-filer
sudo nginx -t
# Kolla requests log
sudo cat /var/log/nginx/access.log
# Kolla error log
sudo cat /var/log/nginx/error.log
```

## Köra program - uWSGI
[uWSGI](https://en.wikipedia.org/wiki/UWSGI) användas för att kommunicera mellan en webbapplikation och en webbserver. Till exempel för att få en python app att korrekt hantera requests och skicka tillbaka data till klienter. Från början användes det bara för python-applikationer men har på senare tid börjat stödja andra språk (e.g. php, ruby, perl, Golang).

Kan vara lite svårt att sätta upp ibland beroende på språk och ramverk. Flask och Django för python brukar oftast fungera på första försöket.

Se detailjer för språk/ramverk i deras [dokumentation](http://uwsgi-docs.readthedocs.io/en/latest/). Det brukar finnas en guide för de mesta.

Kolla logfilen för att debugga din app, i min exempelfil så görs detta med
```bash
sudo cat /srv/tddd82/uwsgi.log
# Alternativt för att få de sista raderna
sudo tail -s 100 /srv/tddd82/uwsgi.log
``` 

# Server setup - fixa själva servern
Mycket copy paste.
```bash
# Gör det möjgligt att ladda ner program för att få HTTPS
echo 'deb http://ftp.debian.org/debian jessie-backports main' | sudo tee /etc/apt/sources.list.d/backports.list

# Uppdatera programlistan
sudo apt-get update

# Installera allt nödvändigt (och lite till)
sudo apt install virtualenv python3 uwsgi uwsgi-emperor uwsgi-plugins-all nginx -y
sudo apt install certbot -t jessie-backports -y

# Klona erat backend repo
sudo git clone https://git..com/...git  /srv/tddd82
cd /srv/tddd82

# Om ni använder en virtuell miljö för python
sudo virtualenv -p python3 .venv
source .venv/bin/activate
sudo .venv/bin/pip install -r requirements.txt
deactivate

# Fixa rättigheter i eran server-mapp
sudo chown -R www-data:www-data /srv/tddd82

# Hämta detta repo, spelar ingen roll vart den sparas
cd ~
git clone https://github.com/JubbeArt/tddd82-server-setup .

# Här kanske ni vill ändra i uwsgi-config filen
# Exempelfilen är designad för att fungera för python-appar skrivet med Flask 
# (och som har namnet app.py och 'app' som huvud-python-object) 
sudo cp uwsgi-flask.ini /etc/uwsgi-emperor/vassals

# Stoppa default programmet som kör uwsgi (den suger)
sudo systemctl stop uwsgi-emperor
sudo systemctl disable uwsgi-emperor

# Installera vår egen uwsgi daemon
sudo cp emperor.uwsgi.service /etc/systemd/system
sudo systemctl daemon-reload
sudo systemctl enable nginx emperor.uwsgi
sudo systemctl start emperor.uwsgi

# Fixa nginx config
sudo cp nginx-default /etc/nginx/sites-enabled/default
sudo systemctl reload nginx

# Fixa HTTPS (ändra domänen till din egen)
sudo certbot certonly --webroot /srv/tddd82 -d itkand-X-X.tddd82-20XX.ida.liu.se

# Notera raderna "Generating key (2048 bits): ..." och
# "Congratulations! Your certificate and chain have been saved at ..."

# Du måste lägga till dessa två filer i nginx-configen
sudo nano /etc/nginx/sites-enabled/default
# Ta bort kommentarerna i andra server-blocket och se till att filvägarna är korrekt
# Spara med ctrl-o och avsluta med ctrl-x
# Testa nginx med
sudo nginx -t

# Ladda om nginx
sudo service nginx reload

# De var allt

```
## Updatera filer på server
```bash
cd /srv/tddd82
sudo git pull
sudo service uwsgi.emperor restart
```

Kolla databasen med:
```bash
mysql -h db-und.ida.liu.se -u itkand_20XX_X_X -pitkand_20XX_X_X_XXXX itkand_20XX_X_X  
```


# Advanced shit

## TODO
* webhooks
* login utan lösenord
* lokal utveckling mot skolans databas


# Rekomendationer (aka de vi gjorde för vårt projekt)
* [Flask](http://flask.pocoo.org/) - lättaste sättet att göra backenden på
* [Flask-RESTful](https://flask-restful.readthedocs.io/en/latest/index.html) - ett till och med lättare sätt att göra backenden på
* [PyMySQL](http://pymysql.readthedocs.io/en/latest/user/examples.html) - helt ok databas-driver för MySQL, lite få exempel kanske
* [Advanced REST client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo) - enkel klient för att göra alla möjliga typer av requests
* [Volley](https://developer.android.com/training/volley/index.html) - request-bibliotek för android
* [Java 8](https://developer.android.com/studio/write/java8-support.html) - mycket skönare syntax för Java/Android, funkar för alla API nivåer
* [Singleton pattern](https://www.javaworld.com/article/2073352/core-java/simply-singleton.html) (glorifierad global varaibel) - använd för klasser som bara behöver instansieras en gång
* [Factory pattern](https://www.tutorialspoint.com/design_pattern/factory_pattern.htm) - för att skapa mer avancerade object (t.ex. olika typer markörer). Om man inte gillar factories kan man istället använda [builder pattern](https://jlordiales.me/2012/12/13/the-builder-pattern-in-practice/)