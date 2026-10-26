# Handoff Instant-Key — à exécuter dans Claude Code LOCAL (SSH déjà configuré)

Contexte : audit public déjà fait depuis le cloud (voir rapport). Reste à faire côté serveur.
Règle : LECTURE SEULE sur WooCommerce. Aucune donnée personnelle client dans les sorties (pas de nom, email, adresse, IP).
Ne jamais afficher wp-config.php ni de secrets.

## A. Commande AutoCAD « claude.ai » (lecture seule)
Via `wp eval` (compatible HPOS ou non). Remplacer ORDER_ID (la commande reçue le 25/10/2026).

```php
$o = wc_get_order(ORDER_ID);
$keys = ['source_type','utm_source','utm_medium','utm_campaign','referrer','session_entry',
         'session_pages','session_count','session_start_time','device_type','user_agent'];
foreach ($keys as $k) echo $k.': '.$o->get_meta('_wc_order_attribution_'.$k)."\n";
echo 'total: '.$o->get_total().' '.$o->get_currency()."\n";
foreach ($o->get_items() as $i) echo $i->get_name().' | variation_id='.$i->get_variation_id()."\n";
```
Durée de session = date de commande − `session_start_time`. `user_agent` : n'en déduire que le type d'appareil.

## B. Historique commandes issues d'assistants IA (toute la période)
Parcourir `wc_get_orders(['limit'=>-1,'return'=>'ids'])` par lots, filtrer sur `utm_source` / `referrer`
contenant : claude.ai, chatgpt.com, chat.openai.com, perplexity.ai, gemini.google.com,
copilot.microsoft.com, bing.com/chat. Sortir : nombre, CA (statuts completed/processing), produits,
`session_entry` (sans query string perso). Agréger, ne pas lister de clients.

## C. À me ramener (cloud ou local)
Page d'entrée de la commande → je vérifie : présence dans llms.txt, FAQPage balisée, position GSC.
GSC est déjà accessible via Windsor ; GA4 (googleanalytics4) n'est PAS connecté à Windsor : à connecter.

## D. Relevé hebdomadaire « trafic et ventes venant des assistants IA »
Source WooCommerce (B) + GA4 (sessions, landing pages, conversions par domaine IA). À brancher dans
l'agent SEO/GEO existant, à côté du taux de citation des sondes. Dépôt/chemin de l'agent : à indiquer.

## E. Reste de l'audit serveur (voir rapport P0/P1)
error_log public, /wp-json/wp/v2/users exposé, robots.txt (groupes bots IA sans les Disallow de `*`),
claim « Microsoft Partner » dans llms.txt à justifier. Sauvegarde avant toute modification ; valider avant robots.txt.
