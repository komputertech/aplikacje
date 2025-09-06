# Greenbone

## Instalacja

Pobieramy listę najnowszych modułów, instrukcja często nie nadąża za aktualizacjami. Trzeba jednak pamiętać, że nie zawsze najnowsze moduły działają. Podczas kompilacji modułów trzeba uważnie sprawdzać informacje i doinstalować brakujące pakiety, które czasami nie są uwzględnione w instrukcji. Poniżej skrypty do pobrania wersji.

### Powershell

```powershell
$greenboneModules = @(
    'gvm-libs',
    'gvmd',
    'pg-gvm',
    'gsa',
    'gsad',
    'openvas-smb',
    'openvas-scanner',
    'ospd-openvas'
)

$greenboneModules | ForEach-Object {
    $apiUrl = "https://api.github.com/repos/greenbone/$_/releases/latest"
    $latestRelease = Invoke-RestMethod -Uri $apiUrl -UseBasicParsing
    Write-Host "$_ - $($latestRelease.tag_name)"
    if($_ -match 'openvas-scanner') { 
        $openvasd = "openvasd - $($latestRelease.tag_name)"
    }
}

Write-Host $openvasd
```

### Bash

```bash
greenboneModules=(
    "gvm-libs"
    "gvmd"
    "pg-gvm"
    "gsa"
    "gsad"
    "openvas-smb"
    "openvas-scanner"
    "ospd-openvas"
)

for module in "${greenboneModules[@]}"; do
    api_url="https://api.github.com/repos/greenbone/$module/releases/latest"
    latest_release=$(curl --silent $api_url | grep -Po '"tag_name": "\K.*?(?=")')
    echo "$module - $latest_release"
    if [[ $module == "openvas-scanner" ]]; then
        openvasd="openvasd - $latest_release"
    fi
done

echo $openvasd
```

## Usługa GSAD

W katalogu `/etc/systemd/system/gsad.service` została zmodyfikowana, tak aby można było dostać się z zewnątrz przez HTTPS (przekierowanie z HTTP), z certyfikatem lokalnego CA. Usługę uruchamiamy z parametrem –drop-privileges, a nie jako użytkownik, żeby móc używać portów niskich, tj. 443. Możemy zmodyfikować też ten fragment tak, aby używać portu 80.

```text
[Unit]
Description=Greenbone Security Assistant daemon (gsad)
Documentation=man:gsad(8) <https://www.greenbone.net>
After=network.target gvmd.service
Wants=gvmd.service

[Service]
Type=exec
# User=gvm
# Group=gvm
RuntimeDirectory=gsad
RuntimeDirectoryMode=2775
PIDFile=/run/gsad/gsad.pid
ExecStart=/usr/local/sbin/gsad --foreground --listen=0.0.0.0 --drop-privileges=gvm --port=443 --rport=80 -c /var/lib/gvm/private/CA/servercert.pem -k /var/lib/gvm/private/CA/serverkey.pem
# ExecStart=/usr/local/sbin/gsad --foreground --listen=0.0.0.0 --drop-privileges=gvm --port=80 --http-only
Restart=always
TimeoutStopSec=10

[Install]
WantedBy=multi-user.target
Alias=greenbone-security-assistant.service
```

## Certyfikat i klucz z pliku PFX

```bash
sudo mkdir -p /var/lib/gvm/private/CA
openssl pkcs12 -in greenbone.pfx -nocerts -out key.pem
openssl rsa -in key.pem -out nkey.pem
sudo mv nkey.pem /var/lib/gvm/private/CA/serverkey.pem
openssl pkcs12 -in greenbone.pfx -clcerts -nokeys -out cert.pem
sudo mv cert.pem /var/lib/gvm/private/CA/servercert.pem
sudo chown -R gvm:gvm /var/lib/gvm/private
```

## Aktualizacja słowników

Ręcznie:

```bash
sudo /usr/local/bin/greenbone-feed-sync
```

Automatycznie z cron:

```bash
echo "0 5 * * * /usr/local/bin/greenbone-feed-sync" | sudo tee -a crontab
```

Aktualizacja słowników trwa bardzo długo, do tego jest ograniczenie na pobieranie, ze strony Greenbone Community, więc jak się zablokuje, to trzeba poczekać. Po pobraniu słowników, usługi Greenbone indeksują zawartość, a to również może trwać długo, czasami dłużej niż pobieranie. Do tego warto zapewnić ok. 70GB wolnego miejsca, podczas pierwszej aktualizacji słowników.

Sprawdzenie indeksacji danych podczas aktualizacji:

```bash
sudo tail /var/log/gvm/ospd-openvas.log
sudo tail /var/log/gvm/gvmd.log
```

### Aktualizacja

Podczas aktualizacji modułów trzeba je wcześniej usunąć. Cała procedura opisana jest [link do instrukcji](https://greenbone.github.io/docs/latest/22.4/source-build/workflows.html). Całość w skrócie to usunięcie wszystkich pakietów i zbudowanie ich od nowa, plus migracja bazy i słowników.

```bash
sudo systemctl stop gsad gvmd ospd-openvas openvasd
sudo python3 -m pip uninstall --break-system-packages ospd-openvas greenbone-feed-sync gvm-tools
```

Teraz przechodzimy do instrukcji instalacji, zaraz po przygotowaniu użytkowników. Omijamy tworzenie bazy, schematów i usług. Kończymy instalację.

```bash
/usr/local/sbin/gvmd --migrate
sudo systemctl start gsad gvmd ospd-openvas openvasd
```
