# Installation d'Apache sur Debian 12

## 1. Mise à jour des paquets
```bash
apt update && sudo apt upgrade -y
```

## 2. Installation d'Apache
```bash
apt install apache2 -y
```

## 3. Vérification du service
```bash
systemctl status apache2
```
