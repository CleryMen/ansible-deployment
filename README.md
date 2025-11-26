# Ansible Deployment DEV

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

---

## 📂 Cloner le dépôt

Se connecter sur la machine où Ansible sera exécuté (ex: `ansible@vbox`) puis cloner le dépôt :

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
├── ansible.cfg          # Configuration Ansible
├── inventories/         # Inventaire des machines
│   └── hosts.yml
├── playbooks/           # Playbooks Ansible
│   └── debian_update.yml
├── roles/               # Rôles Ansible
│   └── debian_update_upgrade/
└── README.md
```

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

Sur la machine **où Ansible est installé** (ton contrôleur) :

```bash
ssh-keygen -t ed25519 -C "ansible"
ssh-copy-id ansible@IP_DE_LA_MACHINE
```

Tester la connexion :

```bash
ssh ansible@IP_DE_LA_MACHINE
```

⚠️ Si tu peux te connecter sans mot de passe, c’est prêt.

---

## ▶️ Exécution d’un playbook

Toujours se placer **à la racine du projet** :

```bash
cd ansible-deployment
```

Lancer un playbook avec inventaire :

```bash
ansible-playbook playbooks/debian_update.yml -i inventories/hosts.yml
```

Simulation (mode check, aucun changement appliqué) :

```bash
ansible-playbook playbooks/debian_update.yml -i inventories/hosts.yml --check
```

---

## 📌 Notes importantes

* Les machines cibles doivent être accessibles en SSH sur le port 22.
* L’utilisateur `ansible` doit exister sur chaque machine distante, avec authentification par clé SSH.
* Les rôles sont définis dans le dossier `roles/` et sont appelés par les playbooks.
* Exécuter Ansible avec l’utilisateur `ansible`, pas `root`.

---

