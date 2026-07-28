# Se connecter au dépôt — guide Windows & Linux

Ce guide explique comment configurer l'accès SSH au dépôt
`ansible-deployment` et le cloner, depuis une machine **Linux/macOS** ou
**Windows**. Chaque machine a besoin de sa **propre clé SSH** : on ne recopie
jamais la clé privée d'un autre poste.

> Principe : GitHub est la source de vérité. Chaque PC n'est qu'une copie locale
> qu'on synchronise avec `git pull` (récupérer) et `git push` (envoyer).

---

## 🐧 Linux / macOS

### 1. Installer Git

```bash
sudo apt update && sudo apt install -y git      # Debian / Ubuntu
```

### 2. Générer une clé SSH dédiée à ce poste

Donnez un nom explicite à la clé (ici `github_lab`) :

```bash
ssh-keygen -t ed25519 -C "github-lab" -f ~/.ssh/github_lab
```

- Appuyez deux fois sur Entrée pour ne pas mettre de passphrase (ou choisissez-en une).
- Deux fichiers sont créés : `github_lab` (privée, secrète) et `github_lab.pub` (publique).

### 3. Indiquer la clé à SSH

Comme la clé a un nom personnalisé, il faut le déclarer dans `~/.ssh/config` :

```bash
cat >> ~/.ssh/config << 'EOF'
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_lab
  IdentitiesOnly yes
EOF
```

### 4. Ajouter la clé publique sur GitHub

```bash
cat ~/.ssh/github_lab.pub
```

Copiez la ligne affichée, puis sur GitHub :
**Settings → SSH and GPG keys → New SSH key** → collez → **Add SSH key**.

### 5. Tester

```bash
ssh -T git@github.com
```

Réponse attendue : `Hi CleryMen! You've successfully authenticated...`

### 6. Cloner

```bash
git clone git@github.com:CleryMen/ansible-deployment.git
cd ansible-deployment
git checkout dev
```

---

## 🪟 Windows (PowerShell)

> ⚠️ Sous PowerShell, le raccourci `~` n'est pas toujours interprété. On utilise
> `$env:USERPROFILE` (équivalent de `C:\Users\<vous>`) pour les chemins.

### 1. Installer Git

Téléchargez **Git for Windows** depuis https://git-scm.com puis installez-le
(les options par défaut conviennent). OpenSSH est inclus dans Windows récent.

### 2. Générer une clé SSH dédiée à ce poste

```powershell
ssh-keygen -t ed25519 -C "github-pc-windows" -f $env:USERPROFILE\.ssh\github_windows
```

- Si un message dit que le dossier `.ssh` n'existe pas, créez-le d'abord :
  `mkdir $env:USERPROFILE\.ssh`
- Appuyez deux fois sur Entrée pour ne pas mettre de passphrase.

### 3. Indiquer la clé à SSH (fichier `config`)

Créez le fichier `config` proprement, **sans Notepad** (qui ajoute un `.txt` piégeux) :

```powershell
@"
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/github_windows
  IdentitiesOnly yes
"@ | Set-Content -Encoding ascii $env:USERPROFILE\.ssh\config
```

> Dans le fichier `config`, la syntaxe `~/.ssh/...` est correcte : c'est OpenSSH
> qui le lit, pas PowerShell.

Vérifiez :

```powershell
Get-Content $env:USERPROFILE\.ssh\config
```

### 4. Ajouter la clé publique sur GitHub

```powershell
Get-Content $env:USERPROFILE\.ssh\github_windows.pub
```

Copiez la ligne entière (de `ssh-ed25519` à `github-pc-windows`), puis sur GitHub :
**Settings → SSH and GPG keys → New SSH key** → collez → **Add SSH key**.

### 5. Tester

```powershell
ssh -T git@github.com
```

Réponse attendue : `Hi CleryMen! You've successfully authenticated...`

### 6. Cloner (hors OneDrive !)

⚠️ Ne clonez **pas** dans un dossier synchronisé OneDrive : la synchro interfère
avec le dossier `.git` (erreurs de suppression, verrous de fichiers). Choisissez un
dossier local :

```powershell
cd $env:USERPROFILE
mkdir projets
cd projets
git clone git@github.com:CleryMen/ansible-deployment.git
cd ansible-deployment
git checkout dev
```

Le dépôt sera dans `C:\Users\<vous>\projets\ansible-deployment`.

---

## 🔧 Configurer son identité Git (les deux systèmes)

À faire une fois par machine ; ce nom apparaît dans vos commits :

```bash
git config --global user.name "CleryMen"
git config --global user.email "votre-email@exemple.com"
```

---

## 🔄 Rythme de travail à plusieurs machines

Règle d'or : **on récupère en arrivant, on envoie avant de partir.**

En début de session :

```bash
git checkout dev
git pull origin dev
```

En fin de session :

```bash
git add -A
git commit -m "type: description"
git push origin dev
```

Ainsi les postes restent synchronisés et aucun travail n'est perdu.

---

## 🆘 Dépannage

### `Permission denied (publickey)`

GitHub ne reconnaît pas la clé. Vérifiez, dans l'ordre :

1. **La clé publique est-elle ajoutée sur GitHub ?**
   → https://github.com/settings/keys
2. **Le fichier `config` existe-t-il et pointe-t-il vers la bonne clé ?**
   Lancez `ssh -T -v git@github.com` et cherchez une ligne
   `Will attempt key: ... github_windows` (ou `github_lab`).
   - Si votre clé n'apparaît pas → le `config` n'est pas lu (mauvais nom ou absent).
   - Si elle apparaît mais échoue → la clé n'est pas (bien) ajoutée sur GitHub.

### Sous Windows, `config.txt` au lieu de `config`

Notepad enregistre souvent en `.txt`. Vérifiez et renommez :

```powershell
Get-ChildItem $env:USERPROFILE\.ssh
Rename-Item $env:USERPROFILE\.ssh\config.txt config
```

### Sous PowerShell, `Saving key ... failed: No such file or directory`

Le dossier `.ssh` n'existe pas, ou le `~` n'a pas été interprété.
Créez le dossier et utilisez le chemin complet :

```powershell
mkdir $env:USERPROFILE\.ssh
ssh-keygen -t ed25519 -C "github-pc-windows" -f $env:USERPROFILE\.ssh\github_windows
```

### Boucle « Deletion of directory '.git/hooks' failed » (Windows/OneDrive)

Tapez `n`, puis supprimez le dossier incomplet et reclonez **hors OneDrive** :

```powershell
Remove-Item -Recurse -Force ansible-deployment
```

---

## 📌 Bonnes pratiques

* Une clé SSH **par machine**, avec un nom explicite (`github_lab`, `github_windows`).
* Ne jamais partager ni copier une **clé privée** ; seule la `.pub` va sur GitHub.
* Sur un poste professionnel, vérifier que l'usage d'un dépôt personnel est autorisé.
* Cloner hors des dossiers synchronisés (OneDrive, Dropbox…).

---

*Guide d'accès au dépôt ansible-deployment — à adapter si les noms de clés changent.*
