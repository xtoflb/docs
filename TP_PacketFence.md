# B3-Act10-TP1 – Authentification 802.1X et serveur RADIUS PacketFence

**BTS SIO – Bloc 3 – Authentification sécurisée sur un réseau (6h)**

---

## 1. Présentation

### 1.1. Le protocole RADIUS

**RADIUS** (Remote Authentication Dial-In User Service) est un protocole client-serveur permettant de **centraliser l'authentification, l'autorisation et l'audit** des accès réseau. Le protocole RADIUS permet de faire la liaison entre des besoins d'identification et une base d'utilisateurs en assurant le transport des données d'authentification de façon normalisée.

L'opération d'authentification est initiée par un **client du service RADIUS** (NAS - Network Access Server), qui peut être :
- Un point d'accès réseau sans fil
- Un commutateur (switch)
- Un pare-feu (firewall)
- Un serveur VPN

Le serveur RADIUS traite les demandes en accédant si nécessaire à une **base externe** : base de données SQL, annuaire LDAP, comptes Active Directory, etc.

Par défaut, le trafic RADIUS utilise le protocole **UDP** sur les ports :
- **1812** : authentification
- **1813** : accounting (comptabilité/audit)
- 1645 et 1646 (ports historiques, encore utilisés)

### 1.2. PacketFence comme serveur RADIUS et NAC

Dans ce TP, le serveur RADIUS sera fourni par **PacketFence**, une solution **NAC (Network Access Control) open source** intégrant :
- **FreeRADIUS** pour l'authentification RADIUS
- Des fonctions avancées : profils d'accès, VLAN dynamiques, portail invité captif, détection d'anomalies, quarantaine, etc.

PacketFence permet de gérer de manière centralisée l'accès au réseau filaire et WiFi en s'appuyant sur l'authentification 802.1X et une intégration native à l'Active Directory.

### 1.3. L'authentification 802.1X

Le standard **802.1X** permet de contrôler l'accès au réseau (filaire ou WiFi) en s'appuyant sur :
- Un **suppliant** (client qui souhaite se connecter)
- Un **authenticator** (commutateur ou point d'accès)
- Un **authentication server** (serveur RADIUS, ici PacketFence)

Le processus se déroule en deux phases :
1. **Phase d'association** : le client s'associe au point d'accès ou au port du switch
2. **Phase d'authentification 802.1X** : échange EAP entre le client et le serveur RADIUS, via l'authenticator

Une fois l'authentification réussie, le serveur RADIUS peut renvoyer des **attributs RADIUS** pour affecter dynamiquement le client à un **VLAN** spécifique en fonction de son rôle (attributs Tunnel-Type, Tunnel-Medium-Type, Tunnel-Private-Group-ID).

---

## 2. Contexte de travail

Vous travaillez sur le contexte de la **SODECAF**, avec l'infrastructure montée précédemment. Votre objectif est de mettre en place une **authentification centralisée** des utilisateurs souhaitant se connecter au réseau de la SODECAF, en utilisant **PacketFence comme serveur RADIUS** à la place de Microsoft NPS.

### 2.1. Schéma de l'infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                      VLAN WIFI (192.168.50.0/24)                │
│  ┌──────────────┐          ┌────────────────┐                   │
│  │  Smartphone  │──WiFi────│  Point d'accès │                   │
│  │     WiFi     │          │   D-Link       │                   │
│  └──────────────┘          │   DAP1665      │                   │
│                            └────────┬───────┘                   │
└─────────────────────────────────────┼─────────────────────────────┘
                                      │
                     ┌────────────────┼────────────────┐
                     │   Commutateur Cisco 2960       │
                     │   IP: 192.168.10.200           │
                     │                                 │
                     │   Ports 1-4:  VLAN ADMIN       │
                     │   Ports 5-8:  VLAN DEV         │
┌──────────────┐     │   Ports 9-12: 802.1X (dynamic) │
│  PC Admin    │─────┤   Ports 13-16: VLAN WIFI       │
│  VLAN ADMIN  │     │   Ports 23-24: Trunk           │
└──────────────┘     └────────────────┬────────────────┘
                                      │ Trunk
┌──────────────┐                      │
│   PC Dev     │              ┌───────┴────────┐
│  VLAN DEV    │              │  Routeur Cisco │
└──────────────┘              │  G0/0/0: VLAN  │
                              │  G0/0/1: LAN   │
                              │  172.16.0.253  │
                              └───────┬────────┘
                                      │
         ┌────────────────────────────┴─────────────────────────┐
         │         Réseau serveurs (172.16.0.0/24)              │
         │                                                       │
         │  ┌──────────────────┐      ┌──────────────────┐    │
         │  │ Windows Server   │      │  PacketFence     │    │
         │  │ SRV-WIN1         │      │  (VM Linux)      │    │
         │  │ 172.16.0.1       │      │  172.16.0.10     │    │
         │  │                  │      │                  │    │
         │  │ - AD DS          │      │ - FreeRADIUS     │    │
         │  │ - DNS            │      │ - NAC            │    │
         │  │ - DHCP           │      │ - Portail invité │    │
         │  │ - (AD CS)        │      │                  │    │
         │  └──────────────────┘      └──────────────────┘    │
         └───────────────────────────────────────────────────────┘
```

### 2.2. Plan d'adressage et VLAN

**Configuration du commutateur Cisco 2960 :**
- IP de management : **192.168.10.200**
- Ports 1 à 4 : **VLAN ADMIN** (192.168.10.0/24)
- Ports 5 à 8 : **VLAN DEV** (192.168.20.0/24)
- Ports 9 à 12 : **802.1X avec VLAN dynamique** (ADMIN ou DEV selon authentification)
- Ports 13 à 16 : **VLAN WIFI** (192.168.50.0/24)
- Ports 23 à 24 : **Trunk** (vers routeur)

**Configuration du routeur :**
- G0/0/0 : sous-interfaces pour chaque VLAN (dernières adresses de chaque réseau)
  - G0/0/0.10 : 192.168.10.254 (VLAN ADMIN)
  - G0/0/0.20 : 192.168.20.254 (VLAN DEV)
  - G0/0/0.50 : 192.168.50.254 (VLAN WIFI)
- G0/0/1 : **172.16.0.253** (réseau serveurs)

**Serveurs :**
- Windows Server : **172.16.0.1** (AD DS, DNS, DHCP, éventuellement AD CS)
- PacketFence : **172.16.0.10** (interface Management, serveur RADIUS/NAC)

---

## 3. Ressources et prérequis

### 3.1. Documentation

- Cours sur l'authentification 802.1X et le protocole RADIUS
- Documentation PacketFence :
  - Installation Guide : https://www.packetfence.org/doc/PacketFence_Installation_Guide.html
  - Network Devices Configuration Guide : https://www.packetfence.org/doc/PacketFence_Network_Devices_Configuration_Guide.html
  - Getting Started : https://www.packetfence.org/support/documentation.html

### 3.2. Vidéos recommandées

- PacketFence configuration for wired connection 802.1x (YouTube)
- Tutoriels sur l'intégration Active Directory avec PacketFence

### 3.3. Matériel nécessaire

- 1 routeur Cisco (routage inter-VLAN)
- 1 commutateur Cisco 2960
- 1 point d'accès WiFi D-Link DAP1665 (ou équivalent supportant WPA2-Enterprise)
- 1 serveur Windows (AD DS, DHCP, DNS)
- 1 serveur PacketFence (VM Linux Debian/Ubuntu ou CentOS/AlmaLinux)
- 2-3 PC clients pour les tests (intégrés au domaine)
- Câbles réseau

---

## 4. Travail à réaliser

### 4.1. Configuration du commutateur et du routeur

**Objectif :** Mettre en place le routage inter-VLAN, le lien avec le réseau serveurs et préparer les VLAN qui seront utilisés pour l'authentification 802.1X.

**Tâches :**

1. **Câblez l'infrastructure** selon le schéma fourni :
   - PC clients vers le switch
   - Point d'accès WiFi vers le switch (port VLAN WIFI)
   - Switch vers routeur (trunk)
   - Serveurs vers routeur

2. **Configurez le routeur** :
   - Créez les sous-interfaces sur G0/0/0 pour chaque VLAN (encapsulation dot1Q)
   - Configurez les adresses IP de passerelle pour chaque VLAN
   - Configurez l'interface G0/0/1 pour le réseau serveurs (172.16.0.253/24)
   - Activez le routage IP

3. **Configurez le commutateur Cisco 2960** :
   - Créez les VLAN (ADMIN, DEV, WIFI)
   - Affectez les ports d'accès aux VLAN correspondants
   - Configurez les ports 23-24 en trunk (vers routeur)
   - Configurez l'adresse IP de management (192.168.10.200)
   - Configurez la passerelle par défaut

4. **Vérifiez la connectivité de base** :
   - Ping entre les VLAN via le routeur
   - Ping depuis les serveurs vers les VLAN

**Commandes Cisco utiles :**

```cisco
! Création des VLAN
vlan 10
 name ADMIN
vlan 20
 name DEV
vlan 50
 name WIFI

! Configuration des ports d'accès
interface range fa0/1-4
 switchport mode access
 switchport access vlan 10

interface range fa0/5-8
 switchport mode access
 switchport access vlan 20

interface range fa0/13-16
 switchport mode access
 switchport access vlan 50

! Configuration du trunk
interface range fa0/23-24
 switchport mode trunk
 switchport trunk allowed vlan 10,20,50

! IP de management
interface vlan 10
 ip address 192.168.10.200 255.255.255.0
ip default-gateway 192.168.10.254
```

**📝 Zone de réponse / captures d'écran :**

_(Insérez ici les captures de configuration du switch et du routeur, ainsi que les tests de connectivité)_

---

### 4.2. Configuration des étendues DHCP sur Windows Server

**Objectif :** Fournir les adresses IP aux clients des différents VLAN.

**Tâches :**

1. Sur le serveur Windows, ouvrez la console **DHCP**
2. Créez une **étendue DHCP** pour chaque VLAN :
   - **VLAN ADMIN** : 192.168.10.0/24 (plage 192.168.10.50-192.168.10.150)
     - Passerelle : 192.168.10.254
     - DNS : 172.16.0.1 (serveur AD)
   - **VLAN DEV** : 192.168.20.0/24 (plage 192.168.20.50-192.168.20.150)
     - Passerelle : 192.168.20.254
     - DNS : 172.16.0.1
   - **VLAN WIFI** : 192.168.50.0/24 (plage 192.168.50.50-192.168.50.150)
     - Passerelle : 192.168.50.254
     - DNS : 172.16.0.1

3. **Configurez le routeur comme agent relais DHCP** pour les VLAN :
   ```cisco
   interface GigabitEthernet0/0/0.10
    ip helper-address 172.16.0.1
   interface GigabitEthernet0/0/0.20
    ip helper-address 172.16.0.1
   interface GigabitEthernet0/0/0.50
    ip helper-address 172.16.0.1
   ```

4. **Testez l'obtention d'une adresse IP** :
   - Connectez un poste client dans chaque VLAN
   - Vérifiez l'adresse IP obtenue (`ipconfig /all`)
   - Testez la connectivité (ping passerelle, ping DNS)

**📝 Zone de réponse / captures d'écran :**

_(Captures des étendues DHCP configurées et tests d'obtention d'adresse IP)_

---

### 4.3. Configuration de l'Active Directory

**Objectif :** Préparer la base d'utilisateurs qui servira à l'authentification 802.1X.

**Tâches :**

1. **Créez des Unités d'Organisation (UO)** dans Active Directory :
   - UO **Utilisateurs_SODECAF**
     - Sous-UO **Admin**
     - Sous-UO **Dev**
     - Sous-UO **Visiteurs**

2. **Créez des groupes de sécurité** dans chaque UO :
   - **Groupe_Admin** (dans UO Admin)
   - **Groupe_Dev** (dans UO Dev)
   - **Groupe_Visiteurs** (dans UO Visiteurs)

3. **Créez des comptes utilisateurs** pour chaque groupe :
   - **Admin** : uadmin1, uadmin2 (membres de Groupe_Admin)
   - **Dev** : udev1, udev2 (membres de Groupe_Dev)
   - **Visiteurs** : uguest1, uguest2 (membres de Groupe_Visiteurs)

4. **Autorisez l'accès réseau** pour ces comptes :
   - Ouvrez les propriétés de chaque compte utilisateur
   - Onglet **"Appel entrant"** ou **"Dial-in"**
   - Sélectionnez **"Autoriser l'accès"** (Control access through NPS Network Policy / Permettre l'accès)

**📝 Zone de réponse / captures d'écran :**

_(Captures de la structure AD, des groupes créés et de l'autorisation d'accès réseau pour un compte)_

---

### 4.4. (Optionnel) Création d'une autorité de certification racine AD CS

> **Note :** Cette partie est **optionnelle** si vous restez sur l'authentification **PEAP-MSCHAPv2** (mot de passe). Elle devient **nécessaire** si vous souhaitez faire un bonus en **EAP-TLS** avec certificats côté client.

**Objectif :** Créer une autorité de certification d'entreprise pour émettre des certificats serveur et client.

**Contexte :** L'utilisation d'une Autorité de Certification (AC) et de certificats avec 802.1X offre une sécurité renforcée par rapport aux mots de passe. Avec **EAP-TLS**, l'authentification repose sur des certificats côté serveur et client, empêchant ainsi les attaques par interception (Man-in-the-Middle). Cela élimine les risques liés aux mots de passe faibles ou volés.

**Tâches :**

1. **Installez le rôle AD CS** sur le serveur Windows :
   - Ouvrez le **Gestionnaire de serveur**
   - Cliquez sur **"Ajouter des rôles et fonctionnalités"**
   - Sélectionnez **"Services de certificats Active Directory (AD CS)"**
   - Installez le rôle

2. **Configurez l'autorité de certification** :
   - Type : **AC d'entreprise racine**
   - Nom commun : **CA-SODECAF**
   - Validez la configuration

3. **Vérifiez le déploiement du certificat racine** :
   - Sur un poste client joint au domaine, vérifiez que le certificat **CA-SODECAF** apparaît dans les autorités de certification racines de confiance
   - `certmgr.msc` → Autorités de certification racines de confiance → Certificats

**Ressource :** Tutoriel IT-Connect sur l'autorité de certification :  
https://www.it-connect.fr/adcs-creer-une-autorite-de-certification-racine-sous-windows-server/

**📝 Zone de réponse / captures d'écran :**

_(Si réalisé : captures de l'installation AD CS et du certificat racine déployé sur les clients)_

---

## 5. Installation et configuration de PacketFence

### 5.1. Installation de PacketFence

**Objectif :** Déployer un serveur PacketFence jouant le rôle de serveur RADIUS et de NAC.

**Tâches :**

1. **Créez une VM Linux** pour PacketFence :
   - OS recommandé : **Debian 12** ou **Ubuntu Server 22.04 LTS** ou **AlmaLinux 9**
   - RAM : **minimum 4 Go** (8 Go recommandés pour production)
   - CPU : **2 vCPU minimum**
   - Disque : **50 Go minimum**
   - Interface réseau : **1 interface** en mode bridge (pour le réseau Management)

2. **Configurez l'adresse IP fixe** de la VM :
   - IP : **172.16.0.10**
   - Masque : **255.255.255.0** (/24)
   - Passerelle : **172.16.0.253** (routeur)
   - DNS : **172.16.0.1** (serveur AD)

   Exemple sur Debian/Ubuntu (`/etc/network/interfaces` ou Netplan) :
   ```yaml
   network:
     version: 2
     ethernets:
       ens18:
         addresses:
           - 172.16.0.10/24
         gateway4: 172.16.0.253
         nameservers:
           addresses:
             - 172.16.0.1
   ```

3. **Installez PacketFence** selon la documentation officielle :

   Pour Debian/Ubuntu :
   ```bash
   # Ajout du dépôt PacketFence
   wget -qO - https://inverse.ca/downloads/GPG_PUBLIC_KEY | sudo apt-key add -
   echo "deb http://inverse.ca/downloads/PacketFence/debian/12 buster buster" | sudo tee /etc/apt/sources.list.d/packetfence.list
   
   # Installation
   sudo apt update
   sudo apt install packetfence
   
   # Configuration initiale
   sudo /usr/local/pf/bin/pfcmd configreload hard
   ```

4. **Lancez l'assistant de configuration** :
   - Accédez à l'interface Web : https://172.16.0.10:1443
   - Suivez l'assistant de configuration initiale
   - Configurez l'interface **Management** (172.16.0.10/24)

**📝 Zone de réponse / captures d'écran :**

_(Captures de l'installation PacketFence et de l'assistant de configuration initiale)_

---

### 5.2. Intégration de PacketFence à l'Active Directory

**Objectif :** Permettre l'authentification 802.1X des utilisateurs AD via PacketFence.

**Tâches :**

1. **Accédez à l'interface Web de PacketFence** :
   - URL : https://172.16.0.10:1443
   - Connectez-vous avec le compte administrateur créé lors de l'installation

2. **Ajoutez une source d'authentification Active Directory** :
   - Menu : **Configuration → Policies and Access Control → Authentication Sources**
   - Cliquez sur **"New Authentication Source"**
   - Type : **"Active Directory"**
   - Configuration :
     - **Identifier** : AD_SODECAF
     - **Host** : 172.16.0.1 (ou sodecaf.local)
     - **Base DN** : `DC=sodecaf,DC=local`
     - **Bind DN** : `CN=Administrator,CN=Users,DC=sodecaf,DC=local` (ou compte service)
     - **Password** : mot de passe administrateur AD
     - **Connection type** : LDAP (ou LDAPS si certificat configuré)

3. **Testez la connexion** :
   - Utilisez le bouton **"Test"** pour vérifier la connexion à l'AD
   - Testez la résolution d'un groupe (ex : Groupe_Admin)

4. **(Optionnel) Joignez PacketFence au domaine** :
   - Pour PEAP-MSCHAPv2 avec authentification NTLM/Kerberos, rejoignez le domaine :
   ```bash
   sudo realm join sodecaf.local -U Administrator
   ```

**📝 Zone de réponse / captures d'écran :**

_(Captures de la configuration de la source d'authentification AD et du test de connexion)_

---

### 5.3. Création des rôles et mapping VLAN

**Objectif :** Définir les rôles PacketFence correspondant aux groupes AD et associer les VLAN à appliquer.

**Tâches :**

1. **Créez des rôles** dans PacketFence :
   - Menu : **Configuration → Policies and Access Control → Roles**
   - Créez les rôles suivants :
     - **role_admin** → VLAN ID : **10** (VLAN ADMIN)
     - **role_dev** → VLAN ID : **20** (VLAN DEV)
     - **role_guest** → VLAN ID : **50** (VLAN WIFI/Visiteurs)

2. **Configurez les règles de mapping** entre groupes AD et rôles :
   - Menu : **Configuration → Policies and Access Control → Connection Profiles**
   - Créez ou modifiez le profil de connexion par défaut
   - Ajoutez des **règles d'accès** (Access Rules) :
     - **Règle 1** :
       - Condition : `memberOf = CN=Groupe_Admin,OU=Admin,OU=Utilisateurs_SODECAF,DC=sodecaf,DC=local`
       - Action : Assign role **role_admin**
     - **Règle 2** :
       - Condition : `memberOf = CN=Groupe_Dev,OU=Dev,OU=Utilisateurs_SODECAF,DC=sodecaf,DC=local`
       - Action : Assign role **role_dev**
     - **Règle 3** :
       - Condition : `memberOf = CN=Groupe_Visiteurs,OU=Visiteurs,OU=Utilisateurs_SODECAF,DC=sodecaf,DC=local`
       - Action : Assign role **role_guest**

3. **Vérifiez la correspondance** :
   - Les VLAN ID configurés dans les rôles doivent correspondre aux VLAN du commutateur
   - Le VLAN par défaut (default role) doit être configuré (par exemple, VLAN d'inscription ou VLAN restreint)

**📝 Zone de réponse / captures d'écran :**

_(Captures de la création des rôles et du mapping groupes AD → rôles)_

---

### 5.4. Configuration de PacketFence comme serveur RADIUS 802.1X

**Objectif :** Activer 802.1X et préparer les profils d'accès pour filaire et WiFi.

**Tâches :**

1. **Vérifiez l'activation du service RADIUS** :
   - Menu : **Configuration → Services**
   - Vérifiez que **radiusd** est activé et démarré
   - Ports RADIUS par défaut : **1812** (auth) et **1813** (accounting)

2. **Configurez les méthodes EAP** :
   - Menu : **Configuration → Policies and Access Control → Connection Profiles**
   - Pour le profil par défaut ou un profil dédié :
     - **Sources** : sélectionnez **AD_SODECAF**
     - **EAP methods** : cochez **PEAP** (et MSCHAPv2)

3. **Configurez un certificat pour le serveur RADIUS** :
   - PacketFence génère automatiquement un certificat auto-signé
   - Pour utiliser un certificat de l'AD CS (optionnel) :
     - Exportez un certificat depuis l'AD CS
     - Importez-le dans PacketFence : **Configuration → System Configuration → SSL Certificates**

4. **Vérifiez la configuration RADIUS** :
   - Menu : **Configuration → Network Configuration → RADIUS**
   - Vérifiez les paramètres par défaut (ports, timeout, etc.)

**📝 Zone de réponse / captures d'écran :**

_(Captures de la configuration RADIUS et des méthodes EAP)_

---

## 6. Configuration des équipements réseau comme clients RADIUS

### 6.1. Déclaration du commutateur Cisco 2960 dans PacketFence

**Objectif :** Faire du Cisco 2960 un client RADIUS de PacketFence.

**Tâches :**

1. **Ajoutez le commutateur comme équipement réseau** :
   - Menu : **Configuration → Network Configuration → Switches**
   - Cliquez sur **"New Switch"**
   - Configuration :
     - **IP Address** : 192.168.10.200
     - **Description** : Commutateur Cisco 2960 - SODECAF
     - **Type** : Cisco → Catalyst 2960
     - **Mode** : Production
     - **RADIUS Secret** : **BtssioPF2026** (clé secrète partagée)
     - **Role by VLAN ID** : activé
     - **Registration VLAN** : 10 (ou VLAN par défaut)

2. **Vérifiez les paramètres RADIUS** :
   - Onglet **RADIUS** :
     - Port : 1812
     - Secret : BtssioPF2026 (même que ci-dessus)

3. **Sauvegardez** la configuration

**📝 Zone de réponse / captures d'écran :**

_(Captures de la déclaration du switch dans PacketFence)_

---

### 6.2. Configuration 802.1X sur le commutateur Cisco 2960

**Objectif :** Utiliser PacketFence comme serveur RADIUS pour l'authentification 802.1X filaire.

**Tâches :**

Ajoutez la configuration suivante sur le commutateur Cisco 2960 (en adaptant l'IP et la clé) :

```cisco
! Activation du modèle AAA
aaa new-model

! Création du groupe de serveurs RADIUS PacketFence
aaa group server radius PACKETFENCE
 server 172.16.0.10 auth-port 1812 acct-port 1813
!

! Configuration de l'authentification et de l'autorisation 802.1X
aaa authentication dot1x default group PACKETFENCE
aaa authorization network default group PACKETFENCE

! Définition du serveur RADIUS
radius-server host 172.16.0.10 auth-port 1812 acct-port 1813 key BtssioPF2026

! Activation globale de 802.1X sur le switch
dot1x system-auth-control

! Configuration des ports 9 à 12 pour 802.1X
interface range FastEthernet0/9-12
 switchport mode access
 switchport access vlan 10
 authentication port-control auto
 dot1x pae authenticator
 dot1x timeout tx-period 10
 ! Activer la réception des attributs VLAN du serveur RADIUS
 authentication periodic
 authentication timer reauthenticate server
```

**Points clés :**
- `authentication port-control auto` : active 802.1X sur le port
- `dot1x pae authenticator` : définit le switch comme authenticator
- Les attributs RADIUS (Tunnel-Private-Group-ID, etc.) permettront l'affectation dynamique de VLAN
- Les ports 9-12 seront utilisés pour les tests filaires 802.1X

**Vérification :**
```cisco
show dot1x all
show aaa servers
show radius statistics
```

**📝 Zone de réponse / captures d'écran :**

_(Captures de la configuration 802.1X sur le switch et des commandes de vérification)_

---

### 6.3. Configuration du point d'accès WiFi

**Objectif :** Utiliser PacketFence comme serveur RADIUS pour l'authentification WiFi WPA2-Enterprise.

**Tâches :**

1. **Déclarez le point d'accès dans PacketFence** :
   - Menu : **Configuration → Network Configuration → Switches**
   - Cliquez sur **"New Switch"**
   - Configuration :
     - **IP Address** : adresse IP du point d'accès (ex : 192.168.50.10)
     - **Type** : Generic → Access Point
     - **RADIUS Secret** : **BtssioPF2026**

2. **Accédez à l'interface Web du point d'accès** :
   - Configurez son adresse IP dans le VLAN WIFI (192.168.50.10 par exemple)

3. **Configurez le SSID** :
   - **SSID** : SISR-x (x = votre numéro de poste, ex. SISR-A1)
   - **Mode de sécurité** : WPA2-Enterprise / 802.1X
   - **Type d'authentification** : RADIUS

4. **Configurez le serveur RADIUS** :
   - **IP du serveur RADIUS** : 172.16.0.10 (PacketFence)
   - **Port** : 1812
   - **Clé secrète** : BtssioPF2026
   - **Port Accounting** : 1813

5. **Activez le mode 802.1X** sur le SSID

6. **Vérifiez la configuration** et appliquez les modifications

**📝 Zone de réponse / captures d'écran :**

_(Captures de la configuration du point d'accès WiFi)_

---

## 7. Configuration du PC client (Windows)

**Objectif :** Préparer le poste pour 802.1X filaire et WiFi avec PEAP.

### 7.1. Intégration au domaine

1. **Intégrez le PC client au domaine SODECAF** :
   - Paramètres système → Système → À propos → Domaine ou groupe de travail
   - Joindre le domaine **sodecaf.local**
   - Redémarrez le PC

### 7.2. Activation du service d'authentification réseau (filaire)

1. **Démarrez le service dot3svc** (Configuration automatique de réseau câblé) :
   - En ligne de commande (administrateur) :
     ```cmd
     net start dot3svc
     ```
   - Ou via `services.msc` :
     - Service : **Configuration automatique de réseau câblé**
     - Type de démarrage : **Automatique**
     - Démarrez le service

### 7.3. Configuration 802.1X filaire

1. Ouvrez **Centre Réseau et partage** → **Modifier les paramètres de la carte**
2. Clic droit sur **Ethernet** → **Propriétés**
3. Onglet **Authentification** :
   - ☑ **Activer l'authentification IEEE 802.1X pour ce réseau**
   - **Méthode d'authentification** : Microsoft : Protected EAP (PEAP)
   - Cliquez sur **Paramètres**

4. **Paramètres PEAP** :
   - ☑ **Valider le certificat du serveur**
   - **Autorité de certification racine de confiance** : sélectionnez **CA-SODECAF** (si AD CS installé) ou le certificat PacketFence
   - **Notifications avant de se connecter** : selon préférence
   - **Méthode d'authentification** : Mot de passe sécurisé (EAP-MSCHAP v2)
   - Cliquez sur **Configurer**

5. **Propriétés EAP MSCHAPv2** :
   - ☑ **Utiliser automatiquement mon nom et mon mot de passe de session Windows** (et éventuellement le domaine)
   - OK

6. **Paramètres avancés** (onglet Authentification) :
   - **Mode d'authentification** : Authentification de l'utilisateur
   - OK

### 7.4. Configuration 802.1X WiFi

1. Connectez-vous au SSID **SISR-x**
2. Windows demandera automatiquement la configuration 802.1X
3. Ou configurez manuellement :
   - **Centre Réseau et partage** → **Gérer les réseaux sans fil**
   - Propriétés du SSID → Onglet **Sécurité**
   - **Type de sécurité** : WPA2-Enterprise
   - **Méthode d'authentification réseau** : Microsoft : Protected EAP (PEAP)
   - **Paramètres** : identiques à la configuration filaire ci-dessus

**📝 Zone de réponse / captures d'écran :**

_(Captures de la configuration 802.1X sur le client Windows, filaire et WiFi)_

---

## 8. Tests et validation WiFi (visiteur)

**Objectif :** Valider l'authentification 802.1X WiFi via PacketFence et l'affectation dynamique de VLAN visiteurs.

### 8.1. Test de connexion avec un compte visiteur

1. **Connectez-vous au SSID SISR-x** avec un compte du groupe **Visiteurs** (ex : uguest1)
2. **Vérifiez l'authentification** :
   - La connexion WiFi doit réussir
   - Le client doit obtenir une adresse IP dans le **VLAN WIFI** (192.168.50.x)
   - Commande : `ipconfig /all`

3. **Vérifiez la connectivité réseau** :
   - Ping de la passerelle (192.168.50.254)
   - Ping d'une ressource externe (si autorisé)

### 8.2. Consultation des logs

1. **Côté PacketFence** :
   - Menu : **Status → Logs → radius**
   - Recherchez les logs d'authentification pour l'utilisateur uguest1
   - Vérifiez le rôle attribué (role_guest) et le VLAN (50)

2. **Côté point d'accès** (si accessible) :
   - Consultez les logs d'authentification RADIUS

### 8.3. Test d'accès refusé

1. **Tentez de vous connecter** avec un utilisateur **non membre** du groupe Visiteurs (ex : compte non autorisé)
2. **Vérifiez** :
   - La connexion doit échouer
   - Ou être placée dans un VLAN par défaut/restreint selon votre configuration

**📝 Zone de réponse / captures d'écran :**

_(Captures des tests WiFi, adresse IP obtenue, logs PacketFence)_

---

## 9. Tests et validation filaire (VLAN dynamiques)

**Objectif :** Valider l'authentification 802.1X filaire avec affectation de VLAN en fonction du groupe AD (Admin / Dev).

### 9.1. Test avec un compte Admin

1. **Branchez le PC client** sur un port **9 à 12** du commutateur
2. **Connectez-vous** avec un compte membre du groupe **Admin** (ex : uadmin1)
3. **Vérifiez l'authentification** :
   - L'authentification 802.1X doit réussir
   - Le port doit être placé dans le **VLAN ADMIN** (VLAN 10)
   - Le client doit obtenir une adresse IP dans la plage **192.168.10.x**

4. **Sur le commutateur, vérifiez** :
   ```cisco
   show authentication sessions
   show authentication sessions interface fa0/9
   show vlan
   ```
   - Le port doit apparaître dans le VLAN 10 (ADMIN)

5. **Consultez les logs PacketFence** :
   - Vérifiez l'attribution du rôle **role_admin** et du VLAN 10

### 9.2. Test avec un compte Dev

1. **Déconnectez-vous** et **reconnectez-vous** avec un compte membre du groupe **Dev** (ex : udev1)
2. **Vérifiez** :
   - Authentification réussie
   - Port placé dans le **VLAN DEV** (VLAN 20)
   - Adresse IP dans la plage **192.168.20.x**

3. **Sur le commutateur, vérifiez** :
   ```cisco
   show authentication sessions interface fa0/9
   show vlan
   ```
   - Le port doit maintenant apparaître dans le VLAN 20 (DEV)

### 9.3. Vérification côté client

1. Sur le client Windows, exécutez :
   ```cmd
   netsh lan show interfaces
   netsh lan show profiles
   ```
   - Vérifiez l'état de la connexion 802.1X

2. Consultez l'**Observateur d'événements** :
   - Journaux Windows → Système
   - Recherchez les événements liés à 802.1X (source : Wired-AutoConfig)

### 9.4. Debug (si nécessaire)

Sur le commutateur Cisco, activez le debug temporairement :
```cisco
debug dot1x events
debug radius
terminal monitor
```

**N'oubliez pas de désactiver le debug après les tests :**
```cisco
no debug dot1x events
no debug radius
```

**📝 Zone de réponse / captures d'écran :**

_(Captures des tests filaires pour Admin et Dev : adresses IP, show authentication sessions, logs PacketFence)_

---

## 10. Synthèse et analyse

### 10.1. Tableau comparatif NPS vs PacketFence

| Aspect                          | NPS (Windows Server)              | PacketFence                        |
|---------------------------------|-----------------------------------|------------------------------------|
| **Rôle RADIUS**                 | Oui (basique, intégré Windows)    | Oui, basé sur FreeRADIUS (flexible)|
| **Intégration Active Directory**| Native Windows                    | Via LDAP/NTLM, jointure domaine    |
| **VLAN dynamique 802.1X**       | Oui (par policy NPS)              | Oui (par rôles et profils)         |
| **Portail invité / BYOD**       | Non natif                         | Oui (captive portal intégré)       |
| **Profiling et détection**      | Limité                            | Oui (fingerprinting, DHCP, etc.)   |
| **Quarantaine / Isolation**     | Via politique réseau              | Natif (VLAN isolation)             |
| **Plateforme**                  | Windows Server                    | Linux (VM dédiée, appliance)       |
| **Coût**                        | Licence Windows Server            | Open source (gratuit)              |
| **Complexité**                  | Simple (environnement Microsoft)  | Plus complexe (Linux, config)      |

### 10.2. Questions de réflexion

1. **Quels sont les avantages et inconvénients** de PacketFence par rapport à NPS pour la SODECAF ?
2. **Quel est l'intérêt** de l'authentification 802.1X par rapport à une simple authentification par clé PSK (WPA2-Personal) ?
3. **Comment PacketFence améliore-t-il la sécurité** du réseau de la SODECAF au-delà de la simple authentification ?
4. **Quelles sont les limitations** de la solution mise en place ? (scalabilité, gestion des exceptions, etc.)
5. **Dans quel contexte** une entreprise choisirait-elle PacketFence plutôt qu'une solution commerciale comme Cisco ISE ou Aruba ClearPass ?

**📝 Zone de réponse :**

_(Rédigez vos réponses aux questions ci-dessus)_

---

## 11. Bonus (optionnel)

### 11.1. Portail invité captif

**Objectif :** Mettre en place un portail invité simple dans PacketFence pour le VLAN visiteurs.

1. Activez le portail captif dans PacketFence
2. Configurez un profil de connexion pour les invités (avec enregistrement par formulaire)
3. Testez l'accès au réseau via le portail Web après connexion WiFi
4. Vérifiez que l'accès est accordé après authentification Web

### 11.2. VLAN de quarantaine

**Objectif :** Isoler automatiquement un compte ou une machine non conforme.

1. Créez un rôle **role_quarantine** avec un VLAN dédié (ex : VLAN 99)
2. Créez une règle dans PacketFence pour placer certains utilisateurs en quarantaine
3. Testez avec un compte spécifique

### 11.3. Authentification EAP-TLS avec certificats

**Objectif :** Remplacer PEAP-MSCHAPv2 par EAP-TLS pour une sécurité maximale.

**Prérequis :** AD CS installé et configuré (partie 4.4)

1. **Émettez des certificats** pour les utilisateurs :
   - Créez un modèle de certificat "Authentification utilisateur"
   - Déployez les certificats via GPO ou demande manuelle

2. **Configurez PacketFence** pour accepter EAP-TLS :
   - Activez EAP-TLS dans le profil de connexion
   - Vérifiez le certificat serveur RADIUS

3. **Configurez le client** pour utiliser EAP-TLS :
   - Méthode : Microsoft : Carte à puce ou autre certificat (EAP-TLS)
   - Sélectionnez le certificat utilisateur

4. **Testez** et **comparez** avec PEAP du point de vue sécurité

**📝 Zone de réponse bonus :**

_(Captures et observations des fonctionnalités bonus testées)_

---

## 12. Conclusion

Dans ce TP, vous avez :
- Mis en place une **authentification 802.1X centralisée** avec PacketFence comme serveur RADIUS
- Intégré PacketFence à l'**Active Directory** pour l'authentification des utilisateurs
- Configuré l'**affectation dynamique de VLAN** en fonction des groupes AD (Admin, Dev, Visiteurs)
- Testé l'authentification **filaire et WiFi** avec PEAP-MSCHAPv2
- Comparé PacketFence à Microsoft NPS

Cette solution offre une **flexibilité accrue** et des **fonctionnalités NAC avancées** (portail invité, profiling, quarantaine) par rapport à NPS, tout en restant basée sur des standards ouverts (FreeRADIUS, 802.1X).

---

**Fin de l'activité B3-Act10-TP1**

**Félix Le Dantec - BTS SIO SISR - 2026**
