# Mise en oeuvre d'un cluster de serveur web
- Installation de corosync, pacemaker et crmsh
```bash
apt install corosync pacemaker crmsh
```
- Création de la clé authkey pour le chiffrement des échanges
```bash
corosync-keygen
ls -l /etc/corosync/
```

