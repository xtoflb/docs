# Synthèse – Mise en œuvre d’IPv6

Cette synthèse rappelle les notions et gestes techniques clés travaillés dans l’activité de mise en œuvre d’IPv6 avec un firewall, un LAN, une DMZ et des postes clients.

---

## 1. Étapes générales de configuration IPv6 sur un firewall

1. **Définir le plan d’adressage IPv6**

- Identifier le préfixe IPv6 fourni par le FAI (souvent un /56 ou un /48).
- Découper ce préfixe en sous-réseaux /64 :
  - au moins un /64 pour le réseau **LAN** (postes internes),
  - un /64 pour la **DMZ** (serveurs exposés),
  - éventuellement d’autres /64 pour d’autres VLANs.

2. **Configurer IPv6 sur les interfaces du routeur / firewall**

- Activer globalement la prise en charge d’IPv6 dans les paramètres généraux.
- Sur chaque interface (LAN, DMZ, etc.) :
  - choisir un type d’adresse IPv6 (souvent « statique » en lab),
  - attribuer une adresse IPv6 lisible pour l’admin (par exemple `…::1/64`).

3. **Activer les Router Advertisements (RA)**

- Activer les RA sur les interfaces où les clients doivent s’auto-configurer.
- Choisir le mode en fonction de la stratégie d’auto-configuration :
  - **SLAAC pur** : les clients construisent leur adresse à partir du préfixe du RA, sans DHCPv6.
  - **SLAAC + DHCPv6 stateless** : adresse via SLAAC, options (DNS, etc.) via DHCPv6.
  - **DHCPv6 stateful** : adresse et options fournies par DHCPv6 (les RA indiquent de contacter DHCPv6).

4. **Définir la politique de filtrage IPv6**

- Appliquer une politique de base « tout bloquer » puis ouvrir les flux nécessaires.
- Sur le LAN : autoriser le trafic sortant IPv6 vers Internet/DMZ et laisser passer les ICMPv6 indispensables (NDP, PMTUD, etc.).
- Sur la DMZ : n’autoriser que les ports nécessaires vers les serveurs (par exemple HTTP/HTTPS) et limiter sévèrement les flux de la DMZ vers le LAN.

---

## 2. Auto-configuration IPv6 : SLAAC et DHCPv6

### SLAAC (Stateless Address Autoconfiguration)

- Le routeur envoie des **Router Advertisements (RA)** ICMPv6 qui contiennent :
  - le **préfixe /64**,
  - la passerelle par défaut,
  - la longueur de préfixe,
  - les indicateurs **M** et **O**.
- Le client fabrique automatiquement son adresse :
  - 64 bits de gauche = préfixe du RA,
  - 64 bits de droite = identifiant d’interface (IID), dérivé de la MAC (EUI-64) ou généré aléatoirement.

Les indicateurs M et O dans le RA :

- **M (Managed address configuration flag)** :
  - M = 0 : l’adresse IPv6 n’est pas gérée par DHCPv6 → le client peut utiliser **SLAAC** pour construire son adresse.
  - M = 1 : l’adresse IPv6 est gérée par DHCPv6 → le client doit utiliser **DHCPv6 stateful** pour obtenir son adresse.
- **O (Other configuration flag)** :
  - O = 0 : pas d’autres informations à récupérer via DHCPv6 (tout est dans le RA ou configuré autrement).
  - O = 1 : le client doit utiliser **DHCPv6** pour récupérer des options (DNS, domaine, NTP, …), même si l’adresse vient de SLAAC (**DHCPv6 stateless**).

Combinaisons usuelles :

- M=0, O=0 : **SLAAC pur** (adresse via SLAAC, pas de DHCPv6).
- M=0, O=1 : SLAAC + **DHCPv6 stateless** (options seulement).
- M=1, O=x : **DHCPv6 stateful** (adresse + options via DHCPv6).

### DHCPv6 stateful (avec état)

- Le serveur DHCPv6 attribue une **adresse IPv6 complète** et les options (DNS, domaine, NTP, etc.).
- Il conserve un **état des baux** (durées, réservations, logs).
- Les RA indiquent au client qu’il doit utiliser DHCPv6 (M=1).

---

## 3. Commandes utiles sur les clients

### Côté Windows
```powershell
ipconfig /release6
ipconfig /renew6
ipconfig
ipconfig /all
