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
- Création d'un nouveau fichier de configuration de corosync
```bash
mv corosync.conf corosync.conf.sav
nano corosync.conf
```
```bash
totem {
        version: 2
        cluster_name: cluster_web
        crypto_cipher: aes256
        crypto_hash: sha1
        clear_node_high_bit:yes
}
logging {
        fileline: off
        to_logfile: yes
        logfile: /var/log/corosync/corosync.log
        to_syslog: no
        debug: off
        timestamp: on
        logger_subsys {
                subsys: QUORUM
                debug: off
        }
}
quorum {
        provider: corosync_votequorum
        expected_votes: 2
        two_nodes: 1
}
nodelist {
        node {
                name: srv-web1
                nodeid: 1
                ring0_addr: 172.16.0.10
        }
        node {
                name: srv-web2
                nodeid: 2
                ring0_addr: 172.16.0.11
        }
}
service {
        ver: 0
        name: pacemaker
}
```                  
- Vérification de la configuration
```bash
corosync-cfgtool -s
```
- Cloner le serveur web, modifier IP et nom du clone
- Visualiser l'état du cluster
```bash
crm status
```

