# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Projet

Dashboard de suivi de campagne digitale pour BIL (Banque Internationale a Luxembourg) - Campagne Branding Belgique 2026. Fichier HTML unique avec donnees live Supabase.

## Memoire projet (a lire avant de travailler)

Le contexte accumule au fil des sessions est versionne dans `docs/memoire/` (index : `docs/memoire/README.md`). Ce CLAUDE.md decrit l'architecture ; `docs/memoire/` decrit le **pourquoi** (decisions, preferences de l'equipe, pieges connus). Trois notes non negociables :

- `docs/memoire/spend-billing-coefs.md` : **CRITIQUE**, ne jamais afficher le spend brut, toujours avec billing coefficients (le dashboard est vu par le client).
- `docs/memoire/budgets-source-de-verite.md` : les budgets viennent du Sheet "Matt - 26" via le sync vers `campaign_tracking`. Avant de toucher au code sur un budget faux, comparer Sheet vs `campaign_tracking`.
- `docs/memoire/design-approche.md` : pas de refonte, garder les sections et le violet BIL.

Quand une decision structurante est prise en session, l'ecrire dans `docs/memoire/` (versionne, partage) plutot que dans la memoire locale de Claude Code (par machine, non partagee).

## Architecture

Tout tient dans `index.html` : CSS inline, HTML, et JS (Chart.js + Supabase). Pas de build, pas de bundler, pas de framework.

- **Lock screen** : code d'acces `BIL2026` (constante `ACCESS_CODE`), session via `sessionStorage`
- **Supabase** : projet `mfqbhpxsuawujnfbcojr`, client code `billuxembourg` (constante `CLIENT_CODE` ; ex-`bil`, renomme pour anticiper un futur `bilsuisse`). Donnees chargees via 7 fonctions RPC (toutes appelees en parallele dans `loadDashboard`) :
  - `get_dashboard_kpis(p_client_code)` - KPIs globaux avec coefs billing
  - `get_dashboard_by_platform(p_client_code)` - breakdown par plateforme avec coefs
  - `get_dashboard_by_platform_daily(p_client_code)` - spend/impr par plateforme et par jour (sert au filtre par phase via `aggregateByPhase`)
  - `get_dashboard_daily(p_client_code)` - impressions par jour (chart timeline)
  - `get_dashboard_by_language_daily(p_client_code)` - breakdown langue x canal x jour
  - `get_dashboard_video_views(p_client_code)` - vues video DV360/YouTube
  - `get_dashboard_planning(p_client_code)` - **planning budgets + objectifs impressions**, agrege depuis `campaign_tracking` par phase et par canal. Source unique des budgets/objectifs : `applyPlanning()` construit `BUDGETS`, `OBJECTIVES`, `PHASE_OBJECTIVES`, `PHASES_DATA.budgetMedia/imprObj`, `TOTAL_BUDGET`, `TOTAL_IMPR_OBJ` et le donut. **Aucun budget/objectif impressions n'est en dur dans le JS.** Mapping canal dans la RPC : `Habillage% -> display_prog`, `pYoutube%/youtube -> google`, sinon `platform`. Bornes de phases (2026-06-30, 2026-08-31) en dur dans la RPC. La RPC matche `client_code = p_client_code OR client_name ILIKE clients.name` (fallback de securite conserve, meme si `campaign_tracking.client_code` vaut maintenant `billuxembourg`).
- **Seules valeurs plan encore en dur** (absentes de `campaign_tracking`) : objectifs de CLICS par phase (`CLICK_OBJECTIVES_PHASE`) et courbe mensuelle cumulee `CUM_TARGETS` (granularite mensuelle non stockee). Commentees dans le code.
- **Perimetre des RPC** : helper `is_dashboard_scope(p_client_code, campaign_name)`, appele dans les 6 RPC qui lisent des insights. Pour `billuxembourg`, ne garde que les campagnes dont le nom contient `BELGIAN` (BDC 202604417) ; pour tout autre client, ne filtre rien. Ajoute le 2026-08-11 parce que `2026 - BIL - BELGIQUE - KNOKKE - WEALTH MANAGEMENT` (Displayce, DV360, LinkedIn, demarree le 2026-07-28) tournait sur le meme client Supabase sans ligne dans `campaign_tracking`, donc sans objectif en face, et gonflait depense et impressions. **Filtrer sur `campaign_tracking.platform_campaign_ids` n'est PAS possible** : cote Displayce ces IDs sont des IDs de lignes, pas le `platform_campaign_id` de la campagne, le DOOH disparaitrait entierement. Le helper est fail-safe (renvoie `true` par defaut) : si le code client rechange, on retombe sur la pollution visible, pas sur une perte de donnees silencieuse.
- **Coefficients billing** : table `billing_coefficients`, le spend client = spend reel / coef. Fonction helper `get_billing_coef(client_id, platform::text)` avec fallback sur coefs par defaut (client_id IS NULL)
- **Charts** : Chart.js 4.x - un line chart (impressions cumulees) et un doughnut (repartition budget, alimente par `applyPlanning`)

## Couleurs BIL

Violet principal `#702F8A`. Les CSS variables vont de `--bil-dark` (#4A1D6B) a `--bil-lighter` (#F5EEF9). Les couleurs des canaux (LinkedIn bleu, Meta bleu, YouTube rouge, etc.) gardent leurs couleurs propres.

## Canaux actifs (8)

DOOH (Displayce), LinkedIn, Meta, DV360 Video, YouTube (Google), Teads, Prog. Display, CTV (Connected TV - YouTube). Pas de Presse (dashboard 100% digital).

## Mapping plateforme Supabase -> tableau

Le JS mappe les plateformes Supabase aux lignes du tableau HTML par index (`rowByPlatform` / `rowPlatformOrder`) :
- `displayce` -> row 0 (DOOH)
- `linkedin` -> row 1
- `meta` -> row 2
- `dv360` -> row 3 (DV360 Video)
- `google` -> row 4 (YouTube / pYoutube)
- `teads` -> row 5
- `display_prog` -> row 6 (Prog. Display / Habillage)
- `ctv` -> row 7 (CTV, couleur teal #00A8A8, sans objectif clics)

Deux canaux sont des sous-ensembles de plateformes mutualisees, scindes par **nom de campagne** (pas par code client, robuste aux renommages) :
- **Habillage** -> `display_prog` : campagne DV360 dont le nom contient `HABILLAGE`. Coef billing `habillage` (0.40) au lieu d'`outstream` (0.20).
- **CTV** -> `ctv` : campagnes Google Ads dont le nom contient `CTV` (ex `BIL - 2604 - BELGIAN - CTV - EN/FR`). Coef billing `ctv` (0.35, ligne ajoutee dans `billing_coefficients`, defaut client_id NULL). Cote planning, repere par `canal_label ILIKE '%CTV%'` (teste AVANT la regle `%youtube%`).

Le split est implemente dans les RPC de spend (`get_dashboard_by_platform`, `get_dashboard_by_platform_daily`, `get_dashboard_by_language_daily`) et dans `get_dashboard_planning` ; `get_dashboard_kpis` et `get_dashboard_daily` n'ont pas besoin du split CTV (la part CTV est sur platform `google` -> coef `pyoutube` 0.35, identique). `get_billing_coef` mappe `display_prog -> habillage` et `ctv -> ctv` (via la branche ELSE). Les 4 RPC sur `campaign_insights` renvoient un total Depenses identique (coherence verifiee).

Objectifs de CLICS (hardcode, `CLICK_OBJECTIVES_PHASE`) et courbe mensuelle (`CUM_TARGETS`) recalcules a la repart v2 (transfert +10k vers P1, Habillage P1 coupe, ligne CTV 3 346 EUR / 185 889 imp). CTV est sans objectif clics.

## Deploiement

Fichier statique, deployable sur Vercel/Cloudflare Pages. `open index.html` pour tester en local.

## Langue

Dashboard en francais. Commits en francais.
