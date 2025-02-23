# Gitea

## Baza danych

### Instalacja bazy MySQL 8.4 LTS

```bash
wget https://repo.mysql.com/mysql-apt-config_0.8.32-1_all.deb
sudo dpkg -i mysql-apt-config_0.8.32-1_all.deb
sudo apt update
sudo apt install mysql-server
sudo systemctl enable --now mysql
sudo mysql_secure_installation
```

### Konfiguracja bazy

```bash
sudo mysql -u root -p
```

```sql
CREATE DATABASE giteadb;
CREATE USER IF NOT EXISTS 'gitea'@'localhost' IDENTIFIED BY 'StrongPassword123';
GRANT ALL PRIVILEGES ON giteadb.* TO 'gitea'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

## Aplikacja

### Instalacja aplikacji

```bash
sudo adduser --system --shell /bin/bash --gecos 'Git Version Control' --group --disabled-password --home /home/git git
wget -O gitea https://dl.gitea.com/gitea/1.23.4/gitea-1.23.4-linux-amd64
chmod +x gitea
sudo mv gitea /usr/local/bin/gitea
sudo mkdir -p /var/lib/gitea/{custom,data,log}
sudo chown -R git:git /var/lib/gitea/
sudo chmod -R 750 /var/lib/gitea/
sudo mkdir /etc/gitea
sudo chown root:git /etc/gitea
sudo chmod 770 /etc/gitea
```

### Przygotowanie usługi

```bash
wget https://raw.githubusercontent.com/go-gitea/gitea/main/contrib/systemd/gitea.service
nano /etc/systemd/system/gitea.service
```

```text
[Unit]
Description=Gitea
After=network.target
Wants=mysql.service
After=mysql.service
 
[Service]
LimitNOFILE=524288:524288
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
RuntimeDirectory=gitea
ExecStart=/usr/bin/gitea web --config /etc/gitea/app.ini
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo mv gitea.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now gitea
```

## Pierwsze uruchomienie GUI

### Strona WWWW

Przejść pod adres serwera, `http://adres-ip:3000` wypełnić dane dotyczące bazy:

- Database type: MySQL
- Host: localhost:3306
- Username: git
- Password: StrongPassword123
- Databse name : giteadb

Resztę zostawiamy jak jest i zatwierdzamy. Po poprawnej instalacji, musimy jeszcze zarejestrować się, utworzyć pierwsze konto, które będzie administratorem. Po poprawnym logowaniu, przechodzimy do zakończenia, musimy jeszcze zablokować możliwość edycji pliku konfiguracyjnego.

### Po instalacji

```bash
sudo chmod 750 /etc/gitea
sudo chmod 640 /etc/gitea/app.ini
sudo nano /etc/gitea/app.ini
```

```text
...
[service]
DISABLE_REGISTRATION = true
...
[openid]
ENABLE_OPENID_SIGNIN = false
ENABLE_OPENID_SIGNUP = false
...
```

```bash
sudo systemctl restart gitea
```

## Certyfikat SSL

### Utworzenie certyfikatu i klucza z PFX

```bash
openssl pkcs12 -in gitea.pfx -nocerts -out key.pem
openssl rsa -in key.pem -out nkey.pem
sudo mv nkey.pem /etc/gitea/key.pem
sudo chown git:git /etc/gitea/key.pem
openssl pkcs12 -in gitea.pfx -clcerts -nokeys -out cert.pem
sudo mv cert.pem /etc/gitea/cert.pem
sudo chown git:git /etc/gitea/cert.pem
```

### Konfiguracja aplikacji

```bash
sudo nano /etc/gitea/app.ini
```

```text
...
[server]
SSH_DOMAIN = gitea.komputertech.local
DOMAIN = gitea.komputertech.local
HTTP_PORT = 443
ROOT_URL = https://gitea.komputertech.local/
PROTOCOL = https
CERT_FILE = /etc/gitea/cert.pem
KEY_FILE = /etc/gitea/key.pem
...
```

### Konfiguracja usługi z certyfikatem

```text
[Unit]
Description=Gitea
After=network.target
Wants=mysql.service
After=mysql.service
 
[Service]
LimitNOFILE=524288:524288
RestartSec=2s
Type=simple
User=git
Group=git
WorkingDirectory=/var/lib/gitea/
RuntimeDirectory=gitea
ExecStart=/usr/bin/gitea web --config /etc/gitea/app.ini
Restart=always
Environment=USER=git HOME=/home/git GITEA_WORK_DIR=/var/lib/gitea
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
PrivateUsers=true

[Install]
WantedBy=multi-user.target
```
