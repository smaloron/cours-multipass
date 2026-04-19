# Module 2 -- Installation et configuration sous Windows

## Introduction

Vous avez maintenant une bonne vision de ce que Multipass peut faire
pour vous. Mais un outil, aussi puissant soit-il, ne sert à rien
s'il n'est pas correctement installé et configuré. C'est un peu
comme acheter un robot de cuisine dernier cri : si vous ne le
branchez pas sur la bonne prise ou si vous ne retirez pas les
sécurités de transport, il ne fonctionnera jamais comme prévu.

L'installation de Multipass sous Windows comporte quelques subtilités
liées à l'hyperviseur sous-jacent. Ce module vous guidera pas à pas
pour que votre environnement soit opérationnel sans accroc. Nous
aborderons également les problèmes les plus fréquents et leurs
solutions, pour que vous ne restiez jamais bloqué.

## Objectifs du module

Au terme de ce module vous serez capable de :

- Vérifier que votre système répond aux prérequis pour Multipass
- Installer Multipass sous Windows pas à pas
- Configurer le backend hyperviseur adapté à votre machine
- Diagnostiquer et résoudre les problèmes d'installation courants

## Prérequis système

### L'hyperviseur, la fondation indispensable

Avant d'installer Multipass, il faut s'assurer que votre machine
peut faire tourner un hyperviseur. Rappelez-vous du module
précédent : Multipass ne virtualise pas directement, il s'appuie
sur un hyperviseur existant. Sous Windows, deux options s'offrent
à vous : Hyper-V et VirtualBox.

**Hyper-V** est l'hyperviseur intégré de Microsoft. Il est disponible
sur les éditions Pro, Enterprise et Education de Windows 10 et 11.
C'est l'option recommandée car elle offre les meilleures performances
et une intégration native avec le système.

**VirtualBox** est une alternative gratuite et open source qui
fonctionne sur toutes les éditions de Windows, y compris l'édition
Home. C'est la solution de repli si Hyper-V n'est pas disponible.

| Critère | Hyper-V | VirtualBox |
|---|---|---|
| Éditions Windows | Pro, Enterprise, Education | Toutes |
| Performance | Excellente | Bonne |
| Intégration système | Native | Tierce |
| Conflit avec d'autres hyperviseurs | Oui | Possible |

### Vérifier et activer Hyper-V

Pour vérifier si Hyper-V est disponible et l'activer, ouvrez
PowerShell en tant qu'administrateur :

```powershell
# Vérifier si Hyper-V est disponible
Get-WindowsOptionalFeature -Online `
  -FeatureName Microsoft-Hyper-V

# Activer Hyper-V si disponible
Enable-WindowsOptionalFeature -Online `
  -FeatureName Microsoft-Hyper-V -All
```

Après activation, un redémarrage de la machine est nécessaire.

Vous pouvez également activer Hyper-V via l'interface graphique :

<procedure>
<step>Ouvrir le Panneau de configuration</step>
<step>Cliquer sur "Programmes et fonctionnalités"</step>
<step>Cliquer sur "Activer ou désactiver des fonctionnalités Windows"</step>
<step>Cocher "Hyper-V" et valider</step>
<step>Redémarrer la machine</step>
</procedure>

#### Exemple pratique {id="exemple-verif-hyperv"}

Voici comment vérifier rapidement si la virtualisation est activée
dans le BIOS de votre machine :

```powershell
# Vérifier le support de la virtualisation
systeminfo | findstr /i "hyper-v"
```

Si vous voyez "Un hyperviseur a été détecté", c'est que la
virtualisation matérielle est bien activée dans votre BIOS/UEFI.
Dans le cas contraire, vous devrez redémarrer votre machine, entrer
dans le BIOS et activer la fonctionnalité de virtualisation (souvent
nommée "Intel VT-x" ou "AMD-V" selon votre processeur).

## Téléchargement et installation

### Installer Multipass pas à pas

L'installation de Multipass sous Windows est directe. Le programme
d'installation gère toutes les dépendances et la configuration
initiale.

<procedure>
<step>
Rendez-vous sur le site officiel :
https://multipass.run/install
</step>
<step>
Téléchargez l'installateur Windows (.exe)
</step>
<step>
Lancez l'installateur et suivez les instructions.
Lors de l'installation, choisissez le backend hyperviseur :
Hyper-V (recommandé) ou VirtualBox.
</step>
<step>
Une fois l'installation terminée, ouvrez un terminal
(PowerShell ou Invite de commandes) pour vérifier.
</step>
</procedure>

### Vérification de l'installation

Une fois Multipass installé, il est essentiel de vérifier que tout
fonctionne correctement avant d'aller plus loin. Deux commandes
suffisent pour s'en assurer :

```bash
# Afficher la version installée
multipass version
```

Le résultat devrait ressembler à ceci :

```
multipass   1.14.1+win
multipassd  1.14.1+win
```

Vous voyez deux lignes : `multipass` est le client en ligne de
commande, et `multipassd` est le daemon (le service en arrière-plan).
Les deux doivent afficher une version.

```bash
# Lister les images Ubuntu disponibles
multipass find
```

Cette commande affiche la liste des images Ubuntu que vous pouvez
utiliser pour créer des VM :

```
Image           Aliases          Version
20.04           focal            20240822
22.04           jammy            20240829
24.04           noble            20240830
daily:25.04     plucky           20240901
```

#### Exemple pratique {id="exemple-premier-lancement"}

Pour confirmer que tout est opérationnel, lancez votre toute
première VM :

```bash
# Lancer une VM de test
multipass launch --name test-install

# Vérifier qu'elle tourne
multipass list

# Se connecter rapidement
multipass shell test-install

# Une fois dans la VM, vérifier le système
lsb_release -a

# Quitter la VM
exit

# Nettoyer
multipass delete test-install
multipass purge
```

Si toutes ces commandes s'exécutent sans erreur, votre installation
est fonctionnelle.

## Configuration du backend hyperviseur

### Changer de backend

Multipass permet de changer de backend hyperviseur après
l'installation. C'est utile si vous avez installé avec VirtualBox
et que vous souhaitez passer à Hyper-V, ou inversement.

```bash
# Voir le backend actuel
multipass get local.driver

# Passer à Hyper-V
multipass set local.driver=hyperv

# Ou passer à VirtualBox
multipass set local.driver=virtualbox
```

<warning>

Changer de backend nécessite de supprimer toutes les instances
existantes au préalable. Sauvegardez votre travail avant de
procéder.
</warning>

### Configurer les ressources par défaut

Vous pouvez également ajuster les ressources par défaut attribuées
à chaque nouvelle instance :

```bash
# Voir tous les paramètres configurables
multipass get --keys

# Exemples de paramètres utiles
multipass get local.driver
multipass get client.gui.autostart
```

## Résolution des problèmes courants

### Les difficultés que vous risquez de rencontrer

Même avec la meilleure volonté du monde, l'installation peut parfois
poser problème. Voici les situations les plus fréquentes et leurs
solutions.

### Hyper-V non activé ou indisponible

**Symptôme** : Multipass affiche une erreur indiquant que
l'hyperviseur n'est pas disponible.

**Diagnostic** :

```powershell
# Vérifier l'état de Hyper-V
Get-WindowsOptionalFeature -Online `
  -FeatureName Microsoft-Hyper-V | `
  Select-Object State
```

**Solutions** :

Si l'état est "Disabled", activez Hyper-V (voir section prérequis).
Si vous êtes sur Windows Home, Hyper-V n'est pas disponible :
basculez vers VirtualBox comme backend.

```bash
multipass set local.driver=virtualbox
```

### Conflit entre Hyper-V et VirtualBox

**Symptôme** : VirtualBox ne démarre plus après l'activation de
Hyper-V, ou inversement.

**Explication** : Hyper-V et VirtualBox (versions anciennes) ne
peuvent pas coexister, car ils tentent tous deux de contrôler les
extensions de virtualisation du processeur. Depuis VirtualBox 6.0,
la coexistence est possible, mais avec des performances réduites
pour VirtualBox.

**Solution** : Choisissez un seul backend et tenez-vous-y.
Si vous avez besoin de VirtualBox pour d'autres projets, utilisez
VirtualBox comme backend Multipass également.

### Le daemon Multipass ne démarre pas

**Symptôme** : Les commandes `multipass` échouent avec un message
de connexion au daemon impossible.

**Solution** :

```powershell
# Vérifier l'état du service
Get-Service -Name "Multipass"

# Redémarrer le service si nécessaire
Restart-Service -Name "Multipass"
```

#### Exemple pratique {id="exemple-diagnostic"}

Voici une séquence de diagnostic complète à exécuter si Multipass
ne fonctionne pas après l'installation :

```powershell
# Étape 1 : Vérifier la version
multipass version

# Étape 2 : Vérifier le service
Get-Service -Name "Multipass"

# Étape 3 : Vérifier le backend
multipass get local.driver

# Étape 4 : Vérifier la virtualisation
systeminfo | findstr /i "hyper-v"

# Étape 5 : Consulter les logs
# Les logs se trouvent dans :
# C:\ProgramData\Multipass\data\logs\
Get-Content `
  "C:\ProgramData\Multipass\data\logs\multipassd.log" `
  -Tail 50
```

## Conclusion

Vous disposez maintenant d'un environnement Multipass fonctionnel
sous Windows. Nous avons couvert l'ensemble du processus : la
vérification des prérequis (Hyper-V ou VirtualBox), le
téléchargement et l'installation, la vérification avec
`multipass version` et `multipass find`, la configuration du
backend hyperviseur, et la résolution des problèmes les plus
fréquents.

L'essentiel à retenir est que Multipass s'appuie sur un hyperviseur
existant, et que le choix entre Hyper-V et VirtualBox dépend de
votre édition de Windows. Une fois ce choix fait et l'installation
validée, vous êtes prêt à créer et gérer vos premières machines
virtuelles, ce que nous aborderons dans le module suivant.
