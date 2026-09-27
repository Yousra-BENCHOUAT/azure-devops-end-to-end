# Azure DevOps End-to-End Platform

Projet DevOps Cloud permettant de provisionner une infrastructure Azure, configurer automatiquement une VM Linux, déployer une application Web conteneurisée via une pipeline CI/CD et superviser l'infrastructure avec Prometheus et Grafana.

## Architecture

```text
Developer
    |
    v
GitHub
    |
    v
GitHub Actions
    |
    v
Azure VM
    |
    +----> Docker ----> Nginx Web Application
    |
    +----> Node Exporter ----> Prometheus ----> Grafana
```

L'infrastructure Azure est provisionnée avec Terraform et la configuration de la VM est automatisée avec Ansible.

```text
Terraform
    |
    v
Microsoft Azure
    |
    +-- Resource Group
    +-- Virtual Network
    +-- Subnet
    +-- Network Security Group
    +-- Public IP
    +-- Network Interface
    +-- Ubuntu VM
             |
             v
          Ansible
             |
             +-- Docker
             +-- Node Exporter
```

## Technologies

- Microsoft Azure
- Terraform
- Ansible
- Docker
- Nginx
- Git
- GitHub
- GitHub Actions
- Linux
- SSH
- Prometheus
- Node Exporter
- Grafana
- PromQL

---

## 1. Infrastructure as Code - Terraform

L'infrastructure Azure est provisionnée automatiquement avec Terraform.

Les ressources suivantes sont créées :

- Resource Group
- Virtual Network (VNet)
- Subnet
- Network Security Group (NSG)
- Règle HTTP sur le port 80
- Règle SSH sur le port 22
- Public IP
- Network Interface
- Machine virtuelle Ubuntu 24.04

### Workflow Terraform

```bash
terraform init
terraform fmt
terraform validate
terraform plan
terraform apply
```

`terraform init` initialise le projet et télécharge les providers nécessaires.

`terraform fmt` formate les fichiers Terraform.

`terraform validate` vérifie la validité de la configuration.

`terraform plan` permet de visualiser les changements avant leur application.

`terraform apply` applique les changements sur Azure.

Terraform permet ainsi de gérer l'infrastructure sous forme de code et de rendre son déploiement reproductible.

---

## 2. Configuration Management - Ansible

Après le provisionnement de la VM avec Terraform, Ansible est utilisé pour automatiser sa configuration.

Ansible se connecte à la VM Azure via SSH avec une authentification par clé.

Le playbook permet notamment de :

- Mettre à jour le cache APT
- Installer les packages nécessaires
- Installer Docker
- Démarrer Docker
- Activer Docker au démarrage
- Ajouter l'utilisateur au groupe Docker
- Installer Node Exporter
- Démarrer Node Exporter
- Activer Node Exporter au démarrage

### Vérification de la connexion

```bash
ansible all -i inventory.ini -m ping
```

### Vérification du playbook

```bash
ansible-playbook -i inventory.ini playbook.yml --syntax-check
```

### Exécution

```bash
ansible-playbook -i inventory.ini playbook.yml
```

Ansible est utilisé comme outil de configuration tandis que Terraform est utilisé pour le provisionnement de l'infrastructure.

---

## 3. Application Web avec Docker et Nginx

L'application est une application Web légère servie par Nginx dans un conteneur Docker.

Le Dockerfile utilise l'image :

```dockerfile
FROM nginx:alpine
```

La page Web est copiée dans le répertoire utilisé par Nginx :

```dockerfile
COPY index.html /usr/share/nginx/html/index.html
```

Le conteneur utilise le port HTTP 80 :

```dockerfile
EXPOSE 80
```

### Construction de l'image

```bash
docker build -t devops-web:v1 .
```

### Lancement du conteneur

```bash
docker run -d \
  --name devops-web \
  -p 80:80 \
  devops-web:v1
```

### Vérification

```bash
docker ps
curl http://localhost
```

L'application est ensuite accessible via la Public IP de la VM Azure.

---

## 4. CI/CD avec GitHub Actions

Une pipeline CI/CD GitHub Actions automatise le build, le test et le déploiement de l'application.

La pipeline est déclenchée lors d'un push sur la branche :

```text
main
```

### Pipeline

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
GitHub Actions
    |
    +---- Build Docker Image
    |
    +---- Test Docker Image
    |
    +---- Deploy
              |
              v
          Azure VM
              |
              v
          Docker Container
```

### Continuous Integration

Le job de build :

1. Récupère le repository.
2. Construit l'image Docker.
3. Vérifie la configuration Nginx.

L'image est identifiée avec le SHA du commit afin d'améliorer la traçabilité.

Exemple :

```bash
docker build -t devops-web:${{ github.sha }} ./app
```

Le test utilise :

```bash
docker run --rm devops-web:${{ github.sha }} nginx -t
```

---

## 5. Continuous Deployment

Lorsque le build réussit, le job de déploiement est exécuté.

GitHub Actions se connecte à la VM Azure via SSH.

Les informations sensibles sont stockées dans GitHub Secrets :

- `VM_HOST`
- `VM_USER`
- `VM_SSH_KEY`

La clé privée SSH n'est jamais stockée directement dans le repository.

Le workflow :

1. Configure la connexion SSH.
2. Transfère l'application vers la VM.
3. Construit la nouvelle image Docker.
4. Supprime l'ancien conteneur.
5. Lance le nouveau conteneur.
6. Vérifie son état.

Le conteneur est lancé avec :

```bash
docker run -d \
  --name devops-web \
  --restart unless-stopped \
  -p 80:80 \
  devops-web:<commit-sha>
```

Le SHA Git permet d'identifier précisément la version du code correspondant à l'image déployée.

---

## 6. Monitoring avec Node Exporter

Node Exporter est installé sur la VM avec Ansible.

Il expose les métriques système Linux, notamment :

- CPU
- RAM
- Disque
- Réseau

Le service peut être vérifié avec :

```bash
systemctl is-active prometheus-node-exporter
```

Node Exporter expose ses métriques sur le port :

```text
9100
```

Test local :

```bash
curl http://localhost:9100/metrics
```

Le port 9100 n'est pas exposé publiquement sur Internet.

---

## 7. Prometheus

Prometheus collecte les métriques exposées par Node Exporter.

Configuration principale :

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "node-exporter"

    static_configs:
      - targets:
          - "host.docker.internal:9100"
```

Prometheus fonctionne dans un conteneur Docker alors que Node Exporter fonctionne directement sur la VM.

L'alias Docker vers l'hôte permet au conteneur Prometheus de communiquer avec Node Exporter.

Prometheus est lancé avec :

```bash
docker run -d \
  --name prometheus \
  --restart unless-stopped \
  --add-host=host.docker.internal:host-gateway \
  -p 9090:9090 \
  -v /home/azureuser/prometheus.yml:/etc/prometheus/prometheus.yml:ro \
  prom/prometheus
```

### Vérification de Prometheus

```bash
curl http://localhost:9090/-/healthy
```

La métrique :

```promql
up{job="node-exporter"}
```

permet de vérifier si Prometheus arrive à collecter les métriques de Node Exporter.

```text
up = 1 → target accessible
up = 0 → target inaccessible
```

---

## 8. Grafana

Grafana est utilisé pour visualiser les métriques collectées par Prometheus.

Grafana fonctionne dans un conteneur Docker avec un volume persistant.

```bash
docker run -d \
  --name grafana \
  --restart unless-stopped \
  --add-host=host.docker.internal:host-gateway \
  -p 3000:3000 \
  -v grafana-data:/var/lib/grafana \
  grafana/grafana
```

Prometheus est configuré comme Data Source Grafana avec :

```text
http://host.docker.internal:9090
```

L'accès à Grafana depuis la machine locale est réalisé avec un tunnel SSH :

```bash
ssh -i ~/.ssh/id_ed25519_azure \
  -L 3000:localhost:3000 \
  azureuser@<PUBLIC_IP>
```

Grafana est ensuite accessible localement sur :

```text
http://localhost:3000
```

Cela permet d'utiliser Grafana sans exposer directement le port 3000 sur Internet.

---

## 9. Dashboard Grafana

Le dashboard `Azure VM Monitoring` permet de superviser la VM Azure.

Il contient notamment :

- CPU Usage
- RAM Usage
- Disk Usage
- Server Status

### CPU Usage

```promql
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

Cette requête calcule le pourcentage d'utilisation du CPU.

### RAM Usage

```promql
100 * (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes))
```

Cette requête calcule le pourcentage de RAM utilisée.

### Disk Usage

```promql
100 * (1 - (
  node_filesystem_avail_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
  /
  node_filesystem_size_bytes{mountpoint="/",fstype!~"tmpfs|overlay"}
))
```

Cette requête calcule le pourcentage d'espace disque utilisé sur la partition racine.

### Server Status

```promql
up{job="node-exporter"}
```

Le panel est configuré en requête Instant :

```text
1 → UP
0 → DOWN
```

---

## 10. Test de panne Monitoring

Une panne contrôlée de Node Exporter a été simulée :

```bash
sudo systemctl stop prometheus-node-exporter
```

Vérification :

```bash
systemctl is-active prometheus-node-exporter
```

Résultat :

```text
inactive
```

Le port 9100 devient également inaccessible.

Prometheus détecte alors :

```text
up = 0
```

Grafana affiche :

```text
DOWN
```

Le service est ensuite restauré :

```bash
sudo systemctl start prometheus-node-exporter
```

Après le prochain cycle de collecte :

```text
Prometheus → up = 1
Grafana    → UP
```

Ce test permet de vérifier que la chaîne de monitoring détecte réellement l'indisponibilité de la target.

---

## 11. Troubleshooting rencontré

Plusieurs problèmes réels ont été rencontrés et diagnostiqués pendant le projet.

### 11.1 Azure SKU indisponible

Lors du provisionnement Terraform, Azure a retourné une erreur indiquant que la taille :

```text
Standard_B1s
```

n'était pas disponible dans la région utilisée.

La taille de VM a été remplacée par :

```text
Standard_B2ats_v2
```

Après modification :

```bash
terraform plan
terraform apply
```

Le déploiement a réussi sans avoir besoin de supprimer toute l'infrastructure existante.

### 11.2 Permissions de la clé SSH

La connexion SSH depuis WSL retournait :

```text
UNPROTECTED PRIVATE KEY FILE
```

La clé présente sur le système de fichiers Windows avait des permissions trop ouvertes pour SSH.

La clé a été copiée dans l'environnement Linux :

```bash
mkdir -p ~/.ssh
cp /mnt/c/Users/Yousra/.ssh/id_ed25519 ~/.ssh/id_ed25519_azure
chmod 600 ~/.ssh/id_ed25519_azure
```

La connexion SSH a ensuite fonctionné.

### 11.3 Pression mémoire sur la VM

Pendant le déploiement du monitoring, la VM disposait de moins de 1 Go de RAM et aucune Swap.

Grafana répondait avec des erreurs de connexion et la VM devenait très lente.

Diagnostic :

```bash
free -h
df -h
```

Après redémarrage de la VM, 1 Go de Swap a été ajouté :

```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

La Swap a été rendue persistante avec :

```text
/swapfile none swap sw 0 0
```

dans `/etc/fstab`.

Grafana a ensuite été vérifié avec :

```bash
curl http://localhost:3000/api/health
```

### 11.4 Grafana affichait UP alors que Prometheus indiquait DOWN

Lors du test de panne Node Exporter :

```text
Node Exporter → inactive
Prometheus    → up = 0
Grafana       → 1
```

La collecte Prometheus fonctionnait correctement.

Le panel Grafana utilisait une requête temporelle alors que le besoin était d'afficher l'état actuel.

La requête du panel Server Status a donc été configurée en mode :

```text
Instant
```

Grafana a ensuite correctement affiché :

```text
0 → DOWN
1 → UP
```

---

## 12. Méthode de Troubleshooting

La méthode utilisée pendant le projet consiste à vérifier les différentes couches progressivement.

Pour l'application :

```text
VM
 ↓
Docker
 ↓
Container
 ↓
Logs
 ↓
Port
 ↓
Application locale
 ↓
NSG Azure
 ↓
Public IP
```

Commandes utiles :

```bash
docker ps
docker logs devops-web
docker inspect devops-web
curl http://localhost
ss -ltnp
```

Pour le monitoring :

```text
Node Exporter
 ↓
Port 9100
 ↓
Prometheus
 ↓
Target
 ↓
PromQL
 ↓
Grafana
```

Commandes utiles :

```bash
systemctl status prometheus-node-exporter
curl http://localhost:9100/metrics
docker logs prometheus
curl http://localhost:9090/-/healthy
```

---

## 13. Sécurité

Plusieurs bonnes pratiques ont été appliquées :

- Authentification SSH par clé
- Pas de mot de passe SSH stocké dans le projet
- Secrets CI/CD stockés dans GitHub Secrets
- Clé SSH privée exclue de Git
- Terraform State exclu du repository
- Ports de monitoring non exposés publiquement
- Accès à Grafana via tunnel SSH
- NSG Azure pour contrôler les accès réseau

Le fichier `.gitignore` permet notamment d'exclure :

```gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
*.tfplan
*.pem
*.key
.env
.env.*
```

---

## 14. Structure du projet

```text
azure-devops-end-to-end/
│
├── terraform/
│   ├── providers.tf
│   └── main.tf
│
├── ansible/
│   ├── inventory.ini
│   └── playbook.yml
│
├── app/
│   ├── Dockerfile
│   └── index.html
│
├── monitoring/
│   └── prometheus.yml
│
├── .github/
│   └── workflows/
│       └── deploy.yml
│
├── .gitignore
└── README.md
```

---

## 15. Workflow End-to-End

Le fonctionnement global du projet est :

```text
1. Terraform
      ↓
Provisionnement Azure

2. Ansible
      ↓
Configuration de la VM

3. Docker
      ↓
Conteneurisation de l'application

4. GitHub
      ↓
Versioning du code

5. GitHub Actions
      ↓
Build + Test + Deployment

6. Azure VM
      ↓
Exécution de l'application

7. Node Exporter
      ↓
Exposition des métriques

8. Prometheus
      ↓
Collecte et stockage

9. Grafana
      ↓
Visualisation
```

---

## 16. Résultat

Le projet permet de mettre en pratique une chaîne DevOps complète :

- Infrastructure as Code
- Configuration Management
- Containerisation
- CI/CD
- Cloud Computing
- Monitoring
- Linux Administration
- Networking
- Troubleshooting

L'objectif principal est de disposer d'un environnement Cloud reproductible, automatisé, déployable et supervisé.