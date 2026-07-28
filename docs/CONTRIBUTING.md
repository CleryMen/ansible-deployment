# Guide de bonnes pratiques — ansible-deployment

Ce guide résume les conventions et réflexes à suivre pour faire évoluer ce dépôt
proprement. Il reflète le fonctionnement mis en place : une seule ligne de code de
référence, des environnements gérés par inventaire, un lint strict et une CI qui
protège tout ça.

---

## 1. Les principes de base

Quatre idées gouvernent tout le reste :

1. **Le code est unique, les données varient.** Un même playbook doit pouvoir tourner
   sur n'importe quel environnement. Ce qui change (adresses IP, utilisateurs,
   variables) vit dans les inventaires, pas dans des branches ou des copies de code.
2. **Branches ≠ environnements.** Les branches Git gèrent le *code dans le temps*
   (`main` stable, `dev` en cours). Les environnements se gèrent par des dossiers
   d'inventaire (`inventories/lab/`, plus tard `inventories/production/`).
3. **La qualité est automatisée.** `ansible-lint` au profil `production` doit rester
   à zéro violation, et la CI le vérifie à chaque push. On ne « fait pas confiance »,
   on mesure.
4. **On travaille toujours avec le bon utilisateur.** Ce dépôt appartient à `ansible`.
   On ne l'édite jamais en `root`, sous peine de casser les permissions.

---

## 2. Structure du projet

```
ansible-deployment/
├── .ansible-lint              # exige le profil "production"
├── .github/workflows/lint.yml # CI : lance ansible-lint à chaque push / PR
├── .gitignore
├── ansible.cfg                # inventaire par défaut = inventories/lab
├── inventories/
│   └── lab/
│       ├── hosts.yml          # machines de l'environnement lab
│       └── group_vars/
│           └── all.yml        # variables communes au lab
├── playbooks/
│   └── debian_update.yml
└── roles/
    └── debian_update_upgrade/
        └── tasks/main.yml
```

**Pour ajouter un environnement plus tard** (une vraie production, par exemple) :
créez simplement `inventories/production/` à côté de `lab/`, avec ses propres
`hosts.yml` et `group_vars/`. Aucune autre partie du projet ne bouge.

---

## 3. Le bon utilisateur et les permissions

C'est la source d'erreurs la plus fréquente sur ce projet. À retenir :

- **Tout le travail sur le dépôt se fait en tant que `ansible`** (Git, édition de
  fichiers, `ansible-lint`, `ansible-playbook`).
- **`root` ne sert qu'aux tâches système** (installer un paquet via `apt`, corriger
  une propriété de fichier). Jamais pour éditer le code du projet.
- Si un jour des fichiers se retrouvent possédés par `root` (message Git
  « dubious ownership », ou `Permission denied`), on répare d'un coup :

  ```bash
  su -
  chown -R ansible:ansible /home/ansible/ansible-deployment
  exit
  ```

- Les outils installés via `pipx` le sont **par utilisateur**. `ansible-lint` doit
  donc être installé en tant que `ansible`, pas `root`.

---

## 4. Workflow Git au quotidien

On garde deux branches longues : `main` (référence stable) et `dev` (travail en cours).

Le cycle type d'une modification :

```bash
# 1. Se placer sur dev, à jour
git checkout dev
git pull origin dev

# 2. Travailler, puis vérifier la qualité en local
ansible-lint

# 3. Committer par petits lots cohérents
git add -A
git status                    # toujours relire ce qui va être commité
git commit -m "type: description courte"

# 4. Pousser
git push origin dev
```

**Pour faire passer le travail de `dev` vers `main`**, on ne fusionne pas à l'aveugle
en local : on ouvre une **Pull Request** sur GitHub (`base: main ← compare: dev`).
La CI tourne sur la PR ; on ne fusionne que si la coche est verte. On garde le mode
« Create a merge commit » pour préserver l'historique.

> Réflexe : lancer les commandes **une par une** et lire la sortie avant d'enchaîner.
> Coller un bloc entier peut masquer l'échec d'un `cd` et exécuter la suite au mauvais endroit.

---

## 5. Convention de messages de commit

On utilise des préfixes explicites (style « Conventional Commits ») : ils rendent
l'historique lisible et prêt à générer un changelog un jour.

| Préfixe     | Quand l'utiliser                                             |
|-------------|-------------------------------------------------------------|
| `feat:`     | Nouvelle fonctionnalité (nouveau rôle, nouveau playbook)     |
| `fix:`      | Correction de bug                                           |
| `refactor:` | Réorganisation sans changement de comportement              |
| `style:`    | Mise en forme, lint, conventions (pas de logique modifiée)  |
| `chore:`    | Tâches d'outillage (gitignore, config)                      |
| `ci:`       | Modifications de la CI                                      |
| `docs:`     | Documentation, README                                      |

Exemple : `refactor: restructure inventory per environment (lab)`

Le message décrit **le pourquoi**, pas juste le quoi. Une ligne courte suffit dans
la plupart des cas.

---

## 6. Qualité : ansible-lint + CI

**En local**, avant de pousser, toujours depuis la racine du projet :

```bash
ansible-lint
```

L'objectif est constant : `Profile 'production' was required, and it passed.`
avec 0 violation.

**En CI**, le workflow `.github/workflows/lint.yml` relance ansible-lint
automatiquement à chaque push sur `dev`/`main` et à chaque Pull Request. Une croix
rouge ❌ signale une régression avant toute fusion.

**Si le lint échoue :** lire le message, il indique le fichier, la ligne et la règle.
Les cas les plus courants sur ce projet :

- `fqcn` → écrire `ansible.builtin.apt` au lieu de `apt`.
- `yaml[truthy]` → écrire `true`/`false`, jamais `yes`/`no`.
- `no-changed-when` → une tâche `command`/`shell` doit préciser quand elle est
  « changée » (voir §8).

**Ignorer une règle** ne se fait qu'à bon escient, tâche par tâche, avec un commentaire
`# noqa: <règle>` qui documente pourquoi. On ne désactive jamais une règle globalement
pour se débarrasser d'un vrai problème.

---

## 7. Gérer les environnements (inventaires)

On désigne toujours l'environnement explicitement au lancement :

```bash
ansible-playbook -i inventories/lab playbooks/debian_update.yml
```

- Les variables spécifiques à une machine vont dans son entrée `hosts.yml`.
- Les variables communes à tout l'environnement vont dans `group_vars/all.yml`.
- On ne répète pas une valeur sur chaque hôte si elle peut être factorisée dans
  `group_vars`.
- Les noms de groupes sont en **minuscules, sans tiret** (`debian`, pas `CL-Debian`),
  pour rester compatibles avec les `group_vars`.

---

## 8. Écrire des playbooks et des rôles propres

- **FQCN partout** : `ansible.builtin.apt`, `ansible.builtin.service`,
  `ansible.builtin.debug`… Cohérent et sans ambiguïté.
- **Booléens explicites** : `true` / `false`, jamais `yes` / `no`.
- **Idempotence** : une tâche relancée sur un système déjà conforme ne doit rien
  changer. Pour une commande brute, encadrer avec `creates:`, `removes:`, ou
  `changed_when:` / `failed_when:`. Exemple :

  ```yaml
  - name: Générer la configuration
    ansible.builtin.command: mon-outil --init
    args:
      creates: /etc/mon-outil/config    # ne s'exécute que si le fichier manque
  ```

- **Sauvegarde avant écrasement** : quand un `template:` ou `copy:` remplace un
  fichier système, ajouter `backup: true` pour pouvoir revenir en arrière.
- **Nommer chaque tâche** avec un `name:` clair et lisible.
- **Handlers** pour les réactions à un changement (redémarrer un service après avoir
  modifié sa conf), plutôt que des tâches conditionnées sur `changed`.

---

## 9. Sécurité

- **Clés SSH** plutôt que mots de passe, pour Git comme pour Ansible. Une clé
  dédiée par usage (une pour GitHub, une par cible Ansible) est plus propre.
- **`host_key_checking`** est actuellement désactivé dans `ansible.cfg` : acceptable
  en lab, mais à réactiver (avec un `known_hosts` pré-rempli) le jour d'une vraie
  production, pour se protéger des attaques d'interception.
- **Secrets** : dès qu'un mot de passe, token ou clé privée doit vivre dans le dépôt,
  utiliser **`ansible-vault`** pour le chiffrer. Jamais de secret en clair dans Git.
  Ajouter le fichier de mot de passe du vault au `.gitignore`.

---

## 10. Versionnage

On suit le versionnage sémantique (SemVer) : `vMAJEUR.MINEUR.CORRECTIF`.

- **MAJEUR** : changement incompatible (ex. réorganisation qui casse l'usage existant).
- **MINEUR** : évolution notable rétrocompatible (nouveau rôle, nouvelle structure).
- **CORRECTIF** : petite correction sans changement d'usage.

On marque une version avec un tag **annoté**, sur `main`, puis on le pousse :

```bash
git checkout main
git pull origin main
git tag -a v0.3.0 -m "Description de la version"
git push origin v0.3.0
```

Les tags ne partent pas avec un `git push` normal : il faut les pousser explicitement.

---

## 11. Checklist avant chaque push

- [ ] Je suis bien l'utilisateur `ansible` (pas `root`).
- [ ] Je suis à la racine du projet.
- [ ] `ansible-lint` passe au profil `production`, 0 violation.
- [ ] `ansible-inventory --list` lit correctement l'inventaire visé.
- [ ] `git status` ne montre que les changements attendus.
- [ ] Mon message de commit a un préfixe clair et explique le pourquoi.
- [ ] Je pousse sur `dev` (le passage vers `main` se fait par Pull Request).

---

## 12. Pistes pour aller plus loin

Sans urgence, dans l'ordre de rentabilité :

1. **Mettre le README à jour** : nouvelle structure d'inventaire, commande
   `-i inventories/lab`, mention de la CI et du profil `production`.
2. **Protéger la branche `main`** (Settings → Branches sur GitHub) : exiger que la CI
   soit verte avant toute fusion. Le filet de sécurité devient un garde-barrière.
3. **`ansible-vault`** dès le premier secret à gérer.
4. **Molecule** pour tester les rôles dans des conteneurs jetables.
5. **Durcir progressivement** : quand tout est stable, envisager d'autres contrôles
   (yamllint dédié, tests de syntaxe `--syntax-check` en CU).

---

*Ce guide accompagne le dépôt à partir de la version v0.2.0. À faire évoluer avec le projet.*
