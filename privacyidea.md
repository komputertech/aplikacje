# Instalacja PrivacyIdea na Ubuntu 24.04

## Instalacja

<https://privacyidea.readthedocs.io/en/stable/installation/ubuntu.html>

Skrótowo:

```bash
wget https://lancelot.netknights.it/NetKnights-Release.asc
sudo mv NetKnights-Release.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository http://lancelot.netknights.it/community/noble/stable
sudo apt update
sudo apt install privacyidea-apache2
sudo pi-manage admin add admin -e admin@localhost
```

Opcjonalnie aktualizujemy MySQL do wersji 8.4 LTS

```bash
wget https://repo.mysql.com/mysql-apt-config_0.8.32-1_all.deb
# po użyciu tej komendy należy wybrać z listy wersje 8.4 LTS
sudo dpkg -i mysql-apt-config_0.8.32-1_all.deb
sudo apt update
sudo apt install mysql-server
sudo apt autoremove
```

Po instalacji zalecam zmianę hasła root do bazy:

```bash
sudo mysql -u root --skip-password
```

```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root-password';
```

Na koniec na wszelki wypadek kończymy instalację bazy:

```bash
sudo mysql_secure_installation -p
```

## Konfiguracja WEB

Przechodzimy na adres naszego serwera w przeglądarce, np. 10.0.0.2 logujemy się podając login admin i hasło.
Następnie zostanie wyświetlony komunikat o utworzeniu pierwszego REALM, zgadzamy się.

### Dodanie nowego REALM

Aby dodać nowe REALM skorzystamy z już zainstalowanej bazy MySQL, logujemy się i zakładamy co potrzeba:

```SQL
CREATE DATABASE dbusers;
CREATE USER dbcon@localhost identified BY 'password';
GRANT all privileges ON dbcon.* TO dbusers@localhost;
USE dbusers;
CREATE TABLE users (
  uid SERIAL PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  mail VARCHAR(255) UNIQUE NOT NULL,
  pass VARCHAR(255) NOT NULL
);
INSERT INTO dbusers (name, mail, pass) VALUES ('testuser', 'test@example.com', 'password123');
```

Teraz wracamy do WEB i przechodzimy do ustawień Config -> Users -> New Sqlresolver

- Resolver name MySQL_Users
- Driver mysql+pymysql
- Server 127.0.0.1
- Port 3306
- Database dbusers
- User dbcon
- Password password123
- Edit user store zaznaczyć i ustawić MD5CRYPT
- Table users
- Limit 500
- Mapping {"userid": "uid", "username": "name", "email": "mail", "password": "pass" }

Przechodzimy do Users -> Edit realms na liście wpisujemy:

- w nowym polu myrealm
- zaznaczamy MySQL_Users (sqlresolver)
- klikamy Create Realm
- klikamy Set Default

### Dodanie tokena

Przechodzimy do Users, lista powinna być pusta, a Realm wybrane myrealm. Klikamy Add user i wypełaniamy pola.
Na liście powinien pojawić się nowy użytkownik, klikamy jego nazwę, a następnie Enroll New Token.
Wybieramy z listy TOTP, wprowadzamy PIN lub hasło do tokena i zatwierdzamy. Skanujemy kod QR i gotowe.
Wracamy na listę użytkowników, klikamy na naszego użytkownika i na liście pojawi nam się token.
Klikamy na token i szukamy przycisku Test Token, obok niego, w pole, wpisujemy hasło i wygenerowany z aplikacji OTP,
np. password453001 i czekamy na odpowiedź, jak jest zielona, wszystko ok.
