# Rôle Ansible `server_bootstrap`

`server_bootstrap` prépare un serveur Linux pour son administration avec
Ansible. Il crée un compte technique, installe ses clés publiques, configure
sudo et OpenSSH, puis applique les réglages système nécessaires.

Le rôle prend volontairement en charge uniquement :

- Debian 13 (`trixie`) ;
- Ubuntu 26.04 (`resolute`).

Toute autre distribution ou version est refusée avant toute installation ou
configuration du serveur. Ce contrôle fonctionne sans Python et s'exécute
également en mode simulation (`--check`).

## Ce que le rôle configure

Lors d'une exécution complète, le rôle :

1. installe Python 3 et `python3-apt` s'ils sont absents ;
2. installe les paquets système déclarés dans `sys_packages` ;
3. crée le compte technique défini par `create_user` ;
4. installe exclusivement les clés de `user_public_keys` pour ce compte ;
5. configure les règles sudo du compte technique et du groupe `admin` ;
6. sauvegarde puis modifie les directives globales de `sshd_config`, et vérifie
   leur syntaxe et leurs valeurs effectives avant rechargement ;
7. configure, si demandé, le nom d'hôte et `/etc/hosts` ;
8. installe une configuration Neovim pour le compte technique et, en option,
   pour root.

## Avant de commencer

Le contrôleur doit disposer d'Ansible Core 2.20 ou d'une version ultérieure
compatible. Les collections nécessaires sont installées avec :

```bash
ansible-galaxy collection install -r requirements.yml
```

`ansible.posix` est utilisée par le rôle pour gérer `authorized_keys`.
`community.docker` sert uniquement au scénario Molecule.

La cible doit disposer :

- de Debian 13 ou Ubuntu 26.04 avec systemd ;
- d'OpenSSH et d'une connexion initiale fonctionnelle ;
- d'`apt-get` et d'un accès aux dépôts APT configurés ;
- d'un accès root ou de privilèges équivalents ;
- de l'inclusion de `/etc/sudoers.d` dans `/etc/sudoers`.

Python peut être absent : le rôle l'installe avec une commande `raw` avant
d'utiliser les modules Ansible habituels.

## Premier déploiement

Le déploiement se déroule en quatre étapes : tester l'accès initial, préparer
l'inventaire, appliquer le rôle, puis vérifier le nouveau compte Ansible.

### 1. Tester l'accès initial

La commande suivante vérifie la connexion SSH sans nécessiter Python sur la
cible :

```bash
ansible all -i inventory.yml -m ansible.builtin.raw -a "id -u"
```

Une sortie `0` indique que la connexion utilise root. Une autre valeur est
l'identifiant numérique de l'utilisateur distant.

### 2. Préparer l'inventaire

Cet exemple utilise une adresse réservée à la documentation :

```yaml
---
all:
  hosts:
    server-01:
      ansible_host: 192.0.2.10
      ansible_user: root
      ansible_ssh_private_key_file: /chemin/securise/cle-initiale
```

### 3. Appliquer le rôle

Le playbook minimal doit fournir la ou les clés publiques du futur compte
technique ainsi qu'un nom d'hôte, puisque sa gestion est activée par défaut :

```yaml
---
- name: Préparer les serveurs pour Ansible
  hosts: all
  gather_facts: false
  become: false

  vars:
    create_user: ansible
    user_public_keys:
      - "{{ lookup('ansible.builtin.file', '/chemin/cles/ansible-ed25519.pub') }}"

    manage_hostname: true
    new_hostname: server-01.example.test

  roles:
    - role: server_bootstrap
```

Seules les clés publiques sont lues et envoyées à la cible. La clé privée reste
sur le contrôleur.

Vérifier d'abord la syntaxe :

```bash
ansible-playbook -i inventory.yml playbook.yml --syntax-check
```

Le mode simulation peut ensuite montrer une partie des changements :

```bash
ansible-playbook -i inventory.yml playbook.yml --check --diff
```

Sur un serveur neuf, la simulation ne peut pas représenter les tâches qui
dépendent d'un paquet, d'un groupe ou d'un répertoire encore absent. La première
exécution réelle doit donc être effectuée sur une cible de validation :

```bash
ansible-playbook -i inventory.yml playbook.yml
```

### 4. Vérifier le nouveau compte

Conserver la session initiale ouverte, puis tester le compte créé dans une
seconde connexion :

```bash
ssh -i /chemin/securise/cle-ansible ansible@192.0.2.10
sudo -n id -u
```

La seconde commande doit afficher `0`. L'inventaire peut ensuite utiliser le
compte technique :

```yaml
ansible_user: ansible
ansible_ssh_private_key_file: /chemin/securise/cle-ansible
```

À partir de cette seconde phase, le playbook doit activer l'élévation de
privilèges, car le rôle modifie des fichiers et des services système :

```yaml
---
- name: Administrer les serveurs avec le compte Ansible
  hosts: all
  gather_facts: false
  become: true

  vars:
    create_user: ansible
    user_public_keys:
      - "{{ lookup('ansible.builtin.file', '/chemin/cles/ansible-ed25519.pub') }}"
    manage_hostname: true
    new_hostname: server-01.example.test

  roles:
    - role: server_bootstrap
```

Le premier passage en root conserve `become: false`. Les passages suivants
avec le compte Ansible utilisent `become: true` et son accès sudo sans mot de
passe.

Relancer enfin ce second playbook. Une exécution stable doit se terminer sans
changement :

```bash
ansible-playbook -i inventory.yml playbook.yml
```

## Comprendre les choix de sécurité

### Clés du compte technique

`user_public_keys` est toujours une liste, même lorsqu'une seule clé est
utilisée :

```yaml
user_public_keys:
  - "{{ lookup('ansible.builtin.file', '/chemin/cles/admin-1.pub') }}"
  - "{{ lookup('ansible.builtin.file', '/chemin/cles/admin-2.pub') }}"
```

La liste doit contenir au moins une clé non vide. Elle est appliquée avec
`exclusive: true` : une clé déjà présente mais absente de la liste est retirée
du fichier `authorized_keys` du compte technique.

### Connexion root et authentification par mot de passe

Par défaut, le rôle désactive les nouvelles connexions SSH root et
l'authentification par mot de passe :

```yaml
ssh_permit_root_login: "no"
ssh_password_authentication: false
ssh_kbd_interactive_authentication: false
```

Pour conserver un accès root par clé uniquement :

```yaml
deploy_root_ssh_key: true
root_public_key: "{{ lookup('ansible.builtin.file', '/chemin/cles/root-ed25519.pub') }}"
ssh_permit_root_login: prohibit-password
```

La clé root est également autoritaire : lorsque `deploy_root_ssh_key` vaut
`true`, `root_public_key` devient l'unique clé gérée dans
`/root/.ssh/authorized_keys`.

`ssh_permit_root_login: "yes"` associé à
`ssh_password_authentication: true` autorise les tentatives de connexion root
par mot de passe. Cette combinaison doit rester une décision explicite et être
protégée par les contrôles réseau appropriés.

Le rôle ne crée aucun mot de passe. Activer l'authentification par mot de passe
ne suffit donc pas à rendre un compte accessible de cette manière.

### Limiter les utilisateurs SSH

Le rôle ne crée pas de directive `AllowUsers` par défaut. Pour l'activer :

```yaml
ssh_manage_allowed_users: true
ssh_allowed_users:
  - ansible
  - root
```

La liste doit être non vide et contenir `create_user`, afin de ne pas bloquer le
compte technique. Lorsque la gestion est désactivée, une directive `AllowUsers`
provenant d'un autre fichier OpenSSH peut toujours rester effective.

Le rôle modifie les directives globales directement dans
`/etc/ssh/sshd_config`. Une ligne existante, même commentée, est remplacée sur
place. Une directive absente est ajoutée avant le premier bloc `Match`, sans
modifier les règles conditionnelles de ce bloc. Une sauvegarde horodatée est
créée avant chaque modification effective et la syntaxe est contrôlée avant le
remplacement du fichier actif.

### Administrateurs humains

Le compte technique reçoit toujours un accès sudo sans mot de passe, nécessaire
aux exécutions Ansible non interactives. Les comptes humains peuvent être
ajoutés au groupe `admin` :

```yaml
admin_group_members:
  - alice
  - bob
admin_group_nopasswd: false
admin_group_members_authoritative: false
```

Ces comptes doivent déjà exister. Avec `admin_group_nopasswd: false`, ils doivent
saisir leur mot de passe pour utiliser sudo.

Lorsque `admin_group_members_authoritative` vaut `true`, tout membre
supplémentaire du groupe est retiré. Le rôle interrompt toutefois l'exécution si
`admin` est le groupe principal d'un compte non déclaré, car ce retrait ne peut
pas être effectué de manière sûre automatiquement.

### Nom d'hôte et contrôleur Ansible

Avec la configuration par défaut, `new_hostname` doit contenir un nom court ou
un FQDN compatible DNS :

```yaml
manage_hostname: true
new_hostname: server-01.example.test
```

Le FQDN précédent produit la ligne suivante :

```text
127.0.1.1 server-01.example.test server-01
```

Pour laisser le nom d'hôte et cette ligne inchangés :

```yaml
manage_hostname: false
new_hostname: ""
```

Une entrée facultative peut rendre le contrôleur résolvable depuis l'hôte
distant. Elle est écrite dans `/etc/hosts` de cet hôte, pas sur le contrôleur :

```yaml
controller_ip: 192.0.2.20
controller_hostname: ansible-controller
```

Les deux variables doivent être renseignées ensemble. `controller_ip` accepte
actuellement une adresse IPv4 uniquement.

Pour faire l'opération inverse et ajouter l'hôte distant dans `/etc/hosts` du
contrôleur, activer explicitement :

```yaml
manage_controller_hosts: true
new_hostname: server-01.example.test
```

L'adresse IPv4 est reprise de `ansible_host` dans l'inventaire. Si celui-ci
utilise un nom DNS, renseigner `controller_hosts_ip` avec l'IP à enregistrer.
Le rôle ajoute ou met à jour la ligne correspondant à `new_hostname`, sans
sauvegarde automatique, et laisse les autres entrées inchangées. Les écritures
sont effectuées une par une pour éviter les conflits entre plusieurs hôtes.

Cette tâche exige les droits root ou sudo sur le contrôleur Linux. Elle utilise
`delegate_to: localhost` : si Ansible tourne dans WSL ou dans un conteneur,
le fichier modifié appartient à cet environnement d'exécution. Dans Molecule,
`controller_hosts_path` pointe vers un fichier temporaire, jamais vers le
véritable `/etc/hosts` du contrôleur.

## Référence des variables

### Compte et paquets

| Variable | Défaut | Description |
| --- | --- | --- |
| `create_user` | `ansible` | compte technique créé par le rôle |
| `user_public_keys` | `[]` | liste exhaustive et obligatoire des clés publiques du compte |
| `sys_packages` | voir `defaults/main.yml` | liste non vide des paquets APT installés |
| `deploy_root_nvim_config` | `false` | déploie la configuration Neovim de root |

La liste complète des paquets par défaut se trouve dans
[`defaults/main.yml`](defaults/main.yml).

### SSH

| Variable | Défaut | Description |
| --- | --- | --- |
| `deploy_root_ssh_key` | `false` | active la gestion de la clé publique root |
| `root_public_key` | chaîne vide | clé publique unique de root |
| `ssh_permit_root_login` | `no` | accepte `no`, `prohibit-password` ou `yes` |
| `ssh_password_authentication` | `false` | contrôle `PasswordAuthentication` |
| `ssh_kbd_interactive_authentication` | `false` | contrôle `KbdInteractiveAuthentication` |
| `ssh_manage_allowed_users` | `false` | demande au rôle d'écrire `AllowUsers` |
| `ssh_allowed_users` | `[]` | entrées placées dans `AllowUsers` |

### Administrateurs

| Variable | Défaut | Description |
| --- | --- | --- |
| `admin_group_members` | `[]` | comptes existants ajoutés à `admin` |
| `admin_group_nopasswd` | `false` | autorise sudo sans mot de passe pour `admin` |
| `admin_group_members_authoritative` | `false` | retire les membres supplémentaires de `admin` |

### Nom d'hôte

| Variable | Défaut | Description |
| --- | --- | --- |
| `manage_hostname` | `true` | gère le nom d'hôte et la ligne `127.0.1.1` |
| `new_hostname` | chaîne vide | nom court ou FQDN appliqué à la cible |
| `controller_ip` | chaîne vide | adresse IPv4 facultative du contrôleur |
| `controller_hostname` | chaîne vide | nom facultatif du contrôleur |
| `manage_controller_hosts` | `false` | ajoute l'hôte distant au fichier hosts du contrôleur |
| `controller_hosts_path` | `/etc/hosts` | fichier existant sur le contrôleur ; temporaire dans les tests |
| `controller_hosts_ip` | `ansible_host`, sinon nom d'inventaire | adresse IPv4 de l'hôte distant à enregistrer |

Les types de l'interface publique sont également déclarés dans
[`meta/argument_specs.yml`](meta/argument_specs.yml). Après la préparation
éventuelle de Python 3 et de `python3-apt`, le rôle contrôle les relations entre
les variables et les formats détaillés avant de configurer les comptes, sudo
et SSH.

## Tags disponibles

| Tag | Périmètre |
| --- | --- |
| `server_bootstrap` | ensemble des tâches principales |
| `packages` | paquets APT |
| `sudo` | sudoers et groupe `admin` |
| `users` | comptes, groupes et clés publiques |
| `ssh` | politique OpenSSH |
| `hostname` | nom d'hôte et `/etc/hosts` |
| `neovim` | configuration Neovim |

La préparation de Python utilise le tag `always` pour rester disponible sur une
cible minimale.

## Tester le rôle

Le serveur de test doit déjà disposer de Python, Docker, Molecule,
`molecule-plugins[docker]` et `ansible-lint`. Aucun fichier de dépendances Python
n'est fourni par le rôle.

Les collections Ansible du scénario sont déclarées dans
`molecule/default/collections.yml`. Le fichier
`molecule/default/requirements.yml` contient `roles: []`, car le scénario ne
dépend d'aucun rôle externe.

Pour récupérer le projet :

```bash
git clone https://github.com/BOURGUITSamuel/Server-Bootstrap-Ansible-Role.git server_bootstrap
cd server_bootstrap
```

Le répertoire doit porter le nom `server_bootstrap` pour que les playbooks
retrouvent le rôle. Depuis ce répertoire :

```bash
ansible-lint --profile production .
ansible-playbook --syntax-check playbook/server_bootstrap.yml
molecule test -s default
```

Le scénario vérifie Debian 13 et Ubuntu 26.04, le mode simulation, la
convergence, l'idempotence, les permissions, sudo, les connexions SSH réelles,
les clés autorisées et refusées, les politiques root et mot de passe, le groupe
`admin`, le nom d'hôte et les principales configurations invalides. Il vérifie
aussi l'ajout et la mise à jour des entrées sur le contrôleur dans un fichier
temporaire, leur idempotence, la préservation des autres hôtes et l'absence de
sauvegarde.

Les images peuvent être remplacées avec les variables d'environnement du
scénario :

```bash
MOLECULE_DEBIAN_IMAGE=registre/debian13-ansible:version \
MOLECULE_UBUNTU_IMAGE=registre/ubuntu2604-ansible:version \
molecule test -s default
```

## Validation automatique sur GitHub

La CI définie dans `.github/workflows/ci.yml` s'exécute à chaque push et pour les
pull requests. Elle contrôle la qualité Ansible puis lance le scénario Molecule
complet sur Debian 13 et Ubuntu 26.04, avec des politiques SSH et sudo différentes
entre les deux hôtes. Les versions des outils Python sont fixées dans ce workflow.

Le runner GitHub utilise Ubuntu 24.04 ; il héberge Docker et ne constitue pas une
distribution cible supplémentaire du rôle. La CI ne déploie sur aucun serveur
réel.

## Limite du test Docker

Docker monte `/etc/hostname` et `/etc/hosts` dans ses conteneurs. Le scénario
crée donc les conteneurs avec leur nom final et vérifie ensuite le nom actif, le
fichier persistant, `/etc/hosts` et l'idempotence.

Une transition réelle d'un ancien nom vers un nouveau doit encore être validée
sur une VM ou un serveur de préproduction. Les écritures non atomiques de
`/etc/hosts` sont utilisées uniquement pour les connexions Docker ; elles
restent atomiques lors d'une exécution standard en SSH.

## Contenu du rôle

| Élément | Usage |
| --- | --- |
| [`defaults/main.yml`](defaults/main.yml) | valeurs publiques par défaut |
| [`vars/main.yml`](vars/main.yml) | chemins et expressions internes |
| [`meta/argument_specs.yml`](meta/argument_specs.yml) | types de l'interface publique |
| [`tasks/`](tasks) | préparation, validations et configuration |
| [`templates/`](templates) | configuration Neovim |
| [`molecule/default/`](molecule/default) | scénario d'intégration |
| [`playbook/server_bootstrap.yml`](playbook/server_bootstrap.yml) | exemple de premier déploiement |

## Licence

Ce projet est distribué sous licence Apache 2.0. Consulter [`LICENSE`](LICENSE).
