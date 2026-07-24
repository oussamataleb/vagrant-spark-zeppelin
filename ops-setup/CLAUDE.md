# CLAUDE.md — Contexte des opérations (~/ops)

> Ce fichier est lu automatiquement par Claude Code quand il tourne dans `~/ops`.
> Il fixe le contexte et les **règles permanentes** à respecter pour toute action.

## Année en cours
- **Nous sommes en 2026.** Ne jamais écrire « 2025 » comme année courante.

## Sites hébergés sur ce VPS
Trois sites WooCommerce partagent le **préfixe de base de données `gILwCKG_`**.

| Site                 | Chemin sur le disque (À CONFIRMER)          |
|----------------------|---------------------------------------------|
| instant-key.com      | `__A_REMPLIR__` (ex. /var/www/instant-key)  |
| ms-server.com        | `__A_REMPLIR__`                             |
| naijasoftwares.com   | `__A_REMPLIR__`                             |

> ⚠️ Les chemins ci-dessus doivent être remplis à partir de la config réelle du
> serveur web (voir la commande de découverte dans RUNBOOK.md, étape 3b).
> Ne pas inventer de chemin.

## Règles permanentes (NE PAS enfreindre)
1. **Année = 2026.** Ne jamais écrire 2025.
2. **Ne jamais dépublier un produit sur la base de « zéro vente ».**
   L'historique de commandes antérieur à 2023 a été purgé : une absence de vente
   dans les données ne prouve rien.
3. **Ne jamais lancer de campagne Search payante Autodesk** (blocage DMCA).
4. **Champs meta Yoast : toujours utiliser `set_values` (au pluriel).**

## Périmètre & prudence
- Ne rien modifier en **production** (PHP, MySQL, nginx/apache, contenu des sites)
  sans confirmation explicite de l'opérateur humain.
- Les scripts d'ops récurrents vivent dans `~/ops/scripts/`, leurs sorties dans `~/ops/logs/`.
- Les secrets sont dans `~/ops/.env` (chmod 600, jamais commité, hors webroot).
