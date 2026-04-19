# Module 4 -- Accéder aux instances et travailler dedans

## Introduction

Créer une machine virtuelle, c'est bien. Pouvoir y travailler, c'est
mieux. Imaginez que vous ayez construit une maison flambant neuve
mais que vous n'ayez pas encore les clés pour y entrer. C'est un peu
la situation dans laquelle nous sommes après le module précédent :
nos VM existent, mais nous n'avons pas encore vu comment interagir
avec elles au quotidien.

Multipass offre deux modes d'interaction principaux : l'ouverture
d'un shell interactif (comme si vous étiez assis devant la machine)
et l'exécution de commandes à distance (comme si vous envoyiez des
instructions par courrier). Chacun a ses avantages selon la
situation, et un bon développeur sait quand utiliser l'un ou l'autre.

## Objectifs du module

Au terme de ce module vous serez capable de :

- Ouvrir un shell interactif dans une instance Multipass
- Exécuter des commandes à distance sans ouvrir de session
- Comprendre le fonctionnement de l'utilisateur par défaut et la
  gestion des droits

## Ouvrir un shell interactif

### La commande `multipass shell`

Le moyen le plus direct d'interagir avec une instance est d'ouvrir
un shell interactif. C'est l'équivalent de se connecter en SSH à un
serveur, sauf que Multipass gère la connexion pour vous sans avoir à
configurer de clés SSH ni à retenir de mot de passe.

```bash
# Ouvrir un shell dans l'instance "dev-server"
multipass shell dev-server
```

Une fois cette commande exécutée, votre invite de commande change
pour indiquer que vous êtes désormais dans la VM :

```
ubuntu@dev-server:~$
```

Vous êtes maintenant dans un environnement Ubuntu complet. Vous
pouvez y exécuter toutes les commandes Linux que vous connaissez :

```bash
# Vérifier la version d'Ubuntu
lsb_release -a

# Voir l'espace disque
df -h

# Voir la mémoire disponible
free -h

# Mettre à jour les paquets
sudo apt update && sudo apt upgrade -y
```

Pour quitter le shell et revenir à votre machine hôte, tapez
simplement `exit` ou utilisez le raccourci `Ctrl+D`.

```bash
# Quitter le shell de la VM
exit
```

#### Exemple pratique {id="exemple-shell-interactif"}

Voici un scénario typique d'utilisation du shell interactif pour
préparer un environnement de développement Python :

```bash
# Depuis la machine hôte, ouvrir un shell
multipass shell dev-server

# Dans la VM : installer Python et pip
sudo apt update
sudo apt install -y python3 python3-pip python3-venv

# Créer un environnement virtuel
python3 -m venv ~/mon-projet/venv
source ~/mon-projet/venv/bin/activate

# Installer des dépendances
pip install flask requests

# Vérifier l'installation
python3 -c "import flask; print(flask.__version__)"

# Quitter la VM
exit
```

Le shell interactif est idéal pour les tâches exploratoires : quand
vous avez besoin de naviguer dans le système de fichiers, de déboguer
un problème, ou de tester des commandes de manière itérative.

## Exécuter des commandes à distance

### La commande `multipass exec`

Parfois, vous n'avez pas besoin d'ouvrir une session interactive
complète. Vous voulez simplement exécuter une commande et récupérer
le résultat. C'est comme envoyer un SMS au lieu de passer un appel
téléphonique : c'est plus rapide et plus direct quand le message est
simple.

La commande `multipass exec` permet d'exécuter une commande dans une
instance sans ouvrir de session :

```bash
# Syntaxe générale
multipass exec <nom-instance> -- <commande>
```

Le double tiret `--` sépare les arguments de `multipass exec` de la
commande à exécuter dans la VM. C'est une convention courante dans
les outils en ligne de commande.

```bash
# Vérifier l'espace disque dans la VM
multipass exec dev-server -- df -h

# Lister le contenu du répertoire home
multipass exec dev-server -- ls -la /home/ubuntu

# Voir les processus en cours
multipass exec dev-server -- ps aux
```

#### Exemple pratique {id="exemple-exec-simple"}

`multipass exec` est particulièrement utile pour les vérifications
rapides et l'automatisation :

```bash
# Vérifier si un service est installé
multipass exec dev-server -- which nginx

# Installer un paquet sans ouvrir de session
multipass exec dev-server -- \
  sudo apt install -y nginx

# Vérifier que le service tourne
multipass exec dev-server -- \
  systemctl status nginx

# Récupérer l'adresse IP depuis la VM
multipass exec dev-server -- \
  hostname -I
```

### Exécuter des commandes complexes

Pour les commandes plus longues ou celles qui contiennent des
caractères spéciaux (pipes, redirections), il est préférable
d'encapsuler la commande dans `bash -c` :

```bash
# Commande avec pipe
multipass exec dev-server -- \
  bash -c "ps aux | grep nginx"

# Commande avec redirection
multipass exec dev-server -- \
  bash -c "echo 'Hello VM' > /tmp/test.txt"

# Commande multiligne
multipass exec dev-server -- bash -c "
  cd /home/ubuntu
  mkdir -p mon-projet
  echo 'Projet initialisé' > mon-projet/README.md
  cat mon-projet/README.md
"
```

<tip>

Utilisez `multipass exec` pour les opérations ponctuelles et les
scripts automatisés. Réservez `multipass shell` pour les sessions
de travail interactives ou le débogage.
</tip>

#### Exemple pratique {id="exemple-exec-avance"}

Voici comment utiliser `multipass exec` dans un petit script
d'automatisation sur votre machine hôte :

```bash
#!/bin/bash
# Script : setup-dev-env.sh
# Configure une instance pour le développement

VM_NAME="dev-server"

echo "Mise à jour du système..."
multipass exec $VM_NAME -- \
  sudo apt update -qq

echo "Installation de Git..."
multipass exec $VM_NAME -- \
  sudo apt install -y -qq git

echo "Installation de Node.js..."
multipass exec $VM_NAME -- \
  sudo snap install node --classic

echo "Vérification des installations..."
multipass exec $VM_NAME -- git --version
multipass exec $VM_NAME -- node --version
multipass exec $VM_NAME -- npm --version

echo "Environnement prêt."
```

Ce type de script peut être lancé depuis votre machine hôte pour
configurer automatiquement une instance fraîchement créée.

## Utilisateur par défaut et gestion des droits

### L'utilisateur `ubuntu`

Lorsque vous vous connectez à une instance Multipass, vous êtes
automatiquement connecté en tant qu'utilisateur `ubuntu`. Cet
utilisateur est créé par défaut lors de la création de l'instance
et dispose de droits `sudo` sans mot de passe.

```bash
# Vérifier l'utilisateur courant
multipass exec dev-server -- whoami
# Résultat : ubuntu

# Vérifier les groupes de l'utilisateur
multipass exec dev-server -- groups
# Résultat : ubuntu adm cdrom sudo dip plugdev
```

L'appartenance au groupe `sudo` signifie que l'utilisateur `ubuntu`
peut exécuter n'importe quelle commande en tant que root en
préfixant avec `sudo`, sans saisir de mot de passe.

### Utiliser sudo

Étant donné que l'utilisateur `ubuntu` a un accès `sudo` sans mot
de passe, l'élévation de privilèges est transparente :

```bash
# Installer un paquet (nécessite les droits root)
multipass exec dev-server -- \
  sudo apt install -y curl wget

# Modifier un fichier système
multipass exec dev-server -- \
  sudo bash -c "echo '127.0.0.1 monapp.local' \
    >> /etc/hosts"

# Redémarrer un service
multipass exec dev-server -- \
  sudo systemctl restart nginx
```

#### Exemple pratique {id="exemple-droits"}

Voici comment configurer un utilisateur supplémentaire dans une
instance, ce qui peut être utile pour simuler un environnement
multi-utilisateurs :

```bash
# Créer un nouvel utilisateur
multipass exec dev-server -- \
  sudo adduser --disabled-password \
  --gecos "" developpeur

# Lui donner les droits sudo
multipass exec dev-server -- \
  sudo usermod -aG sudo developpeur

# Exécuter une commande en tant que cet utilisateur
multipass exec dev-server -- \
  sudo -u developpeur whoami
```

<note>

Dans la plupart des cas d'utilisation de Multipass, l'utilisateur
`ubuntu` par défaut suffit amplement. La création d'utilisateurs
supplémentaires n'est nécessaire que dans des scénarios spécifiques
comme la simulation d'environnements multi-utilisateurs ou le test
de permissions.
</note>

## Conclusion

Ce module vous a présenté les deux modes d'interaction principaux
avec les instances Multipass : le shell interactif avec
`multipass shell` et l'exécution de commandes à distance avec
`multipass exec`. Vous avez également découvert le fonctionnement
de l'utilisateur par défaut `ubuntu` et sa capacité à utiliser
`sudo` sans mot de passe.

La règle d'or est simple : utilisez `multipass shell` quand vous
avez besoin d'explorer, de déboguer ou de travailler de manière
interactive, et `multipass exec` quand vous devez exécuter des
commandes précises ou automatiser des tâches.

Le prochain module abordera un sujet passionnant : le provisioning
automatique avec cloud-init, qui vous permettra de configurer vos
instances automatiquement dès leur création.
