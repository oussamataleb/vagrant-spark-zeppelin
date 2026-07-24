# RUNBOOK — Héberger Claude Code en Remote Control sur le VPS

> **Pourquoi ce runbook et pas une exécution SSH directe ?**
> Claude Code tourne ici dans un conteneur cloud isolé, sans tes clés SSH
> (`digitlicense`, `instant-key`) et sans accès réseau sortant vers le VPS
> (`176.123.6.211:22` et `213.190.7.180:65002` sont bloqués). Je ne peux donc
> **pas** ouvrir de session SSH ni exécuter/vérifier de commande sur le serveur.
> Ce runbook te met aux commandes : pour **chaque étape**, la commande à lancer,
> puis **le contrôle à faire** avant de passer à la suivante. Rien qui touche la
> prod n'est inclus sans que tu le déclenches toi-même.

## Cible

Le VPS visé est **`instant-key` = `root@176.123.6.211`** (accès root → systemd
possible ; il héberge instant-key.com, ms-server.com, naijasoftwares.com).

> L'autre hôte, `digitlicense` (`u507704156@213.190.7.180:65002`), est un
> hébergement mutualisé (port 65002, pas de root) : **pas** la cible pour un
> service systemd. Si je me trompe de machine, dis-le avant de continuer.

Depuis ton Mac :
```bash
ssh instant-key
```
Toutes les commandes ci-dessous s'exécutent **sur le VPS**, sauf mention contraire.

---

## Étape 1 — Audit préalable (lecture seule, ne casse rien)

```bash
# OS + version
cat /etc/os-release | grep -E '^(PRETTY_NAME|VERSION)='
uname -srm
# RAM libre / dispo
free -h
# Espace disque
df -h /
# User courant + droits sudo
whoami; id
sudo -n -l 2>/dev/null | head -n 20 || echo "sudo interactif requis"
# Outils déjà présents + versions
for b in node npm tmux git claude; do printf '%-6s ' "$b"; command -v "$b" >/dev/null && "$b" --version 2>/dev/null | head -n1 || echo "ABSENT"; done
# Ne rien toucher aux services en prod : juste constater qu'ils tournent
systemctl is-active nginx apache2 php*-fpm mysql mariadb 2>/dev/null
```

**Vérification avant de continuer :**
- Tu connais l'OS et sa version (Ubuntu/Debian probable).
- Il reste de la RAM et de l'espace disque (installer Node.js ≈ 150–200 Mo).
- `whoami` = `root` (ou un user avec `sudo`).
- Tu notes quelles versions de `node`/`npm`/`tmux`/`git`/`claude` existent déjà.
- nginx/apache + php-fpm + mysql/mariadb sont `active` → **on n'y touche pas**.

> ⚠️ **On ne modifie aucun paquet PHP/MySQL/nginx/apache existant.** Node.js
> s'installe à côté et n'entre pas en conflit avec la stack web des sites.

---

## Étape 2 — Installation (Node.js 22 LTS, tmux, git, Claude Code)

### 2a. Node.js 22 LTS — **seulement si absent ou trop ancien**

Vérifie d'abord :
```bash
node --version 2>/dev/null   # cible : v22.x (ou plus récent LTS)
```
- Si `node` ≥ v22 → **saute 2a**.
- Sinon, sur **Debian/Ubuntu** (adapte si l'audit révèle une autre distro — RHEL/Alma utiliseraient `dnf` et le repo NodeSource `rpm`) :

```bash
# Dépôt officiel NodeSource pour Node.js 22.x
curl -fsSL https://deb.nodesource.com/setup_22.x -o /tmp/nodesource_setup.sh
less /tmp/nodesource_setup.sh          # (optionnel) inspecte avant d'exécuter
sudo -E bash /tmp/nodesource_setup.sh
sudo apt-get install -y nodejs
```

**Vérif :** `node --version` affiche `v22.x` et `npm --version` répond.

### 2b. tmux + git — seulement si absents

```bash
sudo apt-get install -y tmux git
```
**Vérif :** `tmux -V` et `git --version` répondent.

### 2c. Claude Code

```bash
sudo npm install -g @anthropic-ai/claude-code
```

**Vérif (obligatoire) :**
```bash
claude --version
which claude          # note bien ce chemin absolu → sert au service systemd
```
- La version doit être une **2.1.x récente**. Ton hypothèse « ≥ 2.1.52 » est
  probablement **trop basse** pour Remote Control : installe la **dernière**
  version et, en cas de doute, mets à jour (`sudo npm update -g @anthropic-ai/claude-code`).
- Le vrai test de compatibilité est fonctionnel : si `/login` puis
  `claude remote-control` marchent (étapes 7a–7b), la version convient.

> Note le chemin renvoyé par `which claude` (souvent `/usr/bin/claude` ou
> `/usr/local/bin/claude`) : tu le mettras dans `__CLAUDE_BIN__`.

---

## Étape 3 — Espace de travail `~/ops`

```bash
mkdir -p ~/ops/scripts ~/ops/logs
cd ~/ops
git init            # versionne tes scripts d'ops (optionnel mais conseillé)
```

### 3a. Poser CLAUDE.md et le .env d'exemple
Récupère les fichiers de ce repo (`ops-setup/CLAUDE.md`, `ops-setup/env.example`)
et pose-les :
```bash
# depuis ton Mac, si tu as cloné ce repo :
scp ops-setup/CLAUDE.md   instant-key:~/ops/CLAUDE.md
scp ops-setup/env.example instant-key:~/ops/.env
# — OU — colle leur contenu à la main dans ~/ops/CLAUDE.md et ~/ops/.env
```

### 3b. Découvrir les vrais chemins disque des 3 sites (pour compléter CLAUDE.md)
```bash
# nginx
grep -RhoE 'root\s+/[^;]+' /etc/nginx/sites-* 2>/dev/null | sort -u
# apache
grep -RhoE 'DocumentRoot\s+/\S+' /etc/apache2/sites-* 2>/dev/null | sort -u
```
Reporte les chemins réels dans les lignes `__A_REMPLIR__` de `~/ops/CLAUDE.md`.
**Ne devine pas** : mets uniquement ce que la config révèle.

**Vérif :** `ls -la ~/ops` montre `scripts/ logs/ CLAUDE.md .env` ; `CLAUDE.md`
ne contient plus de `__A_REMPLIR__`.

---

## Étape 4 — Secrets (`~/ops/.env`, chmod 600, hors webroot)

```bash
chmod 600 ~/ops/.env
ls -l ~/ops/.env            # doit afficher -rw------- (600)
# .env est déjà dans ~/ops (le home), donc hors de tout webroot des sites — OK.

# .gitignore de ~/ops
printf '.env\nlogs/\n' >> ~/ops/.gitignore
```

Puis **édite `~/ops/.env` toi-même** et renseigne les 5 clés Google Ads
(je n'en génère ni n'en affiche aucune) :
```
GOOGLE_ADS_DEVELOPER_TOKEN=
GOOGLE_ADS_CLIENT_ID=
GOOGLE_ADS_CLIENT_SECRET=
GOOGLE_ADS_REFRESH_TOKEN=
GOOGLE_ADS_LOGIN_CUSTOMER_ID=
```

**Vérif :** `stat -c '%a' ~/ops/.env` → `600` ; `git -C ~/ops check-ignore .env`
renvoie `.env` (donc ignoré). Le fichier est dans `~/ops`, pas sous
`/var/www` → **hors webroot**.

---

## Étape 5 — Service systemd `claude-ops` (activé, PAS démarré)

Récupère `ops-setup/claude-ops.service`, **remplace les 4 marqueurs**, puis :

```bash
# Renseigne les valeurs (exemple pour root ; adapte à l'audit) :
#   __USER__       = root
#   __HOME__       = /root
#   __CLAUDE_BIN__ = $(which claude)
#   __TMUX_BIN__   = $(which tmux)

sudo tee /etc/systemd/system/claude-ops.service >/dev/null < ton_fichier_edite.service
sudo systemctl daemon-reload
sudo systemctl enable claude-ops        # ACTIVE au boot, mais NE DÉMARRE PAS
```

**Vérif :** `systemctl is-enabled claude-ops` → `enabled` ;
`systemctl is-active claude-ops` → `inactive` (normal, on ne démarre pas encore).

> **Variante plus robuste (optionnelle, sans tmux) :** si tu préfères que
> `Restart=on-failure` relance réellement Claude quand il crashe, remplace le
> bloc `[Service]` par un `Type=simple` qui lance `claude` directement :
> ```
> [Service]
> Type=simple
> User=__USER__
> Environment=HOME=__HOME__
> WorkingDirectory=__HOME__/ops
> ExecStart=__CLAUDE_BIN__ remote-control --name "VPS ops"
> Restart=on-failure
> RestartSec=5
> StandardOutput=append:__HOME__/ops/logs/remote-control.log
> StandardError=append:__HOME__/ops/logs/remote-control.log
> ```
> Tu perds l'attache tmux interactive mais tu gagnes un vrai redémarrage auto
> et l'URL/QR de session est journalisée dans `~/ops/logs/remote-control.log`.

---

## Étape 6 — Sécurité SSH & firewall (audit ; diff proposé AVANT toute modif)

### 6a. Auth par mot de passe désactivée + root login interdit ?
```bash
sudo sshd -T 2>/dev/null | grep -Ei '^(passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication|challengeresponseauthentication)'
```
Lecture attendue pour un serveur durci :
- `passwordauthentication no`
- `permitrootlogin no` (ou `prohibit-password` si tu te connectes en root par clé)
- `pubkeyauthentication yes`

> ⚠️ **Attention** : tu te connectes actuellement en **root par clé**
> (`User root` dans ton `~/.ssh/config`). Passer `permitrootlogin no`
> **te couperait l'accès** si tu n'as pas d'autre user sudo avec clé. Le choix
> `prohibit-password` (root autorisé **par clé seulement**) est le compromis sûr
> pour ta config actuelle. **Ne modifie rien sans un 2e accès de secours ouvert.**

Si un durcissement est nécessaire, **voici le diff proposé** (à appliquer via un
fichier drop-in, non destructif, qui n'écrase pas `/etc/ssh/sshd_config`) :
```bash
# NE PAS exécuter à l'aveugle — relis, garde une session SSH ouverte en secours
sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
EOF
sudo sshd -t            # teste la config AVANT de recharger
# Depuis une SECONDE session de secours déjà ouverte, puis :
sudo systemctl reload ssh    # (ou 'sshd' selon la distro)
```
**Vérif :** rouvre une **nouvelle** session `ssh instant-key` dans un autre
terminal ; si elle passe, c'est bon. Ne ferme jamais ta session actuelle avant
d'avoir validé la nouvelle.

### 6b. Firewall & ports ouverts
```bash
# Firewall applicatif
sudo ufw status verbose 2>/dev/null || echo "ufw absent"
sudo firewall-cmd --list-all 2>/dev/null || echo "firewalld absent"
# Ports réellement en écoute
sudo ss -tulpen | grep LISTEN
```
**Vérif / signalement :** attends-toi à voir 22 (SSH), 80/443 (web), 3306
(MySQL — **doit être en `127.0.0.1`**, pas `0.0.0.0`). Signale tout port
inattendu exposé publiquement (ex. 3306 sur `0.0.0.0`, un panneau d'admin, etc.).
Remote Control **n'ouvre aucun port entrant** (tout est sortant HTTPS).

---

## Étape 7 — Finalisation

### 7a. `/login` interactif (à faire par TOI, depuis Windows/navigateur)
Remote Control exige une auth OAuth claude.ai **au préalable** (les clés API ne
sont pas acceptées). Depuis ton PC Windows :

1. Ouvre un terminal SSH vers le VPS (PowerShell : `ssh instant-key`, ou PuTTY).
2. Accepte d'abord la confiance de l'espace de travail dans `~/ops` :
   ```bash
   cd ~/ops
   claude
   ```
   Réponds « oui » au dialogue de confiance (Remote Control ne peut pas démarrer
   depuis le home `~` nu ; `~/ops` est un dossier de projet → OK).
3. Toujours dans cette session, lance :
   ```
   /login
   ```
   Claude affiche une **URL OAuth**. Copie-la dans ton navigateur Windows,
   connecte-toi à **claude.ai** (le **même compte** que tu utiliseras sur le
   téléphone/navigateur), autorise, colle le code de retour si demandé.
4. Quand « Logged in » s'affiche, quitte cette session (`Ctrl-C` / `/exit`).
   Les identifiants sont désormais en cache dans `~/.claude`.

### 7b. Démarrer le service et récupérer l'URL de session
```bash
sudo systemctl start claude-ops
systemctl status claude-ops --no-pager
# Attache-toi à la session tmux pour voir l'URL + le QR code :
tmux attach -t ops
```
Dans le panneau tmux, `claude remote-control` affiche l'**URL de session**
(`https://claude.ai/code?...`) ; appuie sur **Espace** pour afficher le **QR code**.
Détache-toi sans tuer la session avec **`Ctrl-b` puis `d`**.

> Avec la variante `Type=simple` (étape 5), l'URL est aussi écrite dans
> `~/ops/logs/remote-control.log` : `tail -f ~/ops/logs/remote-control.log`.

### 7c. Se connecter depuis n'importe quel appareil
- **Navigateur** (Windows/Mac/mobile) : ouvre l'URL, ou va sur
  **claude.ai/code** et retrouve la session par son nom (« VPS ops »), pastille
  verte = en ligne.
- **App mobile Claude** : scanne le QR code.
- Aucune contrainte réseau : tout passe par le cloud Anthropic en HTTPS sortant.
  Ton Mac n'est plus nécessaire. ✅

---

## Récapitulatif — ce qui reste MANUEL (moi je ne peux pas le faire à ta place)

1. **Exécuter chaque étape sur le VPS** (je n'ai ni tes clés SSH ni l'accès réseau).
2. **Étape 2c** : confirmer `claude --version` (dernière 2.1.x) et noter `which claude`.
3. **Étape 3b** : remplir les vrais chemins disque des 3 sites dans `~/ops/CLAUDE.md`.
4. **Étape 4** : saisir toi-même les 5 secrets Google Ads dans `~/ops/.env` (je n'en fournis aucun).
5. **Étape 5** : remplacer les 4 marqueurs du fichier `.service`, `enable` (sans `start`).
6. **Étape 6** : décider du durcissement SSH **en gardant une session de secours** ;
   attention à `PermitRootLogin` puisque tu es en root-par-clé.
7. **Étape 7a** : faire le `/login` OAuth dans ton navigateur (interactif, toi seul).
8. **Étape 7b** : `systemctl start claude-ops` puis récupérer l'URL/QR.

> **Garde-fou** : aucune commande de ce runbook ne modifie les sites en prod
> (PHP/MySQL/nginx/apache/contenu). Tout ajout se fait « à côté ». Demande-moi
> avant toute action touchant la prod.
