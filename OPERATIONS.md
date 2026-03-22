# Mailcow selfmail — mise en service

Fichiers sensibles (ne pas commiter) : `mailcow.conf`, `docker-compose.override.yml`, `data/assets/ssl/`.

## Git : remotes

- **`origin`** → ton dépôt : `git@github.com:Shayajs/mailcow-perso.git` (push / pull de ta copie).
- **`upstream`** → Mailcow officiel : mises à jour avec `git fetch upstream` puis `git merge upstream/master` (ou rebase), puis `git push origin master`.

Le dépôt a été passé en **non-shallow** pour pouvoir pousser tout l’historique ; un clone `--depth 1` seul provoquait une erreur côté GitHub.

## 0. Local (Cursor / WSL) vs VPS OVH

Ce dépôt peut être préparé **en local** : les conteneurs Docker et `mailcow.conf` avec mots de passe générés ne tournent pas sur ton serveur tant que tu n’as pas déployé là-bas.

**Sur le VPS**, après `rsync` / `git pull` / copie du dossier :

1. Si tu n’emportes pas ton `mailcow.conf` : `cp mailcow.conf.example mailcow.conf`, éditer les secrets et `MAILCOW_HOSTNAME`, puis `chmod 600 mailcow.conf` et `ln -sf mailcow.conf .env`.
2. Vérifier **`IPV4_NETWORK`** : `docker network ls` puis `docker network inspect <nom>` — choisir un préfixe `/24` Mailcow **sans chevauchement** (voir commentaire dans `mailcow.conf`).
3. `docker compose pull` et `docker compose up -d` **sur le serveur** uniquement quand DNS / pare-feu sont prêts ou pour un test avec IP directe.

## 1. Sur le VPS OVH

- Installer les prérequis Mailcow : Docker 24+, Docker Compose plugin, **jq**, git, curl.
- Copier tout le dossier `selfmail/` sur le serveur (ou `git pull` si dépôt privé).
- `chmod 600 mailcow.conf` et `ln -sf mailcow.conf .env` (déjà fait en local si tu synchronises tel quel).
- `cp docker-compose.override.example.yml docker-compose.override.yml` puis vérifier les noms de réseaux (`docker network ls`).
- Lancer : `docker compose pull` puis `docker compose up -d`.

Interface admin Mailcow : **https://IP_DU_VPS:8443** (ou via NPM, voir §3).  
Identifiants par défaut : **admin** / **moohoo** — les changer immédiatement.

## 2. Pare-feu

Ouvrir au minimum : **25, 465, 587, 143, 993, 4190** (et **8080, 8443** si accès direct sans NPM).  
Laisser NPM gérer **80/443** pour les sites existants.

## 3. Nginx Proxy Manager + Let’s Encrypt (mail.brightshell.fr)

Mailcow écoute en **8080** (HTTP) et **8443** (HTTPS) sur l’hôte pour ne pas prendre 80/443.

Pour que **ACME HTTP-01** de Mailcow réussisse, NPM doit exposer le domaine **mail.brightshell.fr** et router le challenge vers Mailcow :

- Créer une **Proxy Host** (ou stream si tu termines le TLS côté NPM uniquement) pour `mail.brightshell.fr`.
- Dans **Advanced** / configuration personnalisée Nginx, ajouter un bloc qui envoie **uniquement** le challenge ACME vers le port HTTP de Mailcow, par exemple :

```nginx
location ^~ /.well-known/acme-challenge/ {
  proxy_pass http://127.0.0.1:8080;
  proxy_set_header Host $host;
}
```

Adapter si Docker écoute sur une autre IP (passerelle du bridge `docker0` ou IP du conteneur nginx-mailcow).

Alternative sans NPM pour le certificat Mailcow : **ACME DNS-01** (Cloudflare API) — activer `ACME_DNS_CHALLENGE=y` dans `mailcow.conf` et suivre la doc Mailcow.

## 4. DNS Cloudflare (brightshell.fr)

| Type | Nom | Contenu | Notes |
|------|-----|---------|--------|
| A | mail | IP du VPS | **DNS only** (nuage gris) |
| MX | @ | `mail.brightshell.fr` | Priorité 10 (ou 20 selon ta politique) |
| TXT | @ | SPF (voir UI Mailcow après ajout du domaine) | Une seule ligne SPF |
| TXT | sélecteur DKIM | valeur Mailcow | Après création du domaine dans Mailcow |
| TXT | `_dmarc` | `v=DMARC1; p=none; rua=mailto:…` | Durcir plus tard |

Dans Mailcow : **Configuration → Ajouter le domaine** `brightshell.fr`, créer les boîtes, copier les enregistrements DNS / DKIM indiqués.

## 5. PTR (rDNS) OVH

Faire pointer le PTR de l’IP publique vers **`mail.brightshell.fr`** (aligné avec `MAILCOW_HOSTNAME`).

## 6. Spamhaus sur OVH

Avec une IP OVH, sans clé **Spamhaus DQS**, certaines listes sont désactivées. Inscription gratuite DQS recommandée, puis renseigner `SPAMHAUS_DQS_KEY` dans `mailcow.conf` et `docker compose up -d`.

## 7. Tests délivrabilité

- [mail-tester.com](https://www.mail-tester.com)
- Envoi vers Gmail / Outlook, vérifier **Authentication-Results** (SPF, DKIM, DMARC).

## 8. Laravel / Brightshell (mail sortant)

Avec l’override réseau, depuis un conteneur sur `brightshell_internal` ou `www_laravel_net` :

- **MAIL_HOST**=`mailcow-smtp`
- **MAIL_PORT**=`587` (ou `465` selon schéma)
- Utilisateur / mot de passe = compte mail dédié (ex. `noreply@brightshell.fr`) créé dans Mailcow.

## 9. Ajouter **allotata.fr** plus tard (même instance)

1. Mailcow : **Ajouter le domaine** `allotata.fr`, boîtes / alias.
2. Zone DNS Cloudflare de `allotata.fr` : **MX** → même cible **`mail.brightshell.fr`** (pas besoin d’un second serveur).
3. SPF / DKIM / DMARC pour **allotata.fr** comme pour brightshell (enregistrements distincts dans cette zone).

Une seule IP et un seul PTR vers `mail.brightshell.fr` restent corrects si SPF/DKIM/DMARC sont bons par domaine.
