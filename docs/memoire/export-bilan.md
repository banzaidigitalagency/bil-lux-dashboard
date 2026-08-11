---
name: Export bilan xlsx / csv
description: "Bouton d'export du dashboard : ce qu'il contient, ce qu'il ne doit jamais contenir (investi réel, line items), et pourquoi il boucle avec le dashboard."
type: project
maj: 2026-08-11
---

Le media planner a demandé un bilan téléchargeable "le plus granulaire possible (langue, créa, canal)".
Livré le 2026-08-11 : deux boutons dans le header, **Bilan Excel** (classeur multi-onglets) et **CSV**
(reprend l'onglet le plus fin, "Par créa").

## Deux règles non négociables

Le fichier est téléchargé par le client, donc :

1. **Aucun investi média réel.** La RPC `get_dashboard_export` divise le spend plateforme par
   `get_billing_coef` : toutes les colonnes monétaires sont du budget client.
   Voir [[spend-billing-coefs]]. Avant d'ajouter une colonne, vérifier qu'elle ne réintroduit
   pas du coût réel (CPM plateforme, coût par vue brut, `cost_per_result`, etc.).
2. **Aucun line item.** Demande explicite de Matthieu le 2026-08-11 : le client ne doit pas voir
   nos LI, en particulier côté DV360 et Teads. Donc pas de nom de campagne, pas de nom de groupe
   d'annonces, pas de nom d'annonce, pas de ciblage (SirData, Equativ, whitelists, Smart Targeting,
   segments Finance / Broad, environnements DOOH).

L'agrégation est faite **dans la RPC** et pas côté JS, précisément pour ça : ces noms ne transitent
même pas jusqu'au navigateur. Une première version envoyait le détail par annonce au front, qui
aurait été lisible dans l'onglet réseau des devtools. Ne pas revenir en arrière là-dessus.

## Dimensions du fichier

La RPC renvoie une ligne par **jour x canal x langue x créa x variante x format**, et rien d'autre.
Le vocabulaire est fermé, extrait du nommage par des helpers SQL :

| Helper | Sort |
|---|---|
| `dashboard_norm_name` | majuscules sans accents, `_` et `/` traités comme séparateurs |
| `dashboard_lang_of` | FR / NL / EN |
| `dashboard_crea_of` | Cadre, Couple, Entrepreneur, Jeune |
| `dashboard_variante_of` | V1 à V4 (`TEXTE Vx`) |
| `dashboard_format_of` | Shorts, 1920x1080, 1280x720, In-feed, Habillage, Instream, Outstream, Non désactivable, Display, Vidéo, DOOH |

Un token non prévu ne sort pas du fichier : c'est ce qui garantit qu'aucun nom interne ne fuit.
Une cellule vide veut dire que la plateforme ne permet pas de distinguer la dimension
(Meta ne porte pas le format dans son nommage, DV360 et DOOH n'ont pas de créa déclinée).

Onglets : Synthèse (par canal, consommé vs plan), Par langue, Par phase (absent si la campagne
n'a pas de phases, cas de Knokke), Par créa, Méthodo. **Pas d'onglet de détail quotidien**,
jugé inutile par Matthieu et trop proche du niveau LI.

## Bouclage avec le dashboard

L'export doit boucler exactement avec ce qu'affiche le dashboard, sinon le planner nous renvoie
l'écart. La référence est `campaign_insights`, la même que le tableau par canal. Quand la somme
du détail par annonce est inférieure au total campagne d'un jour, la différence est réinjectée
dans une ligne sans créa (annonces supprimées de la plateforme : 9 285 impressions YouTube sur
13,1 M au 2026-08-11). Les écarts sous 10 impressions ou 1 euro sont ignorés, c'est du bruit
d'arrondi. Contrôle au 2026-08-11 sur Belgian : écart d'1 impression et de 5 centimes sur
13 091 918 impressions et 74 004,70 euros.

## Pièges de données constatés

- **Teads campagne BELGIAN : aucune métrique vidéo** (`video_start`, `video_complete` à 0 partout,
  alors que Knokke les remonte). Le sync Teads de cette campagne a tourné sans les métriques vidéo.
  Colonnes vidéo vides pour Teads sur Belgian. À corriger côté sync si le planner en a besoin.
- **Nommage Teads inconstant** : `INFEED` présent sur les annonces FR et sur une partie des NL
  seulement. `dashboard_format_of` retombe donc sur `ads.format`
  (`single image performance` -> In-feed), fiable, plutôt que sur le nom.
- **Underscores dans le nommage DV360 Knokke** (`KNOKKE_NL_BROAD`) : les bornes de mot Postgres
  ne détectent pas `NL`. D'où `dashboard_norm_name`. Attention,
  `get_dashboard_by_language_daily` (le tableau langue du dashboard) utilise encore l'ancienne
  regex sans normalisation : si un jour on veut la carte langue sur Knokke, c'est là qu'il faut passer.

## Ce qui n'est pas dans le fichier

Le reach et la couverture (non sommables entre plateformes), les conversions (campagne branding,
aucune n'est trackée), le détail quotidien, et tout ce qui touche au line item.
