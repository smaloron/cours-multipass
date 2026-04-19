# Module 8 -- Installation et utilisation de Docker dans une VM

## Introduction

Vous maîtrisez maintenant la création et la gestion de machines
virtuelles avec Multipass. Il est temps d'aller plus loin en
combinant deux technologies complémentaires : la virtualisation
(Multipass) et la conteneurisation (Docker). Pourquoi utiliser
Docker dans une VM alors que Docker peut tourner directement sur
votre machine ? La réponse tient en un mot : isolation.

Pensez à un laboratoire de chimie. Vous pourriez faire vos
expériences directement dans la cuisine (Docker sur votre machine),
mais un laboratoire dédié (Docker dans une VM) vous offre un espace
contrôlé, reproductible et sans risque pour le reste de la maison.
De plus, sous Windows, utiliser Docker dans une VM Multipass est une
alternative gratuite et légère à Docker Desktop, qui nécessite une
licence commerciale pour les entreprises de plus de 250 employés.

## Objectifs du module

Au terme de ce module vous serez capable de :

- Expliquer l'intérêt d'utiliser Docker dans une VM Multipass
- Installer Docker manuellement dans une instance
- Automatiser l'installation de Docker via cloud-init
- Utiliser l'image Docker optimisée de Multipass
- Gérer les permissions Docker dans l'instance

## Pourquoi Docker dans Multipass ?

### Les avantages de cette combinaison

L'utilisation de Docker dans une VM Multipass présente plusieurs
avantages concrets :

**Isolation complète** : Docker partage le noyau de l'hôte. Si un
conteneur exploite une faille du noyau, votre système hôte est
potentiellement compromis. Dans une VM, le conteneur est doublement
isolé : par le moteur Docker et par l'hyperviseur.

**Reproductibilité** : avec cloud-init, vous pouvez créer en une
commande une VM avec Docker préconfiguré, prête à l'emploi.
Partagez le fichier YAML avec votre équipe et tout le monde dispose
du même environnement.

**Alternative à Docker Desktop** : sous Windows, Docker Desktop
nécessite une licence payante pour un usage commercial dans les
grandes entreprises. Docker dans une VM Multipass est entièrement
gratuit.

**Environnement jetable** : si vos expériences Docker tournent mal
(images corrompues, volumes encombrants, configuration cassée), vous
détruisez la VM et en recréez une. Votre machine hôte reste intacte.

## Installation manuelle de Docker

### Procédure pas à pas

Commencez par créer une instance avec suffisamment de ressources
pour Docker :

```bash
# Créer une instance adaptée à Docker
multipass launch --name docker-vm \
  --cpus 2 --memory 4G --disk 20G
```

Puis connectez-vous et installez Docker :

```bash
# Se connecter à l'instance
multipass shell docker-vm
```

Dans la VM, exécutez les commandes suivantes :

```bash
# Mettre à jour les paquets
sudo apt update

# Installer les prérequis
sudo apt install -y \
  ca-certificates \
  curl \
  gnupg \
  lsb-release

# Ajouter la clé GPG officielle de Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  | sudo gpg --dearmor \
  -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Ajouter le dépôt Docker
echo \
  "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") \
  stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list \
  > /dev/null

# Installer Docker
sudo apt update
sudo apt install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-compose-plugin
```

### Vérification de l'installation

Toujours dans la VM, vérifiez que Docker fonctionne :

```bash
# Vérifier la version de Docker
sudo docker version

# Lancer le conteneur de test
sudo docker run hello-world
```

Si vous voyez le message "Hello from Docker!", l'installation est
réussie.

#### Exemple pratique {id="exemple-verif-docker"}

Voici une série de vérifications pour confirmer que Docker est
pleinement opérationnel :

```bash
# Vérifier que le service Docker tourne
sudo systemctl status docker --no-pager

# Vérifier la version complète
sudo docker info | head -20

# Lancer un conteneur interactif Ubuntu
sudo docker run -it --rm ubuntu:24.04 bash -c \
  "echo 'Docker fonctionne dans la VM !'"

# Lancer un serveur web temporaire
sudo docker run -d -p 8080:80 --name test-nginx nginx
curl http://localhost:8080
sudo docker rm -f test-nginx
```

## Installation automatisée via cloud-init

### Un fichier cloud-init pour Docker

L'installation manuelle est instructive, mais en pratique, vous
voudrez automatiser ce processus. Voici un fichier cloud-init qui
installe Docker automatiquement :

```yaml
#cloud-config

package_update: true
package_upgrade: true

packages:
  - ca-certificates
  - curl
  - gnupg
  - lsb-release

runcmd:
  # Ajouter la clé GPG Docker
  - install -m 0755 -d /etc/apt/keyrings
  - |
    curl -fsSL \
      https://download.docker.com/linux/ubuntu/gpg \
      | gpg --dearmor \
      -o /etc/apt/keyrings/docker.gpg
  - chmod a+r /etc/apt/keyrings/docker.gpg

  # Ajouter le dépôt Docker
  - |
    echo "deb [arch=$(dpkg --print-architecture) \
      signed-by=/etc/apt/keyrings/docker.gpg] \
      https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && \
      echo $VERSION_CODENAME) stable" \
      | tee /etc/apt/sources.list.d/docker.list \
      > /dev/null

  # Installer Docker
  - apt-get update
  - |
    apt-get install -y \
      docker-ce docker-ce-cli \
      containerd.io docker-compose-plugin

  # Ajouter ubuntu au groupe docker
  - usermod -aG docker ubuntu

  # Démarrer Docker
  - systemctl enable docker
  - systemctl start docker
```

Sauvegardez ce fichier sous le nom `docker-cloud-init.yaml`, puis
lancez une instance :

```bash
multipass launch --name docker-auto \
  --cpus 2 --memory 4G --disk 20G \
  --cloud-init docker-cloud-init.yaml
```

Après quelques minutes, Docker est installé et opérationnel sans
aucune intervention manuelle.

#### Exemple pratique {id="exemple-cloudinit-docker"}

Vérifiez que l'installation automatisée a fonctionné :

```bash
# Attendre que cloud-init ait terminé
multipass exec docker-auto -- \
  cloud-init status --wait

# Vérifier Docker
multipass exec docker-auto -- docker version
multipass exec docker-auto -- docker run hello-world
```

## Image Docker optimisée de Multipass

### La solution la plus rapide

Multipass propose une image préconstruite qui inclut Docker et Docker
Compose. C'est la méthode la plus rapide pour obtenir un
environnement Docker fonctionnel :

```bash
# Lancer l'image Docker optimisée
multipass launch docker --name docker-express
```

Cette image contient Docker et Portainer (une interface web de
gestion Docker que nous verrons en détail dans le module suivant)
préinstallés et préconfigurés.

#### Exemple pratique {id="exemple-image-docker"}

Comparons les trois méthodes d'installation en termes de temps et
de simplicité :

```bash
# Méthode 1 : Image optimisée (la plus rapide)
time multipass launch docker --name m1-express

# Méthode 2 : Cloud-init (automatisé mais plus long)
time multipass launch --name m2-cloudinit \
  --cloud-init docker-cloud-init.yaml

# Méthode 3 : Manuelle (la plus longue)
time multipass launch --name m3-manuelle
# + installation manuelle ensuite
```

L'image optimisée est généralement la plus rapide, car Docker est
déjà inclus dans l'image. Cloud-init est plus long, car il télécharge
et installe Docker au premier démarrage. L'installation manuelle
est la plus longue, mais offre le plus de contrôle.

| Méthode | Temps | Personnalisation | Simplicité |
|---|---|---|---|
| Image Docker | ~1 min | Faible | Maximale |
| Cloud-init | ~5 min | Élevée | Bonne |
| Manuelle | ~10 min | Totale | Moyenne |

## Gestion des permissions Docker

### Le groupe `docker`

Par défaut, seul l'utilisateur root peut exécuter des commandes
Docker. Pour éviter de préfixer chaque commande par `sudo`, il faut
ajouter l'utilisateur `ubuntu` au groupe `docker` :

```bash
# Ajouter ubuntu au groupe docker
sudo usermod -aG docker ubuntu

# Appliquer le changement (nécessaire pour la session actuelle)
newgrp docker

# Vérifier que ça fonctionne sans sudo
docker run hello-world
```

<note>

Si vous avez utilisé le fichier cloud-init proposé plus haut, cette
configuration est déjà appliquée. L'image Docker optimisée de
Multipass inclut également cette configuration.
</note>

#### Exemple pratique {id="exemple-permissions-docker"}

Voici comment vérifier et corriger les permissions Docker :

```bash
# Vérifier les groupes de l'utilisateur
multipass exec docker-auto -- groups ubuntu

# Le résultat doit contenir "docker"
# Exemple : ubuntu adm sudo docker

# Si "docker" n'apparaît pas, l'ajouter :
multipass exec docker-auto -- \
  sudo usermod -aG docker ubuntu

# Tester sans sudo
multipass exec docker-auto -- docker ps
```

### Utiliser Docker au quotidien dans la VM

Une fois les permissions configurées, voici quelques commandes
Docker courantes que vous utiliserez régulièrement dans votre VM :

```bash
# Lancer un conteneur en arrière-plan
multipass exec docker-auto -- \
  docker run -d --name mon-nginx \
  -p 8080:80 nginx

# Vérifier les conteneurs en cours
multipass exec docker-auto -- docker ps

# Consulter les logs
multipass exec docker-auto -- \
  docker logs mon-nginx

# Arrêter et supprimer
multipass exec docker-auto -- \
  docker stop mon-nginx
multipass exec docker-auto -- \
  docker rm mon-nginx

# Nettoyer les images non utilisées
multipass exec docker-auto -- docker system prune -f
```

Pour accéder au serveur Nginx depuis votre machine hôte, utilisez
l'adresse IP de la VM :

```bash
# Récupérer l'IP de la VM
multipass info docker-auto | grep IPv4

# Accéder au serveur web
# (Remplacez IP_VM par l'adresse réelle)
curl http://IP_VM:8080
```

## Conclusion

Ce module vous a montré trois façons d'installer Docker dans une VM
Multipass : manuellement, via cloud-init, et en utilisant l'image
Docker optimisée. Chaque méthode a ses avantages selon le contexte.
L'image optimisée est idéale pour démarrer rapidement, cloud-init
pour des configurations reproductibles et personnalisées, et
l'installation manuelle pour comprendre les rouages en profondeur.

Vous avez également appris à gérer les permissions Docker pour
travailler confortablement sans `sudo`. Ces compétences vous
préparent au module suivant, où nous ajouterons Portainer pour
gérer Docker via une interface web intuitive.
