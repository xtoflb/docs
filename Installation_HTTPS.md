# Mise en oeuvre de l'HTTPS sur un serveur web
# Utilisation d'une autorité de certification interne

## 1. Préparation de la machine CA
- Configuration IP : /etc/network/interfaces
```bash
allow-hotplug ens33
iface ens33 inet static
        address 172.16.0.20/24
        gateway 172.16.0.254
```
- Installation d'openssl
```bash
apt update && sudo apt upgrade -y
apt install openssl
```
## 2. configuration d'openssl
- Editez le fichier /etc/ssl/openssl.cnf
```bash
dir = ./sodecaf
```
- Création des dossiers et fichiers nécessaires
```bash
mkdir /etc/ssl/sodecaf
cd /etc/ssl/sodecaf/
mkdir certs
mkdir private
touch index.txt
echo "01" > serial
```
## 3. Génération du certificat de l'autorité de certification
- Création de la clé privée de l'autorité de certification
```bash
openssl genrsa -des3 -out /etc/ssl/sodecaf/private/cakey.pem 4096
chmod 400 /etc/ssl/sodecaf/private/cakey.pem
```
- Création du certificat auto-signé de l'autorité de certification
```bash
cd /etc/ssl/sodecaf/
openssl req -new -x509 -days 1825 -key private/cakey.pem -out certs/cacert.pem
```
## 4. Génération du certificat du serveur web
On travaille sur le serveur web1
- Editez le fichier /etc/ssl/openssl.cnf
```bash
dir = /etc/ssl
```
- Création de la clé privée du serveur web : 
```bash
openssl genrsa -out /etc/ssl/private/srvwebkey.pem 4096
```
