# Installation d'Apache sur Debian 12

## 1. Mise à jour des paquets
```bash
sudo apt update && sudo apt upgrade -y
```

## 2. Installation d'Apache
```bash
sudo apt install apache2 -y
```

## 3. Vérification du service
```bash
sudo systemctl status apache2
```

## 4. Configuration du firewall
```bash
sudo ufw allow "Apache Full"
```

## 5. Test de fonctionnement
Ouvrez un navigateur et accédez à :
```
http://votre_ip_serveur
```

## Schéma de flux
```mermaid
graph LR
    Client -->|HTTP:80| Apache
    Apache -->|Fichiers| /var/www/html
```

> **Note** : Pour les TP avancés, configurez les hôtes virtuels dans `/etc/apache2/sites-available/`.
