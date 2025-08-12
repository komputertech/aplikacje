# Instalacja PrivacyIdea MFA

Całość oparta o Ubuntu 24.04 instalacja z pakietów apt. Przedstawiona konfiguracja bazy danych MySQL, wdrożenie podstawowych polityk, tworzenie użytkowników, dodawanie tokenów. Na koniec przykładowe zastosowanie, czyli użycie w zabezpieczeniu dostępu do zasobów strony www, poprzez konfigurację Apache2. Nie podejmuję szczegółowej konfiguracji i zabezpieczenia.

## Instalacja

Cała instrukcja znajduje się pod adresem: [link do instrukcji](https://privacyidea.readthedocs.io/en/stable/installation/ubuntu.html). Skrócona wersja:

```bash
wget https://lancelot.netknights.it/NetKnights-Release.asc
sudo mv NetKnights-Release.asc /etc/apt/trusted.gpg.d/
sudo add-apt-repository http://lancelot.netknights.it/community/noble/stable
sudo apt update
sudo apt install privacyidea-apache2
sudo pi-manage admin add admin -e admin@localhost
```

Opcjonalnie aktualizujemy MySQL do wersji 8.4 LTS, wersja 8.0 wkrótce nie będzie wspierana, warto po aktualizacji wyczyścić system ze zbędnych pakietów. Podczas dodawania repozytorium, pojawi się mini konfigurator, w którym należy wybrać wersję MySQL:

```bash
wget https://repo.mysql.com/mysql-apt-config_0.8.32-1_all.deb
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

Przechodzimy do PrivacyIdea w przeglądarce, np. 172.16.0.18 Logujemy się podając login admin i hasło. Hasło ustawiamy za pomocą poleceń pi-manage. Następnie zostanie wyświetlony komunikat o potrzebie utworzenia pierwszego Realm, zgadzamy się.

### Dodanie nowego Realm

Aby dodać nowe Realm, które będzie wykorzystywało bazę danych MySQL, musimy najpierw utworzyć bazę danych. Wykorzystamy do tego już działającą instancję, wykorzystywaną do PrivacyIdea. Logujemy się i zakładamy nową bazę użytkowników oraz użytkownika do połączeń z bazą:

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
```

Teraz wracamy do WEB i przechodzimy do ustawień Config -> Users -> New Sqlresolver, który pozwoli nam spiąć bazę danych z PrivacyIdea. Musimy po kolie przeklikać kolejno:

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

Przechodzimy do Users -> Edit Realms na liście wpisujemy:

- w nowym polu myrealm
- zaznaczamy MySQL_Users (sqlresolver)
- klikamy Create Realm
- klikamy Set Default

### Dodanie polityk

Teraz powinniśmy utworzyć pierwsze polityki, czyli zasady dotyczące dostępów lub funkcji. Pierwsze polityka usunie komunikat powitalny, oraz rozszerza czas, zanim zostaniemy wylogowani. Przechodzimy do Config -> Policies i klikamy Create New Policy i zaznaczamy kolejne opcje:

- Condition
  - Nazwa GUI_conf
  - Scope webui
- Action
  - miscellaneous
    - hide_welcome_info
    - logout_time 360

Kolejna polityka pozwoli na wyświetlanie administratorowi strony podsumowującej po zalogowaniu, ustawi hasło jako PIN oraz doda reset ilości błędnych tokenów, po prawidłowym logowaniu:

- Condition
  - Nazwa User_auth
  - Scope authentication
- Action
  - miscellaneous
    - admin_dashboard
    - otppin userstore
    - reset_all_user_tokens
    - webauthn_timeout 120

I ostatnia, zezwala na modyfikację tokena przez użytkownika, jego danych, w tym hasła oraz zezwala na dodanie tokena TOTP o określonych z góry parametrach:

- Condition
  - Nazwa User_user
  - Scope user
- Action
  - token
    - delete
    - disable
    - enable
    - resync
    - setdescription
  - miscellaneous
    - password_reset
    - totp_force_server_generate
    - totp_hashlib sha1
    - totp_otplen 6
    - totp_timestep 30
    - updateuser
    - userlist
  - enrollment
    - enrollTOTP

### Dodanie użytkownika

Przechodzimy do Users, lista powinna być pusta, a Realm wybrane myrealm. Klikamy Add user i wypełaniamy pola. Na liście powinien pojawić się nowy użytkownik.

## Dodanie tokena przez użytkownika

Logujemy się jako użytkownik. Klikamy Enroll New Token. Zgodnie z polityką, można ustawić jedynie TOTP. Skanujemy kod QR poprzez np. FreeOTP lub inną aplikację (SHA1 pozwoli na użycie praktycznie każdej) i gotowe.

Przechodzimy na listę tokenów, klikamy na nasz token i szukamy przycisku Test Token, obok niego jest pole, w które wpisujemy hasło i wygenerowany z aplikacji TOTP, np. password453001 i czekamy na odpowiedź.

## Instalacja Radius

### Serwer PrivacyIdea

Instalujemy PrivacyIdea Radius plugin:

```bash
apt-get install privacyidea-radius
```

Konfigurujemy FreeRadius, edytujemy plik /etc/freeradius/3.0/clients.conf i dodajemy na końcu pliku wpis z adresem naszego serwera Apache2 oraz hasłem, które służy do potwierdzenia połączenia. Po dodaniu wpisu, należy zrestartować freeradius.service:

```text
client webapp {
        ipaddr = 172.16.0.19
        secret = secret123
}
```

### Serwer Apache2

Instalujemy pakiety, wykorzystamy mod-auth-radius bo to najłatwiejsza metoda na połączenie z Radius:

```bash
sudo apt install apache2 libapache2-mod-auth-radius
sudo systemctl enable apache2
sudo a2enmod auth_radius
sudo systemctl restart apache2
```

 Konfigurujemy domyślny wpis Apache2 w pliku /etc/apache2/sites-enabled/000-default.conf Na samym dole konfiguracji musimy wprowadzić adres naszego serwera PrivacyIdea oraz hasło do połączenia. Port 1812/udp służy do autoryzacji Radius.

 ```text
 <VirtualHost *:80>
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        <Directory "/var/www/html">
                Options Indexes FollowSymlinks
                AuthType Basic
                AuthName "Apache Radius authentication"
                AuthBasicAuthoritative Off
                AuthBasicProvider radius
                AuthRadiusAuthoritative On
                AuthRadiusActive on
                Require valid-user
        </Directory>

        AddRadiusAuth 172.16.0.18:1812 secret123 5:3
</VirtualHost>
 ```

Pozostało tylko przetestowanie działania całości, poprzez przejście w przeglądarce do adresu 172.16.0.19 Powinno pojawić się okienko z prośbą o zalogowanie, wpisujemy dane naszego użytkownika oraz hasło+token. Po poprawnej weryfikacji zobaczymy stronę startową Apache2.
