## 1. Installation et configuration des outils

### Objectifs

- Installer et configurer l’environnement complet avec VirtualBox, Vagrant, Git, WSL2 + Linux
- Créer une VM locale avec Vagrant pour la pratique du provisioning


### Exemple de commandes et scripts

- Installation manuelle (explications orales) de Git, VirtualBox, Vagrant, WSL2 (Ubuntu)
- Dans WSL2, installer Terraform et Ansible :

```bash
sudo apt update && sudo apt install -y terraform ansible
```

- Exemple de commande Vagrant pour init VM Ubuntu :

```bash
vagrant init ubuntu/bionic64
vagrant up
vagrant ssh
```

- Vérifier que la VM est accessible (IP, ping, SSH)


### Questionnements

- Pourquoi utiliser une VM locale pour déployer et tester son infrastructure ?
- Quels avantages apporte WSL2 pour un étudiant sous Windows ?

***

## 2. Provisionnement de l’infrastructure locale avec Terraform

### Objectifs

- Écrire un fichier Terraform simple pour provisionner une VM
- Réaliser l’opération de provisioning avec commandes Terraform


### Template fichier `main.tf` minimal (exemple local)

```hcl
terraform {
  required_providers {
    local = {
      source = "hashicorp/local"
      version = "~> 2.1"
    }
  }
}

resource "local_file" "example" {
  content  = "Provisionnement réussi via Terraform"
  filename = "result_provision.txt"
}
```

(Les providers VirtualBox ou autres peuvent être plus complexes à mettre en place, préférez un exemple simple ici)

### Commandes Terraform

```bash
terraform init
terraform plan
terraform apply
cat result_provision.txt
terraform destroy
```


### Questionnements

- Que fait exactement Terraform lors d’un `terraform apply` ?
- Pourquoi est-il important de versionner l’état (`state`) Terraform ?

***

## 3. Automatisation avec Ansible

### Objectifs

- Créer un inventaire Ansible pointant vers la VM
- Écrire un playbook pour installer Nginx et déployer une page HTML


### Template inventaire `hosts.ini`

```ini
[webserver]
192.168.56.101 ansible_user=vagrant ansible_ssh_private_key_file=.vagrant/machines/default/virtualbox/private_key
```


### Template playbook `nginx.yml`

```yaml
---
- name: Installer et configurer Nginx
  hosts: webserver
  become: yes
  tasks:
    - name: Installer Nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Démarrer le service Nginx
      service:
        name: nginx
        state: started
        enabled: yes

    - name: Déployer une page index HTML simple
      copy:
        dest: /var/www/html/index.html
        content: "<h1>Déploiement automatisé avec Ansible</h1>"
        mode: '0644'
```


### Commande de lancement

```bash
ansible-playbook -i hosts.ini nginx.yml
```


### Questionnements

- Quel est l’intérêt d’un playbook Ansible pour la gestion d’un parc serveur ?
- Quelle différence entre une configuration manuelle et une configuration Ansible ?

***

## 4. Gestion des versions avec Git

### Objectifs

- Initialiser un dépôt git pour versionner les fichiers Terraform et Ansible
- S’exercer aux commandes basiques de suivi de version


### Commandes et étapes

```bash
git init
git add main.tf nginx.yml hosts.ini
git commit -m "Ajout fichiers Terraform et Ansible initial"
```

Faire modification dans `nginx.yml` (ajout d’une tâche par exemple), puis :

```bash
git add nginx.yml
git commit -m "Ajout page HTML personnalisée"
git log --oneline
git status
```


### Questionnements

- Pourquoi versionner le code d’infrastructure et de configuration ?
- Comment Git facilite la collaboration et la traçabilité dans les projets DevOps ?

***

## 5. (Option) Conteneurisation avec Docker et Docker Compose

### Objectifs

- Construire un Dockerfile simple pour application web Nginx
- Déployer avec Docker Compose une application composée


### Template Dockerfile

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```


### Template `index.html`

```html
<h1>Application Docker déployée</h1>
```


### Template `docker-compose.yml`

```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "8080:80"
```


### Commandes Docker

```bash
docker build -t webapp .
docker-compose up -d
```


### Questionnements

- Quels bénéfices apporte la conteneurisation dans le déploiement d’infrastructures ?
- Pourquoi utiliser Docker Compose pour orchestrer plusieurs conteneurs ?

***


