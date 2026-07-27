# Ansible Deployment 0.2.0

Déploiement et maintenance de machines Debian/Ubuntu avec Ansible.
Le projet suit une structure par environnement, un lint strict (profil `production`)
et une intégration continue qui vérifie la qualité à chaque push.

---

## 🚀 Installation

### 1. Installer Git

Sur Debian / Ubuntu :

```bash
sudo apt update && sudo apt install -y git
```

### 2. Installer Ansible

Toujours sur Debian / Ubuntu :

```bash
sudo apt update && sudo apt install -y ansible
```

Vérifier l’installation :

```bash
ansible --version
```

### 3. (Optionnel) Installer ansible-lint

Pour contrôler la qualité en local avant de pousser, en tant qu’utilisateur `ansible` :

```bash
sudo apt install -y pipx
pipx ensurepath
pipx install ansible-lint
```

---

## 📂 Cloner le dépôt

Se connecter sur la machine où Ansible sera exécuté (ex : `ansible@vbox`) puis cloner le dépôt :

```bash
git clone git@github.com:CleryMen/ansible-deployment.git
```

Aller dans le projet :

```bash
cd ansible-deployment
```

---

## 🏗️ Structure du projet

```
ansible-deployment/
├── .ansible-lint              # Exige le profil "production"
├── .github/
│   └── workflows/
│       └── lint.yml           # CI : lance ansible-lint à chaque push / PR
├── .gitignore
├── ansible.cfg                # Configuration Ansible (inventaire par défaut : lab)
├── inventories/               # Un dossier par environnement
│   └── lab/
│       ├── hosts.yml          # Machines de l’environnement lab
│       └── group_vars/
│           └── all.yml        # Variables communes au lab
├── playbooks/
│   └── debian_update.yml
├── roles/
│   └── debian_update_upgrade/
└── README.md
```

> **Ajouter un environnement plus tard** (une vraie production, par exemple) : créez
> simplement `inventories/production/` à côté de `lab/`, avec ses propres `hosts.yml`
> et `group_vars/`. Le reste du projet ne change pas.

---

## 👤 Préparer les machines cibles

Sur chaque machine distante (Debian/Ubuntu) :

### 1. Créer l’utilisateur `ansible`

```bash
sudo adduser ansible
```

### 2. Donner les droits sudo sans mot de passe

Éditer la configuration sudo :

```bash
sudo visudo
```

Ajouter à la fin :

```
ansible ALL=(ALL) NOPASSWD:ALL
```

### 3. Activer l’accès SSH par clé

Sur la machine **où Ansible est installé** (le contrôleur) :

```bash
ssh-keygen -t ed25519 -C "ansible_r1" -f ~/.ssh/ansible_r1
ssh-copy-id -i ~/.ssh/ansible_r1.pub ansible@IP_R1
```

Tester la connexion :

```bash
ssh ansible@IP_DE_LA_MACHINE
```

⚠️ Si la connexion se fait sans mot de passe, c’est prêt.

---

## ▶️ Exécution d’un playbook

Toujours se placer **à la racine du projet**, et **désigner l’environnement** via son
dossier d’inventaire :

```bash
cd ansible-deployment
ansible-playbook -i inventories/lab playbooks/debian_update.yml
```

Simulation (mode check, aucun changement appliqué) :

```bash
ansible-playbook -i inventories/lab playbooks/debian_update.yml --check
```

Vérifier la lecture de l’inventaire :

```bash
ansible-inventory -i inventories/lab --list
```

---

## ✅ Qualité et intégration continue

Le dépôt vise **zéro violation** au profil `production` d’ansible-lint.

Vérification en local (depuis la racine, en tant qu’utilisateur `ansible`) :

```bash
ansible-lint
```

À chaque `git push` sur `dev`/`main` et à chaque Pull Request, la CI GitHub Actions
(`.github/workflows/lint.yml`) relance automatiquement ansible-lint. Une croix rouge
signale une régression avant toute fusion.

---

## 🌿 Modèle de branches

* `main` — branche de référence, stable et à jour.
* `dev` — branche de travail.

Le passage de `dev` vers `main` se fait par **Pull Request** sur GitHub, une fois la
CI au vert. Les versions sont marquées par des tags annotés (ex. `v0.2.0`).

---

## 📌 Notes importantes

* Les machines cibles doivent être accessibles en SSH sur le port 22.
* L’utilisateur `ansible` doit exister sur chaque machine distante, avec
  authentification par clé SSH.
* Les rôles sont définis dans `roles/` et sont appelés par les playbooks.
* **Exécuter Ansible avec l’utilisateur `ansible`, pas `root`.**
* Utiliser les noms de modules complets (FQCN, ex. `ansible.builtin.apt`) et des
  booléens explicites (`true`/`false`).

---
