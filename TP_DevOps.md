# 🚀 TP DevOps : Déploiement Automatisé avec Ansible

**Durée :** 4h  
**Niveau :** BTS SIO 2ème année  
**Objectif :** Déployer automatiquement un serveur web Nginx avec une page personnalisée en utilisant Ansible.

## 📋 Table des matières
- [Architecture du Lab](#-architecture-du-lab)
- [Prérequis & Installation](#-prérequis--installation)
- [Étape 1 : Mise en place de l'environnement](#-étape-1--mise-en-place-de-lenvironnement-30-minutes)
- [Étape 2 : Configuration d'Ansible](#-étape-2--configuration-dansible-30-minutes)
- [Étape 3 : Premier Playbook](#-étape-3--premier-playbook-1-heure)
- [Étape 4 : Déploiement du site personnalisé](#-étape-4--déploiement-du-site-personnalisé-1-heure)
- [Étape 5 : Amélioration avec les Handlers](#-étape-5--amélioration-avec-les-handlers-45-minutes)
- [Étape 6 : Versionning avec Git](#-étape-6--versionning-avec-git-30-minutes)
- [Validation Finale](#-validation-finale)
- [Dépannage](#-dépannage---aide-mémoire)
- [Compte-rendu](#-compte-rendu-à-rendre)

## 📁 Architecture du Lab

```
Controller (Ansible) → SSH → Webserver (Cible)
     ↓
Playbook Ansible
```

## 🛠 Prérequis & Installation

**Logiciels nécessaires :**
- VirtualBox
- Vagrant
- Navigateur web

**Vérification de l'installation :**
```bash
vagrant --version
```

## 📥 Étape 1 : Mise en place de l'environnement (30 minutes)

### Tâche 1.1 : Télécharger et démarrer les VMs

1. **Créer un dossier de travail :**
   ```bash
   mkdir tp-devops
   cd tp-devops
   ```

2. **Créer le fichier `Vagrantfile` :**
   ```ruby
   # Vagrantfile
   Vagrant.configure("2") do |config|
     # Machine contrôleur Ansible
     config.vm.define "controller" do |controller|
       controller.vm.box = "ubuntu/jammy64"
       controller.vm.hostname = "controller"
       controller.vm.network "private_network", ip: "192.168.60.10"
       controller.vm.provider "virtualbox" do |vb|
         vb.memory = "1024"
         vb.cpus = 1
       end
     end

     # Machine cible pour le déploiement
     config.vm.define "webserver" do |web|
       web.vm.box = "ubuntu/jammy64"
       web.vm.hostname = "webserver"
       web.vm.network "private_network", ip: "192.168.60.20"
       web.vm.provider "virtualbox" do |vb|
         vb.memory = "1024"
         vb.cpus = 1
       end
     end
   end
   ```

3. **Démarrer les machines :**
   ```bash
   vagrant up
   ```

### Tâche 1.2 : Installation d'Ansible sur le controller

1. **Se connecter au controller :**
   ```bash
   vagrant ssh controller
   ```

2. **Installer Ansible :**
   ```bash
   sudo apt update
   sudo apt install -y ansible
   ```

3. **Vérifier l'installation :**
   ```bash
   ansible --version
   ```

## 🔧 Étape 2 : Configuration d'Ansible (30 minutes)

### Tâche 2.1 : Configurer l'inventaire

1. **Créer le dossier du projet :**
   ```bash
   mkdir ~/ansible-project
   cd ~/ansible-project
   ```

2. **Créer le fichier d'inventaire `inventory.ini` :**
   ```ini
   # inventory.ini
   [webservers]
   webserver ansible_host=192.168.60.20

   [all:vars]
   ansible_user=vagrant
   ansible_ssh_private_key_file=~/.ssh/id_rsa
   ```

### Tâche 2.2 : Configuration de la connexion SSH

1. **Générer une clé SSH :**
   ```bash
   ssh-keygen -t rsa -b 2048 -f ~/.ssh/id_rsa -N ""
   ```

2. **Copier la clé publique vers le webserver :**
   ```bash
   ssh-copy-id vagrant@192.168.60.20
   # Mot de passe : 'vagrant'
   ```

### Tâche 2.3 : Test de connexion

1. **Tester la connexion avec ping :**
   ```bash
   ansible webservers -i inventory.ini -m ping
   ```

2. **Tester une commande shell :**
   ```bash
   ansible webservers -i inventory.ini -m shell -a "uptime"
   ```

## 📜 Étape 3 : Premier Playbook (1 heure)

### Tâche 3.1 : Créer la structure du playbook

**Créer le fichier `deploy-website.yml` :**
```yaml
# deploy-website.yml
---
- name: Déploiement du serveur web Nginx
  hosts: webservers
  become: yes  # Devient root pour les tâches
  tasks:
```

### Tâche 3.2 : Ajouter les tâches de base

**Compléter le playbook :**
```yaml
---
- name: Déploiement du serveur web Nginx
  hosts: webservers
  become: yes
  
  tasks:
    - name: Mettre à jour le cache des paquets
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Installer le paquet nginx
      apt:
        name: nginx
        state: present

    - name: Démarrer le service nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

### Tâche 3.3 : Exécuter le playbook

```bash
ansible-playbook -i inventory.ini deploy-website.yml
```

**✅ Validation :** Ouvrir un navigateur sur `http://192.168.60.20`. Vous devriez voir la page par défaut de Nginx.

## 🌐 Étape 4 : Déploiement du site personnalisé (1 heure)

### Tâche 4.1 : Créer le contenu du site

1. **Créer le dossier pour les fichiers :**
   ```bash
   mkdir files
   ```

2. **Créer le fichier `files/index.html` :**
   ```html
   <!DOCTYPE html>
   <html>
   <head>
       <title>TP DevOps BTS SIO</title>
       <style>
           body { font-family: Arial, sans-serif; margin: 40px; }
           header { background: #2c3e50; color: white; padding: 20px; text-align: center; }
           .container { max-width: 800px; margin: 0 auto; }
           .student-info { background: #ecf0f1; padding: 15px; margin: 20px 0; }
       </style>
   </head>
   <body>
       <header>
           <h1>🚀 TP DevOps - Déploiement Ansible</h1>
           <p>BTS SIO - Votre Nom Ici</p>
       </header>
       
       <div class="container">
           <div class="student-info">
               <h2>Informations de déploiement</h2>
               <p><strong>Serveur:</strong> Webserver</p>
               <p><strong>Déployé le:</strong> Aujourd'hui</p>
           </div>
           
           <h2>🎯 Objectifs du TP</h2>
           <ul>
               <li>Déployer automatiquement avec Ansible</li>
               <li>Configurer un serveur web Nginx</li>
               <li>Gérer le contenu déployé</li>
               <li>Utiliser les handlers pour les services</li>
           </ul>
       </div>
   </body>
   </html>
   ```

### Tâche 4.2 : Modifier le playbook pour déployer le site

**Ajouter cette tâche AVANT la tâche de démarrage du service Nginx :**
```yaml
    - name: Déployer la page HTML personnalisée
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
```

### Tâche 4.3 : Tester le déploiement complet

```bash
ansible-playbook -i inventory.ini deploy-website.yml
```

**✅ Validation :** Rafraîchir la page `http://192.168.60.20`. Vous devriez voir votre page personnalisée.

## ⚡ Étape 5 : Amélioration avec les Handlers (45 minutes)

### Tâche 5.1 : Créer un handler pour redémarrer Nginx

**Ajouter cette section en fin de playbook (après `tasks`) :**
```yaml
  handlers:
    - name: Redémarrer nginx
      service:
        name: nginx
        state: restarted
```

### Tâche 5.2 : Modifier la tâche de déploiement

**Remplacer la tâche de déploiement de la page HTML par :**
```yaml
    - name: Déployer la page HTML personnalisée
      copy:
        src: files/index.html
        dest: /var/www/html/index.html
        owner: www-data
        group: www-data
        mode: '0644'
      notify: Redémarrer nginx
```

### Tâche 5.3 : Tester l'idempotence

**Exécuter le playbook plusieurs fois :**
```bash
ansible-playbook -i inventory.ini deploy-website.yml
```

**✅ Validation :** Le handler ne doit se déclencher que si le fichier a changé.

## 📊 Étape 6 : Versionning avec Git (30 minutes)

### Tâche 6.1 : Initialiser un dépôt Git

```bash
git init
git config user.name "Votre Nom"
git config user.email "votre.email@edu.fr"
```

### Tâche 6.2 : Créer le fichier .gitignore

```bash
# .gitignore
*.retry
.vagrant/
*.vdi
*.vbox*
```

### Tâche 6.3 : Committer le projet

```bash
git add .
git commit -m "TP DevOps - Déploiement Nginx avec Ansible"
```

## 🎯 Validation Finale

### Checklist de validation

- [ ] Les machines Vagrant démarrent sans erreur
- [ ] La connexion SSH entre controller et webserver fonctionne
- [ ] Le playbook s'exécute sans erreur
- [ ] Le site web est accessible sur http://192.168.60.20
- [ ] La page personnalisée s'affiche correctement
- [ ] Le playbook est idempotent
- [ ] Le handler redémarre Nginx seulement si nécessaire
- [ ] Le projet est versionné avec Git

## 🔍 Dépannage - Aide Mémoire

### Problème de connexion SSH
```bash
# Vérifier que la clé est bien copiée
ssh vagrant@192.168.60.20

# Régénérer la clé si besoin
ssh-keygen -R 192.168.60.20
ssh-copy-id vagrant@192.168.60.20
```

### Problème de syntaxe Ansible
```bash
# Vérifier la syntaxe YAML
ansible-playbook --syntax-check deploy-website.yml

# Exécuter en mode verbose
ansible-playbook -i inventory.ini deploy-website.yml -v
```

### Redémarrer depuis le début
```bash
# Depuis l'hôte physique
vagrant destroy -f
vagrant up
```

## 📝 Compte-rendu à Rendre

### Éléments à fournir :
1. Le playbook Ansible final (`deploy-website.yml`)
2. La page HTML déployée (`files/index.html`)
3. Un screenshot du site accessible
4. Les logs d'exécution du playbook
5. Les réponses aux questions de réflexion

### Questions de réflexion :

1. **Quel est l'avantage d'utiliser un playbook plutôt que des commandes manuelles ?**

2. **Expliquez l'intérêt des handlers dans Ansible.**

3. **Comment pourriez-vous adapter ce playbook pour déployer sur 10 serveurs ?**

4. **Quels sont les avantages de versionner le code d'infrastructure avec Git ?**

---

## 📞 Support

En cas de problème, vérifiez dans cet ordre :
1. Les machines Vagrant sont-elles démarrées ? (`vagrant status`)
2. La connexion SSH fonctionne-t-elle ? (`ssh vagrant@192.168.60.20`)
3. L'inventaire Ansible est-il correct ? (`cat inventory.ini`)
4. La syntaxe du playbook est-elle valide ? (`ansible-playbook --syntax-check`)

**Bon TP ! 🚀**
