---
name: Export bilan xlsx / csv
description: "Bouton d'export du dashboard : granularité, garantie de bouclage avec les chiffres affichés, et ce qui ne doit jamais y figurer."
type: project
maj: 2026-08-11
---

Le media planner a demandé un bilan téléchargeable "le plus granulaire possible (langue, créa, canal)".
Livré le 2026-08-11 : deux boutons dans le header, **Bilan Excel** (classeur multi-onglets) et **CSV**
(le seul onglet de détail quotidien).

## Règle non négociable

**Le fichier est téléchargeable par le client, donc aucun investi média réel ne doit y figurer.**
La RPC `get_dashboard_export` divise déjà le spend plateforme par `get_billing_coef` : toutes les
colonnes monétaires du fichier sont du budget client. Voir [[spend-billing-coefs]].
Avant d'ajouter une colonne à l'export, vérifier qu'elle ne réintroduit pas du coût réel
(un CPM plateforme, un coût par vue brut, un `cost_per_result`, etc.).

## Granularité

Une ligne par jour et par annonce, avec les dimensions déduites du nommage :
langue (FR / NL / EN), créa (Cadre, Couple, Entrepreneur, Jeune), variante de texte (V1 à V4),
format (In-feed, 1280x720, Shorts, Instream, Outstream, Habillage, DOOH), ciblage
(SirData, Equativ, Whitelist, Smart Targeting, Finance, Broad, Outdoor, Business, Transports).

Deux niveaux de repli, dans cet ordre :

1. **Annonce** quand la plateforme remonte `ad_insights` (Meta, LinkedIn, Google, DV360, Teads).
2. **Groupe d'annonces** pour les campagnes sans niveau annonce. C'est le cas du DOOH Displayce :
   il n'a aucune ligne dans `ads`, mais le groupe porte l'environnement (Outdoor, Business,
   Transports) et la langue, ce qui reste exploitable.

## Pourquoi une ligne "Non ventilé"

L'export doit boucler **exactement** avec ce qu'affiche le dashboard, sinon le planner nous
renvoie l'écart. La référence est `campaign_insights` (la même que le tableau par canal).
Quand la somme du détail est inférieure au total campagne d'un jour, une ligne "Non ventilé"
porte la différence. Cas connu au 2026-08-11 : 9 285 impressions YouTube sur 13,1 M
(annonces supprimées de Google Ads, dont les insights ne redescendent plus au niveau annonce).
Les écarts sous 10 impressions ou 1 euro ne génèrent pas de ligne : c'est du bruit d'arrondi.

## Pièges de données constatés

- **Teads campagne BELGIAN : aucune métrique vidéo** (`video_start`, `video_complete` à 0 sur
  toutes les lignes, alors que la campagne Knokke les remonte). Le sync Teads de cette campagne
  a tourné sans les métriques vidéo. Les colonnes vidéo sont donc vides pour Teads sur Belgian.
  À corriger côté sync si le planner en a besoin.
- **Teads, 17 au 20 avril 2026 : attribution groupe d'annonces décalée.** Les insights annonce
  sont rattachés à SMART TARGETING alors que les insights groupe les mettent sur FINANCE. Les
  totaux par canal et par jour restent justes, seule la répartition FINANCE / SMART TARGETING
  de ces 4 jours est fausse dans l'onglet "Par créa".
- **Underscores dans le nommage DV360 Knokke** (`KNOKKE_NL_BROAD`) : les bornes de mot Postgres
  ne détectent pas `NL` dans ce cas. La fonction `dashboard_lang_of()` normalise `_` et `/` en
  espaces avant de lire la langue. `get_dashboard_by_language_daily` (le tableau langue du
  dashboard) utilise encore l'ancienne regex sans normalisation : si un jour on veut la carte
  langue sur Knokke, c'est là qu'il faut passer.

## Ce qui n'est pas dans le fichier

Le reach et la couverture (non sommables entre plateformes), les conversions (la campagne est
en branding, aucune n'est trackée), et le détail par créa du DOOH (la plateforme ne le remonte pas).
